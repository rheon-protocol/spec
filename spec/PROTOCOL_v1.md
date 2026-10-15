# Rheon Protocol — v1

Version: 1
Status: Stable / Frozen

An NDJSON event-stream protocol. v1 defines **syntax**; semantics (generation /
lineage) are deferred to v2. This document is the canonical spec.

## 1. Purpose

Stream meaningful events between tools as NDJSON, with a minimal, stable syntax:

- a uniform event envelope,
- `type` / `event` classification,
- a Signal Registry for per-event payload schemas,
- conservative validation,
- room for v2 (generation / lineage) without breaking v1.

## 2. Envelope (required fields)

One line = one event. Four top-level fields are **required**:

| Field | Type | Description |
|---|---|---|
| `v` | number | Schema version. Always `1` in v1. |
| `source` | string | Origin of the event (tool / process name). |
| `type` | string | Classification of the subject (`file`, `cmd`, `telemetry`, …). |
| `event` | string | What happened to the subject (`created`, `run`, `collected`, …). |

All other fields are **payload**, interpreted via the Signal Registry.

## 2.05 Time — three names, and only two of them are timestamps

v1 deliberately left "when" out of the envelope: a signal arrives *now*, so the
receiver already knows. That holds while the stream is live and stops holding the
moment it is stored, and many producers store it. A producer that stores
needs a time field, and producers left to choose one on their own do not
agree. This section settles the vocabulary. It **adds no field and makes nothing
non-conforming**; renames are staged separately (§2.05.3).

### 2.05.1 The three names

| Name | What it is | Whose clock |
|---|---|---|
| `ts` | **When the event happened**, as recorded by the producer | the producer's |
| `rx_ts` | **When a collecting layer received it** | the collector's |
| `mtime` | **The observed object's last-modification time** | 🚨 **not a clock — data** |

⭐ **Only the first two are timestamps of the record.** `mtime` is a property of
the thing being described, in the same column as `size` and `extension`. It is
listed here because it currently occupies a name (`modified`) that looks like a
record timestamp, which is the whole reason this section exists.

⚠️ **The three do not rhyme, and that is the design.** `ts` and `rx_ts` share a
stem because they are the same kind of thing — one unmarked (the default "when",
needing no word for whose clock) and one marked (the prefix names the clock).
`mtime` deliberately does not join them: making all three uniform would spend the
one visual cue that separates a record's timestamp from a field of its payload,
and this section exists precisely because that line was already crossed once.
Do not "tidy" `mtime` into `m_ts`. Its foreignness is load-bearing, and the
spelling is borrowed from POSIX `st_mtime` so that it carries that struct's
precision about modification-not-creation along with it.

⚠️ `mtime` is **last modification, not creation**. Unix creation time is not
portable, so no producer is expected to read it. The Unix spelling is used precisely
because `modified` reads as "when it was made".

**These line up with prior art**, which is why the names are not invented here:

| Rheon | OpenTelemetry Logs Data Model | CloudEvents |
|---|---|---|
| `ts` | `Timestamp` — "time when the event occurred … at the source" | `time` |
| `rx_ts` | `ObservedTimestamp` — "time when the event was observed by the collection system" | — |

### 2.05.2 Which to write

- A producer describing something that happened **SHOULD** write `ts`.
- A collector that receives records from elsewhere **SHOULD** write `rx_ts`,
  and **MUST NOT** overwrite `ts`.
- A transform **MUST NOT** reuse a name that carried an upstream fact. Setting
  its own emission time into a field the upstream used for the observed object
  destroys that value with no way to recover it.
- Neither is required. A high-volume producer may omit `ts` — a scan emitting
  half a million records in three seconds writes the same value on every line —
  and a collector's `rx_ts` then supplies the floor.

**Format**: RFC 3339 with an explicit offset. A producer **SHOULD** write at
least microseconds — six fractional digits or more,
`2026-10-15T17:35:46.581772+09:00`: fine enough to keep one producer's events
apart, and every platform's clock gives it.
⚠️ Records written before this line vary (seconds, milliseconds, microseconds),
and different producers' clocks are not one clock, so **records from different
producers can tie or cross**; do not order a merged stream on time alone.

### 2.05.3 Reading across producers, while the names are still mixed

A consumer **SHOULD** take the first field present:

```
ts  ??  modified  ??  timestamp
```

⭐ This mirrors OpenTelemetry's own rule ("use `Timestamp` if present, otherwise
`ObservedTimestamp`") — a fallback order written into the spec is the standard
way to migrate a renamed field.

⚠️ **Renaming does not fix history.** Journals already written keep the old
spelling. The fallback order above is
what makes old records readable, and it is not a temporary measure.

