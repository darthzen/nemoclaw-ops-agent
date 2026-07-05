# T-004 — OpenShell sandbox on k3s: go/no-go + port design

**Date:** 2026-07-05 · **Verdict: GO, and easier than PLAN.md assumed.**
Sources at bottom. Grounded in NVIDIA OpenShell + NemoClaw docs (latest as of
July 2026) and the live ash4d cluster (k3s v1.35.5, containerd 2.2.3).

## Bottom line (read this first)

- **OpenShell has a first-class Kubernetes compute driver.** You do **not** have
  to reimplement OpenShell's sandbox semantics on k8s primitives by hand. Set
  `compute_drivers = ["kubernetes"]` and the gateway creates each sandbox as a
  **pod** in a configured namespace.
- **This corrects a core PLAN.md rev-2 assumption.** The plan framed the k8s
  port as bespoke portfolio work ("translate egress→NetworkPolicy,
  fs→securityContext, by hand"). NVIDIA already ships that translation. The
  honest portfolio claim shifts from *"I re-built the sandbox on k8s"* to
  **"I ran OpenShell's Kubernetes driver against a real, busy homelab cluster
  (not the throwaway embedded-k3s demo), routed inference to on-prem Ollama, and
  wrapped it in defense-in-depth RBAC + NetworkPolicy + PSA."** Still real,
  still hard in the integration seams — just not a from-scratch reimplementation.
- **The difficulty Rick wanted is still there**, just relocated: installing the
  Agent Sandbox controller, solving supervisor sideload on k8s 1.35, and getting
  NemoClaw's Docker-assumptive default topology to target an external cluster.

## How OpenShell's k8s driver actually works

- Driver selection: `[openshell.gateway] compute_drivers = ["kubernetes"]`.
  When unset, the gateway auto-detects **Kubernetes → Podman → Docker**.
- Sandboxes are created as namespaced **`agents.x-k8s.io` `Sandbox`** resources
  from the Kubernetes SIG project **kubernetes-sigs/agent-sandbox**. A Sandbox
  controller reconciles those into pods + storage. The gateway probes the served
  API version (`v1beta1`, falls back to `v1alpha1`).
- Per-sandbox knobs map to native pod fields: `--cpu`/`--memory` → requests
  **and** limits; `--gpu N` → `nvidia.com/gpu` limit; `pod.runtime_class_name`
  → `runtimeClassName` (so kata-containers / gVisor via RuntimeClass is a
  one-field upgrade if we want a VM boundary later).
- Sandbox identity: a dedicated **sandbox ServiceAccount**; the gateway uses a
  projected SA token (TokenReview bootstrap). UID defaults to 1000 on
  non-OpenShift clusters.

## The real integration risks on THIS cluster (k3s v1.35.5)

1. **Supervisor sideload method.** OpenShell delivers the `openshell-sandbox`
   supervisor binary into each pod either via `image-volume` (needs k8s **1.33+
   ImageVolume feature gate; GA in 1.36**) or `init-container` (older clusters).
   We're on **1.35**, so ImageVolume exists but is still behind a feature gate.
   **Decision: set `supervisor_sideload_method = init-container`** — no feature
   gate, works today, one less moving part. Revisit image-volume after a 1.36
   bump.
2. **NemoClaw's default topology is NOT external-k8s.** Per NVIDIA's architecture
   doc, stock NemoClaw runs: host `nemoclaw` CLI → **Docker daemon** → gateway
   **container that embeds its own k3s** → sandbox pod inside that embedded
   cluster. That embedded k3s is a private, throwaway control plane — it is not
   the ash4d cluster. Two viable paths:
   - **(A) Point the OpenShell gateway's k8s driver at ash4d** (external cluster,
     `namespace = nemoclaw`, our `sandboxServiceAccount`). This is the "real
     cluster" story and what the portfolio should show. Gateway can run as a
     Deployment in `nemoclaw` (in-cluster kubeconfig via SA) or off-cluster with
     the exported `sa-nemoclaw-deploy` kubeconfig.
   - **(B) Run stock NemoClaw's Docker+embedded-k3s** on a lab host. Faster to
     first-light, but it's the demo topology, not "k8s-native on my cluster."
   **Decision: build toward (A); keep (B) as a fallback for first smoke test if
   the Agent Sandbox controller install fights us.**
3. **Agent Sandbox controller must be installed on ash4d.** The k8s driver has a
   hard dependency on the `agents.x-k8s.io` CRD + controller
   (kubernetes-sigs/agent-sandbox). This is a new install on the cluster and a
   net-new M1 sub-task. **Add ticket: T-011a "install + verify agent-sandbox
   controller (Sandbox CRD served, v1beta1)."**
