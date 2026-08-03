# Platform Workflows

`platform-workflows` centralizes reusable, versioned CI/CD workflows for portfolio
projects. It keeps common automation consistent while allowing each application to
own its dependencies and source-specific configuration.

Reusable CI contracts are implemented for Python APIs and React/Vite web
applications. Security scanning and Docker image publishing to GHCR are also
implemented. All four planned v1 reusable workflows have been validated in a
real consumer. Deployment is outside the v1 scope.

## Python API CI

The reusable workflow at
[`.github/workflows/python-api-ci.yml`](.github/workflows/python-api-ci.yml) checks
out the consumer repository, installs its Python dependencies, and runs the same
quality gates for every API:

```bash
python -m ruff check .
python -m ruff format --check .
python -m pytest
```

The central workflow owns runner selection, dependency caching, commands, timeout,
and baseline permissions. The consumer owns application code, dependency files,
tool configuration, and fixes for failed checks.

### Consumer contract

At minimum, the consumer must contain:

```text
.
├── requirements.txt
└── application and test files
```

`requirements.txt` is required. `requirements-dev.txt` is optional and is installed
when present. The resolved dependencies must provide Ruff and Pytest; the workflow
does not install undeclared tools. Ruff and Pytest configuration may live in the
consumer's standard project configuration files.

The workflow accepts these string inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `python-version` | `3.12` | Python version used by the job |
| `working-directory` | `.` | Directory containing the Python API |
| `requirements-file` | `requirements.txt` | Runtime requirements path, relative to the working directory |
| `dev-requirements-file` | `requirements-dev.txt` | Optional development requirements path, relative to the working directory |

If the required dependency file does not exist, CI stops with an explicit error.
A missing development dependency file does not fail the job.

### Usage

Copy [`examples/python-api.yml`](examples/python-api.yml) into the consumer
repository as a workflow, then replace `REPLACE_WITH_FULL_COMMIT_SHA` with the
published commit SHA:

```yaml
jobs:
  ci:
    uses: Matheus-TecDev/platform-workflows/.github/workflows/python-api-ci.yml@REPLACE_WITH_FULL_COMMIT_SHA
```

Only pass inputs that differ from, or intentionally document, the defaults.

## React/Vite Web CI

The reusable workflow at
[`.github/workflows/react-web-ci.yml`](.github/workflows/react-web-ci.yml) supports
React/Vite applications using Node.js, npm, and a committed `package-lock.json`.
It checks out the consumer, restores setup-node's native npm cache, runs
installation and lint, optionally runs an explicitly configured test command,
builds the production application, and optionally uploads only the declared build
output.

### Consumer contract

The workflow accepts these inputs:

| Input | Type | Default | Purpose |
| --- | --- | --- | --- |
| `node-version` | string | `22` | Node.js version used by the job |
| `working-directory` | string | `.` | Directory containing the application and lockfile |
| `install-command` | string | `npm ci` | Reproducible dependency installation |
| `lint-command` | string | `npm run lint` | Required lint command |
| `test-command` | string | empty | Optional test command; empty means no suite is configured and skips the step |
| `build-command` | string | `npm run build` | Required production build |
| `artifact-path` | string | `dist` | Build output path relative to `working-directory` |
| `upload-artifact` | boolean | `false` | Whether to upload the build output |
| `artifact-name` | string | `react-web-dist` | Uploaded artifact name |

`package-lock.json` must exist directly inside `working-directory`; it is used by
both `npm ci` and the npm cache. Commands run in `working-directory`.
`artifact-path` is also relative to that directory. When upload is enabled, a
missing artifact fails the job and the artifact is retained for seven days.

The command inputs are trusted CI code maintained by the consumer repository, not
untrusted user input. They are passed through environment variables to Bash and
are not evaluated with `eval`. Callers should keep workflow-edit permissions
restricted and must not interpolate untrusted event data into these inputs.

### Usage

Copy [`examples/react-web.yml`](examples/react-web.yml) into the consumer
repository. It demonstrates an application in `frontend/`. Replace
`REPLACE_WITH_FULL_COMMIT_SHA` with the published 40-character commit SHA:

