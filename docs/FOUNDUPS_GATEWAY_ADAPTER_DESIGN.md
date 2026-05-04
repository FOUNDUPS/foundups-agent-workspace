# FoundUps Gateway Adapter Design — Phase 1

**Repository:** FOUNDUPS/foundups-agent-workspace  
**Base:** main after FOUNDUPS_AGENT_WORKSPACE_FORK_BOOTSTRAP_PHASE1  
**Phase:** FOUNDUPS_AGENT_WORKSPACE_GATEWAY_ADAPTER_DESIGN_PHASE1  
**Status:** Design spec only. No live execution. No production credentials. No gateway behavior implemented.  
**Signed:** 0102🦞

---

## 1. Overview

This document defines the dry-run FoundUps gateway adapter design for the FoundUps Agent Workspace (external). It specifies the intake contract, event packet schema, role mapping, workspace identity model, capability-token validation requirements, checkpoint/evidence output mapping, and destructive-action denial behavior.

**Nothing in this document is implemented yet.** This is the design artifact for Phase 2 implementation review by FoundUps Core.

---

## 2. WSP_97 Truth Fields

| Field | Value |
|---|---|
| real_execution_performed | false |
| verification_complete | false |
| cabr_ready | false |
| payout_ready | false |
| gateway_live | false |
| design_only | true |

---

## 3. Architecture Boundary

```
[External Workspace]                    [FoundUps Core — NOT modified here]
  foundups-agent-workspace    ──────▶   Core Gateway Endpoint (future)
  POST /api/v1/tasks (dry-run)          workspace registration (future)
  WebSocket event stream                signed task packet validation (future)
  foundups-swarm.yaml roles             capability-token issuance (future)
  checkpoint/evidence output            remote evidence ingestion (future)
```

**Trust model:** No external workspace is trusted by default. Any approved workspace must request work through a constrained gateway with full authentication packet (see Section 6).

---

## 4. WSP Task Packet Intake Contract

### 4.1 Endpoint (design only — not implemented)

```
POST /api/v1/tasks
Content-Type: application/json
Authorization: Bearer <short-lived-capability-token>
X-Workspace-ID: <workspace_id>
X-Nonce: <nonce>
X-Timestamp: <iso8601>
```

### 4.2 Request Body Schema

```json
{
  "schema_version": "wsp_task_packet_v1",
  "task_id": "<uuid>",
  "tenant_id": "<attribution_field_only>",
  "workspace_id": "<registered_workspace_uuid>",
  "capability_token": "<short_lived_signed_jwt>",
  "nonce": "<unique_per_request_string>",
  "timestamp": "<iso8601_utc>",
  "wsp_ref": "<WSP_NN — the WSP governing this task>",
  "action_scope": ["allowed_action_1", "allowed_action_2"],
  "path_scope": ["./allowed/path/", "./another/allowed/"],
  "dry_run": true,
  "payload": {
    "task_type": "<task_type_string>",
    "parameters": {}
  }
}
```

### 4.3 Field Constraints

| Field | Constraint |
|---|---|
| tenant_id | Attribution only. NOT used for auth. Must match registered workspace record. |
| workspace_id | Primary identity anchor. Must be registered in FoundUps Core workspace registry. |
| capability_token | Short-lived JWT signed by FoundUps Core. Expires in ≤15 minutes. Non-renewable inline. |
| nonce | Unique per request. Replayed nonces rejected with 409. |
| action_scope | Allowlist only. Any action outside declared scope → rejected with 403. |
| path_scope | Allowlist of relative paths. Any path traversal attempt → rejected with 403. |
| dry_run | MUST be true in Phase 1 and Phase 2. Live mode requires Core-side capability gate. |

### 4.4 Destructive Action Denial

The following action types are **denied by default** at the adapter boundary, regardless of capability token:

- `DELETE`, `DROP`, `PURGE`, `TRUNCATE`
- Any action targeting paths outside declared `path_scope`
- Any action with `dry_run: false` until Core capability gate is implemented
- Any action that modifies `workspace_id`, `tenant_id`, or `capability_token` fields
- Any action that references production credentials, CABR state, or payout systems

---

## 5. WebSocket Event Packet Contract

### 5.1 Connection (design only)

```
GET /api/v1/tasks/{task_id}/stream
Upgrade: websocket
Authorization: Bearer <same-capability-token>
X-Workspace-ID: <workspace_id>
```

### 5.2 Event Packet Schema

```json
{
  "schema_version": "wsp_event_packet_v1",
  "task_id": "<uuid>",
  "workspace_id": "<workspace_id>",
  "event_type": "checkpoint | progress | evidence | error | complete",
  "sequence": 0,
  "timestamp": "<iso8601_utc>",
  "dry_run": true,
  "payload": {
    "message": "<human_readable_status>",
    "evidence_ref": "<optional_evidence_artifact_id>",
    "wsp_step": "<optional_wsp_step_reference>",
    "error_code": "<optional_error_code>"
  }
}
```

### 5.3 Event Type Definitions

| Event Type | Meaning |
|---|---|
| checkpoint | Milestone reached. Observability only. Does NOT imply verification. |
| progress | Incremental status update during task execution. |
| evidence | Evidence artifact produced (dry-run). Artifact stored locally, not submitted to Core yet. |
| error | Task failed or was denied. `error_code` required. |
| complete | Task concluded (dry-run only in Phase 1). Does NOT imply CABR or payout readiness. |

