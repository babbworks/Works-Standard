# Works-Standard — 02: Stream Protocol

**Document:** 02-stream-protocol  
**Version:** 0.1-draft  
**Status:** Draft  
**Date:** 2026-05-04  
**Depends on:** 00-concepts.md

---

## 1. Scope

This document specifies the Stream protocol: the encoding of Stream events, the
contract of the three Stream layers (Log, Bus, Replay), and the wire encoding
options for bus and replay delivery.

This document does not specify which buildings emit which events — that is
`03-buildings.md`. It does not specify the full BitPads binding — that is
`12-bitpads-binding.md`. It specifies the Stream layer itself: what a conforming
Stream implementation must accept, store, deliver, and recover.

A Works installation that does not implement a conforming Stream is not a conforming
Works installation.

---

## 2. The Log Layer

### 2.1 The Stream Log

The Stream Log is a single append-only text file maintained by the Works installation.
It is the authoritative record of all events that have occurred in the compound.
All derived state — Store contents, Actor sessions, Gate decisions, Office records —
is reconstructible from the Stream Log by replay. The Log is the truth.

A conforming implementation MUST maintain exactly one Stream Log per installation.
The Log MUST be stored as UTF-8 encoded plain text. The Log MUST be human-readable
without any tool other than a text reader.

The Log file MUST persist across process restarts, system reboots, and implementation
upgrades. A conforming implementation MUST NOT truncate, rotate, or partition the
Log file except through the documented reset procedure (§2.5).

### 2.2 Log Entry Format — Hollerith Encoding

Every entry in the Stream Log is a Hollerith-encoded event line. Hollerith encoding
is positional: five fields in fixed order, separated by single spaces, terminated
by a newline character (`\n`). The positions of the fields are the schema.

```
<ts> <op> <action> <object> [<ctx>]
```

| Position | Field    | Type              | Required | Constraints |
|----------|----------|-------------------|----------|-------------|
| 1        | `ts`     | integer           | MUST     | Unix timestamp (seconds); positive integer; primary sort key |
| 2        | `op`     | single char       | MUST     | Uppercase letter; registered op code |
| 3        | `action` | string            | MUST     | Lowercase; no whitespace; describes what happened |
| 4        | `object` | string            | MUST     | Identifier of the entity acted upon; no whitespace |
| 5        | `ctx`    | minified JSON     | MAY      | Optional occurrence context; valid JSON if present; no bare whitespace between tokens |

**Field constraints:**

`ts` MUST be a positive integer. Implementations SHOULD use the current UTC unix
timestamp at the moment of event creation. Implementations MUST NOT use timestamps
in the past to fabricate historical events after the fact; the reset procedure
(§2.5) is the only authorized mechanism for log modification.

`op` MUST be a single uppercase ASCII letter. It MUST be either a core op code (§3)
or a registered extension op code. An unrecognized `op` value MUST cause the line
to be treated as a system event (S) by replay processors that do not recognize it,
not rejected.

`action` MUST be a non-empty lowercase string containing no whitespace and no
embedded newlines. Action values are defined per op code (§3). An unrecognized
action value for a known op code MUST be accepted by conforming replay processors
and passed through unchanged.

`object` MUST be a non-empty string containing no whitespace and no embedded
newlines. Object values are typically UUIDs, 12-character SHA-256 hash prefixes,
or human-assigned identifiers. The Works-Standard does not constrain object value
format beyond these character constraints.

`ctx` if present MUST be valid JSON. It MUST be minified — no whitespace between
tokens outside of string values. The `ctx` field MUST NOT contain embedded newlines.
If `ctx` is absent, the line has exactly four space-separated fields.

**Line constraints:**

A log entry MUST be terminated by a single `\n` character. A log entry MUST NOT
contain embedded newline characters in any field. A log entry MUST be encodable
in UTF-8. The Works-Standard does not specify a maximum line length, but
implementations SHOULD keep `ctx` content under 4096 bytes to preserve readability
and tool compatibility.

**Worked example:**

