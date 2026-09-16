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
| Lint, tests, coverage | Each microservice repo (`ci.yml`) | Genuinely different per language (Go, Java, Node, Python, Vue) |
| Docker build check, Trivy source and image scan | This repo (`reusable-security-scan.yml`) | Identical for every service; only the image name changes |
| Semantic versioning, ECR login, image build & push | This repo (`reusable-build-and-push.yml`) | Identical for every service; only the ECR repository name changes |

## Reusable workflows

### `reusable-build-and-push.yml`

`.github/workflows/reusable-build-and-push.yml` runs, in order:

1. Checkout (full history, needed by semantic-release).
2. Authenticate to AWS via **GitHub OIDC** — no long-lived AWS keys.
3. Log in to Amazon ECR.
4. Run **semantic-release**: analyzes Conventional Commits, computes the next
   semantic version, updates `CHANGELOG.md`, tags the release and publishes a
   GitHub release.
5. Build the Docker image from the caller repo's own `Dockerfile`.
6. Scan the image with **Trivy**, upload the SARIF to GitHub code scanning and
   optionally fail before the push.
7. Push it to ECR with an **immutable** `${version}-${sha}` tag. `latest` is
   never used. If semantic-release produces no new version, the push is skipped.

#### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `ecr_repository` | yes | — | ECR repository name, e.g. `docket/todos-api` |
| `aws_region` | no | `us-east-1` | Region of the ECR registry |
| `aws_role_arn` | no | `arn:aws:iam::429418377318:role/GitHubActionsECRPush` | IAM role assumed via OIDC |
| `node_version` | no | `22` | Node.js version used to run semantic-release |
| `trivy_severity` | no | `HIGH,CRITICAL` | Severities the image gate considers |
| `trivy_exit_code` | no | `"0"` | `"1"` blocks the push on findings, `"0"` only reports |

### `reusable-security-scan.yml`

Replaces the per-repo `docker-build-check` job. It builds the image from the
caller's `Dockerfile` (so the build is still verified) and then runs Trivy
twice over two targets:

- `trivy fs` over the source tree — dependency vulnerabilities, hardcoded
  secrets and Dockerfile misconfigurations.
- `trivy image` over the image just built.

Each target is scanned once in SARIF format (uploaded to the Security tab,
never fails) and once in table format (readable log, enforces the gate). Trivy
ignores `exit-code` when writing SARIF, which is why a single pass cannot both
report and block.

Unlike the release workflow, this one runs on **pull requests**, so problems
surface before merge rather than at push-to-ECR time.

#### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `image_name` | yes | — | Local image name to build and scan, e.g. `todos-api` |
| `trivy_severity` | no | `HIGH,CRITICAL` | Severities the gate considers |
| `trivy_exit_code` | no | `"0"` | `"1"` fails the build on findings, `"0"` only reports |
| `ignore_unfixed` | no | `true` | Ignore vulnerabilities with no fix available |

> **Where findings appear.** Every finding is printed in full in the job log,
> under the `Gate on findings` steps. They are *also* uploaded to the Security
> tab, but code scanning requires GitHub Advanced Security on private
> repositories — the five service repos are private, so the upload is marked
> `continue-on-error` and is expected to be skipped there. The job log is the
> reliable place to read results.

> Both scan gates ship in **warn mode**. Flip `trivy_exit_code` to `"1"` per
> repository once the existing findings have been triaged, so enabling security
> scanning does not break all five pipelines on day one.

## How a microservice consumes it

Each service keeps its own `.github/workflows/ci.yml` with stack-specific
`lint` / `test` jobs, then delegates the last two stages to this repository:

```yaml
  build-and-scan:
    name: Build and Scan Image
    needs: test
    permissions:
      contents: read
      security-events: write
    uses: Sintratel-Docket/.github/.github/workflows/reusable-security-scan.yml@main
    with:
      image_name: <service-name>

  release:
    name: Release and Publish Image
    needs: build-and-scan
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      id-token: write
      contents: write
      security-events: write
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

- `docket-infra` — Terraform for AWS/EKS/ECR, the OIDC role this workflow
  assumes. Scans its own Terraform with `trivy config`.
- `gitops-manifests` — Argo CD desired state; consumes the images this workflow
  pushes. Scans its own manifests with `trivy config`.
