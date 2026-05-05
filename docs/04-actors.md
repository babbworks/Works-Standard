# Works-Standard — 04: Actors

**Document:** 04-actors  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md, 02-stream-protocol.md, 03-buildings.md

---

## 1. Scope

This document specifies the four Works Actor types, the session model, capability
tokens, and the identity requirements that apply to all Actor interactions with
the compound. It defines what distinguishes each Actor type structurally, what
each may do by default, and how Office constrains or extends those defaults through
capability tokens.

Actor behavior within specific buildings (which buildings an Actor may invoke,
what operations require which token) is covered here. The Office process for
issuing and revoking tokens is in `07-office.md`.

---

## 2. The Actor Model

An Actor is a named entity that interacts with the Works compound. The Actor model
has three foundational rules:

**Identity**: every action in a conforming Works installation is attributed to a
named Actor. Anonymous actions are not permitted. A building that accepts an
operation without a valid Actor identity in the request is not conforming.

**Session scope**: every Actor operates within a session. A session has a beginning
(authentication), a duration (active operation), and an end (explicit close or
timeout). All actions within a session are attributable to that session's Actor.
Bench contents are tied to a session and cleared when the session ends.

**Capability bounds**: every Actor carries a capability token issued by Office that
defines exactly what it may do. No Actor may perform an operation its token does not
permit. No Actor may issue, modify, or revoke its own token.

These three rules are not features — they are structural requirements. A Works
installation where any action can occur without identity, outside a session, or
beyond capability bounds is not conforming.

---

## 3. Actor Types

### 3.1 Human

```
Human { name, session_id, capability_token }
```

A person interacting with the compound directly — through a CLI, a browser UI,
a TUI, or any other interface that a person operates in real time.

**Default capabilities** (subject to Office token):
- Pull any chunk from any Store to Bench
- Commit chunks from Bench to any Store
- Invoke any Mill with chunks from Bench or Store
- Approve, archive, reject, hold, or reroute at Gate
- Trigger any Barrel
- Read Vault entries
- Request Stream replay

**Session behavior**: Human sessions are finite and explicitly bounded. A Human
Actor MUST authenticate before any operation. Session duration SHOULD be
configurable per installation. On session end — whether explicit logout, timeout,
or unexpected disconnect — Bench contents are cleared. The Human SHOULD be warned
before session close if uncommitted Bench chunks would be lost.

**Gate authority**: Human Actors have Gate approval authority by default. A Human
Actor approving at Gate MUST be identified by name in the G event ctx:
`"actor": "human:mp"`. A Gate approval attributed to an unnamed or generic human
identity is not conforming.

**Notes**: the Human Actor is the primary override mechanism in the compound. When
an Automated Actor holds at Gate because it cannot self-approve, it is a Human
Actor who resolves the hold. When an Agent Actor's output is held for review, a
Human Actor reviews and approves. The Human is not the only Actor type, but the
compound is designed so that a Human can always intervene at any point.

---

### 3.2 Agent

```
Agent { name, model, session_id, capability_token, context_limit }
```

A named AI agent — LLM-based or otherwise — interacting with the compound within
a session. Agent Actors are first-class participants in Works: they read Stores,
produce chunks, trigger Mills, commit to Stores, and submit to Gate. They are
subject to the same identity, session, and capability constraints as all other
Actor types.

**Required fields**:

`name` — a human-assigned identifier for this agent in this installation.
`agent:claude-sonnet-4-6` or `agent:summarizer` or `agent:intake-classifier`.
Names MUST be stable for a given agent role — changing a name changes the audit
trail attribution.

`model` — the specific model identifier for LLM-based agents. Recorded in the
session record and in Stream events attributed to this Agent. Enables audit
reconstruction to know not just which agent but which model version acted.

`context_limit` — the bounded amount of Stream history or Store content this
Agent can hold in working memory during a session. Works installations MUST NOT
present an Agent with more content than its declared context limit in a single
operation. Implementations SHOULD chunk large Store reads into context-sized
batches when operating with Agent Actors.

**Default capabilities** (subject to Office token):
- Read any Store chunk
- Pull chunks to Bench
- Commit chunks from Bench to Stores
- Invoke Mills
- Submit chunks to Gate
- Subscribe to Stream Bus with filters

