# Works-Standard — 09: Extension Protocol

**Document:** 09-extension  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-05  
**Depends on:** 00-concepts.md, 02-stream-protocol.md, 03-buildings.md, 04-actors.md

---

## 1. Scope

This document specifies the extension protocol: the mechanism by which new building
types emerge, declare themselves, register with Office, and participate in Works
conveyance as first-class compound members.

The nine core buildings are defined once and frozen in this version of the standard.
The extension protocol is how Works grows beyond them without the core standard
changing. A building type that registers under this protocol is not a second-class
participant — it is subject to the same Stream, Gate, and Pacioli obligations as
any core building, and once registered it participates in conveyance on equal terms.

This document does not specify what extension buildings should do — that is outside
the standard's scope. It specifies the structural requirements they must meet and
the process by which they become registered.

---

## 2. Why Extensions Exist

The core nine buildings cover the general compound structure. They do not cover
every specific operational need that a Works installation may require. A legal firm
may need a Docket building — a specialized tracker for court deadlines that monitors
Stores and emits events on approaching dates. A medical installation may need a
Consent building that enforces patient authorization before any chunk containing
personal health information passes Gate. A financial services firm may need a
Clearance building that verifies regulatory requirements before Dispatch.

None of these can be anticipated in the core standard. The extension protocol
provides the structural path for them to emerge and register without modifying
Works-Standard itself.

The extension protocol has one non-negotiable requirement: every extension building
must be as auditable, as Stream-connected, and as Pacioli-compliant as the core
buildings. Extension is not a path around the compound's constitutional guarantees
— it is a path into them.

---

## 3. Extension Building Categories

An extension building must declare one of six roles at registration. The role
determines which conveyance paths the extension may participate in.

| Role | Description | Example |
|------|-------------|---------|
| `input` | Receives data from outside the compound; an Intake subtype or alternative | Email Intake, IoT sensor Intake |
| `storage` | Holds chunks at addressed locations; a Store subtype or variant | Encrypted Store, distributed Store |
| `transformation` | Transforms chunks via a lens pipeline; a Mill subtype | LLM-powered Mill, classification Mill |
| `output` | Delivers Gate-authorized chunks to external destinations; a Dispatch subtype | Regulatory filing Dispatch, e-signature Dispatch |
| `control` | Schedules or sequences compound operations; a Barrel subtype or variant | Calendar-aware Barrel, dependency-tracking Barrel |
| `audit` | Observes Stream events and maintains specialized records; an Office subtype | Compliance recorder, domain-specific audit trail |

An extension building with role `input` participates in the Intake position in
conveyance paths. Role `storage` participates where Store participates. Role
`transformation` where Mill participates. And so on. The role declaration is
binding — a registered `storage` extension MUST NOT attempt to act as an `output`
building.

An extension building MAY declare only one role. A building that performs multiple
functions MUST be registered as multiple extensions, each with its own role.

---

## 4. Registration Requirements

A building type that wishes to register with Office as a Works extension MUST
satisfy five requirements before registration is granted.

### 4.1 Declaration

The building type MUST provide a written declaration containing:

```json
{
  "name":        "unique-building-name",
  "role":        "input | storage | transformation | output | control | audit",
  "version":     "semver string, e.g. 0.1.0",
  "maintainer":  "contact identity or organization",
  "op_codes":    ["X"],
  "description": "one paragraph describing what this building does",
  "stream_events": [
    {"op": "X", "action": "event-name", "when": "description of when emitted"}
  ],
  "gate_dependency": true | false,
  "pacioli_state":   true | false
}
```

`op_codes`: the set of op codes this building emits to Stream. If the building
requires a new op code (a letter not in the core set), it MUST declare it here
and request assignment from Office. If it uses only existing core op codes, it
lists those. An extension building MUST NOT emit op codes not declared at
registration.

`gate_dependency`: `true` if this building produces output that must pass Gate
before leaving the compound. An `output`-role extension MUST declare `true`.
A `storage`-role extension typically declares `false`.

`pacioli_state`: `true` if this building maintains internal state that must be
recoverable from Stream replay. If `true`, the building MUST describe in its
declaration how its state is reconstructed from its declared Stream events.

### 4.2 Stream Compliance

The building MUST demonstrate Stream compliance before registration. Stream
compliance means:

- Every significant action the building performs emits a Stream event using the
  op codes and actions declared in its declaration.