```yaml
jobs:
  react-ci:
    uses: Matheus-TecDev/platform-workflows/.github/workflows/react-web-ci.yml@REPLACE_WITH_FULL_COMMIT_SHA
    with:
      working-directory: frontend
      test-command: ""
```

The placeholder is intentional: this example is configuration documentation and
is not executable until the implementation is published and the reference is
replaced. Do not use `main`; no `v1` alias exists yet.

## Security Scan

The reusable workflow at
[`.github/workflows/security-scan.yml`](.github/workflows/security-scan.yml)
provides blocking security checks for secrets, dependency vulnerabilities, and
configuration weaknesses. Consumers call it through `workflow_call` with
read-only access to repository contents and no secrets.

The enabled scanners run as independent jobs after contract validation, so they
can execute in parallel. All three scanners are enabled by default, and at least
one must remain enabled. The workflow fails with an explicit configuration error
if all three are disabled.

### Scanners

1. **Secrets:** Gitleaks checks the consumer's complete Git history, independently
   of `working-directory`. It redacts detected secret values from output and
   blocks the pipeline when it finds a leak.
2. **Vulnerabilities:** Trivy filesystem scans dependencies under
   `working-directory`, applies `severity` and `ignore-unfixed`, and blocks the
   pipeline when a vulnerability violates the configured policy.
3. **Misconfigurations:** Trivy config scans supported Dockerfiles and
   infrastructure-as-code files under `working-directory`, applies `severity`,
   and blocks the pipeline when a misconfiguration violates the policy.

### Consumer contract

| Input | Type | Default | Description |
| --- | --- | --- | --- |
| `working-directory` | `string` | `.` | Path scanned by Trivy for vulnerabilities and misconfigurations |
| `severity` | `string` | `HIGH,CRITICAL` | Severities that cause Trivy scans to fail |
| `scan-secrets` | `boolean` | `true` | Enables Gitleaks scanning across the Git history |
| `scan-vulnerabilities` | `boolean` | `true` | Enables Trivy filesystem vulnerability scanning |
| `scan-misconfigurations` | `boolean` | `true` | Enables Trivy configuration scanning |
| `ignore-unfixed` | `boolean` | `true` | Ignores vulnerabilities without an available fix |

Copy [`examples/security-scan.yml`](examples/security-scan.yml) into the consumer
repository for a minimal configuration that uses all defaults. Override inputs
only when the consumer needs a different policy:

```yaml
jobs:
  security:
    uses: Matheus-TecDev/platform-workflows/.github/workflows/security-scan.yml@25a2735378c5e6782b42a8bf80feb3310ebf254e
    with:
      working-directory: .
      severity: CRITICAL
      scan-secrets: true
      scan-vulnerabilities: true
      scan-misconfigurations: true
      ignore-unfixed: true
```

This reusable workflow was validated in Sentinel through
`.github/workflows/security.yml`, which calls
`security-scan.yml@4741c2c5a01a194a0caa3f066d9433e278173660` with the default
inputs, `contents: read`, and no secrets. Successful pull request and `main`
push runs executed contract validation, Gitleaks secret scanning, Trivy
filesystem vulnerability scanning, and Trivy configuration scanning.

It does not generate or upload SARIF, run CodeQL, produce SBOMs or attestations,
sign artifacts, publish images, or deploy applications.

## Docker Publish

The reusable workflow at
[`.github/workflows/docker-publish.yml`](.github/workflows/docker-publish.yml)
builds one local Docker image, scans that local image with Trivy in blocking
mode, and publishes to GitHub Container Registry only for `push` events on
`refs/heads/main`. Pull requests and any other context run build and scan only;
they do not log in to GHCR and do not publish an image.

Publication uses the same local image that passed the scan. The workflow tags the
image as `sha-<full 40-character commit SHA>`, pushes that tag, resolves the
remote digest, and exposes the remote digest plus immutable image reference as
outputs. It does not publish `latest`, abbreviated SHAs, branch names, semantic
versions, or multiple tags.

### Consumer contract

The reusable workflow has only `workflow_call`; `pull_request` and `push` events
belong to the consumer workflow. During validation, pin the call to a published
full 40-character commit SHA:

