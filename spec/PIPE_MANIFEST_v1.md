# Rheon Pipe Manifest — v1

A **pipe** is an external command that speaks the Rheon Protocol. A **manifest** is the declaration
that travels with it: it says what the command is called, where its body lives, and
what dot-verbs — if any — it offers.

`PROTOCOL_v1.md` says what a record must contain. `CLI_PROFILE_v1.md` says how a process
carrying those records must behave. **This says what the declaration beside that process
must contain, and where the two of them live on a machine.**

> ⚠️ **Why this file exists.** The manifest format has more than one reader — a package
> manager that fetches and places manifests, and a shell that turns them into dot-verbs — and
> they live in different code bases. Neither can see the other's parser, so no guard inside one
> of them can keep the two in agreement. ⭐ A written contract plus a shared corpus is what
> keeps them aligned.

---

## 0. Two shelves, and why they are two

| | what it is | where it lives |
|---|---|---|
| **the binary** | a program that emits NDJSON. **It works without any of this** | `~/.local/bin` |
| **the manifest** | the contract that builds `rheye(path).head(5)`. **Meaningless outside Rheon** | `~/.flux/manifests` |

⭐ **This is why a package manager exists at all: to carry the declaration.** Someone who
only wants the command drops the binary on `PATH` by hand and needs no manifest — which is
also why a POSIX shell can use these tools while having no package manager of its own.

🚨 **`~/.local/bin` is a shared shelf.** Other programs live there. ⭐ **Being on the shelf
is not evidence of ownership.** A reader lists, updates, or removes **only what it holds a
declaration for**, and never enumerates the directory. A tool that scans the shelf will
offer to delete its neighbours.

⚠️ **Nothing prepends this shelf to `PATH`.** It was chosen *because* it is usually there
already; manufacturing reachability the environment did not grant breaks `command -v` parity.

⚠️ The manifest shelf is **not** hidden (`.manifests`) and **not** under a shell's directory.
Manifests are not one program's private business: more than one reader consumes them, so the
shelf belongs to none of them.

### 0.1 Carrying an existing box to the new address

Every implementation **carries the old box once**, by these rules:

| state | action |
|---|---|
| old present, new absent | **move it, and say so** |
| **both present** | 🚨 **do not touch** — a person has been here, and this is not the tool's call |
| old absent | nothing |

⭐ **Every implementation, not one of them.** It is tempting to make the migration one
implementation's job and let the others assume it ran, but the reader who runs is not
predictable: someone may install the package manager and never run the shell, or run the
shell and never install the manager. Whoever gets there first should be able to fix it.

⚠️ That makes the move racy by construction, so it must be written to survive losing:
between the check and the rename, another implementation may have done it. **Losing that
race is not an error and has nothing to report** — the outcome is the one you wanted.
🚨 But a failure for **any other reason** (permissions, a full disk) **must be said out
loud.** Folding "someone else did it" and "I could not do it" into the same silence is how
the second one stays invisible forever.

⭐ Announce on **stderr**, not stdout: a migration is an aside, and stdout is where results
go. Announce **once per process**.

⚠️ Do not run the migration for a command that only asks a question (`--version`, `--help`).
Moving files as a side effect of being asked what version you are is a surprise.

---

## 1. Required fields

| field | meaning |
|---|---|
| `id` | the name in a registry. **The filename of the manifest is this** |
| `name` | human-facing name. 🚨 **Keep it equal to `id`** — see below |
| `bin_name` | **what you type.** Defaults to `id` when absent |
| `description` | one line — 🚨 see below |
| `verbs` | array — **may be empty, but must be present** (see §3) |

### 🚨 `description` is one line; the explanation goes in `notes`

A listing (`rheon list`, a registry search) shows one row per pipe, and `description` is that
row. ⭐ Keep it to **one line of at most 80 characters** — the shape of Debian's synopsis or
Homebrew's `desc`. Everything a reader needs before using the tool (options, edge cases,
platforms, history) belongs in **`notes`**, which `rheon show` prints in full.

⚠️ Descriptions tend to grow one sentence at a time, each worth saying, until no listing can
show them — so the limit is worth enforcing with a checker. ⭐ A reader still cuts a longer one
to one line rather than let a single foreign manifest break its table.

