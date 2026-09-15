# SINTRATEL Docket - Organization CI

This is the organization-level `.github` repository for **Sintratel-Docket**. It
hosts the shared, reusable GitHub Actions workflow that every microservice CI
pipeline builds on, so the continuous-delivery logic lives in one place instead
of being copied into each service.

## Why this repository exists

The `todos-api` pipeline was originally written standalone. Replicating it
service by service would duplicate the security-sensitive delivery logic (AWS
OIDC, ECR push, semantic versioning) five times. Instead, that logic lives here
once and each microservice calls it.

The split follows a clear boundary:

| Stage | Lives where | Why |
| --- | --- | --- |
| Lint, tests, coverage, docker build-check | Each microservice repo (`ci.yml`) | Genuinely different per language (Go, Java, Node, Python, Vue) |
| Semantic versioning, ECR login, image build & push | This repo (`reusable-build-and-push.yml`) | Identical for every service; only the ECR repository name changes |

## Reusable workflow

`.github/workflows/reusable-build-and-push.yml` runs, in order:

1. Checkout (full history, needed by semantic-release).
2. Authenticate to AWS via **GitHub OIDC** — no long-lived AWS keys.
3. Log in to Amazon ECR.
4. Run **semantic-release**: analyzes Conventional Commits, computes the next
   semantic version, updates `CHANGELOG.md`, tags the release and publishes a
   GitHub release.
5. Build the Docker image from the caller repo's own `Dockerfile`.
6. Push it to ECR with an **immutable** `${version}-${sha}` tag. `latest` is
   never used. If semantic-release produces no new version, the push is skipped.

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `ecr_repository` | yes | — | ECR repository name, e.g. `docket/todos-api` |
| `aws_region` | no | `us-east-1` | Region of the ECR registry |
| `aws_role_arn` | no | `arn:aws:iam::429418377318:role/GitHubActionsECRPush` | IAM role assumed via OIDC |
| `node_version` | no | `22` | Node.js version used to run semantic-release |

## How a microservice consumes it

Each service keeps its own `.github/workflows/ci.yml` with stack-specific
`lint` / `test` / `docker-build-check` jobs, then adds a final `release` job that
delegates to this workflow:

```yaml
  release:
    name: Release and Publish Image
    needs: docker-build-check
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      id-token: write
      contents: write
    uses: Sintratel-Docket/.github/.github/workflows/reusable-build-and-push.yml@main
    with:
      ecr_repository: docket/<service-name>
    secrets: inherit
```

The service must also contain a `.releaserc.json` (the semantic-release
configuration; identical across all services) so the release step knows how to
version and tag.

> The double `.github/.github/` in the `uses:` path is expected: the first is the
> repository name, the second is the workflows directory inside it.

## Related repositories

- `docket-infra` — Terraform for AWS/EKS/ECR, the OIDC role this workflow assumes.
- `gitops-manifests` — Argo CD desired state; consumes the images this workflow pushes.