```yaml
uses: Matheus-TecDev/platform-workflows/.github/workflows/docker-publish.yml@729836ae9c9b001051c4f4a46d60ef37f87f5380
```

The workflow exposes exactly these public inputs:

| Input | Type | Required | Default | Description |
| --- | --- | ---: | --- | --- |
| `context` | `string` | yes | none | Docker build context |
| `dockerfile` | `string` | yes | none | Dockerfile path relative to the consumer repository |
| `image-name` | `string` | yes | none | Full GHCR image name without a tag or digest |

Example `image-name`: `ghcr.io/matheus-tecdev/example-api`.

`image-name` is rejected when it is empty, outside `ghcr.io`, contains a tag,
contains a digest, or is incompatible with the implemented image-reference
validation. The implemented validation accepts lowercase GHCR names only; use
lowercase owner, organization, repository, and image path components.

The workflow declares no `workflow_call` secrets, does not require a PAT, and
does not use `GHCR_TOKEN`. GHCR authentication uses `github.token`. No
credential is passed to `docker build`, and login occurs only after Trivy passes.
Do not use `secrets: inherit` for this workflow.

### Events and permissions

The reusable workflow decides whether to validate or publish from the caller's
event context:

| Caller event context | Internal job | Behavior |
| --- | --- | --- |
| `pull_request` | `validate` | Check out, validate inputs, build the local image, and run the blocking Trivy image scan. No GHCR login or publication occurs. |
| Any event other than `push` to `refs/heads/main` | `validate` | Same validation-only path as pull requests. |
| `push` to `refs/heads/main` | `publish` | Check out, validate inputs, build the local image, run the blocking Trivy image scan, log in to GHCR, push the scanned image, resolve the remote digest, and set outputs. |

For validation-only contexts, the reusable job uses read-only contents access:

```yaml
permissions:
  contents: read
```

For `push` to `main`, the publish job also needs package write access:

```yaml
permissions:
  contents: read
  packages: write
```

That path runs:

```text
checkout -> single build -> scan -> login -> push same image -> remote digest
```

The reusable workflow cannot elevate permissions that the caller did not grant,
so consumer workflows that expect to publish must make `packages: write`
available to the calling job or workflow. Any blocking Trivy failure prevents
GHCR login and push.

If the GHCR package already exists and is not automatically linked to the
consumer repository, GitHub may reject publication even with `GITHUB_TOKEN`. In
that case, grant the consumer repository `Write` access in the package settings:
`Package settings -> Manage Actions access`.

### Image metadata and scan policy

The published tag format is:

```text
ghcr.io/owner/image:sha-0123456789abcdef0123456789abcdef01234567
```

The build adds these OCI labels:

| Label | Value source |
| --- | --- |
| `org.opencontainers.image.source` | `github.server_url` and `github.repository` from the caller |
| `org.opencontainers.image.revision` | caller `github.sha` |
| `org.opencontainers.image.created` | UTC build timestamp |
| `org.opencontainers.image.title` | final path component of `image-name` |

Trivy scans the local image with this policy:

```text
scanners=vuln
severity=HIGH,CRITICAL
ignore-unfixed=true
exit-code=1
```

The scan is blocking. The image is not rebuilt after scanning; only the same
approved local image can be pushed.

### Outputs

| Output | Description |
| --- | --- |
| `digest` | Remote published digest in `sha256:...` format |
| `image-reference` | Immutable reference in `image@sha256:...` format |

Both outputs are populated only when publication occurs. They can remain empty in
pull requests and other contexts without a push to `main`. The digest is the
remote registry digest, not the local Docker image ID.

When using a caller job such as
[`examples/docker-publish.yml`](examples/docker-publish.yml), later jobs can read
the immutable reference with the relevant job id:

```yaml
needs:
  - backend
steps:
  - run: echo "${{ needs.backend.outputs.image-reference }}"
```

Current limits are intentional scope boundaries: no deploy, multiple platforms,
configurable cache, `latest`, semantic tags, SBOM, provenance, attestations,
signing, Cosign, or SARIF publication.

## Security