### 🚨 `id` and `name` must be the same string

A registry search matches on `id`, `name`, `description` and tags; **installing resolves
`id` alone**. ⭐ If the two differ, a search prints a name that install cannot accept —
the user types back what they were just shown and it fails.

⚠️ Requiring `name` to be *present* is not enough; its value must be checked. A corpus whose
pipes all happen to agree is safe by accident, not by rule.

📌 This is also why a CLI should spell the argument **`<NAME>`** rather than `<ID>` —
users read `ID` as a number, and `id` is the one field they never see.

`source` is required **in a source repository** (a repo-relative path to the body, e.g.
`bin/rheon-eye`). ⚠️ It is **absent from a published manifest**, where the address is derived
from `id` and the platform instead — a repo-relative path means nothing once distributed.

`runtime` declares what the body needs: `binary` (self-contained) or a runtime's name
(e.g. `python`). ⭐ **A distributor may refuse anything that is not self-contained** — a
tool that installs tools cannot itself require an interpreter the machine may not have.

Optional and free to omit: `version`, `author`, `license`, `tags`, `usage`, `category`,
`notes`, `min_hsh_version`, `platforms`, `side_effects`, `secret_flags`, `replaces`, `$schema`.

### `side_effects` — what running the pipe changes

An array of words naming what changes when the pipe runs, e.g. `["fs:cwd"]` (files appear under
the working directory) or `["fs:home", "net"]` (the user's home is rewritten, the network is used).
A pipe that only reads says so with an **empty array**: `"side_effects": []`.

🚨 **A missing field means "does not say", not "changes nothing".** A runner that sends a pipe's
output somewhere else (an MCP tool) must run only pipes that declare `[]`, and must refuse both a
non-empty declaration and a missing one. Otherwise a tool that writes — a package manager, say —
whose declaration simply lacks the field is accepted as a reader, and its writing commands can be
run through such a runner. ⭐ The field is optional for a reader of manifests, but a pipe that wants
to be run by such a runner must write it — the same rule as `verbs` (write `[]`, do not omit).

### `replaces` — the names this pipe takes over when it is renamed

An array of earlier ids, e.g. `["flux-eye"]` on `rheon-eye`. A distributor installing this pipe
where a declaration of an earlier id is present takes over that declaration's names (its binary
name and verbs) as its own, places this pipe, and then removes the earlier declaration. A catalog
carries the field so that an updater can tell an installed earlier id has a successor.

⭐ To keep the old names working for a while, list them as verbs as well; dropping them from the
verbs later removes them from the shelf.

### `secret_flags` — the flags that make a pipe print secrets

An array of the pipe's own flags that print secret values, e.g. `["--reveal"]` for a pipe that
hides the values of variables whose names say they are secrets unless asked.

⭐ A runner that sends a pipe's output somewhere else — an MCP tool, whose output goes to a model —
must refuse these flags however they are spelled (quoted, split by quotes, `--flag=value`, a
one-letter flag bundled with others, or a glob or brace that would expand to the flag). The
author of the pipe is the one who knows which flags are dangerous, so the knowledge lives in the
declaration, next to `side_effects`.

---

## 2. 🚨 Unknown fields are carried, not corrected

A reader **must ignore fields it does not know**, and a tool that moves a manifest **must
move the bytes**, not a re-serialised struct.

⭐ The reason is concrete: published manifests still carry `kind` and `output`, which were
retired from the format. A tool that parsed into a struct and wrote it back would **silently
drop them**, and the loss would be invisible until some other reader wanted them. Round-trip
through a type you control is a way of quietly editing other people's data.

---

## 3. Verbs

```json
"verbs": [
  { "verb": "rheye", "template": "rheon-eye {path} --detail",
    "args": [ { "name": "path", "type": "path" } ] }
]
```

- `verb` — the dot-chain name. **Must be unique across every manifest on a machine**: two
  pipes claiming one verb makes it impossible to say which runs.
- `template` — the command line, with `{argname}` placeholders. **Must begin with `bin_name`.**
- `args[].type` — one of `str`, `path`, `int`. **This decides quoting** (§4).
- Several entries may share a `verb` (overloads). They are distinguished by **the sequence of
  argument types**, not by how many there are.

