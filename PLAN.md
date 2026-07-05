# NemoClaw Ops Agent — Portfolio Plan

**Status:** Design confirmed — pre-build (rev 2)
**Date:** 2026-07-05 (rev 1: 2026-07-04)
**Purpose:** A genuine, self-hosted autonomous-agent build on OpenClaw/NemoClaw, deployed against the ash4d k3s lab. Reusable portfolio proof that converts the "do you have OpenClaw/NemoClaw experience?" gate into "yes — here's the repo, the writeup, and a live demo" for any future posting in that category.

> Built for the asset and the next opportunity. The reference job (Senior Python / Claw-style agents) is already deep in interviews; this outlives it.

## Decisions locked (this rev)

| Decision | Outcome |
|---|---|
| Scenario scope | Delivery + runtime ops, confirmed. The "too limited / CI-bot" concern is resolved by behavior breadth (§2), keeping cluster ops as the domain. |
| First MVP signal | cert-manager cert expiry (deterministic, stageable, zero blast radius) |
| v1 autonomous allowlist | Read + annotate only; everything mutating requires approval (§7 autonomy ladder) |
| Deployment target | k3s-native, hard requirement. Difficulty is a feature — porting OpenShell's sandbox model to k8s primitives is itself portfolio material. |
| Stack | NemoClaw specifically (enterprise-hardened path). Plain OpenClaw rejected on security grounds; any k8s port must preserve OpenShell semantics, never bypass them. |
| Claude operator access | Desktop connector pattern (§8), gated on the ash4d.com NS flip. Zero GCP dependency. |
| Approval channel | **Slack** (decided 2026-07-05): workspace `vibecoding-wu42136.slack.com`. Rick has no Telegram and wants no new platforms. Note: Telegram is NemoClaw's default path, so the Slack channel adapter is a small extra integration item for the build. Goal: **Hermes and NemoClaw both live in this workspace as separate Slack apps/bot identities** (Hermes deployment exists in ns `hermes`, Socket Mode, connection not yet working — diagnosis runbook in session notes; `NEMOCLAW_AGENT=hermes` is a further config worth a writeup line). |

---

## 1. The core decision — what the agent does

**An autonomous Delivery & Runtime Operations agent (SRE-style) for the ash4d lab.**

It oversees the software-delivery process end to end and the runtime health of what that process ships:

- Watches the **GitOps delivery pipeline** (Fleet bundle sync/status) and **runtime health** (Prometheus/Alertmanager alerts, Longhorn volume state, cert-manager expiry, pod restarts, DCGM/GPU signals).
- Maintains **state and context over time** via Milvus memory plus a run/incident store — it knows baseline-normal and tracks a single incident across multiple steps.
- On a change, it **correlates signals, forms a hypothesis, and runs read-only diagnostics** through the existing least-privilege MCP tools.
- Chooses actions from an **explicit permission policy** (§7).
- **Escalates via an operator-approval flow** (OpenShell egress approval + chat channel) when confidence is low or approval is required.
- **Logs what it did and why** (audit trail), recovers/retries transient failures, and keeps all data on-prem (routed inference to local Ollama, OpenShell egress lock).

## 2. Behavior breadth — what makes it read as autonomous oversight

The domain stays cluster ops; the breadth comes from four distinct agent capability surfaces sharing one infrastructure:

1. **Reactive incident loop** — detect → diagnose → escalate → act (on approval) → audit. The MVP slice.
2. **Conversational ops interface** — ask it "why is bundle X stuck?" over the chat channel. Uses OpenClaw's multi-channel design for interactive Q&A, beyond approval traffic alone.
3. **Proactive scheduled work** (heartbeat) — weekly cluster health report, cert inventory, drift summary. Demonstrates "monitors over time" without waiting for a failure.
4. **Incident memory recall** — "similar to the June 12 Longhorn incident; that one was resolved by X." Makes Milvus a working component with visible value.

## 3. Requirement → capability mapping