```
1746348000 T add 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"src":"task","prof":"work","proj":"alpha","tags":["next"],"name":"Review contract"}
1746348060 F start 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"src":"task","prof":"work","proj":"alpha","tags":["next"],"name":"Review contract"}
1746349800 B stop a3f9c12d8e4b {"src":"timew","prof":"work","proj":"alpha","tags":["alpha","work"]}
1746350100 A write 7c2e9f1a3b5d {"src":"jrnl","prof":"work","proj":"alpha","tags":["standup"],"name":"Morning notes"}
1746350200 G approve 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"actor":"human:mp","decision":"approve","reason":"ready for dispatch"}
```

### 2.3 The H Event — Schema Header

The first entry in every Stream Log MUST be an H event declaring the schema version
and field map. This entry is written once at log creation and MUST NOT be modified.

```
0 H v0 stream.log {"fields":"ts op action object ctx","encoding":"utf8"}
```

The timestamp of the H header event is `0` (not the creation time). This ensures
the header sorts to the top of any timestamp-ordered output.

A conforming replay processor encountering a Log without an H header at position 0
MUST treat the log as Works-Standard v0 format and proceed. A conforming replay
processor encountering an H header with an unrecognized version MUST halt and
report the version mismatch rather than proceeding with potentially incorrect parsing.

On schema version changes, a new H event MUST be appended at the current timestamp
(not at position 0). Replay processors MUST apply the new schema from that timestamp
forward and the previous schema before it.

### 2.4 Sort Invariant

The Stream Log MUST be sortable by field 1 (`ts`) at any point using a numeric sort
(`sort -n`). This is both a property and a tool: any implementation can verify log
integrity, produce a chronological view, or merge two logs using only a standard
sort utility.

Events MAY arrive out of timestamp order — for example, from batch ingestion of
historical data. A conforming implementation MUST sort the log after any batch
append operation. Real-time single-event appends from a single writer are inherently
ordered and do not require re-sort.

When two events share the same `ts` value, their relative order within that timestamp
is implementation-defined. Replay processors MUST NOT assume a specific sub-second
ordering for same-timestamp events.

### 2.5 Log Reset

The Stream Log MUST NOT be truncated or deleted in normal operation. The only
authorized log modification is the reset procedure:

1. A reset S event MUST be appended to the log before any truncation:
   ```
   <ts> S reset stream.log {"reason":"<operator-supplied reason>","actor":"<actor-id>"}
   ```
2. Only after the S reset event is durably written MAY the log be truncated.
3. The reset event itself is lost in the truncation — this is acknowledged and
   intentional. The act of reset is recorded in the Office audit log even when
   it is lost from Stream.

A conforming implementation MUST require explicit operator confirmation (equivalent
to `--confirm` flag or equivalent interactive acknowledgement) before executing
a reset. A reset MUST be logged to the Office audit log with the operator's Actor
identity before the truncation occurs.

---

## 3. Op Codes

Op codes classify Stream events. Each op code is a single uppercase ASCII letter.
The core op codes are fixed for Works-Standard v0. New op codes MAY be registered
through the extension protocol (`09-extension.md`) and MUST be assigned by Office.

### Core Op Codes

#### T — Task

Emitted by: Intake, Barrel, TaskWarrior hook adapters  
Semantics: task lifecycle events — creation, modification, completion, deletion

| Action   | Meaning |
|----------|---------|
| `add`    | Task created |
| `modify` | Task fields changed |
| `done`   | Task completed |
| `delete` | Task removed |
| `post`   | Financial transaction posting (ledger source) |

Required `ctx` fields: `src`, `prof`, `proj`, `tags`, `name`

```
1746348000 T add 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"src":"task","prof":"work","proj":"alpha","tags":["next","home"],"name":"Fix the login bug"}
```

#### F — Frick

Emitted by: Mill, Barrel, shell wrapper adapters  
Semantics: discrete state transition events — markers at the boundary between
operational states. Named for Frederick W. Frick's discrete-event time recorders.

| Action    | Meaning |
|-----------|---------|
| `start`   | Work on an object begins |
| `stop`    | Work on an object ends |
| `pause`   | Work suspended (resumable) |
| `resume`  | Work resumed after pause |
| `switch`  | Context switch to a different object |

Required `ctx` fields: `src`, `prof`; `proj` and `tags` SHOULD be present

```
1746348060 F start 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"src":"task","prof":"work","proj":"alpha","tags":["next"]}
```

#### B — Bundy

Emitted by: Mill, Barrel, timew adapters  
Semantics: interval boundary events — the clock-in and clock-out of bounded time
units. Named for Willard L. Bundy's start/stop time clock. Bundy intervals are
always derived from a matching start/stop pair; they are never stored as facts.