⚠️ **An empty `verbs` array is a legitimate, finished shape** — a pipe can be a plain command
with no dot-verb (the package manager `rheon` itself is one). ⭐ To keep "deliberately none" distinguishable from
"not written yet", a manifest with `"verbs": []` **must carry `notes` saying why**.

---

## 4. Rendering a verb (for readers that build command lines)

Substitute each `{name}` with its value, quoted **by declared type**:

| type | quoting |
|---|---|
| `str` | POSIX single quotes; an embedded `'` becomes `'\''` |
| `path` | a **leading** `~` or `~/` is replaced with **the home directory's value**, and the result is then quoted as `str`. A `~` elsewhere is a filename and is left alone |
| `int` | **bare — not quoted.** A quoted number does not reach arithmetic |

🚨 **For `path`, substitute the value, not the string `$HOME`** — and do it **before** quoting.
The rendered line is wrapped in single quotes, and a shell does not expand anything inside
them, so `'$HOME/x'` arrives at the command verbatim and the path does not exist.

- Supplying the right number of values is the **caller's** job (overload resolution happens
  before rendering).
- ⚠️ A placeholder with no value is **left in place**. Removing it hides the omission.

📌 A tool that only *moves* manifests does not implement this section. ⭐ Not implementing it
is preferable: a rule that exists once cannot drift.

---

## 5. Placement

**Write the declaration first, then the body.**

⭐ If the run dies in between, what remains is a manifest with no binary — which a reader can
name (`missing`) and offer to repair. The reverse leaves an **unattributable file on a shared
shelf**: nobody can say who put it there or whether it is safe to remove.

🚨 **Do not resolve symlinks when placing.** Write to a temporary name in the destination
directory and `rename` onto the final path. `rename` replaces **the path**, so a developer's
link is replaced rather than written through.

> ⚠️ This is not a detail. A shell's atomic-write helper typically *does* follow a terminal
> symlink — correct for config files, where following the link is the whole point of a
> dotfiles repo. Borrowed for package installation, it writes **through a developer's link
> into their source tree**. ⭐ The two operations look identical and differ in exactly this.

⭐ Replacing a link is legitimate; doing it **silently** is not. Say what was replaced and how
to get it back.

**Removal is the reverse order** — body, then aliases, then declaration — so an interrupted
removal also lands on "declaration survives", the state that can be named.

---

## 6. Distribution

A registry entry may declare `platforms` (e.g. `["linux-x86_64", "macos-aarch64"]`).

⭐ **The catalog declares the address shape, so clients need no branch:**

| `platforms` | address |
|---|---|
| absent | `<id>/<bin_name>` |
| present | `<id>/<platform>/<bin_name>` (`.exe` appended on Windows) |

The platform word is **`<os>-<arch>`**, deliberately **not** a target triple: a statically
linked binary makes the libc half meaningless as an address.

⚠️ **A machine whose face was never built must be told so by name**, listing what does exist.
"That pipe does not exist" and "your platform was not built" must not arrive as the same 404.

### 6.1 🚨 The published version is **derived from the body, never written by hand**

A catalog entry carries `build_ids` — a map from platform to the build identifier embedded in
that face's binary:

```json
{ "id": "rheon-eye", "platforms": ["linux-x86_64", "macos-aarch64"],
  "build_ids": { "linux-x86_64": "a95ceb4b9", "macos-aarch64": "5f08e6510" } }
```

⭐ **It is the git identity of the source the body was built from**, and it exists because
`bin/` is outside git: the moment a binary is placed, its origin vanishes from the record.

⚠️ **It is not the manifest's `version` field.** That field is optional, and a declaration and
its body can disagree about it — a manifest may omit the version its binary announces. A version
a human maintains is a second copy of a fact, and the second copy is the one that goes stale.

🚨 **A distributor must derive `build_ids` from the artifacts it is about to publish**, in the
same pass that builds the catalog. Accepting the field from a submitter's manifest reintroduces
exactly the drift this replaces.

