# CI Pipeline

The pipeline is split into a reusable workflow and a small service-specific caller:

- `.github/workflows/golden-path-ci.yml` defines the jobs and optional deployment stages.
- `.github/workflows/todo-service-ci.yml` selects when to run them and passes workflow inputs and secrets.

The reusable workflow uses Node.js 20 and Terraform 1.7.0 by default. Callers can override either version. Terraform planning, security scanning, applying, and publishing are controlled by boolean inputs.

## Jobs

| Job               | What it validates or does                                                                                                                                                                                           | Why it exists                                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `lint`            | Installs npm workspace dependencies and runs ESLint for the backend and frontend.                                                                                                                                   | Catches code-quality and syntax issues before merge.                                                                |
| `test`            | Runs backend Jest tests with coverage and adds line, branch, function, and statement percentages to the GitHub Actions summary. Jest enforces the 80% line and branch thresholds configured in the backend package. | Protects backend behavior and prevents coverage from falling below the project's configured minimum.                |
| `security-scan`   | Runs Checkov on `infra/` and fails on HIGH severity findings. Runs when `run_terraform_plan` is true.                                                                                                               | Detects high-severity infrastructure configuration risks.                                                           |
| `terraform-plan`  | Initializes Terraform with `-backend=false`, validates the dev stack, runs TFLint, and creates a plan summary. Runs when `run_terraform_plan` is true.                                                              | Checks Terraform syntax, style, and whether the stack can be planned without connecting to remote state or AWS.     |
| `docker-build`    | On pull requests, builds both backend and frontend Docker images after `lint` and `test` succeed.                                                                                                                   | Verifies both Dockerfiles and image build contexts before merge without publishing images or using AWS credentials. |
| `terraform-apply` | On a main-branch push when enabled, uses GitHub OIDC credentials, initializes the S3 backend, plans and applies the dev stack, and reports the service URL.                                                         | Deploys approved infrastructure changes without storing long-lived AWS credentials in GitHub.                       |
| `build-and-push`  | After `terraform-apply`, authenticates to ECR, builds and pushes both images with the commit SHA and `latest` tags, then triggers an ECS deployment. It runs only on a main-branch push when enabled.               | Publishes the service images and deploys them to ECS after infrastructure is ready.                                 |

## Required Checks

The project requires these four checks for the golden path:

- **`lint`** checks both JavaScript workspaces with ESLint so style and source errors are caught consistently.
- **`test`** runs backend Jest tests and enforces the configured 80% line and branch coverage minimum, guarding the API behavior.
- **`security-scan`** checks `infra/` with Checkov and fails on HIGH findings to catch risky infrastructure before deployment.
- **`terraform-plan`** runs Terraform validation, TFLint, and a plan against `infra/stacks/dev`; these catch invalid configuration, common Terraform issues, and unintended infrastructure changes.

The caller sets `run_terraform_plan: true`, so the security scan and Terraform checks run on both pull requests and pushes to `main`. The Docker build runs on pull requests. Apply and publishing are enabled only for pushes to `main`.

## Adopt the Workflow

A service team adds a caller workflow that triggers on pull requests and pushes to its deployment branch. This repository's minimum caller is:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Keep `id-token: write` in the caller's top-level permissions: the reusable workflow cannot grant the caller an OIDC token by itself. For a new service, adjust the branch, paths, and workflow inputs to match that service's build and deployment setup.

## AWS OIDC Secret

The current `terraform-plan` job does **not** use an AWS role or the `aws_role_arn` secret. It runs `terraform init -backend=false` and plans with the stack's mock provider configuration, so pull requests can validate infrastructure without AWS credentials.

The caller passes the repository secret `AWS_ROLE_ARN` into the reusable workflow's `aws_role_arn` secret input. The `terraform-apply` and `build-and-push` jobs consume that input through `aws-actions/configure-aws-credentials@v4`; they require OIDC permissions and run only on pushes to `main` when enabled.

To configure deployment credentials:

1. In the AWS account, configure the GitHub Actions OIDC provider and create an IAM role with a trust policy restricted to the intended GitHub repository and branch.
2. In the GitHub repository, open **Settings → Secrets and variables → Actions**, choose **New repository secret**, name it `AWS_ROLE_ARN`, and set its value to the IAM role ARN.
3. Keep the caller's `id-token: write` permission and the `aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}` mapping. Do not hardcode the ARN in either workflow.

If Terraform planning later needs AWS access or remote state, update the reusable job to configure OIDC explicitly; adding the secret alone does not make the current `terraform-plan` job use AWS credentials.