4. **AppArmor.** OpenShell's Helm defaults the sandbox agent container's AppArmor
   profile to `Unconfined` so the supervisor can set up the network namespace.
   openSUSE Leap 16 ships AppArmor — confirm the sandbox pod isn't blocked; if
   PSA `restricted` rejects `Unconfined`, we reconcile PSA vs. the supervisor's
   netns needs (see §PSA note).

## Isolation model — what we actually get, stated honestly

- Default OpenShell sandbox isolation = **Landlock + seccomp + network
  namespace**, enforced by the in-pod supervisor. It is **not** a VM boundary and
  **not** gVisor by default; sandbox containers **share the host kernel** via the
  standard OCI runtime (containerd here). NVIDIA is tracking stronger options
  (gVisor / Firecracker / cluster-in-VM) in OpenShell issue #4.
- The **L7 "Privacy Router"** in the gateway strips caller credentials and
  injects backend credentials at egress — the sandbox never holds raw provider
  keys. This is the piece worth emphasizing: **credential proxying is preserved
  exactly, unchanged, by using the real driver** (not reimplemented).

## Our defense-in-depth wrap (this is the portfolio surface)

The driver gives process-level isolation; we harden the platform around it. Each
guarantee below is already applied in `deploy/` or is a named build step:

| OpenShell guarantee | Native k8s reinforcement (ours) | Where |
|---|---|---|
| Proxy-only egress (netns) | `default-deny-all` + explicit egress pinholes (DNS, mcpo, Ollama, Slack-443) | `deploy/netpol/` (applied) |
| Filesystem confinement (Landlock) | PSA `restricted` on ns + `readOnlyRootFilesystem`/non-root in the pod spec | `deploy/rbac/00-namespace.yaml` + T-011 |
| No raw provider keys (Privacy Router) | inference routed to in-cluster Ollama; provider creds in gateway store / k8s Secret, never in the pod | T-012, `deploy/netpol/20` |
| Least privilege | pod runs under `sa-nemoclaw-*`, never admin; SA scopes proven by negative tests | `deploy/rbac/TESTS.md` (20/20) |
| Stronger boundary (optional) | `runtimeClassName` → kata/gVisor is one field if we want a VM boundary | future |

### PSA note (a real tension to resolve in T-011)

The ns is labelled `pod-security.kubernetes.io/enforce: restricted`. OpenShell's
supervisor needs netns setup and Helm sets AppArmor `Unconfined`, which a strict
`restricted` policy can reject. Resolution options, in order of preference:
(1) keep `restricted` and set AppArmor to `RuntimeDefault` if the supervisor
tolerates it; (2) if netns setup demands it, drop the sandbox pods to a
`baseline`-labelled sub-namespace while keeping everything else `restricted`;
(3) last resort, targeted PSA exemption documented in the writeup. Decide with a
real pod in hand (T-011), not on paper.

## Concrete next steps (feed into M1 tickets)

- **T-011a (new):** install `kubernetes-sigs/agent-sandbox` controller on ash4d;
  verify `Sandbox` CRD served at `v1beta1`. Blocks T-011.
- **T-011:** OpenShell gateway Deployment in `nemoclaw` with
  `compute_drivers=["kubernetes"]`, `namespace=nemoclaw`,
  `service_account_name=<sandbox SA>`, `supervisor_sideload_method=init-container`,
  `default_image=ghcr.io/nvidia/openshell-community/sandboxes/openclaw`. Gateway
  auth uses the in-cluster SA (not admin).
- **T-012:** point OpenShell inference routing at `http://ollama.ai:11434`
  (Ollama/Qwen3) as the provider; smoke-test a turn.
- Reconcile PSA vs. AppArmor with the first real pod (§PSA note).

## Sources

- OpenShell — Sandbox Compute Drivers (Kubernetes driver, agent-sandbox CRD, sideload, GPU): https://docs.nvidia.com/openshell/reference/sandbox-compute-drivers
- NemoClaw — Architecture (Docker+embedded-k3s default topology, L7 Privacy Router, sandbox image): https://docs.nvidia.com/nemoclaw/latest/reference/architecture
- OpenShell — repo + isolation issue (Landlock/seccomp/netns; gVisor/Firecracker eval): https://github.com/NVIDIA/OpenShell and https://github.com/NVIDIA/OpenShell/issues/4
- kubernetes-sigs/agent-sandbox: https://github.com/kubernetes-sigs/agent-sandbox