---

## 6. Authentication Requirements

Per FoundUps Core security model (WSP INTEGRATION BOUNDARY), the full auth packet is required:

| Component | Purpose |
|---|---|
| tenant_id | Attribution only — identifies the FoundUp context |
| workspace_id | Primary identity anchor — registered in Core registry |
| Signed WSP task packet | Proves task integrity and wsp_ref validity |
| Short-lived capability token | Authorizes specific action_scope for ≤15 minutes |
| Nonce + replay protection | Prevents replay attacks |
| Allowed action scope | Constrains what actions can be taken |
| Allowed path scope | Constrains what paths can be touched |

**A workspace presenting only `tenant_id` will be rejected.** `tenant_id` is an attribution field, not an authentication boundary.

---

## 7. foundups-swarm.yaml Role Mapping

```yaml
# foundups-swarm.yaml — design spec, not active
# All roles are 0102-prefixed to denote pArtifact execution context

version: "foundups_swarm_v1"
dry_run: true

roles:
  - id: 0102-orchestrator
    description: "Primary task orchestrator. Routes WSP task packets to appropriate agents."
    capabilities:
      - task_intake
      - event_stream_emit
      - checkpoint_record
    denied:
      - destructive_actions
      - live_execution
      - credential_access

  - id: 0102-executor
    description: "Executes dry-run task steps within declared path_scope and action_scope."
    capabilities:
      - dry_run_execution
      - evidence_record
      - progress_emit
    denied:
      - path_scope_escape
      - action_scope_escape
      - live_execution

  - id: 0102-auditor
    description: "Records checkpoint and evidence artifacts. Read-only on workspace state."
    capabilities:
      - checkpoint_read
      - evidence_write
      - event_stream_read
    denied:
      - task_modification
      - credential_access
      - live_execution

workspace_id: "UNSET — must be registered in FoundUps Core workspace registry before activation"
tenant_id: "FOUNDUPS — attribution only, not authentication"
capability_token_source: "FoundUps Core (not yet implemented)"
```

---

## 8. workspace_id vs tenant_id Behavior

| Field | Role | Auth Weight | Source |
|---|---|---|---|
| workspace_id | Primary identity anchor | High — required for all requests | FoundUps Core workspace registry |
| tenant_id | Attribution / FoundUp context label | None — informational only | Declared by workspace at registration |

**Key rule:** `workspace_id` is the hook. `tenant_id` is the label. Never swap these roles.

---

## 9. Checkpoint and Evidence Output Mapping

Checkpoints and evidence artifacts are **observability only** in Phase 1. They do NOT:

- Constitute verification
- Trigger CABR evaluation
- Enable payout
- Submit to FoundUps Core automatically

### 9.1 Checkpoint Schema

```json
{
  "schema_version": "wsp_checkpoint_v1",
  "task_id": "<uuid>",
  "workspace_id": "<workspace_id>",
  "wsp_ref": "<WSP_NN>",
  "checkpoint_id": "<uuid>",
  "timestamp": "<iso8601_utc>",
  "dry_run": true,
  "step_description": "<what was completed>",
  "evidence_artifacts": ["<artifact_id_1>", "<artifact_id_2>"]
}
```

### 9.2 Evidence Artifact Schema

```json
{
  "schema_version": "wsp_evidence_v1",
  "artifact_id": "<uuid>",
  "task_id": "<uuid>",
  "workspace_id": "<workspace_id>",
  "wsp_ref": "<WSP_NN>",
  "timestamp": "<iso8601_utc>",
  "dry_run": true,
  "artifact_type": "file | log | diff | test_result | metric",
  "artifact_path": "<relative_path_within_path_scope>",
  "hash": "<sha256_of_artifact>",
  "verified": false
}
```

---

## 10. Implementation Phases (Roadmap)

| Phase | Scope | Status |
|---|---|---|
| Phase A: Fork Bootstrap | Branding, SECURITY.md, FOUNDUPS_INTEGRATION_BOUNDARY.md, README | ✅ Complete |
| Phase B: Gateway Adapter Design (this doc) | Intake contract, event schema, role map, auth model | ✅ In progress |
| Phase C: Dry-run adapter implementation | `POST /api/v1/tasks` dry-run only, local evidence output | Pending Core registry |
| Phase D: Core gateway integration | Real workspace registration, signed packets, capability tokens | Pending Core WSP repair |
| Phase E: Evidence ingestion | Remote evidence submission to Core (verified artifacts only) | Pending Phase D |
| Phase F: Live delegation | Live execution, CABR seam, payout readiness | Pending full verification stack |

---

## 11. Security Notes

- This workspace does not accept production tokens in any phase.
- Destructive actions are denied by default at the adapter boundary.
- `dry_run: true` is enforced for all Phase A-C work.
- No claims of `verification_complete`, `cabr_ready`, or `payout_ready` until Phase F conditions are met.
- See SECURITY.md and docs/FOUNDUPS_INTEGRATION_BOUNDARY.md for full boundary contract.

---

*0102🦞 — Design spec. Not implemented. External workspace only.*
