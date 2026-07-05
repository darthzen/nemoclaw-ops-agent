# OpenShell gateway (T-011)

Gateway deployed via the official experimental Helm chart
(`oci://ghcr.io/nvidia/openshell/helm-chart`, dev channel) into its own
`openshell` namespace; sandboxes are created in `nemoclaw`.

## Applied state (2026-07-05, ash4d k3s v1.35.5)

- `openshell-0` StatefulSet pod **Ready 1/1**; PVC `openshell-data-openshell-0`
  **Bound → longhorn**.
- PKI pre-install hook generated `openshell-server-tls`, `openshell-client-tls`,
  `openshell-jwt-keys` (no secrets in Helm history).
- Sandbox identity `openshell-sandbox` SA + Role/RoleBinding created in `nemoclaw`.
- Chart's `openshell-sandbox-ssh` NetworkPolicy (SSH ingress from gateway) added
  in `nemoclaw`; our `egress-openshell-gateway` pinhole (nemoclaw → openshell:8080)
  added so sandbox supervisors can call back.

## Remaining before a sandbox can launch (T-011 acceptance: pod Ready in nemoclaw)

1. **PSA decision (owner: Rick).** OpenShell sets sandbox pods to AppArmor
   `Unconfined` for supervisor netns setup — incompatible with the `restricted`
   PSA currently on `nemoclaw`. The namespace must move to **`baseline`**.
   Compensating controls that remain: default-deny NetworkPolicy + egress
   allowlist, least-privilege RBAC (sandbox SA), the sandbox's own
   Landlock/seccomp/netns, and (optional) a stronger `runtimeClassName`
   (kata/gVisor) via `server.defaultRuntimeClassName`.
2. Launch a sandbox and confirm `Ready` in `nemoclaw`, then wire inference
   (T-012 → `ollama.ai:11434`) and the OpenClaw sandbox image.

## Notes

- Gateway PVC bound to `longhorn` because it is the newest of TWO default
  StorageClasses on ash4d (local-path + longhorn both marked default — a latent
  cluster misconfig worth fixing separately).
- mTLS is on by default; CLI access needs the `openshell-client-tls` material.

---

## T-011 resolved (2026-07-05) — real sandbox running, isolation proven

The gateway-created OpenShell supervisor pod is **privileged by design**
(`runAsUser:0`, caps `SYS_ADMIN,NET_ADMIN,SYS_PTRACE,SYSLOG`, AppArmor
Unconfined) — it builds the agent's Landlock/seccomp/netns sandbox from inside
the pod. PSA `restricted` **and** `baseline` both reject it. Resolution:

- Sandbox pods run in a dedicated **`nemoclaw-sandboxes`** namespace at PSA
  **`privileged`** (`deploy/sandboxes/00-namespace-and-netpol.yaml`); the
  control-plane ns `nemoclaw` stays `restricted`.
- Gateway repointed via `server.sandboxNamespace=nemoclaw-sandboxes`.
- Gateway auth: **dev-principal lab mode** (`server.auth.allowUnauthenticatedUsers=true`)
  — mTLS still enforced for transport; LAN-only. OIDC is the production path.
- **Cross-ns secret dependency:** the sandbox mounts `openshell-client-tls`
  (its mTLS creds to the gateway). When `sandboxNamespace` != release namespace,
  that secret must be replicated into the sandbox ns (copied from `openshell`).
- Base sandbox image `ghcr.io/nvidia/openshell-community/sandboxes/base:latest`
  is ~1.4 GB (first pull ~2m20s; cached after).

### Isolation proven (from inside the running privileged pod)

| Target | Policy | Result |
|---|---|---|
| Ollama `10.43.151.251:11434` | allow | **REACHABLE** |
| kube-dns `10.43.0.10:53` | allow | **REACHABLE** |
| LAN k8s API `192.168.7.149:6443` | deny (RFC1918) | **blocked/timeout** |

So even a root + `SYS_ADMIN` pod cannot reach the LAN — confinement comes from
the egress NetworkPolicy, not pod privilege. Windowed hardening still to add
(T-011b): node SELinux + a kata/gVisor `runtimeClass` microVM boundary.
