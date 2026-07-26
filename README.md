# Platform Workflows

`platform-workflows` centralizes reusable, versioned CI/CD workflows for portfolio
projects. It keeps common automation consistent while allowing each application to
own its dependencies and source-specific configuration.

Reusable CI contracts are implemented for Python APIs and React/Vite web
applications. Both have been validated in a real consumer. Security scanning and
container publishing are planned but not implemented; deployment is outside the
v1 scope.

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

## Security

The workflows grant read-only access to repository contents, receive no secrets,
and use only official GitHub actions pinned to full commit SHAs. The Python
workflow uses fixed quality commands. React workflow command inputs are trusted
configuration controlled by the consumer repository.

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
3. **Security Scan** — planned and not yet implemented.
4. **Docker Build and Publish to GHCR** — planned and not yet implemented.

Docker Publish means building the image, scanning it, and pushing it to GitHub
Container Registry (GHCR). It does not include deployment. Deployment to EC2,
production, or any other environment is outside v1. Neither `v1.0.0` nor the `v1`
alias has been published.

## Current limitations

- Python API dependency installation supports pip requirements files only.
- React web CI supports npm only, requires a lockfile, and does not discover
  missing lint or test scripts.
- CI runs on GitHub-hosted Ubuntu runners and does not collect coverage, scan
  security, build images, publish containers, or deploy applications.
- React build artifact upload is optional; other workflows do not publish
  artifacts.

## Roadmap

1. Implement Security Scan.
2. Document and validate Security Scan in Sentinel using a full commit SHA.
3. Implement Docker Build and Publish to GHCR.
4. Document and validate Docker Publish in Sentinel using a full commit SHA.
5. Review permissions, actions pinned to full commit SHAs, contracts, and
   documentation.
6. Validate the complete v1 scope.
7. Publish `v1.0.0` and establish the `v1` alias.
8. Migrate Sentinel from validation SHAs to `@v1`.

### Future evolution outside v1

Potential capabilities outside v1 include Node.js API CI, Flutter CI, controlled
EC2 deployment with Docker Compose, advanced supply-chain security such as
CodeQL, SBOMs, attestations, and signing, plus release automation and governance.
Go API CI is also deferred until a real consumer exists.

New capabilities will only be implemented when a real consumer is available for
validation.
