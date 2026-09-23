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
| Static analysis and coverage gate | This repo (`reusable-sonarqube-scan.yml`) | Identical for every service; only the project key and coverage report path change |
| Semantic versioning, ECR login, image build & push | This repo (`reusable-build-and-push.yml`) | Identical for every service; only the ECR repository name changes |

## Reusable workflows

### `reusable-build-and-push.yml`

Runs, in order:

1. Checkout (full history, needed by semantic-release).
2. Authenticate to AWS via **GitHub OIDC** — no long-lived AWS keys.
3. Log in to Amazon ECR.
4. Run **semantic-release**: analyzes Conventional Commits, computes the next
   semantic version, updates `CHANGELOG.md`, tags the release and publishes a
   GitHub release.
5. Build the Docker image from the caller repo's own `Dockerfile`.
6. Push it to ECR with an **immutable** `${version}-${sha}` tag. `latest` is
   never used. If semantic-release produces no new version, the push is skipped.

#### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `ecr_repository` | yes | — | ECR repository name, e.g. `docket/todos-api` |
| `aws_region` | no | `us-east-1` | Region of the ECR registry |
| `aws_role_arn` | no | `arn:aws:iam::429418377318:role/GitHubActionsECRPush` | IAM role assumed via OIDC |
| `node_version` | no | `22` | Node.js version used to run semantic-release |

### `reusable-sonarqube-scan.yml`

Runs `sonar-scanner` against the self-hosted SonarQube Community Edition
deployed by `gitops-manifests` (`dev/apps/sonarqube.yaml`), then waits on its
quality gate. The job is guarded by `if: vars.SONAR_HOST_URL != ''`, so it is
skipped (not failed) in every repository until the organization variable
`SONAR_HOST_URL` is set to an address GitHub Actions can reach and a
`SONAR_TOKEN` secret is configured.

The quality gate step ships with `continue-on-error: true`, the same warn-mode
spirit as the Trivy gate: flip it off per repository once existing findings
have been triaged.

#### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `project_key` | yes | — | SonarQube project key, e.g. `docket-todos-api` |
| `coverage_artifact` | no | `""` | Name of the coverage artifact uploaded by the `test` job, if any |
| `sonar_args` | no | `""` | Extra `sonar-scanner` properties, e.g. coverage report paths |

## How a microservice consumes it

Each service keeps its own `.github/workflows/ci.yml` with stack-specific
`lint` / `test` / `docker-build-check` jobs, then calls the shared workflows:

```yaml
  sonarqube:
    name: SonarQube Scan
    needs: test
    uses: Sintratel-Docket/.github/.github/workflows/reusable-sonarqube-scan.yml@main
    with:
      project_key: docket-<service-name>
      coverage_artifact: coverage-<service-name>
    secrets: inherit

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
- `gitops-manifests` — Argo CD desired state; consumes the images this workflow pushes,
  and deploys the self-hosted SonarQube instance `reusable-sonarqube-scan.yml` targets.