⚠️ **Per platform, not per pipe.** The faces of one pipe legitimately differ — one may be
rebuilt alone for a platform-specific reason. ⭐ There is therefore **no single "version of
this pipe"**, and an implementation that reduces the map to one value will report a machine as
up to date against a face it never checked.

⚠️ A face whose version cannot be read is **omitted, not blanked**. "Has no version" and "has
an empty version" are different facts, and a reader must be able to say the first one out loud.

### 6.1.1 🚨 The declaration is a **second axis**, and it moves on its own

A catalog entry must also carry a fingerprint of the published manifest bytes.

⭐ **Manifest-only releases are ordinary, not exceptional.** Adding `category` or `usage`, or
renaming a short name, changes no binary at all. A reader comparing only the body's version says
"up to date" while holding a declaration that is missing fields it would display.

⭐ **The two axes must stay separate.** "The body is current but the declaration is not" is a
real, common state; a reader that folds the axes into one answer cannot say it. And a reader
that reports a row as current on one axis while counting it as unmeasured on the other is
lying in one of the two places — **derive the mark, the tally, and the repair decision from a
single verdict.**

⚠️ **Fingerprint the bytes; do not canonicalise first.** The question is "is the same thing on
this machine", not "does it mean the same". A canonicalisation rule lives in every
implementation independently, and the day two of them disagree, identical declarations get
different fingerprints and every machine is told, permanently, that its declaration is stale —
🚨 **and reinstalling will not fix it.**

⭐ This only works if §5's placement rule is honoured: **a reader that re-serialises the
manifest cannot be compared by fingerprint.** Writing a parsed value back reorders keys; no
data is lost, so nothing looks wrong, and only the fingerprint makes it visible.

🚨 **When the fingerprint is computed in more than one language, hold every implementation to
published test vectors, not only to each other.** Agreement between two implementations proves
nothing if they are wrong in the same way — an error in a hash constant can leave the low bits
matching, so only a check against known values catches it.

⚠️ A pipe installed from a source repository must be **excluded from this comparison too** —
its declaration is the developer's, and differing from the published one is the normal case.
⭐ Comparing it usefully means comparing against **that repository**, which the manifest does
not record — so the repository must be named by the caller rather than guessed.

### 6.2 ⚠️ Comparing is three answers, not two

A reader that answers "is what I have current?" must distinguish **up to date**, **stale**, and
**could not tell** — and, for the third, **say why**. 🚨 Folding "could not tell" into "up to
date" recreates the failure the comparison exists to prevent, now with a machine's endorsement
behind it. At minimum these are separate: *installed from a source repo* (its version is not
meant to match), *absent from the registry*, *no published version for this face*, and *the
local body would not say*.

⭐ Whatever announces this must also report the size of each group, so that **"all current"
and "nothing was compared" cannot arrive as the same sentence.**

⚠️ **Do not read a process's exit status as "it answered."** A pipe that does not implement
`--version` may ignore its arguments, read stdin instead, and exit `0` with no version in its
output. ⭐ The answer must be recognised **by shape** (a semver
immediately before a parenthesised identifier), and anything that asks a body for its version
must close stdin, or a body that reads stdin will hang.

⚠️ **Publishing order is the opposite of installing order: body first, then manifest.** An
existing user picks up a new manifest and immediately invokes it; if the body is still the old
one, the new verb fails. (Installing is per-user and atomic-ish; publishing is a race against
everyone already installed.)

---

## 7. Conformance

Neither reader can see the other, so both are held to the same corpus instead:

- **The manifests of the reference tool family are the reference corpus.** A reader is conformant
  if, over that corpus, it agrees on: the `bin_name` of each manifest, the ordered de-duplicated
  list of verb names, and (for readers that render) the rendered command line.
- 🚨 **A conformance run over an empty corpus is a failure, not a pass** — "read nothing" and
  "everything agreed" must not produce the same result.
- ⚠️ The corpus is a live directory in a working repo, not a frozen fixture. That is deliberate:
  a frozen fixture stops containing whatever the format grew last month.

Known implementations: the package manager `rheon` (moves manifests and compares versions per
§6.1–6.2; does **not** implement §4), and the `hsh` shell (renders dot-verbs; implements §4).
