# Software Supply Chain Security

Covers OWASP Top 10:2025 **A03 Software Supply Chain Failures** and the dependency side of **A10 Mishandling of Exceptional Conditions** (failing open when a check cannot run). Every dependency, build step and CI action is code you ship.

## Principles

- Add a dependency only when it is cheaper than writing and maintaining the code. Prefer the standard library and well-maintained, widely used packages.
- Pin what you build from: commit lockfiles, install with the frozen/locked flag in CI, and review lockfile diffs like code.
- Verify before you trust: advisories, licences, maintainers, install scripts and transitive size.
- Fail closed: if a security check (audit, signature, checksum) cannot run, the pipeline fails.

## Before Adding A Dependency

1. Is it actively maintained (recent releases, responsive issues)?
2. Is the name exactly right? Typosquats (`reqeusts`, `colourama`) are a common attack.
3. What does it pull in transitively, and does it run code at install or build time (npm `postinstall`, Python `setup.py`, Rust `build.rs` and proc-macros)?
4. Is the licence compatible with your distribution?
5. Is there a smaller or built-in alternative?

## Lockfiles And Reproducible Installs

| Ecosystem | Commit | CI install | Audit |
|-----------|--------|-----------|-------|
| npm | `package-lock.json` | `npm ci` | `npm audit --omit=dev` |
| pnpm / yarn / bun | `pnpm-lock.yaml` / `yarn.lock` / `bun.lock` | `pnpm i --frozen-lockfile` / `yarn --immutable` / `bun install --frozen-lockfile` | `pnpm audit` |
| Python | `uv.lock` / `poetry.lock` / hashed `requirements.txt` | `uv sync --frozen` / `pip install --require-hashes -r requirements.txt` | `pip-audit` |
| Go | `go.sum` | `go mod verify` | `govulncheck ./...` |
| Rust | `Cargo.lock` | `cargo build --locked` | `cargo deny check`, `cargo audit` |
| Java | dependency locking / BOM | `--locked` where supported | OWASP Dependency-Check |

## Dependency Cooldown

Most malicious releases are detected and yanked within days. A short delay before adopting brand-new versions removes most of that exposure.

```toml
# Cargo (1.100+): .cargo/config.toml
[registry]
global-min-publish-age = "7 days"

[resolver]
incompatible-publish-age = "deny"
```

```bash
# uv: ignore packages published after a date
uv pip install --exclude-newer 2026-11-01 -r requirements.in
# npm, pnpm and yarn offer a minimum release age setting in recent versions; check your package manager's docs
```

Automate updates with Dependabot or Renovate configured with a cooldown, group non-security updates, and apply security updates promptly.

## CI And Build Pipeline

- Pin third-party CI actions to a full commit SHA, not a mutable tag; let Dependabot update them.
- Give workflows least privilege: `permissions: contents: read`, and grant `id-token: write` only to the deploy job.
- Prefer OIDC federation to long-lived cloud credentials. Never echo secrets; mask them.
- Do not run untrusted code (forks, PR titles, branch names) with secrets: avoid `pull_request_target` checkout of PR code and never interpolate `${{ github.event.* }}` directly into shell.
- Build releases in CI, not on a laptop. Publish with trusted publishing (npm, PyPI, crates.io) and provenance attestations where supported.

```yaml
permissions:
  contents: read

steps:
  - uses: actions/checkout@<full-commit-sha>   # v4.x, pinned
    with:
      persist-credentials: false
  - run: npm ci --ignore-scripts
  - run: npm audit --omit=dev --audit-level=high
```

## Install Scripts And Build-Time Code

- Install with scripts disabled where possible (`npm ci --ignore-scripts`, `pnpm` blocks by default) and allow-list the few packages that need them.
- Treat build scripts and procedural macros as arbitrary code execution; review them for new crates.
- Do not `curl | sh` installers in Dockerfiles or CI. Download a pinned version and verify its checksum.

## Containers And Artifacts

- Pin base images by digest (`image@sha256:...`) and rebuild regularly for patches.
- Use minimal bases (distroless, slim), run as non-root, scan images (Trivy, Grype).
- Generate an SBOM (CycloneDX or SPDX) for released artifacts and keep it with the release.
- Sign artifacts and verify signatures at deploy time (Sigstore/cosign).

## Responding To A Compromised Dependency

1. Identify affected versions and where they are deployed (lockfile and SBOM search).
2. Pin or roll back to a known-good version; rebuild and redeploy.
3. Rotate every secret the build or runtime could have read.
4. Check logs for the indicators published in the advisory.
5. Record the incident and tighten the control that missed it.

## Checklist

- [ ] Lockfiles are committed and CI installs from them (frozen/locked)
- [ ] Audit and licence checks run on every pull request and fail the build
- [ ] New dependencies are reviewed (maintenance, name, scripts, licence)
- [ ] CI actions are pinned by SHA; workflow permissions are minimal
- [ ] Releases are built in CI with provenance and an SBOM
