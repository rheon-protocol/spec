# Rheon CLI Profile — v1

An **optional conformance profile** for producers and transforms that are Unix
external commands. `PROTOCOL_v1.md` says what a record must contain; this says
how a *process* carrying those records must behave.

Not every Rheon producer is a command. A service, or a library that writes a
journal from inside an application, has no stdout, no exit status, and no
downstream reader — so this is a named profile a command opts into, not a
requirement of v1.

A program can be both. A shell, for example, may write its journal from inside a
library *and* ship commands that put records on stdout. Those commands have an
exit status a consumer reads, so they can claim this profile; the journal cannot.

**Scope.** The profile governs commands a person or a program runs whose stdout
carries records. It does not govern parts that only adapt a stream for its two
ends — a stage a host inserts at a pipeline's head to wrap plain text into records,
or one at its tail to unwrap records back into text. Their only reader is the host
that wrote the other end, and a stage that emits plain text is a sink, not a
producer. Deciding a command is out of scope should be written down rather than
left implicit: a command left out without a decision is a command never measured.

A command claims conformance in its manifest:

```json
{ "id": "rheon-todo", "profile": "rheon-cli/v1" }
```

A command that has no manifest (a shell builtin, say) can make the claim by
measurement instead: run a checker for this profile against its build as part of
its own tests.

## 0. Why the profile exists

The protocol says what a record is. **A producer's behaviour as a process does not
follow from that**, and tools written from the record format alone do not inherit
it: a tool that handles a closed pipe quietly, or that sorts its output, does so
because someone decided it should. This profile writes those decisions down.

## 1. The criterion

Every rule below answers one question:

> **Can a naive consumer — one that does not know this tool — handle the stream
> correctly?**

A consumer is naive in the way `head`, `grep`, `sort`, `wc -l` and a shell `if`
are naive. NDJSON is chosen precisely so those tools work on a Rheon stream
without understanding it: one record is one line, so `head -3` yields three
*valid records*, not a truncated document.

Anything that breaks that is a defect, whatever it looks like locally.

The criterion decides cases where general Unix advice and this profile disagree.
NUL-separated output (`-print0` style) is good general practice for paths — and is
**rejected here**, because it stops `head` and `grep` from working, which is the
whole point. JSON string escaping already solves the problem NUL solves: a
filename containing a newline stays on one line.

## 2. Rules

Each rule is stated as **observable behaviour**, so it can be checked from
outside the process and holds for any implementation language.

### 2.1 A closed downstream is a normal ending

When the reader goes away (`… | head -1`), the command must terminate **without
emitting a panic, traceback, or any other implementation-level text**.

Acceptable: death by `SIGPIPE` (the shell reports `141`), or a quiet exit `0`.
Not acceptable: a stack trace on stderr, or an exit status that a consumer would
read as "this tool failed".

**Both Rust and Python default to violating this.** Each sets `SIGPIPE` to
`SIG_IGN` before `main`, turning the condition into a write error the runtime
then reports loudly. With 200k lines into `| head -1`:

| | exit | stderr |
|---|---:|---:|
| Rust, untouched | 101 | 204 B (panic) |
| Python, untouched | **1** | 271 B (traceback) |
| either, disposition restored | **141** | 0 B |
| Python, `BrokenPipeError` caught | **0** | 0 B |

The Python default is the worse of the two: **exit 1 is indistinguishable from
an ordinary failure**, so a consumer cannot tell "the pipeline ended" from "the
tool broke".

### 2.2 Exit status is a value a consumer reads

- `0` — success.
- `1`–`125` — available for the command's own meanings. `2` for a usage error
  follows GNU practice and is what `clap` and `argparse` already do for free.
- `126`, `127`, `128+N`, `255` — **not available**; the shell and convention own
  them.

The status is 8 bits: `exit 256` becomes `0`. Never pass a computed count.

