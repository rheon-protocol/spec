# Rheon Actor Identity — v1

A **policy** for producers that record *who acted*: `actor`, `actor_kind`,
`actor_src`, `claimed_actor`, `claimed_actor_src`, and the observed fields beside
them (`uid`, `auid`, `login`, `from`, `peer_uid`, `peer_pid`). The Signal Registry
(`registry/signals.v1.json`) says what those fields are; this document says **where
their values should come from**.

It is followed by `hsh` (a shell whose command journal is a Rheon stream) and
`berthd` (a daemon that owns terminal sessions and records who attaches to them).

## 0. Why this exists

Self-declared identity only works when somebody configures it, and in practice
nobody does. People do not set a variable to say who they are, and a server
installed with the minimum set-up ships with nothing set. A design whose useful
answer depends on configuration therefore gives the wrong answer **by default**:
an agent's commands filed under the person whose account it ran in.

So the default must come from what can be **observed**. Declaration stays, but as
a knob for the people who want it — administrators naming a bot, a service or a CI
job so they can aggregate by it.

**The principle: fill the journal with values that can be had without
configuration.** A value that exists only after someone edits every machine is,
in practice, absent — an organisation with hundreds of machines will not edit them
all, nor notice when one was missed.

## 1. Three layers

| Question | Source | Configuration |
|---|---|---|
| **Which tool** acted, on this machine | **Observation**: the names of the ancestor processes of the acting (or attaching) process | None |
| **Who came in** over ssh | **Observation**: the key sshd authenticated — sshd already logs it by default; the journal records the key to join on, a collector resolves it (§2.7) | None on the servers (the collector needs to read sshd's log) |
| **A name to aggregate by** (a bot, a service, a CI job) | **Declaration**: `AUDIT_ACTOR` in the environment | An administrator sets it once in that environment |

Each layer answers a question the others cannot. Observation cannot tell two
agents apart that run as the same user in the same way; a declaration cannot be
trusted and is absent whenever nobody set it; a key says nothing about processes on
the local machine.

## 2. Rules

### 2.1 Observation first, and never overwritten by a declaration

Record a declaration **beside** the observation, not instead of it. Agreement is
dull; disagreement is the interesting case, and it is only visible when both are on
the same record.

**And the converse: never fill the declaration field from an observation.** The
temptation is obvious — most records carry no declaration, and the ancestry sitting
next to them is never empty — but a name moved into `actor` stops saying how it was
obtained, and the two layers exist because they answer different questions.

It also gives the wrong answer. In practice the immediate parent of an acting
process is a shell or a terminal (`bash`, `WindowsTerminal.exe`, `pwsh.exe`), almost
never the agent that started the work; the agent, when it appears at all, is further
up the chain. Deriving an actor from the parent therefore names the shell,
confidently, in the one column a reader trusts to be a name somebody chose.

Reaching the agent further up the chain would mean recognising agent names — the
vendor table §2.2 forbids. A producer may still decide to pay that price, but it is
that decision, not this one.

So: an agent is found by **reading the ancestry**, not by reading `actor`. A reader
that wants "which tool attached" queries the chain; `actor` answers "what name did
somebody choose to aggregate by", and stays empty when nobody chose one (§2.3).

### 2.2 No vendor tables

A producer must not keep a list of agent names in order to classify what it sees.
A list is a race the producer loses every time a new tool ships, and until it
catches up the ledger files the unknown tool as a person. Record **raw names** and
leave classification to the reader, who can update a query without changing any
ledger.

Self-declarations that agents set for themselves (`AI_AGENT`, and
`CLAUDE_CODE_ENTRYPOINT` as the one vendor-specific fallback admitted) are still
accepted — as **declarations**, with their own `*_src` value.

### 2.3 Empty means nobody declared

An absent declaration means *nobody claimed a name*. It does not mean a person was
there, and a producer must not fill a declaration field from `USER` / `LOGNAME`:
the OS name is already recorded as an observation, and copying it into the claim
column adds no information while making an unclaimed action look claimed.

A shell's `actor` may fall back to `USER` / `LOGNAME`, labelled `actor_src: user` /
`logname`. That is an OS-given name reported as such, not a claim, and is allowed.
On Windows neither variable exists; the OS gives the name in `USERNAME`, which is
read in `USER`'s place and labelled the same way (`actor_src: user` — the label says
what kind of name it is, not which variable spelled it). It is not read on unix,
where `USERNAME` is not something the OS sets. Without this, every record a person
makes from cmd, PowerShell or an ssh session on Windows carries `actor: unknown`.

