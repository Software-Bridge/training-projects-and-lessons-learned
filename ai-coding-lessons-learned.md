# AI Coding — Lessons Learned

A running log of things that trip up AI coding agents (like Claude) and what to
do about them.

---

## Lesson 1 — Why "just move/rename a bunch of files" is error-prone for an AI agent

**Context.** Reorganizing files and folders — promoting records through a
pipeline of directories, renaming a batch to a new convention, restructuring a
project tree — *looks* like the simplest possible task. Move a file from here to
there. Yet AI agents repeatedly get stuck or get it subtly wrong on exactly
these jobs. This explains *why* a "simple" bulk move is a poor fit for ad-hoc
LLM execution, and what to do instead.

Two running examples used throughout:

- **Example A — staged pipeline.** A directory tree where items advance through
  stages, and some stages store an item as a bare file
  (`incoming/<item>-<date>.ext`) while others wrap each item in its own
  subfolder (`active/<item>/<item>-<date>.ext`) because related files will join
  it later.
- **Example B — bulk rename.** Renaming hundreds of assets from
  `IMG_1234.jpg` to `<event>-<seq>.jpg`, where the sequence resets per event
  and a sidecar `.json` must be renamed in lockstep with each image.

### The move isn't one operation — it's a conditional, multi-step transform

What reads in English as "move item X to the next stage" or "rename these to the
new scheme" is actually a small piece of business logic with branches and an
implicit naming convention:

- In Example A, advancing an item is *not* a uniform relocate. Moving a bare
  file into a folder-style stage means "create `<item>/` (named *without* the
  date), then move the file *inside* it (date *kept*)." But an item that is
  already a folder just relocates as-is, and moving into another flat stage is a
  plain relocate too. That's three distinct behaviors selected by the *type* of
  the source and the *style* of the destination.
- In Example B, the new name depends on state the filename doesn't carry (which
  event? what's the next sequence number for *that* event?), and every image
  drags a sidecar that must move with it.

None of this logic is written down at the point of action; it lives only in
convention. Each time the agent does it by hand, it re-derives the whole
decision tree from memory and applies it with nothing checking the result.

### The specific failure modes

1. **No atomic "move" primitive.** An agent effects a move as a sequence of
   independent shell commands (`mkdir`, `mv`/`git mv`, sometimes
   read→write→delete). There's no transaction. A batch is N× fallible steps; a
   partial failure leaves the tree half-migrated and the next step reasoning
   about an inconsistent state.

2. **Silent naming conventions.** In Example A the folder is named *without* the
   date while the file inside *keeps* it; in Example B the sequence resets per
   event. It's trivial to attach the date to the folder, strip the wrong
   segment, or continue a global counter — and the mistake is invisible until
   something downstream breaks on it.

3. **State must be discovered, not assumed.** An item might already be a bare
   file or already a folder, in any of several stages; the "next sequence
   number" depends on what's already been renamed. The agent works from a
   *snapshot* — directory listings captured earlier, plus whatever is in
   context — which drifts out of date as the session goes on. Acting on a stale
   model yields "not found," moves the wrong instance, or re-processes
   something already done.

4. **No idempotency by default.** If a step *looks* like it failed (or the
   conversation got summarized and the agent lost track of what it already
   did), a naive retry can duplicate an item, nest a folder inside itself
   (`active/<item>/<item>/…`), or clobber an existing destination. An agent's
   memory of "did I already do this?" is weak across a long session — exactly
   when retries happen.

5. **Tool-specific moves (`git mv` vs plain `mv`).** To preserve history you
   want `git mv`, but only for tracked files; untracked files need plain `mv`
   or git errors out. Each path needs a tracked/untracked check first. Get it
   wrong and you either lose history or the command fails mid-batch.

6. **Hidden side effects.** The move often isn't done when the file lands —
   an index, manifest, or summary that references the old location has to be
   reconciled too (the sidecar in Example B, a catalog file in Example A).
   Nothing about a manual `mv` reminds you of that, so it's routinely forgotten.

7. **Name/path ambiguity.** Humans refer to items by short names; the agent has
   to resolve a fuzzy name to a concrete path, picking the right stage and the
   right date/variant. Another place to guess wrong.

8. **Re-derivation tax + no verification loop.** Because none of this is
   encoded, *every* run re-executes the same multi-branch reasoning — burning
   tokens and re-introducing the same risks — and then the agent typically
   *assumes* success rather than cheaply confirming the end state matches
   intent.

### Why this is structurally a poor fit for an LLM

These operations are **deterministic, rule-governed, multi-step, and carry
implicit invariants** (the naming convention, idempotency, tool-awareness,
the downstream reconciliation). That is precisely the kind of work a computer
should do the *same way every time* — and precisely the kind of work an LLM
does *slightly differently every time*, because it re-infers the rules from
context on each invocation instead of executing fixed code. The task "feels
simple" to a human because the human holds the convention effortlessly; for an
agent, holding and flawlessly re-applying that convention under context drift is
the hard part, and there's no safety net to catch a miss.

### The fix (and the general principle)

Encode the convention **once**, in a small script, instead of re-deriving it on
every run. A good migration/rename script is:

- **Deterministic** — the branch logic (flat-vs-folder, per-event sequencing,
  date-stripping) is one code path, applied identically every time.
- **Self-locating** — it discovers where an item actually lives *now*, so the
  agent doesn't act on a stale snapshot.
- **Idempotent** — an item already at the destination is reported and skipped,
  never duplicated or nested; safe to re-run after a partial failure.
- **Tool-aware** — it uses `git mv` for tracked files and falls back to `mv`
  otherwise, automatically.
- **Self-reporting** — it prints exactly what changed and any follow-up needed
  (e.g. "reconcile the index"), closing the verification loop.

**General principle:** when a task is a deterministic, multi-step filesystem
shuffle with implicit naming rules, that's a signal to *stop hand-executing it
and write a small script.* An LLM is genuinely good at the thing it should do
here — author the rule once, correctly — and genuinely unreliable at the thing
we keep asking it to do — execute that rule by hand, identically, dozens of
times under shifting context. Move the cognition into tested code; let the agent
call the tool.