The workflows grant read-only access to repository contents, receive no secrets,
and use only official GitHub actions pinned to full commit SHAs. The Python
workflow uses fixed quality commands. React workflow command inputs are trusted
configuration controlled by the consumer repository. Docker Publish authenticates
to GHCR with `github.token` only after the image has passed the blocking local
scan.

## Versioning

Consumers should pin the workflow to a full commit SHA during validation. After a
real integration succeeds, an immutable SemVer release such as `v1.0.0` can provide
a human-readable exact version. A movable major alias such as `v1` should be
introduced only after compatibility and update practices have proven stable.

## Real-world validation

[Sentinel](https://github.com/Matheus-TecDev/Sentinel) is the first real consumer
of these reusable workflows. Its integrations use immutable, full 40-character
commit SHAs:

- Python API CI was integrated in
  [`f57eaa612f598aba7dc941acd353af86be28f730`](https://github.com/Matheus-TecDev/Sentinel/commit/f57eaa612f598aba7dc941acd353af86be28f730),
  referencing `python-api-ci.yml@b4ec59acecdabe9bc18efe0f0de18b92f2ba49c7`.
- React/Vite Web CI was integrated in
  [`593333ce872b23763a4cb73194ff1c21f86b4d4d`](https://github.com/Matheus-TecDev/Sentinel/commit/593333ce872b23763a4cb73194ff1c21f86b4d4d),
  referencing `react-web-ci.yml@55e78600eda5a676b08227d85c0398f864f8bd3a`.
- Security Scan was integrated in
  [`c45b4f2563b5366cfdbeac97d613b3b2b44c1812`](https://github.com/Matheus-TecDev/Sentinel/commit/c45b4f2563b5366cfdbeac97d613b3b2b44c1812),
  referencing `security-scan.yml@4741c2c5a01a194a0caa3f066d9433e278173660`.
  Later successful Security workflow runs executed all default scanner jobs in
  the reusable workflow.
- Docker Publish was validated in Sentinel with pull requests running build and
  Trivy without publication, and pushes to `main` running build, Trivy, and GHCR
  publication with `GITHUB_TOKEN` and no custom credentials.

The complete
[Sentinel GitHub Actions run](https://github.com/Matheus-TecDev/Sentinel/actions/runs/30163835829)
finished successfully after rerunning a transient PostgreSQL image pull failure.
It proved the backend quality gates, frontend lint and production build, backend
integration with PostgreSQL, Sentinel's existing Docker build and publication,
and Docker Compose validation. Sentinel explicitly declared that no frontend test
suite was configured with `test-command: ""`, and kept artifact upload disabled
with `upload-artifact: false`.

The reusable CI jobs preserved the consumer pipeline's other jobs, dependencies,
and execution flow. This validation completes **Scope 1 — Reusable CI
Foundation**.

## Planned v1.0.0 scope

The planned first stable release has four capabilities:

1. **Python API CI** — implemented and validated.
2. **React/Vite Web CI** — implemented and validated.
3. **Security Scan** — implemented and validated.
4. **Docker Build and Publish to GHCR** — implemented and validated.

Docker Publish means building the image, scanning it, and pushing it to GitHub
Container Registry (GHCR) only on `push` to `main`. It does not include
deployment. Deployment to EC2, production, or any other environment is outside
v1. Neither `v1.0.0` nor the `v1` alias has been published.

## Current limitations

- Python API dependency installation supports pip requirements files only.
- React web CI supports npm only, requires a lockfile, and does not discover
  missing lint or test scripts.
- CI runs on GitHub-hosted Ubuntu runners and does not collect coverage or
  deploy applications.
- React build artifact upload is optional; other workflows do not publish
  artifacts.

## Roadmap

The complete planned v1 workflow scope has been implemented, documented, and
validated in Sentinel. Remaining release steps:

1. Publish `v1.0.0` and establish the `v1` alias.
2. Migrate Sentinel from validation SHAs to `@v1`.

### Future evolution outside v1

Potential capabilities outside v1 include Node.js API CI, Flutter CI, controlled
EC2 deployment with Docker Compose, advanced supply-chain security such as
CodeQL, SBOMs, attestations, and signing, plus release automation and governance.
Go API CI is also deferred until a real consumer exists.

New capabilities will only be implemented when a real consumer is available for
validation.
