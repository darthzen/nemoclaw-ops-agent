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