The observation it stands beside has a different spelling per OS: `uid` / `auid` on
unix; on Windows, where there is no uid, `user_sid` (the process token's SID) and
`login` (`GetUserNameW`), with `identity_src: token_user` naming the mechanism. Both
are asked of the OS, not read from the environment.

### 2.4 Process ancestry

- **Read it at the moment of the action or attachment.** Processes exit and pids
  are reused, so a later read can describe a different process.
- **Names only** — `comm`, or the basename of `argv[0]`; for an interpreter
  (`node`, `python`, …) the basename of `argv[1]`. **Never later arguments**: they
  can carry secrets.
- The executable's file name can be a version string rather than a name (a tool
  installed as `…/versions/2.1.268` still has `comm` set to its own name). Prefer
  `comm` / `argv[0]`.
- On Linux, `comm` may contain spaces and parentheses (`tmux: server`). Parse
  `/proc/<pid>/stat` **after the last `)`**, or the parent pid is read from the
  wrong column.
- Stop at a small depth (8) or at pid 1. **Do not stop at the first `sshd`**: under
  OpenSSH's privilege separation, the pid sshd writes in its log is the `[priv]`
  process one level *above* the session's own `sshd` (§2.7), so the chain has to
  walk through them to be joinable. Both are named `sshd`; only the pid tells them
  apart.
- **Say why the walk stopped, and never guess the reason.** A field that records
  where the chain came from also records why it ended: reached the end, cut at the
  depth limit, the next parent was **gone**, the next parent was **refused**, or the
  read failed for some other reason. **A permission failure is not a
  disappearance.** The kernels answer with the reason — `ESRCH` when the process is
  not there, `EPERM` when it is there and unreadable — so collapsing them is the
  reader's error, and it puts a false statement in an append-only ledger. Suggested
  spelling, as a suffix on the mechanism: `:depth_limit`, `:vanished` (gone),
  `:denied` (refused), `:unreadable` (failed, reason unknown, including a line that
  was read but could not be parsed).
- **On Windows, take one snapshot, and expect two values to behave differently.**
  `CreateToolhelp32Snapshot` returns the whole process table at once, so a walk
  built from a single snapshot is one moment rather than a sequence of questions a
  parent can exit in the middle of. Two consequences a reader has to know, because
  they are not defects:
  - **`:denied` has no producer** — the snapshot hands over names and parent pids
    for other users' processes without opening them, so its absence means nothing
    was asked, not that nothing refused.
  - **`:vanished` is the ordinary ending** — Windows does not reparent to pid 1, so
    walking up reaches a parent that has exited and is simply absent from the table.
    Do not read it as an anomaly, and do not require reaching the root the way a
    unix walk can.
- **On Windows, check creation times, or pid reuse walks the chain into a
  stranger.** An exited parent leaves its pid in the child's record, and pids are
  recycled quickly (a burst of process creation can reuse one within seconds). A
  fabricated chain ends in `toolhelp` or `toolhelp:vanished` exactly like an honest
  one, so the check has to be made during the walk: a parent **younger than the
  child** that claims it is a recycled pid. Drop it and end the walk as `:vanished`
  — the real parent is gone, which is why the pid came free, so no new value is
  needed. When the creation time cannot be read, continue unchanged: there is no
  evidence of a lie, and "could not verify" must not become "was a lie" (which is
  also why `:denied` still has no producer here).
- The name keeps its extension (`claude.exe`) — it is what was observed, and the
  producer does not reshape observations.
- **Joining names: the reader strips a trailing `.exe`, ignoring case.** The same
  tool shows up as `claude` on one machine and `claude.exe` on another, so anything
  that counts tools has to bring the two spellings together. The split is not by
  OS: the name comes from the **file**, and how a file is named is the distributor's
  choice (a binary shipped as `bin/claude.exe` reports `claude.exe` on macOS too).
  Windows merely guarantees the suffix; it does not own it. The rule belongs to the
  reader, not the producer — a stripped name is a string no API returned — and the
  cost of that is that every reader has to know it, which is why it is written here.
  - **Ignore case**, or the rule quietly misses half of them: Windows file names are
    case-insensitive and the snapshot returns the spelling on disk, and system
    directories hold both `PING.EXE` and `where.exe`.
  - **Stripping can collide.** On a typical Windows machine almost every process
    name ends in `.exe`, the few that do not are kernel-side (`System`, `Registry`),
    and stripping produces no collisions — but nothing forbids a real `foo` next to a
    `foo.exe`. A reader that cannot accept a collision should join on the unstripped
    name as well.
- **On macOS, ask with the flavor that can read other users' processes.**
  `proc_pidinfo(PROC_PIDTBSDINFO)` refuses a process owned by root with `EPERM`, and
  a terminal starts its shell through `login(1)`, which is root — so the chain stops
  one step below the terminal, exactly where the useful name is.
  `PROC_PIDT_SHORTBSDINFO` carries the same two fields a walk needs (`pbsi_ppid`,
  `pbsi_comm`) and is readable by an ordinary user; the chain then reaches `login` →
  the terminal → `launchd`. Both flavors report failure as a **short return, not
  `-1`**, so test the returned length and clear `errno` before the call.
  `sysctl(KERN_PROC_PID)` can also read it, but it returns success with a zero
  length for a pid that does not exist — the failure never reaches the return value.
