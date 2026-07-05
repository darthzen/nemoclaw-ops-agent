# T-005 — OpenClaw MCP path, Slack adapter maturity, prebuilt MCP servers

**Date:** 2026-07-05. Sources at bottom. Three questions from PLAN.md §13.

## 1. Does OpenClaw consume MCP natively? — YES

- OpenClaw has **native MCP support** (core, via `@modelcontextprotocol/sdk`,
  ~v1.25.3 as of Feb 2026). It speaks both **stdio** and **HTTP/SSE** transports,
  so it works with the whole published MCP ecosystem.
- Managed with `openclaw mcp add|set|configure|tools|probe|doctor|list ...`;
  servers live under `mcp.servers` in `openclaw.json`.
- **Consequence for our design:** the **mcpo OpenAPI-translation hop is
  optional.** The agent can call our in-cluster MCP servers (`k8s-mcp-server`,
  `github-mcp-server`, a Prometheus MCP) **directly over HTTP/SSE**. Keep mcpo
  only where a tool is OpenAPI-only, or for a single bearer-auth choke point.
  This simplifies PLAN §5.
- **Sandbox caveat:** in NemoClaw, OpenClaw runs inside the OpenShell sandbox
  under a default-deny egress policy. Every MCP endpoint the agent reaches must
  be **allowlisted in the OpenShell network policy** (separate from, and in
  addition to, our k8s NetworkPolicy). Two policy planes, both must permit
  mcpo/MCP/Ollama. Note this in the T-013/T-014 tickets.

## 2. Slack adapter maturity — natively supported, with a known boot bug

- Slack is a **first-class OpenClaw/NemoClaw channel** over **Socket Mode**
  (outbound-only; ideal for on-prem hosts that can reach `*.slack.com` but take
  no inbound HTTPS — exactly ash4d). **No custom adapter/shim is needed** — this
  retires the PLAN.md "Slack is a small extra integration item" worry.
- Requires **two tokens**: `SLACK_BOT_TOKEN` (`xoxb-…`) and `SLACK_APP_TOKEN`
  (`xapp-…`, app-level Socket Mode). NemoClaw validates both before saving.
- Enable at `nemoclaw onboard` or later via `nemoclaw <sandbox> channels`.
- **Known risk — NVIDIA/NemoClaw issue #2085:** Slack Socket Mode can crash on
  boot with `invalid_auth` because the `appToken` is missing from the baked
  `openclaw.json` and placeholder resolution is broken (seen on v0.0.20). This
  mirrors the exact symptom Hermes hit ("runs but Slack never fully connects").
  **Mitigation:** pin a NemoClaw version past the #2085 fix (check release
  notes at build time), and verify `xapp` placeholder resolution before wiring
  the approval flow. This is a real T-014 acceptance gate, not a footnote.
- **Two bots, one workspace:** Slack may deliver a payload to any Socket Mode
  connection of a shared app, so separate gateways sharing one app need
  identical routing/authz. NVIDIA's guidance: **use a separate Slack app per
  gateway.** This confirms the plan's "Hermes and NemoClaw as separate Slack app
  identities" — keep them as two apps, not one shared app. (T-006.)

## 3. Prebuilt Prometheus / Fleet / Longhorn MCP servers — mixed

- **Prometheus: adopt, don't author.** Mature options exist:
  - `giantswarm/mcp-prometheus` — **18 read-only tools** wrapping the Prom HTTP
    API (instant + range PromQL, metric/label/series discovery, targets, TSDB
    stats, **alerting rules**, exemplars), **ships a Helm chart**. Best fit for
    ash4d's rancher-monitoring stack.
  - `pab1it0/prometheus-mcp-server` and **AWS Labs prometheus-mcp-server**
    (FastMCP-based) are alternates.
  - **Alertmanager:** `ntk148v/alertmanager-mcp-server` (status, alerts,
    silences, receivers, groups).
  - **Revises T-030:** the plan's "author a Prometheus MCP from scratch" becomes
    **"deploy `giantswarm/mcp-prometheus` read-only against rancher-monitoring,
    scoped + allowlisted."** Integrating a real component is honest portfolio
    work and faster. Author only if a gap appears.
- **Fleet MCP: none found → author** typed CRD tools (`get_bundle_status`) as
  planned (T-031). Extra wrinkle from T-002 recon: **Fleet CRDs aren't even
  registered on ash4d** (only a `fleet-agent` runs; control-plane CRDs live on
  the Rancher management cluster). The Fleet signal (T-032) must observe wherever
  the CRDs live, or lean on `gitjob`/pod state locally.
- **Longhorn MCP: none found → author** `get_volume_health` typed tool against
  `volumes.longhorn.io` (T-031), which our `sa-nemoclaw-observe` can already
  read. Longhorn also has a REST API if CRD state proves insufficient (deferred).

## Net effect on the plan

- MCP: drop the mandatory mcpo hop; call MCP servers directly over HTTP/SSE;
  allowlist each in **both** policy planes.
- Slack: native, no shim; gate T-014 on the #2085 fix; two separate Slack apps.
- Prometheus MCP: adopt `giantswarm/mcp-prometheus` (Helm, read-only) instead of
  authoring from scratch — reallocate T-030 effort to Fleet/Longhorn typed tools.

## Sources

- OpenClaw MCP (native, stdio + HTTP/SSE, CLI): https://docs.openclaw.ai/cli/mcp
- NemoClaw Messaging Channels (Slack Socket Mode, two tokens): https://docs.nvidia.com/nemoclaw/latest/manage-sandboxes/messaging-channels
- NemoClaw issue #2085 (Slack Socket Mode invalid_auth / appToken placeholder): https://github.com/NVIDIA/NemoClaw/issues/2085
- giantswarm/mcp-prometheus: https://github.com/giantswarm/mcp-prometheus
- ntk148v/alertmanager-mcp-server: https://github.com/ntk148v/alertmanager-mcp-server
- AWS Labs Prometheus MCP: https://awslabs.github.io/mcp/servers/prometheus-mcp-server
