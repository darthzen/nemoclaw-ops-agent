# T-002 acceptance — scoped-credential contract

Verified against live cluster (k3s v1.35.5, ash4d) on 2026-07-05.
Two identities, no admin ever handed to an agent:

- **sa-nemoclaw-deploy** — namespace-admin in `nemoclaw` only + cluster-read on CRD *definitions*.
- **sa-nemoclaw-observe** — cluster-wide `get/list/watch` on named diagnosis resources only.

## Negative + positive suite (`kubectl auth can-i --as=...`)

sa-nemoclaw-deploy (`system:serviceaccount:nemoclaw:sa-nemoclaw-deploy`):

| Action | Expected |
|---|---|
| `create deployment -n nemoclaw` | yes |
| `create secret -n nemoclaw` | yes |
| `* * -n nemoclaw` | yes |
| `list customresourcedefinitions` | yes |
| `delete pod -n kube-system` | **no** |
| `get secret -n kube-system` | **no** |
| `create deployment -n default` | **no** |
| `get nodes` | **no** |
| `delete certificates.cert-manager.io -n cert-manager` | **no** |

sa-nemoclaw-observe (`system:serviceaccount:nemoclaw:sa-nemoclaw-observe`):

| Action | Expected |
|---|---|
| `get pods -A` | yes |
| `list events -A` | yes |
| `get certificates.cert-manager.io -A` | yes |
| `list volumes.longhorn.io -A` | yes |
| `get prometheusrules.monitoring.coreos.com -A` | yes |
| `get pods/log -n kube-system` | yes |
| `delete pod -n kube-system` | **no** |
| `create configmap -n nemoclaw` | **no** |
| `patch certificate -n cert-manager` | **no** |
| `get secret -A` | **no** |
| `create deployment -n nemoclaw` | **no** |

**Result: 20/20 passed.**

## Live end-to-end (real token auth, exported kubeconfigs)

- observe kubeconfig: `get certificates.cert-manager.io -A` → returns rows; `get secret -A` → **Forbidden**.
- deploy kubeconfig: `auth can-i create deployment -n nemoclaw` → yes; `get pods -n kube-system` → **Forbidden**.

## Reproduce

```bash
D=system:serviceaccount:nemoclaw:sa-nemoclaw-deploy
O=system:serviceaccount:nemoclaw:sa-nemoclaw-observe
kubectl auth can-i delete pod -n kube-system --as=$D   # -> no
kubectl auth can-i get secret -A --as=$O               # -> no
kubectl auth can-i get certificates.cert-manager.io -A --as=$O  # -> yes
```

## Exported kubeconfigs (secrets — NOT in this repo)

Long-lived SA-token kubeconfigs for build-time agents live outside git in the
private portfolio folder (`_portfolio/nemoclaw-ops-agent/_creds/`), one per SA.
The running agent in-cluster uses its projected SA token, not these files.
Rotate by deleting `nemoclaw/<sa>-token` and re-running the export.