| Client-style requirement | How this build demonstrates it | Lab component |
|---|---|---|
| Monitors a process over time | Continuous watch on Fleet + Alertmanager + volume/cert/GPU state; heartbeat reports | rancher-monitoring, Fleet |
| Notices when something changes | Signal correlation against tracked baseline | Prometheus/Alertmanager |
| Knows what actions it's allowed to take | Explicit autonomy ladder: autonomous-safe vs approval-required | OpenShell policy + MCP RBAC |
| Uses APIs/tools safely | Read-only diagnostics through least-privilege MCP servers | 09-mcp / mcpo |
| Maintains state and context | Incident/run store + Milvus memory + recall in diagnoses | Milvus, Longhorn PVs |
| Logs what it did and why | Structured audit trail per action + reasoning | agent audit log |
| Escalates when confidence low | Operator-approval flow with human accept/deny | OpenShell egress approval + chat |
| Recovers from failures | Idempotent action design + retry with backoff | agent loop |
| Prevents unsafe actions | Sandbox isolation + egress lock + action allowlist | OpenShell sandbox |
| Useful in real operations | Runs against live k3s infra | ash4d cluster |

This table doubles as the answer sheet for technical screenings.

## 4. Architecture spine

```
Chat channel (Slack)
        │  (operator approval + conversational ops Q&A)
        ▼
NemoClaw (CLI: onboarding, hardened blueprint, routed inference, lifecycle)
        │
        ▼
OpenShell sandbox  ──── egress/network policy + approval gate
        │                (k8s port: NetworkPolicy + securityContext/PSA,
        │                 credential proxying preserved — see §6)
        ▼
OpenClaw agent (plan → tool-use → reflect → self-correct)
   ├── routed inference → local Ollama / Qwen3 (V100)   [zero egress]
   ├── memory → Milvus + incident/run store
   ├── tools → mcpo / MCP servers (§5)
   └── audit log → structured what/why/outcome per step
```

- Inference stays local; the whole agent plane is on-prem with no data leaving.
- The OpenShell egress/approval flow doubles as security control and human-in-the-loop gate.
- Agent egress allowlist reduces to: mcpo, Ollama, chat channel.

## 5. Tool layer

- **Existing (use as-is):** mcpo fronting k8s read-only, systemd, host introspection MCP servers.
- **Verify:** whether OpenClaw consumes MCP natively by now or needs mcpo's OpenAPI translation. Either works; native skips a hop. (Research task.)
- **Extend (cheap wins):** typed convenience tools on the k8s MCP for Fleet/Longhorn CRDs — `get_bundle_status`, `get_volume_health` — returning digested state, friendlier to a local model's context budget than raw CRD YAML.
- **Author (Python/FastMCP):** a **Prometheus MCP** (PromQL queries + active alerts) — highest-value addition; metric correlation is the diagnosis backbone.
- **Defer:** Longhorn REST MCP (only if CRD state proves insufficient); Rancher/Steve API MCP (unnecessary for v1).
- **Auth requirement:** mcpo gets bearer-token auth (minimum) before any exposure beyond the cluster (§8).

## 6. Deployment — k3s-native, OpenShell semantics preserved

- OpenShell's isolation is documented around Docker on Linux. Whether it runs k8s-native (containerd/pods) is **the unverified item and step 1 of the build** — everything downstream depends on it.
- If it's Docker-assumptive, the port translates its guarantees to k8s primitives: egress control → NetworkPolicy, filesystem isolation → securityContext / readOnlyRootFilesystem / restricted PSA, credential proxying kept as-is. The writeup documents each guarantee and its k8s equivalent — "translated NVIDIA's hardened blueprint to k8s-native without weakening it" is a core portfolio claim.
- Bypassing or weakening the sandbox to make k8s work is off the table; that would undercut the reason for choosing NemoClaw.

## 7. Autonomy ladder (permission policy)

Staged, documented, shipped as a design artifact:

- **v1 — read + annotate.** Diagnose, correlate, annotate resources, open issues, write audit entries. Every mutation requires approval. (MVP ships here.)
- **v2 — approval-gated mutations.** Fleet re-sync, cert renewal, named-deployment restarts — proposed by the agent, executed on human approve via chat.
- **v3 — allowlisted autonomous heals.** The v2 set (or a subset) runs autonomously with auto-rollback and post-hoc notification. This is the "self-diagnosing, self-healing system" end state — full-stack analysis (OS via systemd tools, k8s, app frameworks) feeding safe remediation.