**Gate authority**: Agent Actors MUST NOT self-approve at Gate by default. An
Agent that produces output and submits it to Gate must wait for a Human Actor or
an explicitly authorized Automated Actor to approve. Office MAY grant a specific
Agent Actor self-approval capability through its token, but this MUST be explicit
— the default is no self-approval.

This restriction is not a limitation on Agent capability — it is the Right to
Assembly applied to AI-generated content. An Agent can produce anything; a Human
confirms it leaves the compound.

**Session behavior**: Agent sessions are bounded by context limit as well as time.
When an Agent's context is exhausted, the session MUST end cleanly: Bench contents
are cleared, a session-end S event is emitted to Stream, and a new session begins
if the Agent continues. The session boundary is a Stream record — Agent continuity
across context resets is reconstructible from Stream replay.

**Identity in Stream events**: every Stream event attributed to an Agent MUST
include both name and model in ctx:
```json
{"actor": "agent:summarizer", "model": "claude-sonnet-4-6", "session_id": "..."}
```

---

### 3.3 Automated

```
Automated { barrel_name, trigger, capability_token }
```

A Barrel executing autonomously — cron-scheduled, event-triggered, or invoked
programmatically — with no human in the loop for that execution. Automated Actors
represent the compound's self-operating layer: recurring exports, nightly archival
runs, triggered document pipelines, scheduled ingestion.

**Identity**: an Automated Actor's identity is the Barrel that is running.
`automated:nightly-export` or `automated:quarterly-close`. The `barrel_name` field
MUST match the name of the Barrel chunk stored in its Store. Stream events
attributed to an Automated Actor carry:
```json
{"actor": "automated:nightly-export", "trigger": "cron:0 2 * * *"}
```

**Trigger types**:

`cron:<expression>` — scheduled by time, e.g. `cron:0 2 * * *` (daily at 02:00).

`event:<filter>` — triggered by a matching Stream Bus event, e.g.
`event:op=G,action=approve` fires when any Gate approval lands on the Bus.

`manual:<actor>` — triggered explicitly by a named Actor. Not fully autonomous
but still runs as an Automated Actor under its own capability token.

**Default capabilities** (subject to Office token — narrowest defaults of all Actor types):
- Read Stores specified in the Barrel definition
- Write to Stores specified in the Barrel definition
- Invoke Mills specified in the Barrel definition
- Submit chunks to Gate
- Emit S events to Stream

**Gate authority**: Automated Actors MUST NOT self-approve at Gate. If a Barrel's
execution reaches a step that requires Gate authorization, it MUST submit to Gate
and hold. If the Gate hold requires Human review, the Automated Actor MUST notify
the relevant Human Actor (via a Stream event that a subscribed interface surfaces)
and suspend that execution branch until the hold is resolved.

An Automated Actor that cannot complete a Gate step MUST NOT skip it, work around
it, or fail silently. It MUST emit `S barrel-fail` with the reason, leave the
chunk in Gate egress with a hold flag, and wait.

**Capability escalation**: MUST NOT occur. An Automated Actor that encounters a
step requiring permissions beyond its token MUST fail that step and emit
`S barrel-fail`. It MUST NOT attempt to acquire additional permissions at runtime.

**Notes**: the Automated Actor is the compound's productivity engine — it does
the recurring work that would otherwise require Human attention for every cycle.
Its narrow default capability token is intentional: an Automated Actor that can
do anything is indistinguishable from a system with no access control. The token
defines exactly what each Automated Actor is allowed to do, and nothing beyond.

---

### 3.4 External

```
External { system_id, protocol, capability_token }
```

An external system interacting with the compound — another Works installation, a
BitLedger node, a third-party API, a legacy tool, or any system that is not a
Human, Agent, or Automated Actor within this installation.

**Identity**: `external:<system_id>`, where `system_id` is a stable identifier
for the external system. `external:bitledger-node-01` or `external:client-erp`.
The `protocol` field records how the external system communicates with Works:
`bitpads`, `rest-api`, `works-standard`, `custom`.

**Permitted interactions**:

External Actors MUST interact with the compound only through:
- **Intake**: sending data into the compound. Data from an External Actor arrives
  at Intake, is verified and classified, and enters the compound as a chunk.
  The External Actor's identity is recorded in the chunk's `src` field and first
  provenance entry.