**An empty result is `0`, not an error.** `grep` returns `1` on no match because
it is a search command; a Rheon tool is a *transform*, and "no records matched" is
a normal result. A naive consumer under `set -e` must not die on an empty stream.

### 2.3 stdout carries records; everything else goes to stderr

One record per line, strict NDJSON, nothing else on stdout — no banners, no
totals, no progress.

Human-facing summaries belong on stderr, and a command that writes one should
offer a way to silence it (`--quiet`). Diagnostics are not free: chaining several
verbs multiplies them.

### 2.4 Say when the output is not the whole answer

If the command narrowed what it emitted, the **output** must say so — not just
the documentation. Two kinds of narrowing exist and both count:

1. **Truncation** — a budget or `--limit` cut the stream short.
2. **Exclusion** — the producer never looked at part of its input.

**`--limit 0` means no limit.** A command that takes `--limit N` must read `0`
that way and say so in `--help`. Taking a range of records (from the 10th to
the 20th) is not this flag's job: a separate filter does it downstream, the way
`head` follows `grep`. A command that reads `0` as "zero results" turns a request
for everything into an empty answer that looks like "no matches".

Documentation does not discharge this. A machine consumer reads the stream, not
the README; and READMEs drift. A total that silently leaves out an excluded part
of the input is carried forward by the reader as the size of the whole.

### 2.5 The same input yields the same output, in the same order

Directory traversal order is whatever the filesystem returned; sort explicitly.
Parallelism must be re-ordered before emission. Sorting for machine output is
byte order, not locale order.

Non-deterministic output cannot be diffed, so it cannot be regression-tested —
"different every time but always correct" means nobody notices when it stops
being correct.

### 2.6 Do not wait for a human

If the command would prompt and stdin is not a terminal, it must **fail with a
message rather than block**. A hang is worse than an error: an error names its
cause, a hang looks like slowness until CI times out.

This does not apply to a transform reading its stdin — waiting for input on an
open pipe is correct filter behaviour, not a prompt.

### 2.7 A write that failed is a failure

The last write happens at exit. Redirecting to a full disk must not produce a
success. Flush explicitly and let the error reach the exit status.

The one exception is 2.1: a closed pipe is not a write failure.

### 2.8 A command that fails leaves nothing on stdout

When a command exits non-zero, stdout must hold nothing a consumer would read as
a result. **Fail before the first record, or finish.**

Once the first record is out the command is committed, because a reader cannot
tell "the stream ended" from "the producer died": a partial stream and a complete
one have the same shape.

The one exception is 2.1. Exit `141` means the *reader* stopped, and a reader
knows what it consumed.

**`set -o pipefail` does not discharge this.** It reports the failure after the
pipeline has run — by which time a sink has already written its file, and a shell
has no way to un-write it. The obligation is inside the tool, while it runs.

A sink that fails mid-render can leave a valid-looking partial result — a header
and a few entries — with a non-zero exit. A sink reads all of its input before it
can start anyway, so building the whole output and writing it once costs nothing
and removes the case.

### 2.9 Say that you finished

A producer that ran to completion **MUST** make its last record a terminator: a
record whose `event` is `complete`.

- The terminator is **positional** — only the last record is one. A `complete`
  in the middle of a stream is ordinary data.
- It **SHOULD** carry `count`: how many records preceded it.
- Its `type` stays the producer's own. Recognition is on `event` and position, so
  a producer that already ends with a summary (`{"type":"eye-stats",
  "event":"complete",…}`) is conformant as it stands.

A filter **MUST NOT** carry a terminator through as data — it must not sort,
score or reshape it — and **MUST** emit its own **only if it saw one**.
That is what makes the absence travel: a truncated upstream produces no
terminator, so the filter emits none, and every stage downstream inherits the gap.

**A filter whose job is to cut** (`head`-like: it stops reading once it has
emitted enough) never sees its upstream's terminator when it cuts, so by the rule
above it emits none. It **MUST** instead end with a record of its own `type` whose
`event` is `truncated`, carrying `count` (records emitted) and `limit` (the cut),
and **SHOULD** say so in one line on stderr (§2.4).

