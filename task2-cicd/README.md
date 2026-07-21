# Task 2 — Secure CI/CD Pipeline & Supply Chain

## Pipeline stages (in order)
1. **secrets-scan** (gitleaks) — hard-block. Any secret detected in code fails the pipeline immediately.
2. **sast-scan** (Semgrep, Python ruleset) — warn-only (`continue-on-error: true`). Findings uploaded but don't block build; reviewed manually.
3. **build-and-push** — builds image from `task1-hardening/ledger-api-assignment/app`, tags with immutable 12-char git SHA (never `:latest`), pushes to GHCR.
4. **image-scan** (Trivy) — two passes: warn on MEDIUM/HIGH (SARIF uploaded to Security tab), hard-block on CRITICAL (with documented `.trivyignore` exceptions for accepted risk).
5. **sign-and-attest** (Cosign keyless + GitHub native attestation) — signs the image digest using GitHub Actions OIDC identity (no key management), generates SLSA-style provenance attestation.

## Fail policy summary
| Gate | Policy |
|---|---|
| Secrets (gitleaks) | Hard block |
| SAST (Semgrep) | Warn only |
| Image CVEs (Trivy) | Hard block on CRITICAL, warn on MEDIUM/HIGH |
| Signing | Required — pipeline fails if signing fails |

CVE with no fix available: documented as an accepted-risk exception in `.trivyignore` with justification, reviewed each release.

## GitOps (ArgoCD)
- ArgoCD watches `task1-hardening/ledger-api-assignment/deploy` in this repo
- `automated: prune + selfHeal: true` — manual `kubectl scale` drift was auto-reverted to git state, proving self-heal
- Evidence: see `screenshots/` (Synced+Healthy status, drift/self-heal before-after)

## Evidence
See `screenshots/` for: GitHub Actions green runs, cosign verify output (Verified OK), ArgoCD application sync status, drift/self-heal proof.
