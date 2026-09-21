# Static site on S3 + CloudFront, deployed by GitHub Actions

A static site that deploys on every push to `main`. No AWS access keys exist
anywhere — GitHub authenticates to AWS with short-lived credentials via OIDC.
All AWS resources are defined in CloudFormation.

📐 **[Architecture diagram](https://samcyanide.github.io/oxbridge/)** — the deploy
path end to end, from push to viewer. Served by GitHub Pages out of
[`docs/`](docs/architecture.html) — separate from the S3 + CloudFront pipeline
described below.

## How it works

```
push to main
    |
    v
GitHub Actions  --(OIDC token)-->  AWS STS  --(15-min creds)-->  assume DeployRole
    |
    +--> aws s3 sync ./site  -->  S3 bucket (private, no public access)
    +--> cloudfront create-invalidation
                                   |
viewer --> HTTPS --> CloudFront --(OAC, SigV4)--> bucket
```

The bucket is **never public**. CloudFront reaches it through Origin Access
Control, and the bucket policy only trusts the CloudFront service principal
narrowed to this one distribution.

## Layout

| Path | Purpose |
|---|---|
| `site/` | Everything here is published. Nothing outside it is. |
| `infra/static-site.yaml` | All AWS resources. |
| `.github/workflows/deploy.yml` | The pipeline. |
| `docs/` | Architecture diagram, published to GitHub Pages. Not deployed to S3. |

Site content is isolated in `site/` deliberately: syncing the repo root would
publish the template, the README and local config to the public internet.

## Required setup

Two steps, in this order. GitHub-side configuration is **one repo variable**
(`AWS_DEPLOY_ROLE_ARN`, set in step 2) and **no secrets at all**.

### 1. Deploy the stack (once, before the first push)

```bash
aws cloudformation create-stack \
  --stack-name oxbridge-site \
  --template-body file://infra/static-site.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

aws cloudformation wait stack-create-complete \
  --stack-name oxbridge-site --region us-east-1
```

This ordering is a real bootstrap dependency and cannot be designed away: the
stack creates the IAM role the pipeline assumes. If you push before the stack
exists, the run fails at *Configure AWS credentials* with a role-not-found
error — loudly, before anything is deployed.

### 2. Point the workflow at that stack

Two values must match your deployment:

| Where | Key | Must be |
|---|---|---|
| `env:` in `.github/workflows/deploy.yml` | `STACK_NAME` | the same `--stack-name` used above |
| GitHub repo variable | `AWS_DEPLOY_ROLE_ARN` | the stack's `DeployRoleArn` output |

```bash
gh variable set AWS_DEPLOY_ROLE_ARN --body "$(aws cloudformation describe-stacks \
  --stack-name oxbridge-site --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DeployRoleArn'].OutputValue" --output text)"
```

That one variable is the **only** GitHub-side configuration. No secrets are
needed at all. Bucket name, distribution id and site URL are read from the stack
outputs on **every run**, so they cannot drift.

The role ARN sits in a repo variable rather than in the workflow because this
repo is public and there is no reason to publish an AWS account identifier. It
is not a credential: the role's trust policy is what makes it safe, restricting
assumption to this repo on this branch.

### 3. Check your OIDC subject prefix

**This is the step most likely to break.** GitHub now issues *immutable* subject
claims that embed numeric owner and repo ids, so the token's `sub` is **not** the
`repo:OWNER/REPO:...` form nearly every tutorial shows:

```
actual:    repo:SamCyanide@9272499/oxbridge@1379066388:ref:refs/heads/main
classic:   repo:SamCyanide/oxbridge:ref:refs/heads/main
```

A trust policy written the classic way is rejected by STS with
`Not authorized to perform sts:AssumeRoleWithWebIdentity`, which gives no hint
that the subject is the problem. Get your real prefix and pass it as the
`GitHubSubjectPrefix` parameter:

```bash
gh api repos/OWNER/REPO/actions/oidc/customization/sub
# -> {"use_immutable_subject": true, "sub_claim_prefix": "repo:OWNER@123/REPO@456"}
```

If `use_immutable_subject` is `false`, use the classic `repo:OWNER/REPO` form.

### Template parameters

- `GitHubOwner` / `GitHubRepo` / `GitHubBranch` — who may assume the deploy role.
- `CreateOIDCProvider` — set `false` if the account already has a GitHub OIDC
  provider. AWS permits only one per URL, so a second stack would otherwise collide.
- `BudgetNotificationEmail` — optional; creates a $1/month budget alarm.

## IAM: what the pipeline can actually do

The deploy role holds exactly four permissions:

| Action | Scope |
|---|---|
| `s3:ListBucket` | the site bucket only |
| `s3:PutObject`, `s3:DeleteObject` | objects in that bucket only |
| `cloudfront:CreateInvalidation` | that one distribution only |
| `cloudformation:DescribeStacks` | this one stack only |

It cannot read objects, touch other buckets, create resources, or modify itself.
`s3:GetObject` is deliberately absent — an upload-only `sync` compares via
`ListBucket` and never needs to read object bodies.

Trust is the part that matters most:

```yaml
StringEquals:
  token.actions.githubusercontent.com:aud: sts.amazonaws.com
StringLike:
  token.actions.githubusercontent.com:sub: repo:OWNER/REPO:ref:refs/heads/main
```

Both conditions are load-bearing. Without `aud`, any GitHub repository in the
world could assume the role. A `sub` of `repo:OWNER/REPO:*` — which is what most
copy-paste examples use — would let **any branch, and any pull request from any
fork**, deploy to production. Pinning the full `ref:refs/heads/main` path is the
difference between a deploy role and an open door.

## Decisions

**Trust pinned to the immutable subject claim.** The role trusts
`repo:OWNER@<id>/REPO@<id>:ref:refs/heads/main`, not `repo:OWNER/REPO:...`. Beyond
being what GitHub actually sends here, the id-based form survives a repo or account
rename and cannot be hijacked by deleting a repo and re-registering its name.

**CloudFront + OAC rather than S3 website hosting.** S3 static hosting needs a
public bucket. OAC keeps the bucket entirely private, and adds HTTPS, which S3
website endpoints don't support.

**No thumbprint on the OIDC provider.** IAM now manages trust for
`token.actions.githubusercontent.com`. Templates that pin the old SHA-1
thumbprint break when it rotates.

**Inline role policy, not a managed policy.** cfn-guard flags this
(`iam_no_inline_policy_check`). Kept deliberately: an inline policy can't be
accidentally attached to another principal and is deleted with the role.

**`DeletionPolicy: Delete` on the bucket.** Contents are disposable and
reproducible from git. `Retain` would orphan the bucket and then block
re-creating the stack under the same deterministic name.

**The workflow resolves infrastructure at run time** rather than storing bucket
and distribution ids in GitHub repo variables. Copied ids are a snapshot: replace
the distribution and the variable silently points at a dead one, so deploys keep
"succeeding" while invalidations go nowhere. Reading the stack outputs each run
also cuts GitHub-side setup to a single variable, rather than four values a human
must remember to re-derive and apply.

**Runs on every push to `main`,** per the brief. A `paths: ['site/**']` filter
would skip deploys that change only docs — a sensible optimisation, omitted so
behaviour matches the spec.

## Cost

Designed to sit inside the AWS Free Tier.

| Resource | Free tier position |
|---|---|
| CloudFormation, IAM role, OIDC provider | No charge, ever |
| CloudFront traffic | 1 TB egress + 10M requests/month, **perpetual** |
| CloudFront invalidations | 1,000 paths/month; this uses **1 per deploy** |
| S3 | 5 GB + 2,000 PUT + 20,000 GET — **12 months only**, not perpetual |
| Budgets | First 2 free |

Falls outside free tier only at >1,000 deploys/month ($0.005/path), >1 TB
egress, or after the 12-month S3 window (pennies at this size).

Deliberately omitted on cost grounds: **WAF** (~$5/month per web ACL) and
**CloudFront access logging** (billable log storage).

Object versioning is on for rollback, but a lifecycle rule expires non-current
versions after 7 days so storage can't quietly grow.

## With more time

- **Remote state / drift detection.** Nothing currently detects console edits to
  the stack. A scheduled `detect-stack-drift` would catch them.
- **Deploy the stack from CI too**, gated behind a manual approval, so infra
  changes follow the same path as content changes. Needs a second, more
  privileged role, which is why it isn't here.
- **A custom domain + ACM certificate.** This would also clear the two cfn-guard
  TLS findings (`minimum_protocol_version`, `sni_enabled`), which are unfixable
  while using the default CloudFront certificate.
- **Cache headers per file type** — long max-age for fingerprinted assets, short
  for HTML — so invalidation becomes a fallback rather than the mechanism.
- **A smoke test after deploy** that curls the distribution and fails the job on
  a non-200, instead of assuming a clean `sync` means a working site.
- **Staging environment** via a second stack with different parameters.

## Teardown

```bash
BUCKET=$(aws cloudformation describe-stacks --stack-name oxbridge-site \
  --region us-east-1 --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" \
  --output text)

# Versioning is on, so every version AND delete marker must go. `aws s3 rm
# --recursive` removes only current versions, which leaves the bucket
# undeletable and the stack deletion fails on "bucket not empty".
aws s3api list-object-versions --bucket "$BUCKET" \
  --query '{Objects: [Versions, DeleteMarkers][].{Key: Key, VersionId: VersionId}}' \
  --output json > /tmp/purge.json

# Skip the next call if the bucket is already empty (Objects comes back null).
aws s3api delete-objects --bucket "$BUCKET" --delete file:///tmp/purge.json

aws cloudformation delete-stack --stack-name oxbridge-site --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name oxbridge-site --region us-east-1
```

The bucket must be fully emptied first: CloudFormation cannot delete a bucket
that still holds objects, including non-current versions.
