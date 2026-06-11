# GitHub Actions OIDC deploy setup

The [Deploy workflow](../.github/workflows/deploy.yml) authenticates to AWS using
OpenID Connect (OIDC) — GitHub assumes a short-lived IAM role per run, so no
long-lived AWS keys are stored in the repository.

This is a **one-time** setup. The OIDC provider, the IAM role, and the
`sts:AssumeRoleWithWebIdentity` calls are all free.

## 1. Create the IAM OIDC identity provider

Skip this step if your account already has a provider for
`token.actions.githubusercontent.com`.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

(Console: IAM → Identity providers → Add provider → OpenID Connect →
Provider URL `https://token.actions.githubusercontent.com`, Audience
`sts.amazonaws.com`.)

## 2. Create the deploy role

### Trust policy

Save as `trust-policy.json`. Replace `<ACCOUNT_ID>` with your AWS account ID.
The `sub` condition restricts the role to this repository (any branch/ref).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:adoptdontstop/image-optimization:*"
        }
      }
    }
  ]
}
```

> To restrict further, replace `:*` with a specific ref, e.g.
> `repo:adoptdontstop/image-optimization:ref:refs/heads/main`.

### Create the role

```bash
aws iam create-role \
  --role-name image-optimization-deploy \
  --assume-role-policy-document file://trust-policy.json
```

### Permissions policy

The role needs enough access to run `cdk deploy`. The simplest starting point
is to let it assume the CDK bootstrap roles (CDK does the privileged work
through those):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/cdk-*"
    }
  ]
}
```

Attach it:

```bash
aws iam put-role-policy \
  --role-name image-optimization-deploy \
  --policy-name cdk-deploy \
  --policy-document file://permissions-policy.json
```

> This assumes the target account/region has been bootstrapped
> (`cdk bootstrap`). If you prefer not to rely on the bootstrap roles, grant the
> role direct CloudFormation, S3, Lambda, CloudFront, and IAM permissions
> instead and tighten as needed.

## 3. Add the role ARN as a repository secret

In the repo: **Settings → Secrets and variables → Actions → New repository
secret**.

- Name: `AWS_DEPLOY_ROLE_ARN`
- Value: `arn:aws:iam::<ACCOUNT_ID>:role/image-optimization-deploy`

## 4. Run a deploy

**Actions → Deploy → Run workflow**, pick the `environment` (`test` / `prod`)
and `aws_region`, then run.

## Adding a production approval gate later (optional)

Go to **Settings → Environments → prod → Required reviewers** and add
reviewers. The workflow already runs under `environment: ${{ inputs.environment }}`,
so prod deploys will then pause for one-click approval — no workflow change
needed.
