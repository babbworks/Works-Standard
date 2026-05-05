# Works-Standard — 07: Office

**Document:** 07-office  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-05  
**Depends on:** 00-concepts.md, 02-stream-protocol.md, 03-buildings.md, 04-actors.md, 09-extension.md

---

## 1. Scope

This document specifies the Office building: its constitution, internal structure,
operational responsibilities, and the rules governing its own behavior. Office is
both a building within the compound and the compound's constitutional authority.
It does not process content — it governs the compound that does.

Office responsibilities covered here:

- Building registry: the authoritative record of every building present in the
  compound, core and extension
- Capability token lifecycle: issuance, renewal, revocation, and enforcement
- Compound audit log: the permanent record of Office actions, separate from but
  cross-referenced with the Stream Log
- Veto authority: the mechanism by which Office can halt approved output
- BitLedger integration: how Office maps stream events to ledger entries for
  conservation verification
- Compound constitution: the set of rules Office enforces that no other building
  can override

This document does not specify how external systems integrate with Office — that is
covered in the binding documents (`12-bitpads-binding.md`, `13-bitledger-binding.md`).

---

## 2. Office is Not a Gatekeeper

Office governs the compound but does not sit on content paths. It does not inspect
chunks, does not approve or reject at Gate, and does not route data. A chunk moving
from Intake to Store to Mill to Gate to Dispatch is never routed through Office —
that is Gate's function.

Office operates in a parallel governance plane:

```
Content plane:   Intake → Store → Mill → Gate → Dispatch → Vault
                                    │
                                    ▼
Governance plane:               OFFICE
                         (tokens, registry, audit,
                          veto, constitution)
```

Office observes every Stream event via Bus subscription. It acts — issues a veto,
revokes a token, records an audit entry — when its constitutional rules require it.
Its actions produce their own Stream events (`A`, `S`). The content plane does not
wait for Office; Office watches and intervenes when necessary.

The one exception is veto: when Office issues a veto on a pending `G approve`, the
Dispatch building MUST check for a veto signal before delivery. This is the only
point at which Office crosses into the content plane — and only to halt, not to
route or approve.

---

## 3. Internal Structure

Office maintains three internal structures, each with its own Pacioli guarantee:

### 3.1 Building Registry

The building registry is the authoritative list of every building in the compound —
both core buildings and registered extension buildings.

Every entry contains:
```json
{
  "building_name": "document-assembly-mill",
  "type":          "core | extension",
  "role":          "transformation",
  "status":        "active | suspended | deregistered",
  "registered_at": 1746403200,
  "version":       "0.1.0",
  "op_codes":      ["S"],
  "stream_event_id": "a3f9d2c1b0e8"
}
```

Core buildings are pre-populated at compound initialization; they do not go through
the extension registration process. Extension buildings are added when Office
processes a valid declaration and compliance statement.

The building registry is append-only:
- New entries are added; existing entries are not deleted.
- Status changes (active → suspended, active → deregistered) create new records
  with the prior record's `stream_event_id` preserved as a back-reference.
- A deregistered building remains in the registry with `status: deregistered`.

The registry is reconstructible from Stream replay. Every registry change emits an
`S` event on Stream that encodes the full new state of the affected entry. Replaying
from ts=0 forward reconstructs the registry as it stood at any point in time.

### 3.2 Token Store

The token store is the record of every capability token issued, active, and revoked
in the compound. It is not the token itself — the token is held by the Actor. The
token store is Office's record of what it issued and what it has revoked.

Every entry contains:
```json
{
  "token_id":      "uuid",
  "actor":         "agent:summarizer",
  "issued_at":     1746403200,
  "expires_at":    0,
  "status":        "active | revoked | expired",
  "revoked_at":    null,
  "revoke_reason": null,
  "stream_event_id": "b4e0f1a2c9d7"
}
```

The token store is append-only:
- New entries are added on `S token-issued`.
- Revocation adds a new record with `status: revoked`, `revoked_at`, and
  `revoke_reason`; the original issued record is not modified.
