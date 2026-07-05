# OpenClaw → in-cluster Ollama (T-012)

The agent's inference runs on-prem against the cluster Ollama, **not** through
OpenShell's inference proxy (that build has no local-endpoint provider type —
see docs/integration-notes.md and issue #9). OpenClaw talks to Ollama directly;
the sandbox egress-lock already permits `ollama.ai:11434`, so inference stays
zero-egress.

`openclaw-ollama-provider.json` is merged into the sandbox's
`~/.openclaw/openclaw.json`. Key points:

- `models.providers.ollama.baseUrl` → the in-cluster endpoint (bundled ollama
  provider schema accepts only `baseUrl` + `models[]`; the configured origin is
  auto-trusted for the private-network guard).
- `env.OLLAMA_API_KEY` is a required non-secret **marker** (local Ollama needs
  no real auth, but OpenClaw treats the provider as unauthed without it).
- Model: `qwen3-coder:30b` (tool-calling, MoE, 256K ctx).

## Verified (2026-07-05)

`openclaw models status --probe --probe-provider ollama` →
`ollama/qwen3-coder:30b | env (api_key) | ok` — a real inference request to the
model succeeded. Inference path live.

## Remaining
- Full `openclaw agent` turn (needs the OpenClaw gateway running as the sandbox's
  main process) + tool-call-reliability check (raw-JSON-leak) under load.
- Optional: speculative-decoding acceleration (issue #15 / T-060).