Escalation tuning (confidence threshold + dedupe) is a design task within v1 to keep the demo signal clean.

## 8. Access paths (management plane, separate from agent plane)

- **Rick, own devices:** Tailscale subnet routing already reaches LAN IPs. Covered today.
- **Claude (Cowork/desktop):** connector pattern — clone of `mcp-ollama.ash4d.com`: new hostname → home Traefik (192.168.7.159), cert via DNS-01, connector in desktop settings. Gated on the ash4d.com NS flip (Dyn → Cloudflare, in flight; watcher scheduled). Requires mcpo auth first.
- **Any browser, anywhere (tradeshow/borrowed machine):** GCP edge portal — IdP-gated (Okta-style) auth screen → landing page linking lab services, proxied into the lab over Tailscale. Planned in `_portfolio/ash4d-site/edge-access-plan.md`; a parallel track that becomes the public demo path for this agent. The agent build has no dependency on it.
- Writeup rule: agent plane (zero egress) and management plane (inbound operator access) are documented as separate planes so the zero-egress claim stays intact.

## 9. Milvus knowledge corpus

Priority order — index what the model can't already know:

1. **Lab-specific (highest value):** lab-fleet repo contents, ARCHITECTURE.md, live-state dumps, incident notes, the Longhorn 502 writeup when done. This grounds the "knows baseline-normal" claim.
2. **Runbooks authored for this agent:** per-signal diagnostic/remediation procedures; these double as the action-policy source.
3. **Version-pinned product docs:** k3s (where it diverges from upstream), Fleet (bundle state machine), Longhorn (volume/replica states, rebuild behavior), cert-manager (challenge/renewal flows), rancher-monitoring chart specifics.
4. **Host layer, selective:** openSUSE (zypper/transactional-update), systemd unit patterns for lab services.
5. **Curated failure signatures:** Longhorn/Fleet GitHub issues matching failure modes the lab can actually hit.

Excluded: generic Kubernetes concept docs and tutorials (corpus bloat degrades retrieval; Qwen3 knows them).

## 10. Lab substrate already in place (confirmed from live-state dump)

Fleet (GitOps) · rancher-monitoring (Prometheus + Alertmanager) · dcgm-exporter (GPU) · cert-manager · Longhorn · MCP servers/mcpo (09-mcp) · Ollama/Qwen3 on V100 · Milvus · Hermes (10-hermes, `NEMOCLAW_AGENT=hermes` alternate) · Node-RED (push-based event routing option)

## 11. MVP vertical slice (build order)

1. **De-risk:** stand up OpenShell/NemoClaw on k3s; confirm the container model k8s-native (§6). Route inference to Ollama.
2. **First loop end to end (cert expiry):** stage a short-lived test cert → agent detects → diagnoses read-only → escalates for approval → takes the one approved action → writes the audit entry.
3. **Fan out signals:** stuck Fleet bundle, Longhorn degraded, pod crash-loop, GPU errors — same loop + policy.
4. **Fan out behaviors:** conversational ops Q&A, heartbeat reports, memory recall (§2).
5. **Writeup + demo:** lab-fleet-style piece; GIF of detect→escalate→approve→act→audit; screening talking points from §3.

## 12. Portfolio deliverables

- Public repo (new, under darthzen) with agent, policy, deploy manifests
- Lab-fleet writeup / architecture note, including the OpenShell-to-k8s translation section
- Short demo GIF of the full cycle
- Autonomy-ladder design doc (§7)
- Later: live demo via the GCP edge portal

## 13. Open items

1. **OpenShell on k3s** — feasibility research + prototype (step 1 of build).
2. **OpenClaw native MCP support** — verify current state.
3. **Prebuilt Prometheus/Longhorn/Fleet MCP servers** — check what exists post-cutoff before authoring.
4. **Slack channel adapter** — confirm OpenClaw/NemoClaw Slack support maturity (Telegram is the default path); fold into the research pass.

## Next steps

- [ ] Research items 1–4 above (one research pass)
- [ ] Draft DEPLOY.md targeting k3s (spine §4, order §11) + repo scaffold