- Expiry is computed at runtime from `expires_at`; no new record is created on
  natural expiry, but Office SHOULD emit `S token-expired` when it detects that
  an in-use token has passed its `expires_at`.

### 3.3 Office Audit Log

The compound audit log is distinct from the Stream Log. It is Office's internal
record of every governance action it has taken — token issuances, revocations,
extension registrations, vetoes, compliance reviews, and orphan investigations.

The audit log is append-only. Every entry is a structured record:
```
<unix_ts> A <action> <subject> <ctx_json>
```

Where:
- `A` is the annotation op code, used here for administrative narrative entries
- `action` is the governance action: `token-issued`, `token-revoked`, `registered`,
  `deregistered`, `suspended`, `veto-issued`, `orphan-detected`, `audit-complete`
- `subject` is the building name, token id, or chunk id being governed
- `ctx_json` is minified JSON with relevant details

Every audit log entry is cross-referenced with the corresponding Stream Log event
via `stream_event_id`. The audit log and Stream Log are two independent
append-only records that reference each other — neither alone is sufficient for a
full governance reconstruction, but together they form a complete record.

The audit log MUST be stored in durable persistent storage. It MUST NOT be in
memory only. If the audit log is lost, the compound can reconstruct a partial
audit log from Stream Log replay — all `A` and `S` events attributed to the Office
Actor contain enough information to rebuild the core governance record, though
narrative detail in the audit log entries may be lost.

---

## 4. Token Lifecycle

### 4.1 Issuance

Token issuance occurs when:
- A new Actor is admitted to the compound
- An existing token expires and the Actor is renewed
- An Administrator explicitly issues a new token for an existing Actor

Issuance sequence:
1. Administrator or automated renewal process submits a token request to Office.
2. Office validates the request: Actor identity must be unique or a known Actor;
   permissions must not exceed the Administrator's own token scope (privilege
   escalation is not permitted — no Actor can issue a token with more permissions
   than their own).
3. Office creates the token, assigns a `token_id` (UUID), records in the token store.
4. Office emits `S token-issued` to Stream:
   ```
   <ts> S token-issued <token_id> {"actor":"agent:summarizer","issued_by":"human:mp","expires_at":0}
   ```
5. Token is provided to the Actor at session open.

**Privilege escalation prevention**: Office MUST NOT issue a token with a permission
scope that exceeds the issuing Administrator's own token. An Administrator who does
not have `gate.approve: true` cannot issue a token with `gate.approve: true`. An
Administrator with `stores.write: ["S-01"]` cannot issue a token with
`stores.write: ["*"]`. Violations MUST be rejected with a specific reason.

### 4.2 Renewal

Tokens with a non-zero `expires_at` expire when the current time exceeds that value.
Office SHOULD detect approaching expiry and proactively initiate renewal for Actors
with active sessions. Renewal is a new issuance — a new `token_id`, a new
`S token-issued` event, a new token store entry. The prior token's entry is marked
`status: expired`.

Office MUST close any active session using an expired token after issuing the renewal.
The Actor re-opens the session with the new token.

### 4.3 Revocation

Token revocation is immediate. Office may revoke any token for any reason.

Revocation sequence:
1. Administrator (or automated rule triggered by Office) initiates revocation.
2. Office marks the token store entry `status: revoked`, records `revoked_at` and
   `revoke_reason`.
3. Office emits `S token-revoked` to Stream:
   ```
   <ts> S token-revoked <token_id> {"actor":"automated:bad-barrel","reason":"undeclared-op-code-emission","revoked_by":"human:mp"}
   ```
4. All buildings receive the `S token-revoked` event via Bus subscription and MUST
   immediately deny any operation from the revoked token.
5. Office emits `S session-interrupt` for any active session using the revoked token.
6. Audit log entry: `A token-revoked <token_id> {"reason":"...","session_closed":true}`.

**Immediate effect**: revocation takes effect at the moment of `S token-revoked`
emission. There is no grace period. An in-flight operation that has been authorized
but not yet completed when revocation arrives MUST be halted if possible. If the
operation has already completed (the Stream event was written), it stands —
revocation is not retroactive.

