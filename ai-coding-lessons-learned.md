# AI Coding — Lessons Learned

A running log of things that have tripped Claude up (or me, working with it) and
what we changed in response.

---

## Lesson 1 — Why "just move some folders around" is error-prone for Claude

**Context.** In the `home-projects/jobs` pipeline, job records flow through
lifecycle stages (`daily` → `in-progress` → `applied-to` → `on-hold` →
`declined` → `closed`). Promoting a job *looks* like the simplest possible
task — move a file from one folder to another. Yet Claude kept getting stuck or
getting it subtly wrong, often enough that we stopped doing it by hand and
wrote a script (`jobs/job-searchinator/jobsctl.sh`) to do it instead. This is
the explanation of *why* a "simple" move is actually a bad fit for ad-hoc LLM
execution.

### The move isn't one operation — it's a conditional, multi-step transform

What reads in English as "move job X to in-progress" is actually a small piece
of business logic with branches and an implicit naming convention:

- **Flat-style stages** (`daily`, `closed`) hold bare files:
  `daily/<job>-<YYYY-MM-DD>.md`.
- **Folder-style stages** (`in-progress`, `applied-to`, `declined`, `on-hold`)
  wrap each job in its **own date-stripped subdirectory**, because the cover
  letter and résumé will eventually live beside it:
  `in-progress/<job>/<job>-<YYYY-MM-DD>.md`.
- So moving a bare daily file into a folder-style stage is **not a relocate** —
  it's "create `<job>/` (name stripped of the date), then move the file
  *inside* it (date kept)." But moving a job that's *already* a folder is a
  plain relocate. And moving into a flat stage is a plain relocate too.

That's at least three distinct behaviors selected by the *type* of the source
and the *style* of the destination. None of it is written down at the point of
action; the rule lives only in convention. Every time Claude does this by hand
it has to re-derive the whole decision tree from memory and apply it without
anything checking the result.

### The specific failure modes

1. **No atomic "move" primitive.** Claude effects a move as a sequence of
   independent shell commands (`mkdir`, `git mv`/`mv`, sometimes
   Read→Write→delete). There's no transaction. A multi-job move is N×3
   fallible steps; a partial failure leaves the tree half-migrated and the next
   step reasoning about an inconsistent state.

2. **The date-stripping convention is silent.** The folder is named *without*
   the date; the file inside *keeps* it. It's trivial to name the folder with
   the date still attached, or strip the wrong segment — and the mistake is
   invisible until something downstream (the next move, a status report) breaks
   on it.

3. **State must be discovered, not assumed.** A given job might already be a
   bare file or already a folder, sitting in any of six stages. Claude works
   from a *snapshot* — the git status captured at session start, plus whatever
   is in context — which drifts out of date as the session goes on. Acting on a
   stale mental model produces "not found," or moves the wrong instance, or
   moves a job that was already moved.

4. **No idempotency by default.** If a move *looks* like it failed (or the
   conversation got summarized and Claude lost track of what it already did), a
   naive retry can duplicate the record, nest a folder inside itself
   (`in-progress/<job>/<job>/…`), or clobber an existing destination. Claude's
   memory of "did I already do this?" is weak across a long session — exactly
   the situation where retries happen.

5. **`git mv` vs plain `mv`.** To preserve history you want `git mv`, but only
   for tracked files; untracked files need plain `mv` or git errors out. So
   each file needs a tracked/untracked check first. Get it wrong and you either
   lose the file's git history or the command fails mid-batch.

6. **Hidden side effects.** A move isn't done when the file lands — the
   `jobs/summary/applications-summary.md` has to be reconciled too. Nothing
   about a manual `mv` reminds you of that, so it's routinely forgotten.

7. **Path/name ambiguity.** I refer to jobs by short names; Claude has to
   resolve a fuzzy name to a concrete path, picking the right stage and the
   right date. Another place to guess wrong.

8. **Re-derivation tax + no verification loop.** Because none of the above is
   encoded, *every* move re-runs the same multi-branch reasoning — burning
   tokens and re-introducing the same risks each time — and then Claude
   typically *assumes* success rather than cheaply confirming the end state
   matches intent.

### Why this is structurally a poor fit for an LLM

These operations are **deterministic, rule-governed, multi-step, and carry
implicit invariants** (the folder convention, idempotency, git-awareness,
the summary reconciliation). That is precisely the kind of work a computer
should do the *same way every time* — and precisely the kind of work an LLM
does *slightly differently every time*, because it's re-inferring the rules
from context on each invocation instead of executing fixed code. The task
"feels simple" to a human because the human holds the convention effortlessly;
for Claude, holding and flawlessly re-applying that convention under context
drift is the hard part, and there's no safety net to catch a miss.

### The fix (and the general principle)

Encode the convention **once**, in code, instead of re-deriving it on every
move. `jobsctl.sh` makes the operation:

- **Deterministic** — the flat-vs-folder + date-strip transform is one code
  path, applied identically every time.
- **Self-locating** — `find_source` discovers where a job actually lives right
  now, so Claude doesn't act on a stale snapshot.
- **Idempotent** — a job already at the destination is reported and skipped,
  never duplicated or nested.
- **git-aware** — it uses `git mv` for tracked files and falls back to `mv`
  otherwise, automatically.
- **Self-reporting** — it prints exactly what moved and a reminder to reconcile
  the summary, closing the verification loop.

**General principle:** when a task is a deterministic, multi-step filesystem
shuffle with implicit naming rules, that's a signal to *stop hand-executing it
and write a small script.* The LLM is genuinely good at the thing it should do
here — author the rule once, correctly — and genuinely unreliable at the thing
we were asking it to do — execute that rule by hand, identically, dozens of
times under shifting context. Move the cognition into tested code; let Claude
call the tool.
