# ci-templates

Central, reusable GitHub Actions pipelines. Every service repository calls these instead
of keeping its own copy of `build.yml`, so a security fix here reaches every service at once.

| Workflow | Called on | What it does |
|---|---|---|
| `.github/workflows/pr-checks.yml` | `pull_request` | Governance, Gitleaks, Semgrep, CodeQL, Dependency Review, Trivy fs, Hadolint, Trivy config, zizmor, unit tests, native audit, optional Sonar, then one aggregate **PR Gate** |
| `.github/workflows/build-deploy.yml` | `push` to `feature/*`, `develop`, `release/*` | Versioning, build, Trivy image gate, SBOM, ZAP DAST, push, Cosign keyless sign, SBOM attestation, SLSA provenance, Hanoi update, Jira comment |
| `.github/workflows/self-check.yml` | changes to this repo | actionlint + zizmor on the templates themselves |

## Using it from a service

```yaml
jobs:
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: <owner>/ci-templates/.github/workflows/pr-checks.yml@v1
    permissions: { contents: read, security-events: write, pull-requests: write, actions: read }
    with:
      language: python            # python | go | node | dotnet
      test-command: pytest -q --cov=app --cov-fail-under=80
      codeql-languages: '["python"]'
    secrets: inherit
  build-deploy:
    if: github.event_name != 'pull_request'
    uses: <owner>/ci-templates/.github/workflows/build-deploy.yml@v1
    permissions: { contents: read, packages: write, security-events: write, id-token: write, attestations: write }
    with: { app-port: 8080 }
    secrets: inherit
```

See each workflow's `inputs:` block for every option.

## Releasing a new template version

1. Open a PR to `main` and get `Self check` green.
2. Squash merge, then move the tag: `git tag -f v1 && git push -f origin v1`.
   For breaking changes, create `v2` and migrate services one by one.

Images are signed with a certificate whose identity is
`https://github.com/<owner>/ci-templates/.github/workflows/build-deploy.yml@<ref>`.
Kyverno and the release bundle only trust that identity, so an image built by any other
workflow will not run in the clusters.