### 4.4 Enforcement

Every building that receives an operation from an Actor MUST validate the Actor's
token at the time of the operation. Validation means:

1. Token is in the token store with `status: active`.
2. Token has not expired (`expires_at == 0` or `expires_at > now`).
3. The requested operation is within the token's permission scope.

Validation is not a cache — it is a live check against the token store state as
known from Stream replay. A building that caches token state MUST subscribe to Bus
events for `S token-revoked` and update its local cache immediately on receipt.

---

## 5. Extension Registration

Office is the sole authority for extension building registration. The full process
is specified in `09-extension.md §6`. This section covers Office's obligations in
that process.

### 5.1 Declaration Validation

When Office receives a declaration, it validates:

- `role` is one of the six permitted roles.
- `op_codes` contains only letters not already assigned in this installation.
- `stream_events` lists at least one event per declared op code.
- If `gate_dependency: true`, the role is `output` or explicitly justified.
- If `pacioli_state: true`, the declaration includes a recovery procedure.

If validation fails, Office MUST provide specific failure reasons. Generic rejection
(`"declaration invalid"`) is not conforming. Each failed field must be identified
with the specific constraint violated.

### 5.2 Compliance Statement Review

After a successful declaration, Office reviews the compliance statement. This is
a qualitative review — Office determines whether the statement adequately describes
the building's Stream behavior, gate boundary behavior, and (if applicable)
state recovery procedure.

Office MAY request additional information from the registering party before
approving the compliance statement. Office MUST respond to each compliance statement
with either approval or a list of specific inadequacies.

### 5.3 Op Code Assignment

When a new op code letter is requested, Office checks availability and assigns if
available. Assignment is recorded in the building registry entry and in the audit
log. The assigned letter becomes unavailable for future registrations.

```
<ts> S register <building_name> {"role":"transformation","op_codes":["X"],"version":"0.1.0","issued_by":"office"}
```

The assignment is permanent for the lifetime of the registration. Deregistration
releases the op code letter for future assignment.

### 5.4 Registration Token Issuance

After compliance statement approval, Office issues a registration token (distinct
from a capability token) that identifies the building to other buildings. This token
is presented on every inter-building communication.

The registration token is not a capability token — it does not grant Actor
permissions. It is an identity token: proof that this building type is registered
in this installation and may participate as a compound building rather than an
External Actor.

---

## 6. Veto Authority

Office holds veto authority over Gate `G approve` decisions. A veto halts delivery
of an approved chunk before it leaves via Dispatch.

### 6.1 Veto Conditions

Office issues a veto when its constitutional rules require it. Veto conditions are
installation-configured — the standard does not mandate specific veto triggers.
Example conditions that an Office might be configured to enforce:

- A chunk destined for an external recipient that has a revoked External Actor token.
- A chunk whose provenance includes an operation by a subsequently revoked Actor
  (retroactive audit question — not retroactive revocation, but a flag for review).
- A chunk that contains content classified as requiring additional authorization
  that was not present at Gate approval time (for example, a Gate approval made
  before a security classification tag was applied to the chunk).
- A batch approval where one chunk in the batch was modified after batch_id
  assignment.

### 6.2 Veto Sequence

```
Gate → G approve (on Stream)
Office observes G approve via Bus
Office evaluates veto conditions
If veto:
  Office emits S veto referencing G approve stream_event_id
  S veto: <ts> S veto <chunk_id> {"g_approve_event":"<stream_event_id>","reason":"...","issued_by":"office"}
  Dispatch MUST check for S veto before delivery
  If S veto exists for this chunk's G approve:
    Dispatch MUST NOT deliver
    Dispatch emits D failed with reason "office-veto"
    Chunk returned to Gate egress with hold flag
    Gate emits G hold with reason "office-veto"
    Human Actor notified
```