- The building never performs a significant action without a Stream record.
- The building's internal state, if any (`pacioli_state: true`), is fully
  reconstructible from its Stream events alone — no external state store is
  required for recovery.

The registering party MUST provide a Stream compliance statement: a written
description of every Stream event the building emits, the conditions under which
each is emitted, and the recovery procedure for reconstructing state from replay.

### 4.3 Gate Boundary Compliance

If the building has role `output` or otherwise produces chunks for external delivery,
it MUST demonstrate that it honors the Gate boundary:

- Output chunks do not leave the compound without a G approve event in their
  provenance chain.
- The building does not accept chunks from any source other than Gate for
  external delivery.

A `transformation`-role extension (Mill subtype) MUST demonstrate that Content Mill
output waits at egress for Gate rather than proceeding directly to any output path.

### 4.4 Pacioli Compliance

If `pacioli_state: true`, the building MUST demonstrate that its internal state
storage is append-only for all operational data:

- Records are added; existing records are not modified or deleted.
- Stage transitions create new records, not overwrites.
- A full recovery from zero state, using only the Stream Log and the building's
  declared Stream events, produces identical operational state to the current
  runtime state.

A building that cannot satisfy this demonstration for its declared state MUST
reduce its `pacioli_state` declaration to `false` and eliminate the state in
question, replacing it with Stream-derived runtime state.

### 4.5 Registration Token Receipt

Registration is only complete when Office issues a registration token to the
building. Until the token is issued, the building is an External Actor. It may
interact with the compound only through Intake and Dispatch.

The registration token contains:

```json
{
  "token_id":       "<uuid>",
  "building_name":  "unique-building-name",
  "role":           "<declared role>",
  "op_codes":       ["X"],
  "issued_by":      "office",
  "issued_at":      <unix timestamp>,
  "version":        "0.1.0"
}
```

The token is stored in the compound's building registry (maintained by Office)
and provided to the extension building at startup. The building presents its
token on every inter-building communication to identify itself.

---

## 5. Op Code Assignment

Core op codes (`T`, `F`, `B`, `D`, `H`, `S`, `A`, `I`, `G`) are reserved by
this standard and MUST NOT be used by extension buildings for new semantics.
Extension buildings that emit only core op codes with new action values do not
require a new op code — they declare the existing op code in their registration.

If an extension building requires a new op code:

1. It declares the requested letter in its registration declaration.
2. Office checks that the letter is not already assigned to another registered
   extension in this installation.
3. If available, Office assigns the letter and records the assignment in the
   building registry with `S register` on Stream.
4. The assigned op code is local to this Works installation. Cross-installation
   interoperability with custom op codes requires both installations to have
   registered the same extension with the same op code assignment.

Op code assignment is installation-local. There is no global op code registry
in Works-Standard v0.1. A future version of the standard MAY establish a global
registry for widely-adopted extension op codes.

Letters available for extension assignment: all uppercase ASCII letters not
in the core set. As of v0.1, available letters include `C`, `E`, `J`, `K`,
`L`, `M`, `N`, `O`, `P`, `Q`, `R`, `U`, `V`, `W`, `X`, `Y`, `Z`.

---

## 6. Registration Process

The registration process is a sequence of interactions between the registering
party and the Office building of the target Works installation.

```
Registering party                    Office
       │                               │
       │── declaration (JSON) ────────►│
       │                               │ validate: role, op_codes,
       │                               │ stream_compliance, gate_boundary,
       │                               │ pacioli_compliance
       │                               │
       │◄── validation result ─────────│
       │    (pass or reject with       │
       │     specific failures listed) │
       │                               │
       │  [if rejected: revise and     │
       │   resubmit declaration]       │
       │                               │
       │── compliance statement ───────►│
       │   (stream events, recovery    │
       │    procedure, gate proof)     │
       │                               │ review compliance statement
       │                               │ assign op codes if requested
       │                               │ emit S register to Stream
       │                               │
       │◄── registration token ────────│
       │                               │
```

**Validation failure**: Office MUST provide specific failure reasons when rejecting
a declaration. Generic rejection is not conforming. The registering party MUST be
able to determine exactly what to fix.

**S register event**:
```
<ts> S register <building_name> {"role":"transformation","op_codes":["X"],"version":"0.1.0","issued_by":"office"}
```