| Action  | Meaning |
|---------|---------|
| `start` | Interval opens |
| `stop`  | Interval closes |

Required `ctx` fields: `src`, `prof`; `proj` and `tags` SHOULD be present.  
`object` for B events MUST be a stable hash of the interval's identity
(e.g., SHA-256 first 12 characters of `start_timestamp|tags`).

```
1746348060 B start a3f9c12d8e4b {"src":"timew","prof":"work","proj":"alpha","tags":["alpha","work"]}
1746349800 B stop  a3f9c12d8e4b {"src":"timew","prof":"work","proj":"alpha","tags":["alpha","work"]}
```

#### D — Dey

Emitted by: Mill (Dey lens only)  
Semantics: continuous signal samples — point-in-time measurements of operational
state. Named for Alexander Dey's dial-based continuous time recorders. D events
represent the analog layer: not what happened, but how work is going.

| Action   | Meaning |
|----------|---------|
| `sample` | Behavioral state sample recorded |

Required `ctx` fields: `intensity` (float 0–1), `stability` (float 0–1),
`fragmentation` (float 0–1). Additional signal dimensions MAY be included.

```
1746348120 D sample session-2026-05-04 {"intensity":0.7,"stability":0.8,"fragmentation":0.2,"prof":"work"}
```

#### H — Hollerith

Emitted by: Stream (internal)  
Semantics: encoding-level markers — schema version declarations, field map events.
H events are not emitted by buildings; they are emitted by the Stream implementation
itself at log creation and on schema migration.

| Action    | Meaning |
|-----------|---------|
| `v0`      | Schema version 0 header |
| `migrate` | Schema version change |

```
0 H v0 stream.log {"fields":"ts op action object ctx","encoding":"utf8"}
```

#### S — System

Emitted by: any building  
Semantics: system and meta events — compound-level operations that do not fit a
more specific op code.

| Action     | Meaning |
|------------|---------|
| `reset`    | Log reset initiated |
| `sync`     | Batch ingest or sync operation completed |
| `handoff`  | Agent session handoff event |
| `disable`  | Building or Barrel disabled |
| `register` | Extension building registered with Office |

```
1746350400 S sync stream.log {"source":"tasks","added":47,"total":203}
```

#### A — Annotation

Emitted by: Intake, Bench  
Semantics: journal entries, notes, and annotations attached to an object or
standing alone as compound-level records.

| Action  | Meaning |
|---------|---------|
| `write` | Annotation created |
| `link`  | Annotation linked to an existing object |

Required `ctx` fields: `src`, `prof`; `name` SHOULD be present (title or
first line of annotation, max 60 characters).

```
1746350100 A write 7c2e9f1a3b5d {"src":"jrnl","prof":"work","proj":"alpha","tags":["standup"],"name":"Morning notes"}
```

#### I — Intake

Emitted by: Intake  
Semantics: arrival events — records every item that reaches Intake, whether
accepted or rejected.

| Action     | Meaning |
|------------|---------|
| `received` | Item arrived at Intake |
| `verified` | Item passed integrity check |
| `rejected` | Item failed verification or classification |
| `routed`   | Item dispatched to a Store or Mill |

Required `ctx` fields: `src`, `format`. `reason` MUST be included on `rejected`.

```
1746348000 I received manifest-0a1b2c3d {"src":"bitpads","format":"rich-record","size":29}
1746348000 I routed  manifest-0a1b2c3d {"src":"bitpads","dest":"S-finance","decomposed":false}
```

#### G — Gate

Emitted by: Gate  
Semantics: authorization events — every Gate decision is recorded on Stream.

| Action    | Meaning |
|-----------|---------|
| `approve` | Output authorized for Dispatch |
| `archive` | Output authorized for direct Vault entry without Dispatch |
| `reject`  | Output denied; returned to Mill egress or Store |
| `hold`    | Output held pending further action |
| `reroute` | Output redirected to a different Store or Mill |

Required `ctx` fields: `actor`, `decision`. `reason` SHOULD be present on
`reject` and `hold`.

```
1746350200 G approve 9f3c2a1b-4d5e-6789-abcd-ef0123456789 {"actor":"human:mp","decision":"approve"}
1746350210 G reject  draft-invoice-001 {"actor":"human:mp","decision":"reject","reason":"missing line items"}
```

