# Security Policy

## FoundUps Agent Workspace — Security Boundaries

This repository is an **external compatible system** for FoundUps Core. It is not a trusted execution environment by default. The following security boundaries apply to all workstations running this software.

---

## Authentication and Identity

**`tenant_id` is attribution, not authentication.**

A `tenant_id` field in a WSP task packet identifies who the work is attributed to in the FoundUps ROC accounting system. It does NOT authenticate a workstation, grant permissions, or replace signed packet requirements.

Actual authentication requires all of the following (Phase 2+):

- `workspace_id` — unique identifier for the registered workstation
- Signed WSP task packet — HMAC-SHA256 signed with a shared secret issued by FoundUps Core
- Short-lived capability token — issued per job type, scoped to allowed actions
- Nonce / replay protection — timestamps and nonces prevent packet replay attacks
- Allowed action scope — only actions explicitly granted in the capability token are executable
- Allowed path scope — only paths permitted by the workspace binding contract are accessible

A workstation claiming `tenant_id="012"` without a valid signed packet and capability token will be rejected by FoundUps Core.

---

## Live Execution

**Live execution is disabled by default.**

All workstations run in dry-run mode (`dry_run_mode: true`) until explicitly authorized by FoundUps Core. Live delegation requires all of:

- `human_approval_received: true`
- `security_gate_passed: true`
- `compute_budget > 0`
- `live_mode_authorized: true`
- `dry_run_mode: false`

Any workstation attempting to self-authorize these flags will be rejected.

---

## Production Tokens

**No production tokens are issued or accepted by this system.**

This repository does not handle UPS tokens, BTC treasury keys, CABR verification payloads, pAVS rewards, or any financial instruments.

WSP 97 truth fields (`real_execution_performed`, `verification_complete`, `cabr_ready`, `payout_ready`) are always `false` until FoundUps Core sets them through verified execution gates.

---

## Destructive Actions

**Destructive actions are denied by default.**

Hard-blocked paths (never accessible regardless of capability scope):

- `.env`, `*.pem`, `*.key` — secrets and cryptographic keys
- `**/secrets/`, `**/credentials/` — credential stores
- `.git/config`, `.git/credentials` — git authentication
- `vendor/`, `.hermes/`, `node_modules/` — system directories

Path traversal escapes (`..`) are rejected. Symlinks are resolved before path validation.

---

## Reporting Vulnerabilities

To report a security vulnerability, open a GitHub Security Advisory in this repository or contact the FoundUps Core team through the FoundUps Discord.

---

## Upstream Attribution

Forked from [outsourc-e/hermes-workspace](https://github.com/outsourc-e/hermes-workspace) (MIT License).

**Source SHA at fork time**: `6485d2002f6a5c615fa000b2d8f0945d7dadc738` (2026-05-02)

---

*WSP compliance: WSP 15 (Secure Operations), WSP 97 (Truth Boundaries) — authored by 0102*
