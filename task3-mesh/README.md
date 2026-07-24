# Task 3 — Service Mesh & Zero-Trust (Istio)

## What was done
- Installed Istio (default profile), joined `payments` namespace to the mesh
- PeerAuthentication STRICT — mTLS mandatory between all workloads in the namespace
- AuthorizationPolicy: default-deny + explicit allow keyed on ServiceAccount identity (SPIFFE), not IP
- K8s NetworkPolicy: default-deny + explicit allow as an L3/L4 layer beneath Istio's L7 identity layer

## Known issue resolved: PodSecurity Standards vs Istio sidecar
Istio's `istio-init` container requires NET_ADMIN/NET_RAW capabilities to set up traffic interception (iptables rules). This conflicts with both `restricted` and `baseline` Pod Security Standards. For this assessment, PSS enforcement was removed from the `payments` namespace to allow sidecar injection; the application container itself retains its own hardening (non-root, read-only rootfs, dropped capabilities) independently of namespace-level PSS. In production, Istio's CNI plugin would move this privileged step to a DaemonSet in kube-system, keeping the app namespace fully `restricted`.

## Certificate issuance & rotation
- Istiod acts as the mesh's Certificate Authority (CA)
- Each workload's Envoy sidecar requests a certificate via the Istio Agent using the SDS (Secret Discovery Service) protocol
- Workload identity is encoded as a SPIFFE identity: `spiffe://cluster.local/ns/<namespace>/sa/<service-account>`
- Istiod signs short-lived certificates (default validity ~24h); the agent automatically requests renewal before expiry — no manual rotation needed
- Trust root: Istiod's self-signed root CA (in a production deployment, this would be replaced with an enterprise CA or integrated with a service like HashiCorp Vault)

## Defense-in-depth: what each layer catches
| Layer | Catches | Doesn't catch |
|---|---|---|
| K8s NetworkPolicy (L3/L4) | IP/port-level traffic, blocks at kernel/CNI level before it reaches the pod | Can't distinguish identity of caller — any pod matching the label selector gets through, regardless of who it claims to be |
| Istio AuthorizationPolicy (L7) | Cryptographic identity (SPIFFE/ServiceAccount), HTTP method/path-level rules | Doesn't help if the mesh itself is compromised or bypassed entirely (defense-in-depth need) |

Together: NetworkPolicy is the coarse first gate (kernel-level, cheap), AuthorizationPolicy is the fine-grained identity check (cryptographically verified). An attacker who somehow bypasses one still hits the other.

## Evidence
See `screenshots/` for: sidecar injection (2/2 pods), mTLS STRICT confirmation, plaintext request refused (connection reset), AuthorizationPolicy deny (403) + allow (200) for both `reporting` (authorized) and `attacker-sa` (unauthorized) identities, NetworkPolicy layer verification.