- `truncated` is **not a terminator** — a consumer that requires completion stops,
  and every filter downstream emits no terminator of its own. That is the point:
  the cut is a narrowing (§2.4), and the absence travels exactly as above.
- **Like the terminator, it is positional.** Only a `truncated` that is the
  last record is the mark; one in the middle is ordinary data. A filter **MUST
  NOT** treat a trailing `truncated` as data — it must not keep, count, sort or
  reshape it — and emits no `complete` after it. A `tail`-like filter that kept
  the mark would return it as "the last record", and a counting filter would count
  it. The mark survives only where the cutting filter is the last stage — which is
  where a person or an agent reads it.
- When such a filter did **not** cut (its input ended within the limit), it is an
  ordinary filter: it ends with `complete` if and only if upstream did.
- Ending a cut stream with `complete` (with a `truncated: true` field, say) is
  the shape this section exists to prevent: a consumer that requires completion
  would take part of the stream for all of it.

A consumer **MAY** require one, and requiring it is opt-in — a stream can
legitimately come from `cat ledger.jsonl`, which has no producer to speak for it.

**This rule adds a line to every producer's output.** A consumer that counts
lines instead of reading records will see one more.

**The need is not symmetric.** A producer that emits a single aggregated record
cannot be partial; a *streaming* producer, whose output is fed to something that
accumulates, can. The rule exists for the accumulating side: a table rendered from
four records of fourteen looks exactly as complete as one rendered from all
fourteen, unless the stream says which it is.

### 2.10 Say what you could not read

The rules above are about what a command writes. This one is about what it
reads: a command that reads Rheon records from its input must not treat a line it
could not read as if the line were not there.

| Input | Behaviour |
|---|---|
| Empty | Nothing on stderr, exit `0` — an empty stream is a normal result (2.2) |
| Some lines unreadable | One line on stderr saying how many, **exit `0`**; keep going |
| Input present, **no** line readable | One line on stderr saying how many, **exit `1`** |

- "Unreadable" means the line is not a record this command can use: not JSON,
  or JSON without what the command needs. **Why** it was unreadable is the
  command's own business; the rule asks for the count, not a classification.
- **The two cases are not a compromise; 2.8 decides them.** A command that read
  nothing usable has emitted nothing, so failing leaves no false stream behind.
  A command that read some lines may already have written records, and after the
  first record it is committed — failing then would be the partial stream 2.8
  forbids.
- `1` is free for this: 2.2 keeps an empty result at `0`, so `1` is not "no
  match" here as it is for `grep`.
- **Scope.** The rule applies to input the command reads as Rheon records. A
  command built to read another format — a unified diff, CSV — is outside it for
  that input; holding it to NDJSON would break a correct tool.

**The silent version looks exactly like an empty result.** A search command fed
plain text that returns exit `0` and nothing at all is indistinguishable from
"searched, found nothing".

## 3. What this profile does not say

**How to satisfy any of it.** The rules are observable behaviour so that one
checker can test a Rust binary and a Python script identically — every verb ×
the same checks, rather than every verb × an implementation review. Per-language
recipes belong with the implementations, not here, which is why this document
names no language.

It also says nothing about **what a command is for**, its flags, or its schema.
The one exception is `--limit`. 2.4 and 2.9 already name it, and its `0` has to
mean the same thing across a family of commands (2.4).
`--help` and `--version` are ordinary Unix expectations, not Rheon ones; general
CLI design guidance is out of scope.

## 4. Checking conformance

All ten rules are black-box. The manifest already carries what a checker needs
in order to invoke each verb — `category` says whether to feed it stdin or a
path, and `input`/`output` say what to feed it.

