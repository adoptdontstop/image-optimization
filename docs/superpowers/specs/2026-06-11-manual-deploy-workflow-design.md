# Manual GitHub Actions Deploy Workflow — Design

**Date:** 2026-06-11
**Status:** Approved

## Goal

Deploy the image-optimization CDK stack to `test` or `prod` from the GitHub
Actions UI (manual trigger) instead of running `cdk deploy` from a local
terminal.

## Scope

- One manually-triggered workflow that deploys the existing CDK stack.
- Reuses the existing `deploy:test:media` / `deploy:prod:media` npm scripts as
  the single source of truth for per-environment config (bucket names,
  `MAX_IMAGE_SIZE`).
- AWS authentication via OIDC (no long-lived secrets in GitHub).
- Documentation for the one-time AWS OIDC setup.

Out of scope: automatic deploys on push/PR, production approval gates (may be
added later via GitHub Environments), multi-account role switching.

## Workflow

**File:** `.github/workflows/deploy.yml`

**Trigger:** `workflow_dispatch` only (manual, from the Actions tab).

**Inputs:**

| Input | Type | Default | Purpose |
|-------|------|---------|---------|
| `environment` | choice (`test`, `prod`) | `test` | Selects which `deploy:<env>:media` script runs. |
| `aws_region` | string | `us-east-1` | Region passed to the credential step. |

**Job:** single job on `ubuntu-latest`.

- `environment: ${{ inputs.environment }}` — associates the run with a GitHub
  Environment of the same name. No approval rule now, but this allows adding
  required reviewers later without touching the workflow, and permits
  per-environment scoping of the role-ARN secret if test/prod ever diverge.
- `permissions:` `id-token: write`, `contents: read` — required for OIDC.

**Steps:**

1. `actions/checkout@v4`
2. `actions/setup-node@v4` with Node 20.
3. `npm install` — `package-lock.json` is gitignored, so `npm ci` is not usable.
4. `npm run prebuild` — installs the **linux** sharp binary into
   `functions/image-processing/`. The `prebuild` lifecycle hook only fires
   before `build`, not before `cdk deploy`, so it must be invoked explicitly or
   the Lambda ships without sharp.
5. `aws-actions/configure-aws-credentials@v4` — assume the deploy role via OIDC
   using `role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}` and
   `aws-region: ${{ inputs.aws_region }}`.
6. Deploy:
   `npm run deploy:${{ inputs.environment }}:media -- --require-approval never`
   — reuses the existing npm scripts; `--require-approval never` prevents CDK
   from blocking on an IAM confirmation prompt in CI.

## AWS Setup (one-time, no cost)

Documented in `docs/deploy-oidc-setup.md`:

1. Create an IAM OIDC identity provider for
   `token.actions.githubusercontent.com` (audience `sts.amazonaws.com`).
2. Create an IAM role with:
   - A trust policy allowing `sts:AssumeRoleWithWebIdentity` from
     `repo:adoptdontstop/image-optimization:*` (the repo, any ref).
   - A permissions policy sufficient to run `cdk deploy` (CloudFormation, S3,
     Lambda, CloudFront, IAM for the stack's resources). Document a starting
     policy; the user can tighten it.
3. Add the role ARN as repository secret `AWS_DEPLOY_ROLE_ARN`.

IAM OIDC providers, roles, and `sts:AssumeRoleWithWebIdentity` calls are free.

## Deliverables

- `.github/workflows/deploy.yml`
- `docs/deploy-oidc-setup.md` (OIDC provider + role + trust-policy JSON)

## Testing / Verification

- `actionlint` (or YAML lint) on the workflow file if available; otherwise a
  manual syntax review.
- Confirm the workflow appears under the Actions tab with the two inputs.
- A real deploy run is the ultimate verification once the AWS role/secret exist
  (depends on AWS access the user controls; not part of this change's automated
  verification).