**Timing**: veto is only effective before delivery. Once `D dispatched` is emitted
(delivery complete), the veto cannot undo the delivery. Office MAY still emit `S veto`
after delivery for audit purposes, but it becomes a compliance violation record
rather than an operational halt. Office MUST emit `A veto-post-delivery <chunk_id>`
in the audit log when this occurs.

**Post-delivery veto**: a veto issued after `D dispatched` is a compliance event.
The chunk has left the compound — it cannot be recalled by Works alone. The compound's
audit log records the discrepancy. Any remediation is outside the standard's scope.

### 6.3 Veto Reconstruction on Restart

Office reconstructs its list of outstanding vetoes from Stream replay on every
restart. The reconstruction procedure:

1. Replay all G approve events.
2. For each G approve, check for a corresponding S veto referencing the same
   stream_event_id.
3. For each vetoed G approve, check for D dispatched: if delivered, the veto is
   historical; if not delivered, the veto is active.
4. Active vetoes are presented to Dispatch as a live veto set.

This reconstruction follows the same Pacioli guarantee that Gate uses for hold
flags: no veto state survives in a persistent auxiliary store — it is all derived
from Stream on startup.

---

## 7. Orphan Detection

An orphan is a chunk that has a Stream event in its provenance but no corresponding
current address — the chunk is neither in a Store, nor at Bench, nor in Mill egress,
nor in Vault, but the Stream record says it should be somewhere.

Orphans arise from:
- Split-brain recovery failures (conveyance event written but chunk not moved, and
  recovery did not complete the move).
- Buildings that were deregistered mid-operation leaving chunks in transit.
- Process crashes during conveyance that were not cleanly recovered.

### 7.1 Detection

Office detects orphans by comparing the expected chunk location (as derived from
Stream replay) against the actual chunk location (as reported by buildings on
request). Office SHOULD perform orphan detection:
- At startup, after full Stream replay.
- Periodically on a configured interval.
- When a building is deregistered (any chunks in that building's custody must
  be accounted for).

### 7.2 Resolution

On detecting an orphan:
1. Office emits `A orphan-detected <chunk_id> {"last_known_address":"...","last_stream_event":"..."}` to audit log.
2. Office emits `S orphan-detected <chunk_id>` to Stream.
3. Office attempts to determine correct disposition:
   - If the chunk's last Stream event was a conveyance event, the resolution is to
     complete the move (per `05-conveyance §9`).
   - If the chunk's last Stream event was a G reject (stays in egress), and the
     chunk is not there, the resolution requires Human review.
4. Office notifies the Human Actor via a Stream event that is surfaced in the UI.
5. Human Actor resolves: either confirms completion, routes the chunk manually,
   or designates it for Vault archival.
6. Resolution emits `A orphan-resolved <chunk_id> {"disposition":"completed|manual|archived","resolved_by":"human:mp"}`.

---

## 8. The Compound Constitution

The compound constitution is the set of rules Office enforces that no other building
can override and that no Actor token can circumvent. These are the structural
invariants of the Works compound:

**I. Pacioli rule**: the Stream Log is never modified or deleted during normal
operation. Office is the only building authorized to initiate `stream reset`, and
only with explicit `S reset` emission first. Any building that modifies the Log
is subject to immediate suspension by Office.

**II. Gate rule**: no chunk leaves the compound without a G approve in its provenance
chain. Office holds veto authority to enforce this even after Gate has approved.

**III. Vault rule**: Vault entries are permanent. No building may modify or delete
a Vault entry. Office enforces this by monitoring all Store write events and rejecting
any that target a Vault address.

**IV. Identity rule**: every action is attributed to a named Actor. Office issues and
maintains all tokens. A building that accepts an unidentified request is in violation
and SHOULD be suspended.

**V. Token rule**: no Actor may issue a token exceeding its own permission scope.
Office enforces this at issuance time.

**VI. Op code rule**: extension buildings emit only their declared op codes. Office
monitors Stream for undeclared op code emissions and SHOULD suspend the violating
building's token on detection.

**VII. Audit rule**: Office's own actions are themselves recorded on Stream and in
the audit log. Office cannot take governance actions without leaving a record.
There is no privileged unrecorded Office action.

