# T-015 — Audit-log schema (what / why / outcome, per step)

**Date:** 2026-07-05. The audit trail is the evidence that the autonomy ladder
(PLAN §7) is real: every step the agent takes — read, annotate, escalate, or
(later) mutate — writes one append-only record. This is both a safety control
and the demo artifact ("here's exactly what it did and why").

## Design principles

- **Append-only.** Records are never updated in place; corrections are new
  records that reference the prior `step_id`. A single incident is a chain.
- **One record per step**, not per incident. Detect, each diagnostic read,
  the hypothesis, the escalation, the human decision, and the action are all
  separate records sharing an `incident_id`.
- **Self-contained.** Each record carries enough context (inputs, evidence
  refs, reasoning) to reconstruct the decision without external lookups.
- **Structured + greppable.** JSON Lines on disk; every record validates
  against the schema below.
- **Credential-aware.** Records name the identity used (`sa-nemoclaw-observe`
  etc.) but **never** contain secrets, tokens, or full kubeconfig material.

## Record schema (JSON Schema, draft 2020-12)

`deploy/audit/audit-record.schema.json` is the machine copy. Fields:

| Field | Type | Notes |
|---|---|---|
| `schema_version` | string | `"1"` |
| `step_id` | string (uuid) | unique per record |
| `incident_id` | string (uuid) | groups a detect→act chain |
| `parent_step_id` | string\|null | previous step in the chain |
| `ts` | string (RFC3339) | UTC |
| `phase` | enum | `detect`\|`diagnose`\|`hypothesize`\|`escalate`\|`decision`\|`act`\|`audit`\|`recover` |
| `signal` | string | e.g. `cert-expiry`, `fleet-stuck-bundle` |
| `identity` | enum | `sa-nemoclaw-observe`\|`sa-nemoclaw-deploy`\|`human`\|`system` |
| `autonomy_level` | enum | `v1-read-annotate`\|`v2-approval-mutation`\|`v3-autonomous-heal` |
| `what` | object | the action: `{action, target_ref, tool, params_digest}` |
| `why` | object | reasoning: `{rationale, evidence_refs[], model, confidence}` |
| `outcome` | object | result: `{status, detail, changed[], duration_ms}` |
| `approval` | object\|null | `{required, channel, requested_ts, decided_ts, decision, decider, slack_ts}` |
| `error` | object\|null | `{kind, message, retryable, attempt}` |

- `what.target_ref`: structured k8s ref `{apiGroup, kind, namespace, name, uid}`
  — never free text, so records join cleanly against cluster state.
- `what.params_digest`: sha256 of tool params (proves what was requested without
  storing payloads that might carry sensitive values).
- `why.confidence`: 0.0–1.0; drives the escalation threshold (PLAN §7 tuning).
- `outcome.status`: `ok`\|`noop`\|`blocked`\|`denied`\|`error`\|`awaiting-approval`.
- `outcome.changed`: list of target_refs actually mutated — **empty for every
  v1 record by construction** (read+annotate only). A non-empty `changed` on a
  `v1-read-annotate` record is a contract violation and should fail CI.

## Persistence

- **On-disk:** JSON Lines at `/audit/incidents/<incident_id>.jsonl` inside the
  agent pod, on a **Longhorn PVC** (`audit-store`, RWO) so history survives pod
  restarts and reschedules.
- **Retention:** rotate per-incident files; keep 90 days hot on the PVC. The
  Milvus incident-memory layer (T-042) indexes summaries for recall; the JSONL
  is the system of record.
- **Egress:** none. The audit store never leaves the cluster (consistent with
  the zero-data-egress claim). Slack gets a human-readable summary, not the raw
  records.

## PVC manifest

`deploy/audit/00-audit-pvc.yaml` — RWO Longhorn PVC in `nemoclaw`, created under
`sa-nemoclaw-deploy` scope. Applied as part of T-015/T-011.