This document does not fix that vocabulary. `category` is optional and free to
omit (`PIPE_MANIFEST_v1.md` §1), and leaving it free is itself the decision: a
profile that named the values would make every family that spells them
differently non-conforming. A family defines its own values (for example `source`
/ `filter` / `sink`, plus a separate `side_effects` for "running it changes
something outside"), and a checker that wants to invoke a verb reads the family's
definition, not this one.

```
2.1  cmd … > <pipe, read end closed> → exit 141 or 0, and stderr free of panics/tracebacks
     cmd … | head -1              → the same (a second path; races, see below)
2.2  cmd <no matches>             → exit 0
     cmd --nonsense-flag          → exit 2
2.3  cmd … | wc -l                → every line parses as JSON
2.4  compare the count emitted against the population, if the tool narrowed
2.5  run twice, diff              → identical
2.6  cmd < /dev/null              → returns; does not hang
2.7  cmd > /dev/full              → non-zero
2.8  cmd <an input it rejects>    → non-zero **and** stdout empty
2.9  cmd … | tail -1              → the last record's `event` is `complete`
                                     (a cutting filter that cut: its own `truncated`)
2.10 yes 'not rheon' | head -40 | cmd
                                  → exit 1, and one line on stderr
```

**2.8 is the one check a generic checker cannot write.** 2.1–2.7 are provoked
by the invocation; 2.8 needs an input the tool refuses, which is per-tool. Where
no failing input is known, the honest report is *not measured* — not a pass.

**2.10 needs one fact the invocation does not carry: whether the command reads
Rheon records.** The check itself is generic (feed plain text, read the status and
stderr), but a command built to read a diff or CSV is out of scope, and nothing
in the manifest says which kind a command is. Until a family declares it, a
checker can only run 2.10 on commands it has been told read Rheon records.

**The deterministic form of 2.1: hand the command a pipe whose read end is
already closed.** §2.1's condition is "when the reader goes away", and
`… | head -1` is one instance of it, not the definition. With the read end closed
before the command starts, **its first write fails** (`EPIPE`, or death by
`SIGPIPE`) every time — whatever the output size, whatever the start-up speed.
This changes how 2.1 is measured, not what it requires.

- A command that writes nothing (0 bytes on stdout) cannot meet the condition at
  all; the honest report is *not measured*, not a pass.
- Run without a shell, death by `SIGPIPE` comes back as a negative status
  (`-13`); normalise it to `128 + signum` (= `141`) so it reads as the shell
  reports it.
- Under the pipe forms the same command can return `0` on one run and `141` on
  the next; on a closed pipe it returns the same status every time.

**The pipe forms race, and the size that matters is in bytes.** Under
`| head -1` or `| head -0` the check only fires if the reader is gone **before
the writer has finished writing into the pipe buffer** — and which comes first is
up to the scheduler.

- **The buffer is measured in bytes, not lines.** 2.1 cannot fire under a pipe
  form while the whole output fits in the pipe's capacity: 64 KiB by default on
  Linux, adjustable with `fcntl(F_SETPIPE_SZ)` and readable with
  `fcntl(F_GETPIPE_SZ)`; macOS starts smaller and grows. A fixture sized in lines
  can fit in the buffer and pass without ever exercising `SIGPIPE`.
- **`head -0` is not "before the first write".** It narrows the race; it does not
  close it. An interpreted command can look reliable under it only because the
  interpreter starts slowly — a result that is stable for that reason is the more
  misleading one.

**Keep `head -1` alongside the closed pipe.** "Could never write" and "wrote,
then failed" are different paths through a program, and only the second is
reached by a reader that leaves after the first line. The verdict rests on the
deterministic form; the pipe forms add paths, not certainty.

**Emission time is not a determinism failure.** When comparing two runs for
2.5 — or an old implementation against its replacement — mask the field carrying
"when this ran" first. The current spelling is `ts` (PROTOCOL_v1 §2.05), but
stored records may carry the earlier spellings `modified` or `timestamp` for the
same idea, and journals are append-only — so a comparison that reaches back over
stored records **must mask all three**. Masking only the current spelling reports
differences that are not there.