This event is permanent on Stream. The registration of an extension building is
part of the compound's history. If the building is later deregistered, a separate
`S deregister` event is emitted — the original S register is not removed.

**Registration scope**: registration is per Works installation. An extension
building that operates across multiple Works installations must register separately
with each installation's Office.

---

## 7. Extension Building Lifecycle

### 7.1 Active

A registered extension building with a valid token is active. It participates in
conveyance according to its declared role. It emits to Stream using its declared
op codes. Other buildings recognize it by its registration token.

### 7.2 Version Update

When an extension building releases a new version, it submits an updated
declaration to Office with a new `version` field. Office validates the update,
assigns any new op codes, and issues a new registration token. The update is
recorded as a new `S register` event on Stream referencing the prior version.

Breaking changes — changes to declared op codes, role, or state recovery procedure
— MUST be treated as a new building registration, not a version update. The prior
version remains registered until explicitly deregistered.

### 7.3 Suspension

Office MAY suspend a registered extension building's token without deregistering
it — for example, if the building is found to be emitting undeclared op codes or
bypassing Gate. Suspension emits `S token-revoked` with reason on Stream. The
building's token becomes invalid. The building falls back to External Actor status.
The S register event is not affected — the building remains in the registry as
suspended.

### 7.4 Deregistration

An extension building may be deregistered by Office at any time. Deregistration
emits `S deregister` on Stream. The building's registration token is revoked.
The building becomes an External Actor.

Deregistration MUST NOT remove the S register event from Stream (Pacioli guarantee).
The complete registration history of every building — including deregistered ones —
is permanently on Stream.

---

## 8. Obligations After Registration

Registration is not a one-time event — it is an ongoing obligation. A registered
extension building MUST continuously satisfy its registration requirements for the
duration of its active status.

**Ongoing Stream compliance**: the building MUST emit to Stream for every significant
action throughout its operational life, not just during registration review.

**Op code discipline**: the building MUST emit only its declared op codes. Emitting
an undeclared op code is a violation that SHOULD trigger Office suspension.

**Version declaration**: when the building changes its behavior in ways that affect
its Stream events, gate dependency, or pacioli state, it MUST submit a version
update before deploying the change.

**Audit cooperation**: when Office requests an audit of the building's Stream events
(for compliance queries, orphan detection, or veto investigation), the building
MUST cooperate by providing access to its state for comparison against Stream replay.

---

## 9. The Extension Ecosystem

The extension protocol is the mechanism by which Works specializes for specific
industries and use cases without the core standard changing.

A Works installation may accumulate a registry of extension buildings over its
lifetime — each registered, versioned, and permanently recorded on Stream. The
registry is itself part of the compound's history: a future operator reading the
Stream Log from the beginning can reconstruct not just what happened in the compound
but what buildings were active at each point in time, and what they were capable of.

Extension buildings developed for one installation MAY be shared, published, or
licensed for use in other installations. The declaration format is designed to be
portable — a building that registers successfully in one Works installation can
register in any other with the same role and op code availability.

Over time, extension buildings that prove widely useful across installations may be
proposed for inclusion in a future version of Works-Standard as core buildings. The
path is: extension → widespread adoption → proposal → standard revision. The
extension protocol is not a permanent second-class tier — it is the incubator.

---

## 10. Conformance Requirements

| Requirement | Section | Level |
|-------------|---------|-------|
| Unregistered buildings participate only as External Actors | 4.5 | MUST |
| Every extension building declares role, op_codes, stream_events | 4.1 | MUST |
| Extension building emits only declared op codes | 8 | MUST |
| Office provides specific failure reasons on declaration rejection | 6 | MUST |
| S register emitted on successful registration | 6 | MUST |
| S register never removed from Stream after registration | 7.4 | MUST |
| S deregister emitted on deregistration; S register preserved | 7.4 | MUST |
| Op code assignment is installation-local in v0.1 | 5 | MUST |
| Core op codes MUST NOT be used for new semantics by extensions | 5 | MUST |
| output-role extensions declare gate_dependency: true | 4.1 | MUST |
| Breaking changes require new registration, not version update | 7.2 | MUST |
| Registered building continuously satisfies registration requirements | 8 | MUST |
| Office MAY suspend token without deregistering | 7.3 | MAY |

---

*Works-Standard 09-extension — end of document*  
*Next: `docs/10-compliance.md`*
