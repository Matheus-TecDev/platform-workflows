# Platform Workflows

`platform-workflows` centralizes reusable, versioned CI/CD workflows for portfolio
projects. It keeps common automation consistent while allowing each application to
own its dependencies and source-specific configuration.

Only reusable CI for Python APIs is implemented today. React CI, security scanning,
container publishing, and deployment are not current capabilities.

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

## Security

The workflow grants read-only access to repository contents, receives no secrets,
and uses only official GitHub actions pinned to full commit SHAs. Lint, formatting,
and test commands are fixed rather than caller-controlled, so the workflow is not
a generic shell executor.

## Versioning

Consumers should pin the workflow to a full commit SHA during validation. After a
real integration succeeds, an immutable SemVer release such as `v1.0.0` can provide
a human-readable exact version. A movable major alias such as `v1` should be
introduced only after compatibility and update practices have proven stable.

## Current limitations

- Dependency installation supports pip requirements files only.
- CI targets Python APIs and runs on GitHub-hosted Ubuntu runners.
- The workflow does not collect coverage, scan security, build images, publish
  artifacts, or deploy applications.

## Roadmap

1. Validate Python API CI in Sentinel using an immutable reference.
2. Add React CI when a real consumer integration is ready.
3. Add security scanning.
4. Add Docker image build and publication.
5. Add deployment only when a real environment is available for validation.
