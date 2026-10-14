# Rheon Protocol

*rh protocol, or rhp, for short.*

A small, neutral contract for streaming meaningful events as **NDJSON** between tools.
Producers and consumers depend on the protocol, not on each other: a shell's command
journal, a file scanner, a sensor and a dashboard can all speak it without importing one
another.

```json
{"v":1,"source":"rheon-eye","type":"file","event":"existing","path":"src/main.rs","size":1234}
{"v":1,"source":"hsh-bash","type":"cmd","event":"end","cmd":"ls -la","exit_code":0,"ms":42,"ts":"2026-06-10T09:00:00+00:00"}
```

Four required fields — `v`, `source`, `type`, `event` — and everything else is payload,
interpreted through the **Signal Registry**. *Syntax strict, semantics loose.*

The stream is UTF-8 (inherited from NDJSON). The world a producer watches is not — a Unix
file name is arbitrary bytes — so the specification says what to do with values that cannot
be spelled: a producer may mangle them, but **must not stop the stream and must not do it
silently**.

## Contents

| Path | What |
|---|---|
| [`spec/PROTOCOL_v1.md`](spec/PROTOCOL_v1.md) | The v1 specification: envelope, encoding, timestamps, the registry |
| [`spec/CLI_PROFILE_v1.md`](spec/CLI_PROFILE_v1.md) | Optional profile for producers that are Unix commands: exit codes, closed pipes, `--limit`, unreadable input, the end-of-stream record |
| [`spec/PIPE_MANIFEST_v1.md`](spec/PIPE_MANIFEST_v1.md) | How a tool declares itself (name, verbs, side effects) so that package managers and shells can install and run it |
| [`spec/ACTOR_v1.md`](spec/ACTOR_v1.md) | Where the *who acted* fields get their values: observation first, one declared name (`AUDIT_ACTOR`), no vendor tables |
| [`spec/validation.md`](spec/validation.md) | Reference validators |
| [`registry/signals.v1.json`](registry/signals.v1.json) | The Signal Registry — known `type:event` pairs and their payload schemas |
| [`schema/rheon-envelope.v1.schema.json`](schema/rheon-envelope.v1.schema.json) | JSON Schema for the required envelope |

## Implementations

- **rheon** and the `rheon-*` tools — a package manager and a family of command-line tools
  (scanner, grep, reader, `head` / `tail` / `sort` / `count` for records, …) that read and
  write Rheon records.
- **hsh** — a shell whose command journal is a Rheon stream.

## Versioning

`v` is the envelope version. Additions to the registry and to payloads are backward
compatible; a reader ignores fields it does not know and reports (rather than rejects) an
unknown `type:event`.

## License

[Apache-2.0](LICENSE).