### 3.1 The ctx Field — Standard Schema

The `ctx` field carries occurrence context. While the full schema is op-code-specific
(see per-op tables above), the following keys are used consistently across op codes
and MUST have the same meaning wherever they appear:

| Key      | Type             | Meaning |
|----------|------------------|---------|
| `src`    | string           | Source adapter or building name |
| `prof`   | string           | Works profile or installation name |
| `proj`   | string           | Primary project identifier |
| `tags`   | array of strings | Classification labels |
| `name`   | string           | Human-readable label (max 60 chars) |
| `actor`  | string           | Actor identity string (`type:name`) |
| `reason` | string           | Human-readable explanation |

Implementations MAY include additional keys in `ctx`. Replay processors MUST
ignore unrecognized `ctx` keys rather than rejecting the event.

---

## 4. The Bus Layer

### 4.1 Contract

The Bus layer delivers Stream events to subscribed buildings and Actors in real
time as events are appended to the Log. The Bus layer does not store events —
storage is the Log layer's responsibility. If a subscriber misses a bus delivery,
it MUST use the Replay layer to catch up.

A conforming Bus implementation MUST:
- Deliver events to all active subscribers matching the event's filter criteria
- Deliver events in Log order for any given object identifier
- Guarantee at-least-once delivery within a session (a subscriber that does not
  acknowledge MAY receive the event again)
- Not guarantee cross-object ordering beyond Log timestamp order

A conforming Bus implementation SHOULD:
- Support filtered subscriptions (by op code, object, prof, proj, tags)
- Deliver events within 500ms of log append under normal operating conditions

### 4.2 Wire Encoding — Preferred: BitPads

The Bus layer wire encoding is implementation-defined. Two options are specified here:
the preferred compact encoding and the plain text fallback. A conforming implementation
MUST support at least one of these two options. A conforming implementation SHOULD
support both and negotiate encoding at connection time.

**Preferred encoding: BitPads compact record format**

BitPads defines binary records from 1 byte (heartbeat/keepalive) to 29 bytes
(full Rich Record with CRC-15 integrity check). For Stream Bus delivery, the
relevant BitPads record types are:

- **Heartbeat (1 byte)**: sent on idle bus connections to confirm liveness.
  A subscriber that receives no heartbeat for 30 seconds SHOULD reconnect.
- **Minimal event record (variable, ~8–12 bytes)**: `op` + `ts` + `object` hash,
  used for high-frequency op codes where ctx is not needed by most subscribers.
- **Full event record (up to 29 bytes)**: all five fields encoded, with CRC-15
  integrity verification on the full record.

BitPads bus encoding provides: lower bandwidth per event, integrity verification
without a higher-level checksum, and wire compatibility with external Works
installations and BitLedger nodes. See `12-bitpads-binding.md` for the complete
encoding specification.

**Fallback encoding: newline-delimited Hollerith text**

A bus subscriber MAY request plain text delivery. In plain text mode, events are
delivered exactly as they appear in the Log: one Hollerith-encoded line per event,
terminated by `\n`, transmitted over a TCP socket or Unix domain socket.

Plain text delivery is appropriate for:
- Development and debugging (events are human-readable in `nc` or `tail`)
- Simple integrations where a full BitPads parser is not warranted
- Intra-machine bus connections where bandwidth is not a constraint

Plain text delivery MUST NOT be used over untrusted networks. A conforming
implementation that offers plain text bus delivery over a network interface
MUST require TLS and authentication.

**Encoding negotiation:**

At connection time, a subscriber sends a `CONNECT` message declaring its preferred
encoding: `CONNECT encoding=bitpads` or `CONNECT encoding=text`. The bus confirms
with `ACCEPT encoding=<chosen>` or `REFUSE encoding=<reason>`. If no `CONNECT`
message is sent within 5 seconds of connection, the bus MUST default to plain text
on local sockets and BitPads on network sockets.

### 4.3 Subscription and Filtering

A subscriber connects to the Bus and optionally declares a filter. Events not
matching the filter are not delivered. Filter dimensions:

| Dimension | Example                   | Matches |
|-----------|---------------------------|---------|
| `op`      | `op=T,F`                  | Events with op T or F |
| `object`  | `object=9f3c2a1b`         | Events for this object (prefix match allowed) |
| `prof`    | `prof=work`               | Events where ctx.prof = "work" |
| `proj`    | `proj=alpha`              | Events where ctx.proj = "alpha" |

Filters are declared at subscribe time and cannot be changed without reconnecting.
An empty filter (no dimensions declared) receives all events.

---

## 5. The Replay Layer

### 5.1 Contract

The Replay layer delivers historical events from the Log to a requesting subscriber.
Replay is the recovery mechanism for the entire compound: any building that loses
internal state MUST be able to reconstruct it by replaying Stream from its last
known good timestamp.

A conforming Replay implementation MUST:
- Accept a `from_ts` parameter (unix timestamp integer)
- Deliver all Log events with `ts >= from_ts` in Log order
- Complete delivery of all historical events before beginning live Bus delivery
  (the catch-up guarantee)
- Accept an optional `to_ts` parameter to bound the replay window

A conforming Replay implementation SHOULD:
- Accept the same filter dimensions as the Bus layer (§4.3)
- Support resumable replay — a subscriber that disconnects mid-replay MAY
  reconnect with an updated `from_ts` to continue without restarting from the beginning

### 5.2 Wire Encoding

Replay delivery uses the same wire encoding options as the Bus layer (§4.2):
BitPads preferred, plain text fallback, with the same encoding negotiation
procedure.

For large replay windows (more than 10,000 events), BitPads encoding is strongly
recommended. Plain text delivery of a large replay window places significant
bandwidth pressure on local sockets and SHOULD be rate-limited by the implementation
to avoid starving live bus delivery.

### 5.3 The Catch-Up Guarantee

When a subscriber requests replay followed by live bus delivery, the Replay layer
MUST provide the catch-up guarantee: the subscriber receives all historical events
up to the current Log head before any new live events are delivered.

The catch-up handoff is signaled by a synthetic S event delivered after the last
historical event and before the first live event:

```
<current_ts> S replay-complete stream.log {"from_ts":<requested_from>,"events_delivered":<count>}
```

This event is NOT written to the Log. It is a protocol signal only, generated by
the Replay layer and delivered on the bus connection. A subscriber receiving this
event knows it is now current and subsequent events are live.

### 5.4 Recovery Pattern

The standard recovery pattern for any building that loses state:

1. Record the timestamp of the last event successfully processed: `last_ts`
2. Request replay from `last_ts` with the relevant op code filter
3. Process all replayed events to reconstruct internal state
4. On receiving the `replay-complete` S signal, resume normal bus subscription
5. Emit an S `sync` event to Stream confirming recovery:
   ```
   <ts> S sync <building-name> {"from_ts":<last_ts>,"events_processed":<count>,"status":"recovered"}
   ```

---

## 6. Conformance Requirements

A conforming Stream implementation MUST satisfy all MUST requirements in this
document. The following table summarizes the primary conformance points:

| Requirement | Section | Level |
|-------------|---------|-------|
| Maintain one append-only Log file per installation | 2.1 | MUST |
| Log stored as UTF-8 plain text, human-readable | 2.1 | MUST |
| Log persists across restarts and upgrades | 2.1 | MUST |
| Every log entry conforms to Hollerith encoding | 2.2 | MUST |
| Log begins with H header event at ts=0 | 2.3 | MUST |
| Log is sortable by field 1 at any point | 2.4 | MUST |
| Batch appends followed by numeric sort | 2.4 | MUST |
| Reset requires explicit confirmation and Office audit entry | 2.5 | MUST |
| Op codes are registered core or extension codes | 3 | MUST |
| Unrecognized op codes treated as S, not rejected | 3 | MUST |
| Unrecognized ctx keys ignored, not rejected | 3.1 | MUST |
| Bus delivers events at-least-once within session | 4.1 | MUST |
| Bus supports BitPads or plain text encoding | 4.2 | MUST |
| Plain text over network requires TLS | 4.2 | MUST |
| Replay delivers all events from from_ts in Log order | 5.1 | MUST |
| Catch-up guarantee honored before live delivery begins | 5.3 | MUST |
| replay-complete signal sent at catch-up handoff | 5.3 | MUST |

---

*Works-Standard 02-stream-protocol — end of document*  
*Next: `docs/01-chunk-format.md`*