These seven rules are the constitution of the compound. They are not configurable
per installation — they are unconditional requirements of a conforming Works compound.

---

## 9. Office's Own Stream Events

Office emits the following S and A events for its own operations:

| Event | When |
|-------|------|
| `S token-issued` | New capability token created |
| `S token-revoked` | Token revoked before expiry |
| `S token-expired` | Token expired at its natural expiry (SHOULD emit) |
| `S register` | Extension building registration approved |
| `S deregister` | Extension building deregistered |
| `S token-revoked` (building) | Extension building token suspended |
| `S veto` | Veto issued on a G approve |
| `S orphan-detected` | Orphan chunk identified |
| `S reset` | Before Stream Log truncation (rare) |
| `A token-issued` | Audit log entry for token issuance |
| `A token-revoked` | Audit log entry for token revocation |
| `A registered` | Audit log entry for extension registration |
| `A veto-issued` | Audit log entry for veto |
| `A veto-post-delivery` | Audit log entry for post-delivery veto (compliance event) |
| `A orphan-detected` | Audit log entry for orphan detection |
| `A orphan-resolved` | Audit log entry for orphan resolution |
| `A audit-complete` | Audit log entry after a periodic orphan/compliance sweep |

Office's own identity in Stream events:
```json
{"actor": "office", "session_id": "office-permanent"}
```

Office does not have sessions in the Actor sense — it is always active. Its
`session_id` is the fixed string `"office-permanent"`. This is the one exception
to the session model: Office is a building, not an Actor, and its presence is
continuous.

---

## 10. Multi-Office Installations

Works-Standard v0.1 specifies a single Office per Works installation. Cross-installation
governance (multiple Works compounds that need coordinated governance) is outside
the scope of this standard.

An installation MUST have exactly one Office. A compound without an Office building
is not conforming. A compound with two Office buildings competing for governance
authority is not conforming.

In a distributed Works installation where the compound spans multiple nodes,
there MUST be a single logical Office, even if its implementation is distributed.
The Stream Log's Pacioli guarantee and Office's Stream-based state reconstruction
mean that Office state can be reconstructed on any node from the shared Log.

---

## 11. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Office operates in governance plane; MUST NOT sit on content paths | 2 | MUST |
| Office subscribes to Stream Bus for all events | 2 | MUST |
| Building registry is append-only; deregistered buildings remain | 3.1 | MUST |
| Building registry reconstructible from Stream replay | 3.1 | MUST |
| Token store is append-only; revoked tokens remain | 3.2 | MUST |
| Office audit log is in durable persistent storage | 3.3 | MUST |
| Privilege escalation MUST NOT be permitted at token issuance | 4.1 | MUST |
| Token revocation takes effect immediately via Bus broadcast | 4.3 | MUST |
| Revocation is not retroactive — completed operations stand | 4.3 | MUST |
| All buildings validate Actor token at operation time | 4.4 | MUST |
| Declaration rejection includes specific failure reasons | 5.1 | MUST |
| `S register` emitted on successful extension registration | 5.3 | MUST |
| `S veto` references G approve stream_event_id | 6.2 | MUST |
| Dispatch MUST check for `S veto` before delivery | 6.2 | MUST |
| Post-delivery veto recorded in audit log as compliance event | 6.2 | MUST |
| Veto list reconstructible from Stream replay on restart | 6.3 | MUST |
| Office detects orphans at startup after full replay | 7.1 | SHOULD |
| Orphan detection performed on building deregistration | 7.1 | SHOULD |
| `S orphan-detected` emitted when orphan found | 7.2 | MUST |
| Human Actor notified of unresolvable orphans | 7.2 | MUST |
| Office's own actions recorded on Stream and in audit log | 8 | MUST |
| No unrecorded Office governance action | 8 | MUST |
| Exactly one Office per Works installation | 10 | MUST |

---

*Works-Standard 07-office — end of document*  
*Next: `docs/08-manifest.md`*
