# Task 1 — Deploy & Harden ledger-api

## Overview
Deployed `ledger-api` (and a neighbour service, `reporting`) onto a local `kind`
Kubernetes cluster and hardened it from an insecure baseline (plaintext secrets,
root container, no network controls) to a production-grade configuration.

## What was done

### 1. Deployment & networking
- `ledger-api` (3 replicas) and `reporting` (neighbour service) deployed via
  Deployments + Services + ConfigMaps.
- `Ingress` (nginx ingress-nginx controller) exposes `ledger-api` at
  `ledger-api.local`, verified end-to-end with `curl` returning `{"status":"ok"}`.

### 2. Pod & container hardening
Both `ledger-api` and `reporting` run with:
- `runAsNonRoot: true`, explicit `runAsUser`/`runAsGroup` (10001) — no root user or group.
- `readOnlyRootFilesystem: true` — verified by attempting `touch /test.txt` inside
  the container, which fails with `Read-only file system`.
- `allowPrivilegeEscalation: false`, all Linux capabilities dropped (`drop: [ALL]`).
- `seccompProfile: RuntimeDefault`.
- Resource `requests`/`limits` and `livenessProbe`/`readinessProbe` on every container.

### 3. Least-privilege identity
- Each service has its own dedicated `ServiceAccount` (default SA not used).
- `automountServiceAccountToken: false` where the pod doesn't need API access.
- Namespaced `Role` scoped to only the permissions the service needs
  (e.g. `get`/`list` on `configmaps`), no wildcard verbs/resources.

### 4. Secrets management
- Secrets (`STRIPE_API_KEY`, `DB_PASSWORD`) are **not** committed in plaintext.
- Used **Bitnami Sealed Secrets**: a plaintext `Secret` is generated locally with
  `--dry-run=client`, immediately encrypted with `kubeseal` into a `SealedSecret`
  (only the in-cluster controller's private key can decrypt it), and the plaintext
  file is deleted and gitignored.
- Only the encrypted `SealedSecret` YAML is committed to git — safe to make public.

### 5. Admission control guardrails (Kyverno)
Three `ClusterPolicy` resources enforce security at admission time:
- `disallow-root-user` — rejects any Pod without `runAsNonRoot: true`.
- `disallow-latest-tag` — rejects any container image tagged `:latest`.
- `require-signed-images` — verifies container images are signed via Cosign
  keyless signing (wired up once Task 2's signing pipeline exists).

**Proof the guardrail works**: applying a deliberately insecure test Pod
(`nginx:latest`, `runAsNonRoot: false`) was rejected by the admission webhook —
see `screenshots/kyverno-block.png` for the exact error output.

### 6. Pod Security Standards
- `payments` namespace labeled `pod-security.kubernetes.io/enforce=restricted`.
- All workloads in the namespace pass the `restricted` profile (confirmed by
  redeploying and checking for PSS admission warnings — none present).

## Verification commands
```bash
kubectl get pods -n payments
kubectl exec -n payments -it <pod> -- id                 # non-root uid/gid
kubectl exec -n payments -it <pod> -- touch /test.txt     # read-only fs proof
kubectl apply -f /tmp/insecure-test-pod.yaml              # Kyverno rejection proof
curl -H "Host: ledger-api.local" http://localhost:8081/health
```

## What I'd do with more time (bonus items not fully completed)
- Developer / operator / admin persona RBAC roles (partially scoped, not fully
  built out across three tiers).
- Wire the `require-signed-images` Kyverno policy live against the actual
  Cosign keyless identity once Task 2's signing workflow is in place.

## Folder structure
```
task1-hardening/
├── ledger-api-assignment/
│   ├── app/                  # source app
│   └── deploy/                # deployment.yaml, service.yaml, neighbour.yaml,
│                               # ingress.yaml, rbac.yaml, sealed-secret.yaml
├── policies/                  # Kyverno ClusterPolicy YAMLs
└── screenshots/                # verification evidence
```