- **Dispatch**: receiving Gate-authorized output from the compound. An External
  Actor may be a Dispatch destination — the compound sends to it.

**MUST NOT**:
- Access Stores directly.
- Access Mills directly.
- Access Barrels directly.
- Access Bench (External Actors have no session-based workspace in the compound).
- Appear in Gate decisions. External Actors do not approve, reject, or hold.

**Capability token**: External Actors carry a capability token specifying which
Intake subtypes they may submit to and which Dispatch destinations they may receive
from. The token does not grant internal compound access.

**Notes**: the External Actor boundary is the compound's perimeter. Everything
outside is an External Actor; everything inside is one of the three internal types
(Human, Agent, Automated). The Works perimeter is enforced at Intake (entry) and
Dispatch (exit). An external system that bypasses Intake to write directly to a
Store is not interacting with a conforming Works installation.

---

## 4. Session Model

### 4.1 Session Lifecycle

A session has four phases:

**Open**: the Actor authenticates with the compound. Office issues or validates
a session token. A session record is created. An S `session-open` event is emitted
to Stream recording the Actor identity, type, and session id.

**Active**: the Actor performs operations. Every operation carries the session id.
Bench contents accumulate. Stream events are attributed to this session.

**Close**: the Actor explicitly ends the session, or the session times out.
Bench contents are cleared. An S `session-close` event is emitted to Stream.
The session record is closed.

**Expired**: sessions that have been closed. The session record and its Stream
events are permanent (Pacioli guarantee). The session id is not reused.

### 4.2 Session Events

| Event | When |
|-------|------|
| `S session-open` | Session begins — records Actor identity, type, model (Agent), trigger (Automated) |
| `S session-close` | Session ends cleanly |
| `S session-timeout` | Session ends due to inactivity or context limit |
| `S session-interrupt` | Session ends unexpectedly (crash, disconnect) |

`S session-interrupt` MUST be emitted by the compound when it detects that a
session ended without a clean close. The compound detects this via heartbeat
absence or process termination. Bench contents are cleared on interrupt as on
any other session end.

### 4.3 Session Identity String

Throughout Stream events, Actor identity is expressed as a string:

```
<type>:<name>
```

Examples:
```
human:mp
agent:claude-sonnet-4-6
automated:nightly-export
external:bitledger-node-01
```

Where additional disambiguation is needed (multiple Agent sessions of the same
model), the session id is appended:

```
agent:claude-sonnet-4-6:sess-7f3a2b
```

The identity string MUST be consistent within a session. An Actor that changes
its identity string mid-session is not conforming.

---

## 5. Capability Tokens

### 5.1 What a Token Is

A capability token is a scoped authorization record issued by Office to an Actor.
It defines exactly what the Actor may do within the compound. Every Actor carries
exactly one token at a time. A token is not a password or a cryptographic key —
it is a structured permission record that the compound enforces at every operation.

Token fields:

| Field | Type | Meaning |
|-------|------|---------|
| `token_id` | UUID | Unique identifier for this token |
| `actor` | string | Actor identity string this token is issued to |
| `issued_by` | string | Office identity that issued the token |
| `issued_at` | integer | Unix timestamp of issuance |
| `expires_at` | integer | Unix timestamp of expiry; 0 = no expiry |
| `permissions` | object | Scoped permission set (see §5.2) |
| `stream_event_id` | string | S token-issued event cross-reference |

### 5.2 Permission Scope

The `permissions` object defines what the token permits. Permissions are
additive — an Actor may do what its token explicitly lists, and nothing else.

```json
{
  "stores": {
    "read":  ["S-01", "S-legal"],
    "write": ["S-01"],
    "fill":  []
  },
  "mills":    ["document-assembly", "time-analytics"],
  "gate":     { "approve": false, "archive": false, "hold": true },
  "barrels":  ["nightly-export"],
  "vault":    { "read": true },
  "stream":   { "subscribe": true, "replay": true, "emit": ["S","A"] },
  "office":   { "register": false, "token_issue": false }
}
```

**Stores**: `read` lists Stores the Actor may pull from; `write` lists Stores it
may commit to; `fill` lists Stores where it may execute fill instructions. An
empty array means no access. `["*"]` means all Stores (use with care — only
appropriate for administrative Actors).

