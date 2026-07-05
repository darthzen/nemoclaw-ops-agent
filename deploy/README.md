# deploy/

k8s-native manifests for the nemoclaw-ops-agent. Applied against the ash4d
k3s cluster (v1.35.5, containerd). Nothing here is Helm — plain manifests,
readable as portfolio material.

## Layout

- `rbac/` — the scoped-credential contract (T-002). Namespace, the two
  ServiceAccounts (`sa-nemoclaw-deploy`, `sa-nemoclaw-observe`), their
  Roles/ClusterRoles/bindings, and `TESTS.md` (the 20-check acceptance suite,
  all passing live).
- `netpol/` — default-deny + explicit egress pinholes (T-010): DNS, in-cluster
  mcpo + Ollama, and Slack-over-443 (RFC1918 excluded). The OpenShell egress
  lock expressed as k8s NetworkPolicy.

## Applied state (2026-07-05)

`rbac/` and `netpol/` are applied and verified server-side
(`kubectl apply --dry-run=server` clean; RBAC negative tests 20/20).

## NetworkPolicy enforcement note

k3s enforces NetworkPolicy via its in-binary controller (the cluster already
runs Rancher-authored policies, so enforcement is active). The policies here
are inert until the first pod lands in `nemoclaw` — full pod-to-pod
enforcement is verified as part of **T-011** (first deployment), by confirming
from inside the agent pod:

```bash
# expected: DNS ok, mcpo/ollama reachable, LAN + arbitrary egress blocked
kubectl -n nemoclaw exec deploy/<agent> -- sh -c 'nslookup ai-ollama || true'
kubectl -n nemoclaw exec deploy/<agent> -- sh -c 'curl -m5 http://ollama.ai:11434/api/tags'   # ok
kubectl -n nemoclaw exec deploy/<agent> -- sh -c 'curl -m5 http://192.168.7.149:6443 || echo BLOCKED'  # BLOCKED
```

## Credentials

Exported SA kubeconfigs are **not** in this repo (see `rbac/TESTS.md`). The
running agent uses its in-pod projected SA token.
