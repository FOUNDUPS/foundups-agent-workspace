# FoundUps Integration Boundary

**Status**: Bootstrap documentation — Phase 1
**WSP Compliance**: WSP 11 (Interface), WSP 15 (Priority), WSP 50 (Pre-Action), WSP 97 (Truth Boundaries)
**Date**: 2026-05-04
**Source SHA**: `6485d2002f6a5c615fa000b2d8f0945d7dadc738` (2026-05-02)

---

## 1. What This Repository Is

**FoundUps Agent Workspace** is an external compatible system for FoundUps Core.

It is:
- A swarm execution environment (tmux workers, Kanban task board, checkpoint protocol)
- Forked from `outsourc-e/hermes-workspace` (MIT License, verified SHA above)
- Designed to receive and execute FoundUp jobs from FoundUps Core through a constrained gateway
- Dry-run only by default — no live delegation until Phase F gates pass

It is NOT:
- A trusted execution environment by default
- A production token issuer or handler
- An authenticated peer without a signed task packet and capability token
- Part of the FoundUps Core governance layer (WSP framework, WRE, OpenClaw)

---

## 2. Architecture Boundary

```
+--------------------------------------------------+
| FoundUps Core (FOUNDUPS/Foundups-Agent)          |
|                                                  |
|  OpenClaw -> FoundUpJob -> WRE Router            |
|                     |                            |
|                     v                            |
|           HermesJobExecutor (dry-run seam)       |
|                     |                            |
+--------------------------------------------------+
                      |
         Gateway API / WSP Task Packet
         POST /api/v1/tasks
         WebSocket /ws/v1/events
                      |
+--------------------------------------------------+
| FoundUps Agent Workspace (this repo)             |
|                                                  |
|  Gateway Adapter -> Workspace Scheduler          |
|                          |                       |
|                          v                       |
|               Swarm Workers (tmux)               |
|                          |                       |
|                          v                       |
|              Checkpoint / Evidence Output        |
|                                                  |
+--------------------------------------------------+
```

**Key principle**: FoundUps Core owns governance (WSP, WRE, CABR). FoundUps Agent Workspace owns execution (swarm workers, tmux sessions, Kanban UI).

---

## 3. WSP Task Packets

WSP task packets are the protocol through which FoundUps Core dispatches work to this workspace.

### 3.1 Packet Schema

```json
{
  "packet_version": "1.0.0",
  "packet_type": "wsp_task",
  "identity": {
    "job_id": "string (UUID)",
    "foundup_id": "string | null",
    "tenant_id": "string",
    "intent_id": "string | null"
  },
  "task": {
    "requested_action": "build_foundup | extract_foundup | validate_foundup | queue_foundup_job",
    "goal": "string",
    "context": "string",
    "workspace_path": "string | null"
  },
  "policy": {
    "dry_run_mode": true,
    "security_gate_checked": false,
    "security_gate_passed": false,
    "human_approval_received": false,
    "live_mode_authorized": false
  },
  "wsp97_truth": {
    "real_execution_performed": false,
    "verification_complete": false,
    "cabr_ready": false,
    "payout_ready": false
  }
}
```

### 3.2 Packet Requirements

Every accepted packet MUST include:

- `workspace_id` — registered workspace identifier (Phase 2+)
- Valid HMAC-SHA256 signature — signed with capability token issued by FoundUps Core
- Nonce — prevents replay attacks
- `dry_run_mode: true` — until live delegation gate passes (Phase F)

Packets missing required fields or with invalid signatures are rejected without execution.

---

## 4. Workspace Identity

### 4.1 workspace_id

A `workspace_id` is a unique identifier for a registered workstation. It is NOT the same as `tenant_id`.

| Field | Purpose | Auth boundary? |
|-------|---------|----------------|
| `tenant_id` | ROC attribution — who gets credit | NO |
| `workspace_id` | Workstation identity | YES (Phase 2+) |

A workstation must register with FoundUps Core to receive a `workspace_id`.

### 4.2 Capability Tokens

Short-lived capability tokens are issued per job type by FoundUps Core. They define:

- Allowed `requested_action` values
- Allowed path scope
- Budget limits
- Expiry time

Tokens are NOT yet implemented (Phase 2). In Phase 1, all execution is dry-run only.

---

## 5. Signed Task Packets (Phase 2+)

All task packets submitted to the gateway must be signed with HMAC-SHA256:

```
signature = HMAC-SHA256(secret_key, canonical_packet_json)
canonical_packet_json = JSON.stringify(packet_without_signature, sort_keys=True)
```

The `secret_key` is issued by FoundUps Core on workspace registration. It is:
- Short-lived (rotated per session or per time window)
- Scoped to the workspace_id and allowed actions
- Never embedded in this repository

---

## 6. Checkpoint and Evidence — Observability Only

Checkpoint and evidence artifacts (`metadata.json`, `checkpoint.json`, `tool_trace.jsonl`) are **observability artifacts only**.

They prove a job was processed through the WRE pipeline. They do NOT:

- Prove real work was executed (`real_execution_performed` is always `false` in dry-run)
- Constitute CABR verification
- Trigger ROC rewards or payout
- Replace WSP 97 truth field verification

Evidence artifacts are written to: `.hermes_evidence/{job_id}/`

---

## 7. Dry-Run Default

All workstations operate in dry-run mode by default:

```
dry_run_mode: true       (default, always)
live_mode_authorized: false  (default, always)
real_execution_performed: false  (WSP 97, invariant until Phase F)
```

Live execution (Phase F) requires ALL gates to pass simultaneously:
1. `human_approval_received: true`
2. `security_gate_passed: true`
3. `compute_budget > 0`
4. `live_mode_authorized: true`
5. `dry_run_mode: false` (explicitly set by FoundUps Core)

No workstation can self-authorize live mode.

---

## 8. Fork Adaptation Phases

| Phase | Scope | Status |
|-------|-------|--------|
| **Phase A** | Fork bootstrap, branding, license, source SHA pin | ✅ COMPLETE (this commit) |
| **Phase B** | Add foundups_gateway_adapter, dry-run only | NOT STARTED |
| **Phase C** | WSP task packet import/export, schema validation | NOT STARTED |
| **Phase D** | Checkpoint/evidence compatibility | NOT STARTED |
| **Phase E** | Dry-run swarm execution demo end-to-end | NOT STARTED |
| **Phase F** | Live delegation gate — requires all auth gates | NOT STARTED |

---

## 9. WSP 97 Truth Boundaries

This documentation does NOT enable:

- Live FoundUps Core execution
- CABR verification claims
- Reward or payout claims
- Token minting or distribution
- Real `delegate_task` calls

All WSP 97 truth fields remain `false` until FoundUps Core explicitly sets them through verified execution gates.

---

## 10. Related Documents (FoundUps Core)

- `docs/architecture/FOUNDUPS_AGENT_WORKSPACE_FORK_PLAN.md` — Fork strategy and boundary
- `docs/architecture/WRE_GATEWAY_ADAPTER_DESIGN.md` — Gateway adapter design
- `docs/audits/hermes_swarm/HERMES_WORKSPACE_BINDING_CONTRACT.md` — Workspace binding contract
- `docs/audits/hermes_swarm/HERMES_WORKSPACE_EXTERNAL_REPO_AUDIT.md` — External repo audit

---

*Boundary documentation authored by 0102 — WSP compliance: WSP 11, WSP 15, WSP 50, WSP 97*