**Mills**: array of Mill names the Actor may invoke. An empty array means the
Actor may not invoke any Mill directly.

**Gate**: booleans for each Gate decision type. `approve` and `archive` grant
authority to make those decisions. `hold` grants authority to place a hold.
Human Actors default to all true. Agent and Automated Actors default to all false.

**Barrels**: array of Barrel names the Actor may trigger. For Automated Actors,
this is typically the single Barrel the token was issued for.

**Vault**: `read: true` grants read access. Write access to Vault is never granted
through a capability token — Vault is written only by Gate and Intake through
protocol paths, not Actor tokens.

**Stream**: `subscribe: true` grants Bus subscription. `replay: true` grants Replay
access. `emit` is an array of op codes the Actor may emit directly (for Actors
that emit their own events, such as Agent Actors writing A annotations to Stream).

**Office**: `register: true` grants extension building registration. `token_issue:
true` grants capability to issue tokens to other Actors. Both default false and
are typically only granted to administrative Human Actors.

### 5.3 Token Lifecycle

**Issuance**: Office emits `S token-issued` to Stream when a token is created.
The token is stored by the compound and provided to the Actor at session open.

**Validation**: every building that receives an operation from an Actor MUST
validate that the Actor's token permits that operation before executing it.
A building that skips token validation is not conforming.

**Revocation**: Office may revoke a token at any time by emitting `S token-revoked`
to Stream. After revocation, any operation attempted with the revoked token MUST
be denied. Active sessions using the revoked token MUST be closed.

**Expiry**: tokens with a non-zero `expires_at` MUST be treated as revoked after
that timestamp. The compound SHOULD proactively close sessions using expired tokens
rather than waiting for the next operation to fail.

**Renewal**: a new token is issued for the same Actor when the prior token expires
or is revoked. Token renewal is a new issuance — a new `token_id`, a new
`S token-issued` event. Prior tokens are not reused.

### 5.4 Default Token Profiles

Office SHOULD provide default token profiles for each Actor type that conforming
installations can use as starting points. These are recommendations, not requirements.

**Human (default)**: read/write all Stores, invoke all Mills, Gate approve/archive/hold,
trigger all Barrels, read Vault, full Stream access.

**Agent (default)**: read all Stores, write to designated output Stores only, invoke
designated Mills, Gate hold only (no approve/archive), no Barrel trigger, read Vault,
Stream subscribe and emit A events.

**Automated (default)**: read/write named Stores only (specified at Barrel registration),
invoke named Mills only, Gate hold only, trigger own Barrel only, no Vault read, Stream
emit S events only.

**External (default)**: no Store access, no Mill access, no Gate access, no Barrel
access, no Vault access, Stream emit I events via Intake only.

---

## 6. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Every action attributed to a named Actor | 2 | MUST |
| Anonymous actions rejected | 2 | MUST |
| Every Actor operates within a session | 2 | MUST |
| Bench cleared on session end regardless of cause | 4.1 | MUST |
| S session-open emitted at session start | 4.2 | MUST |
| S session-close or session-interrupt emitted at session end | 4.2 | MUST |
| Agent Actor carries name, model, context_limit | 3.2 | MUST |
| Agent Actor identity includes model in Stream ctx | 3.2 | MUST |
| Agent Actor MUST NOT self-approve at Gate without explicit token grant | 3.2 | MUST |
| Automated Actor MUST NOT self-approve at Gate | 3.3 | MUST |
| Automated Actor MUST NOT escalate capabilities at runtime | 3.3 | MUST |
| Automated Actor on Gate hold MUST notify Human and suspend | 3.3 | MUST |
| External Actor interacts only through Intake and Dispatch | 3.4 | MUST |
| External Actor MUST NOT access Stores, Mills, Barrels, or Bench directly | 3.4 | MUST |
| Every building validates Actor token before executing operations | 5.3 | MUST |
| Revoked tokens denied immediately; active sessions closed | 5.3 | MUST |
| Expired tokens treated as revoked | 5.3 | MUST |
| Token permissions are additive — not granted means not permitted | 5.2 | MUST |

---

*Works-Standard 04-actors — end of document*  
*Next: `docs/05-conveyance.md`*
