# CI Pipeline Guide

This repository uses a reusable GitHub Actions pipeline to standardize quality gates for service teams. The shared workflow lives in [.github/workflows/golden-path-ci.yml](../.github/workflows/golden-path-ci.yml), and service-specific callers use it from their own workflow file, such as [.github/workflows/todo-service-ci.yml](../.github/workflows/todo-service-ci.yml).

The goal is simple: every service gets the same minimum checks for code quality, tests, IaC safety, and deployment readiness without duplicating workflow logic in each repo.

## What the reusable workflow does

The reusable workflow is configured with `on: workflow_call`, which means it is designed to be invoked by another workflow rather than run directly. It accepts a few inputs and a single secret:

- `node_version` — default Node.js version used for lint/test jobs
- `terraform_version` — Terraform version used for validation and planning
- `run_terraform_plan` — turns on the infrastructure safety checks
- `run_terraform_apply` — allows apply to the dev stack
- `build_and_push` — toggles image build and push to ECR
- `aws_role_arn` — the GitHub OIDC role ARN used by the Terraform jobs

At the workflow level, it also sets minimal permissions:

- `contents: read`
- `pull-requests: write`

That keeps the workflow aligned with the least-privilege pattern required by the lab.

### Job-by-job breakdown

#### 1) `lint`

Purpose:

- Install dependencies
- Run ESLint for both the backend and frontend workspaces

Why it exists:

- It catches syntax mistakes, unused code, unsafe patterns, and style violations before a PR is merged.
- It gives a fast feedback loop because it runs early and does not require cloud access.

#### 2) `test`

Purpose:

- Install dependencies
- Run the backend Jest suite with coverage enabled
- Write a summary to the GitHub step summary

Why it exists:

- It validates runtime behavior for the service and ensures regressions are caught.
- It enforces coverage visibility so the project can track whether the app remains robust as it grows.

#### 3) `security-scan`

Purpose:

- Install Python and Checkov
- Scan the infrastructure under `infra/` for risky IaC patterns

Why it exists:

- This catches common infrastructure anti-patterns such as publicly open resources or weak security defaults before a plan or apply is attempted.
- It reduces the risk of deploying insecure AWS resources from Terraform.

This job only runs when `run_terraform_plan` is enabled.

#### 4) `terraform-plan`

Purpose:

- Configure AWS credentials using OIDC
- Install the selected Terraform version
- Initialize Terraform for the dev stack
- Provide mock network values for PR validation when a real AWS deployment is not intended
- Generate a plan artifact and post a summary

Why it exists:

- It proves the Terraform configuration is syntactically valid and can be planned against the dev stack.
- It gives reviewers a concrete change set before apply is allowed.
- It is the critical gate for IaC safety because it ensures resource changes are reviewed, not just assumed to work.

This job also requires `id-token: write` because GitHub OIDC tokens are only issued when a workflow explicitly asks for them.

#### 5) `docker-build`

Purpose:

- Build the backend and frontend Docker images on pull requests

Why it exists:

- It validates that the Dockerfiles still build even when no deployment occurs.
- It helps catch broken container packaging before a merge.

This is valuable validation, but it is not the minimum required branch-protection gate in the same way as `lint`, `test`, `security-scan`, and `terraform-plan`.

#### 6) `terraform-apply`

Purpose:

- Run the saved Terraform plan against the dev environment
- Show the deployed service URL

Why it exists:

- This is the environment deployment path for the dev stack, triggered only when the caller sets `run_terraform_apply`.
- It is intentionally separated from the validation-only plan step so teams can review infrastructure changes before actual deployment.

#### 7) `build-and-push`

Purpose:

- Resolve ECR repository URLs
- Log in to Amazon ECR
- Build and push backend and frontend images
- Trigger a new ECS deployment

Why it exists:

- It is the production-style release pipeline for a service after infrastructure is in place and the dev environment is ready.
- It ensures the built container images are pushed to ECR and the ECS service picks them up.

## Required checks and why they matter

The required checks for the golden path are:

- `lint`
- `test`
- `security-scan`
- `terraform-plan`

Each of them protects a different failure mode:

### `lint`

Validates:

- JavaScript/Node quality issues
- Code smells and common logic problems before merge

Why required:

- Prevents broken or unmaintainable code from entering the main branch.
- Keeps the repo consistent and easier for new engineers to read.

### `test`

Validates:

- Real service behavior through Jest tests
- Coverage is generated and surfaced in CI output

Why required:

- Prevents regressions and confirms that business behavior still works.
- Coverage makes the risk of silent breakage visible instead of invisible.

### `security-scan`

Validates:

- Terraform security misconfiguration using Checkov
- High-risk AWS resource settings

Why required:

- Prevents insecure cloud resources from being introduced by IaC drift or oversight.
- Makes infrastructure review part of the standard delivery process.

### `terraform-plan`

Validates:

- Terraform configuration still initializes and can be planned
- AWS provider and remote state configuration are usable for the dev stack
- Proposed changes are visible before apply

Why required:

- It ensures infrastructure changes are reviewed before they hit AWS.
- It protects the team from broken or accidental provisioning changes and helps catch environment mismatch early.

## Minimum adoption pattern for a new service team

A service team only needs a caller workflow that invokes the reusable workflow and provides the required secret.

Example minimum pattern:

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

This is the minimum adoption pattern used in [.github/workflows/todo-service-ci.yml](../.github/workflows/todo-service-ci.yml):

- It triggers on pushes to `main` and on all pull requests.
- It calls the shared pipeline rather than rewriting the checks.
- It passes the AWS OIDC role ARN so the Terraform plan/apply jobs can authenticate without static credentials.
- It enables plan validation for pull requests and includes apply/push only for the `main` branch.

This gives each team a standard golden path while still allowing service-specific behavior through workflow inputs.

## How to configure the AWS OIDC role secret

The Terraform plan job authenticates to AWS using OIDC. The workflow sets:

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.aws_role_arn }}
    aws-region: us-east-1
```

### Required setup steps

1. Create an IAM identity provider for GitHub in AWS.
   - Trust provider type: GitHub Actions OIDC
   - Audience: `sts.amazonaws.com`

2. Create an IAM role that allows the GitHub workflow to assume it.
   - The trust policy should allow access from your repository, for example:
   - `repo:OWNER/REPO:ref:refs/heads/main`
   - and optionally `repo:OWNER/REPO:pull_request` for PR contexts if your policy requires it

3. Attach the minimum AWS permissions needed for Terraform plan and apply.
   - For the plan job, the role needs read access to the Terraform state bucket and the AWS resources the provider will touch.
   - For apply/build jobs, allow the additional permissions needed for ECS, ECR, and related deployment resources.

4. In GitHub, open the repository settings and create a repository secret named `AWS_ROLE_ARN`.
   - Store the full IAM role ARN as the value.

5. In the caller workflow, pass it as:

```yaml
secrets:
  aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

### Why this matters

Using OIDC eliminates the need to store long-lived AWS access keys in GitHub. It is safer than static credentials, aligns with IaC best practices, and makes cloud access tightly scoped to the workflow and branch that needs it.

## Summary

The golden path gives every team the same minimum safety and quality bar:

- `lint` catches code-quality regressions
- `test` verifies behavior and coverage
- `security-scan` checks IaC risk
- `terraform-plan` validates the real AWS deployment config before apply

By adopting the reusable workflow, a new service team can add the same controls without rewriting CI logic or creating inconsistent branch protection policies across repos.