- Limits, accepted: the chain ends at `sshd` for anything that came over ssh
  (layer 2 answers that); a shell running under a session daemon such as `berthd`
  has the daemon as its parent, so the attaching client's chain is recorded by the
  daemon in its attach record and joined to the shell's records by the session id
  (`berthd_session`).

### 2.5 Environment variables are read by name, never by prefix

Pick each variable explicitly. A prefix such as `CLAUDE_*` also matches secrets
(Claude Code exports a messaging token next to its session id).

### 2.6 `AUDIT_ACTOR`

- A neutral name, read by every producer that follows this policy. **It is the only
  variable an administrator needs to know.**
- Resolution order for the claimed name:
  `AUDIT_ACTOR` → tool self-declaration (`AI_AGENT` → `CLAUDE_CODE_ENTRYPOINT`) → none.
- A reader may still meet `hsh_actor` as an `actor_src` value. It came from an older,
  product-specific variable that no producer reads any more; the ledger is
  append-only, so the value survives in old records. Retiring an input is not the
  same as retiring the value it wrote.
- **There is no `AUDIT_ACTOR_KIND`.** A category is a fact about the name — the
  same on every record that carries it — so an administrator who names a bot
  `deploy-bot` has already said what it is, and a reader groups by name. A second
  variable would only be one more thing to set on every machine.
  - A producer may keep an `actor_kind` field, filled **only from values that need
    no configuration**: `agent` from an agent's self-declaration, `ci` from `CI`.
  - **`service` is reachable, but not from the environment.** An embedder sets it
    per session in code: a service knows it is a service, so it declares that rather
    than asking an administrator to set a variable on every machine.
- Read from the **process environment** the producer receives: a shell reads it
  once, when it starts (an `export` inside a running shell does not change that
  shell's own records); a client such as `berth attach` reads it when it runs, so an
  `export` in the shell that launches it does apply.
- An administrator can bind a name to an ssh key on the server side:
  `environment="AUDIT_ACTOR=deploy-bot"` in `authorized_keys` (requires
  `PermitUserEnvironment`). It is still a declaration in form, but the server's
  administrator — not the client — decides it.

### 2.7 Zero configuration first — and the ssh key without touching sshd

A field whose value needs per-machine configuration is a knob, not a default
(§0). Before adding one, look for where the value already exists.

The ssh key is the worked example. sshd at its default `LogLevel INFO` already
writes, for every public-key login (OpenSSH 9.6):

```
sshd[259809]: Accepted publickey for alice from 192.0.2.6 port 61906 ssh2: ED25519 SHA256:…
```

That line holds the sshd pid, the client address **and port**, and the key's
fingerprint. From inside the same session, the ancestry reads

```
sh  ←  sshd: alice@notty  ←  sshd: alice [priv] (pid P)  ←  sshd [listener]
SSH_CONNECTION=<client> <port> <server> 22
```

and the log line for that login carries `sshd[P]` and the same `<port>`. sshd hands
every session `SSH_CONNECTION` (client address, client port, server address, server
port) without any setting. So:

- the **producer** records the join key — the client address and port from
  `SSH_CONNECTION`, and/or the pid of the `sshd` process found in the ancestry
  (§2.4) — and nothing else; it does not read the log;
- a **collector** that can read sshd's log (on Linux `auth.log` needs the `adm`
  group; macOS uses the unified log, Windows OpenSSH the event log) joins the two
  and resolves the fingerprint;
- which key belongs to whom is **one table per organisation**, not a setting per
  machine.

Telling people from agents this way still needs them to use **different keys** — a
cost per user or tool on the client side, not per server.
`ExposeAuthInfo` puts the same fact into the session directly, but only on servers
where someone enabled it — which is exactly the kind of value §0 says will be
missing.

## 3. Not decided yet

- **ssh key observation** — the join-key field (client port and/or sshd pid), and
  which collector resolves it. The log join of §2.7 is the zero-configuration
  candidate. The `ExposeAuthInfo` route has a producer, and the registry carries
  `auth_src` / `auth`: it costs nothing where nobody enabled `ExposeAuthInfo` (the
  session records `auth_src: unset`, itself an observation in the sense of §2.3), but
  it is still the route that needs every server edited. Which route a collector
  should rely on, and whether the log join makes the other redundant, is open.
- **Renaming mid-session.** Under this policy a declaration is set once in an
  environment, which points to "not supported".

## 4. Status

- `AUDIT_ACTOR` is read by both producers (`actor_src: audit_actor`), and the
  ancestry has its own fields (`ancestry`, `ancestry_src`). Both entered the registry
  in the same round as their first producer, as every closed enum here does.
- `actor` is a declaration-only column (§2.1). The producers record the ancestry in
  its own field.
- Ancestry has a producer on every desktop OS — Linux, macOS and Windows
  (`toolhelp` is registered as the Windows mechanism, with `:denied` marked as having
  no producer there). iOS and Android record `unimplemented:`.
