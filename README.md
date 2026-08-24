# Push Docker Image to ECR — Node.js + GitHub Actions OIDC

A minimal Node.js API, containerized and shipped to a private Amazon ECR
repository through a GitHub Actions pipeline that authenticates to AWS with
**OIDC only** — no long-lived access keys are stored anywhere.

## Architecture

```mermaid
flowchart LR
    Dev[Developer git push] --> GHA[GitHub Actions job]
    GHA -- "1: mint OIDC JWT" --> GHOIDC[GitHub token endpoint]
    GHOIDC -- "2: JWT" --> GHA
    GHA -- "3: sts:AssumeRoleWithWebIdentity + JWT" --> STS[AWS STS]
    STS -- "4: validate signature/audience/sub" --> Provider[IAM OIDC provider<br/>token.actions.githubusercontent.com]
    Provider -. "trusted by (repo-scoped condition)" .-> Role[IAM Role<br/>github-actions-ecr-push]
    STS -- "5: temporary credentials" --> GHA
    GHA -- "6: docker build" --> Image[node-api image]
    Image -- "7: docker push, checked against<br/>role's IAM policy + repo policy" --> ECR[(Private ECR<br/>node-api)]

    subgraph aws["AWS Account 447558491229 · eu-west-1"]
        STS
        Provider
        Role
        ECR
    end
```

The IAM role trusts GitHub's OIDC provider only for
`repo:Iradukunda54/docker-ecr-oidc-lab:*` and grants nothing beyond
pushing/pulling layers to the single `node-api` ECR repository — it cannot
touch any other AWS resource.

## Repository layout

```
app/                          Node.js (Express) source
Dockerfile                    Multi-stage, non-root, minimal alpine image
.dockerignore
infrastructure/template.yaml  CloudFormation: ECR repo + OIDC IAM role
.github/workflows/deploy.yml  CI/CD pipeline
```

## Containerization

- Base image: `node:22-alpine` (multi-stage — build deps discarded from
  the final image).
- Runs as the non-root `node` user (built into the official image).
- `npm ci --omit=dev` for reproducible, dependency-pinned installs.
- `HEALTHCHECK` hits `/health`.
- `.dockerignore` excludes `node_modules`, git metadata, CI config, and docs
  from the build context.

## AWS authentication (OIDC, no stored secrets)

Provisioned via [`infrastructure/template.yaml`](infrastructure/template.yaml):

- **ECR repository** `node-api` — image scanning on push, AES256 encryption,
  lifecycle policy expiring untagged images after 7 days and keeping the last
  20 tagged images, tagged with `Project` / `Owner` / `ManagedBy`.
- **IAM role** `github-actions-ecr-push` — trust policy restricted to this
  exact GitHub repository via the `token.actions.githubusercontent.com:sub`
  condition (pinned to the repository's immutable owner/repo IDs — see
  below); permission policy limited to `ecr:GetAuthorizationToken` (`*`, as
  required by the action) plus the push/pull actions scoped to the single
  repository ARN. No `ecr:DeleteRepository`, no IAM, no other service.
- **ECR repository policy** — a second, independent layer that allows only
  the `github-actions-ecr-push` role to push/pull layers on this repository,
  regardless of what else might later be granted to it via IAM.

GitHub's OIDC token embeds the numeric, immutable GitHub owner/repo IDs
(`repo:Iradukunda54@267279209/docker-ecr-oidc-lab@1344933210:ref:...`)
rather than the plain names once an account has ever been renamed, so the
trust condition is pinned to those IDs — matching on names alone caused the
first deploy to be rejected with `AccessDenied` (see below).

Deploy/update it with:

```bash
aws cloudformation deploy \
  --template-file infrastructure/template.yaml \
  --stack-name docker-ecr-oidc-lab \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides GitHubOrg=Iradukunda54 GitHubRepo=docker-ecr-oidc-lab EcrRepositoryName=node-api Owner=KevineIradukunda \
  --region eu-west-1
```

## CI/CD pipeline

[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml):

1. Triggers on every push to any branch.
2. `permissions: { id-token: write, contents: read }` — the minimum needed
   for OIDC, nothing else.
3. `aws-actions/configure-aws-credentials@v4` assumes
   `arn:aws:iam::447558491229:role/github-actions-ecr-push` via OIDC.
4. `aws-actions/amazon-ecr-login@v2` logs the Docker client into ECR.
5. Builds the image and tags it `kevineiradukunda_node-api` (plus a
   `-<short-sha>` variant for traceability) and pushes both tags.
6. `set -euo pipefail` and no `continue-on-error` anywhere — any failed step
   (build, login, push) aborts the job immediately, so the pipeline fails
   securely rather than silently continuing. This is not just a design
   claim: [run #32727405949](https://github.com/Iradukunda54/docker-ecr-oidc-lab/actions/runs/32727405949)
   is a real example — the OIDC trust condition was initially wrong, the
   `Configure AWS credentials` step failed, and the job stopped before the
   build or push steps ever ran.

## Deliverables

- GitHub repository: https://github.com/Iradukunda54/docker-ecr-oidc-lab
- Private ECR repository (console):
  https://eu-west-1.console.aws.amazon.com/ecr/repositories/private/447558491229/node-api?region=eu-west-1
  — image URI: `447558491229.dkr.ecr.eu-west-1.amazonaws.com/node-api:kevineiradukunda_node-api`