Producers that write an emission time use `ts`. Earlier records may carry
`modified` or `timestamp` for the same meaning, which is why the fallback chain is
permanent. Renaming `file:*.modified` (an object's mtime) to `mtime` is deferred: it
would resolve only the created-vs-modified misreading, which §2.05.1 already
addresses in prose.

## 2.06 Paths — one separator

`path` is the field nearly every producer writes and v1 never specified. Like
§2.05 this section **adds no field and makes nothing non-conforming**; it says how
to spell the value so that records from different producers and platforms can be
matched, compared and joined.

- A producer that writes a filesystem path in `path` **SHOULD** separate its
  components with `/` — **on Windows too** (`./sub/a.txt`, `C:/Users/x/a.txt`).
  The consumers of these records match paths with `/`: `grep '/target'`,
  `find -path '*/sub'`, glob patterns, and a person at a POSIX shell, where `\` is
  an escape character. Windows accepts `/` as a separator, so the value still
  opens the file. A path spelled two ways cannot be compared, and a path spelled
  with both (`C:/x\a.txt`) matches neither way.
- A relative path is relative to the producer's working directory, and the part
  the caller named keeps the spelling they gave it, separators aside: asked about
  `sub`, a producer writes `sub/a.txt`; asked about `.`, `./a.txt`. It is not
  rewritten from a project root, nor made absolute or relative on the producer's
  own initiative. A producer whose paths are always absolute (so that a record can
  be used from anywhere) **SHOULD** say so in its documentation.
- A transform passing a record on **SHOULD NOT** rewrite the upstream's `path`:
  it is the upstream's fact, as `ts` is (§2.05.2).
- A consumer **SHOULD** accept `\` as a separator in records from Windows
  producers written before this section.

This is about the `path` field of records. A command whose plain output imitates
another program's prints what that program prints there.

## 2.1 Encoding, and what to do with bytes that do not fit

**A Rheon stream is UTF-8.** This adds no new syntax — it writes down a constraint
v1 already inherited and never stated. §1 calls this "an NDJSON event-stream
protocol", and NDJSON §3.1 says *"All serialized data MUST use the UTF8
encoding"* — unconditionally, which is where the binding requirement comes from.
RFC 8259 §8.1 agrees for anything crossing a boundary (*"JSON text exchanged
between systems that are not part of a closed ecosystem MUST be encoded using
UTF-8"*); a Rheon stream is exactly such an exchange, and its "closed ecosystem"
escape hatch does not apply. RFC 8259 §8.1 also forbids adding a byte order mark,
so a Rheon line **MUST NOT** start with U+FEFF; a reader **MAY** ignore one.

Every producer and consumer is bound by this since v1. Stating it settles the
interesting case.

The interesting case is that **the world a producer observes is not UTF-8.** On
Unix a filename is an arbitrary byte string, and so is a line of command output.
Many registered signals carry a field that can receive those bytes — `path`,
`cmd`, `cwd`, `line`, `dest`. A producer will meet a value it cannot put on
the wire, and needs a rule for that moment. Any signal with such a field is bound
by this section, including signals registered later.

### 2.1.1 Carrying bytes

A JSON string holds Unicode code points, not bytes; there is no escape for an
unpaired byte. So **do not loosen the encoding to carry one.** If a signal must
transport raw bytes, that belongs in the payload, as a sibling field with an
explicit encoding (`<field>_b64`, base64 per RFC 4648) and a registry entry that
says so. No signal does this today. The envelope stays UTF-8 either way.

### 2.1.2 When a value does not fit

A producer that cannot represent a value **MAY** replace the offending bytes
(U+FFFD is the conventional choice), and:

1. **MUST NOT stop the stream.** A byte it could not spell is a fact about one
   value, not a reason to end the run. Later events are unrelated and still true.
2. **MUST NOT be silent.** Say on stderr that a value was mangled and where.
   Once per boundary is enough — the point is that the loss is discoverable, not
   that every instance is announced.
3. **SHOULD NOT** report success in a way that implies completeness. If a
   consumer cannot tell "nothing was lost" from "something was quietly dropped",
   the record is worse than missing.

Rule 1 is here because getting it wrong is cheap and invisible. A reader that
decodes each line into a UTF-8 string by default ends its loop at the first
undecodable line, and then exits 0 with nothing on stderr: a truncated stream that
looks complete.

This is the same shape as §7's rule for unknown signals — *warn and continue,
do not reject* — applied one layer down, to a value rather than a signal.

### 2.1.3 Consumers

A line that is not valid UTF-8 is not valid NDJSON. A consumer **SHOULD** report
it and **SHOULD** continue with the next line, for the same reason as §7.7: one
malformed line says nothing about the rest of the stream.

### 2.1.4 Prior art in the registry

The constraint had already surfaced in one place before it was written here:
`cmd:run.body_preview` is documented as *"First N bytes of the body, cut back to
a UTF-8 boundary."* That is this rule solved locally for one field. It is now
the general case, and new signals should not re-derive it.

## 3. `type`

The subject's classification. Rules:

- lowercase, kebab-case (`a-z0-9-`),
- a **noun**,
- the essential classification only — no action or state.

Examples: `file`, `dir`, `symlink`, `dataset`, `artifact`, `cmd`, `telemetry`.

## 4. `event`

What happened to the subject. Rules:

- lowercase, kebab-case,
- past participle or state name,
- prefer subject-independent, generic words.

Examples: `existing`, `created`, `modified`, `deleted`, `validated`, `run`, `end`, `collected`.

## 5. Examples

```json
{"v":1,"source":"rheon-eye","type":"file","event":"existing","path":"/tmp/a.txt","modified":"2026-02-04T12:00:00Z","size":1234}
{"v":1,"source":"hsh-bash","type":"cmd","event":"end","cmd":"ls -la","exit_code":0,"ms":42,"ts":"2026-06-10T09:00:00+00:00"}
{"v":1,"source":"node-1","type":"telemetry","event":"collected","node_kind":"local","collected_at":"2026-06-11T07:00:00Z","hardware":{}}
```

## 6. Signal Registry

A dictionary mapping `type:event → schema`:

```json
{
  "version": 1,
  "signals": {
    "<type>:<event>": {
      "description": "Human-readable explanation",
      "schema": { "type": "object", "properties": { }, "required": [] }
    }
  }
}
```

The canonical registry of known events ships as `registry/signals.v1.json`.
`schema` is JSON-Schema-compatible. v1 checks only `required` strictly; property
types are checked loosely; unknown fields are allowed.

### 6.1 `plain` — the line a plain command would print (optional)

An entry may carry `plain`: a template for the one line of text that the plain
command this record stands for would print — `grep -rn` for a match, `wc` for a
count, `sha256sum` for a digest. A consumer that turns records back into text
(hsh's `unsig`) reads it from here, so the form lives once, next to the schema,
instead of in every consumer.

```json
"line:matched": {
  "description": "…",
  "schema": { … },
  "plain": "{path}:{line}:{text}"
}
```

The template is substitution only:

- `{field}` is the field's value. A string is written as it is; any other value as
  its JSON text.
- `{field?fallback}` writes `fallback` when the field is `null` or absent. It is a
  default value, not a branch: an empty string is still written empty. (`var:observed`
  uses `{value?<redacted>}`, since its `value` is `null` only when withheld.)
- A field that is `null` or absent writes nothing, and a separator next to it that is
  only spaces or tabs is dropped with it — so `wc`'s template
  `{lines} {words} {bytes} {path}` reads `4 f` for a record that only counted lines.
  A field's own value is never changed (its spaces and indentation stay).
- No arithmetic and no formatting. Where a plain command reports a unit the record
  does not carry, the template names the plain command that matches the record as it
  is (`usage:observed` is `du -B1`, bytes, not `du`'s 1K blocks).
- One record is one line, except a field that is itself text with line endings
  (`file:read`'s `content`, `cat`'s form), which is written as it is.

An entry for a record that wraps another one may carry `plain_of` instead: the name
of the field that holds the wrapped record, whose own `plain` is used
(`priority:scored` → `"plain_of": "payload"`).

`plain` is optional. A record whose entry has none — or that is not registered — is
up to the consumer; `unsig` falls back to `line`, then `path`, then the record as it
is. ⚠️ An unregistered type has no template by construction, which is one more reason
to register a type before emitting it.

## 7. Validation philosophy

**Syntax strict, semantics loose.**

1. Parse JSON.
2. Require `v`, `source`, `type`, `event`.
3. Require `v == 1`.
4. Check `type` / `event` format (lowercase / digits / `-`).
5. Look up `type:event` in the registry.
6. If found, require the schema's `required` fields.
7. Unknown `type:event` → **warn and continue** (do not reject).
8. Unknown extra fields → allowed.

Reference validators (Python / Rust) are in `spec/validation.md`.

## 8. Out of scope for v1

generation, lineage, rollback, SPP steps, semantic dependencies — all deferred to v2.

**Process behaviour is out of scope here, but not undefined.** This spec governs
what is in a record; it says nothing about exit status, a closed downstream, or
what a producer does when its reader goes away — because a producer need not be a
command (a service, or a desktop application's backend). Producers that
*are* Unix commands can opt into `spec/CLI_PROFILE_v1.md`, which states those rules
as observable behaviour.

A program can be both: a shell may write its journal from a library and also ship
commands that put records on stdout. Those commands are commands, and can opt into
the profile like any other.

"SPP" (§8, §9) names the data pattern in which events stream through a pipeline
rather than being stored; it is unrelated to process behaviour.

## 9. v2 roadmap (forward compatibility)

v1 is the *syntax* foundation; v2 adds the *semantic* layer:

- `gen.{id,parent,root,step}` (generation), lineage tracking, rollback,
- generation-aware schemas (`requiresGeneration: true`), SPP integration.

Compatibility: v1 syntax stays valid in v2; v2's added `gen` field is ignored by
v1 tools; the v1 registry is a subset of the v2 registry.

## 10. License

This specification is licensed under the Apache License 2.0 (see `LICENSE`). It is
free to implement and extend; conformance is recommended for interop.
