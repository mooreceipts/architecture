# Operator's Guide

Third of three. The plan says what to build. The companion piece says why harness configuration
matters. This one is about what *you* do differently once it exists.

The harness changes your job. You stop being the person who checks the work and become the
person who specifies it and owns the boundaries. Almost everything below follows from that.

---

## 1. State the acceptance criterion before the task

The most common way to waste a session is to describe what to build without describing what
correct means. The model then optimizes for plausible completion, because that's the only
target available to it.

The shift is from this:

> Add retry logic to the S3 upload path.

to this:

> Add retry to the S3 upload path. Done means: `scripts/fitness/check` passes, there's a test
> that fails without the retry, transient 503s retry with backoff and 4xx do not, and it
> follows the retry pattern already in `internal/artifactory/client.go`.

Forty extra seconds of typing, and it removes the single most expensive failure mode in the
system.

If you can't state the acceptance criterion, that's useful information rather than a reason to
proceed. The task isn't ready. You should be having a design conversation, not an
implementation one.

### The task template

Worth internalizing until it's automatic:

1. **What.** The change, in one sentence.
2. **Done means.** The verifiable condition. Gate passes, plus whatever the gate can't see.
3. **Look like.** The sibling file or module it should resemble.
4. **Don't touch.** Anything adjacent that's out of scope.

---

## 2. Point at code, not at conventions

Your instinct will be to describe your standards in prose. Prose gets followed roughly half the
time. A pointer to the nearest file that already does the thing gets matched almost every time,
including details you would never have thought to write down.

So build the reflex: before starting, name the file it should look like.

> Same shape as `roles/artifactory_proxy`.
> Mirror the error handling in `cmd/sync/main.go`.
> Follow the tool registration in `.pi/extensions/verify.ts`.

When you catch yourself explaining the same convention a third time, stop and write a skill.
Repeated explanation is a specification bug you're paying for on every session.

---

## 3. Scope to one verify cycle

Rework scales worse than linearly with diff size, because a large diff means late detection and
an entangled fix. The right unit of work is one thing that can be verified and committed on its
own.

The test: if you can't describe done in one sentence, it's two tasks. Split first, and hand
over the second half only after the first is committed.

This also gives you a clean intervention point. A session that went wrong on a small task is
discarded with a worktree delete. A session that went wrong on a large task is a negotiation.

---

## 4. Kill sessions early, without sentiment

You will develop an instinct to let a struggling session continue because it seems almost
there. Resist it.

The repair loop is bounded at three for a reason. Attempt four is simultaneously the most
expensive and the least likely to work, because a session that has accumulated failures is
reasoning from its own confusion. Failed output stays in context, raising the price and
lowering the quality of everything after it.

**The rule:** two failed repair loops on the same problem means stop, delete the worktree, and
restart with a better specification. Not more instruction to the same session. A fresh one,
informed by what you just learned.

Sunk cost in a context window is genuinely sunk. Continuing preserves nothing.

---

## 5. Review the diff, not the explanation

The model's summary of what it did is a generated artifact, produced by the same process that
produced the code, and it reads as convincing whether or not it's accurate.

Read the diff. If the diff is too large to read, that's the scoping problem from section 3.

### What to look for, because the gates can't

* **Invented APIs.** Any call you don't recognize, check against real docs. This is the failure
  class that reaches production.
* **Tutorial shape.** Code that would fit any project rather than this one. Correct, passing,
  and still wrong.
* **Over-engineering.** An abstraction for a single caller. A config option nobody asked for. An
  interface with one implementation.
* **Scope creep.** Files changed that the task didn't call for.

### What to stop looking at

Formatting and lint-level issues. Whether the tests pass. The gates own these now. If you're
still checking them by hand, the gate isn't doing its job, and the fix is the gate.

---

## 6. Treat the harness as a product

This is the behavior that compounds, and the one most people skip.

When something goes wrong, the useful question isn't how to fix the code. It's **what would
have caught this, and at which rung of the enforcement ladder.** Then build that. Every
incident is a free specification for a control.

Give it a rhythm or it won't happen. A monthly fifteen minutes:

* Read the token-per-task and turns-per-task numbers. Is the trend right?
* Delete controls that never fire. Dead weight with maintenance cost.
* Fix controls that fire on legitimate work. Friction you've already started routing around.
* Re-run the eval set. Did the last month of changes help or hurt?

**Corollary, and it matters:** fix the harness immediately when it's wrong, but never during a
task. Working around a bad gate to ship something is exactly how the gate dies. Note it, finish
the task, fix it after.

---

## 7. Own the boundary decisions personally

A short list of judgments that don't delegate:

* Anything touching real infrastructure credentials.
* Any change to `.pi/**`, `scripts/fitness/**`, `.githooks/**`, or `.fitness/baseline.json`.
* Any new pi extension. It's arbitrary code about to run with your privileges.
* Any third-party pi package. Never installed from inside an agent session, always read first.
* Every commit. There is no auto-commit, by design.

The list stays short on purpose. If you find yourself in the approval loop constantly, the
constraints are miscalibrated. Loosen them deliberately rather than drifting into
rubber-stamping, which is the same outcome with none of the thought.

---

## 8. Don't argue with the model

When the agent is wrong, the productive move is almost never to explain why at greater length.
It's to change the input: point at a different file, restate the acceptance criterion, or
restart clean. Extended back-and-forth fills context with failed attempts, which raises cost and
lowers quality for every turn after.

Same principle, different register: don't ask it to double-check its work. Self-review is a
generation task, priced like generation, returning an unreliable answer. Run the script.

---

## 9. Session hygiene

* **One task, one session, one worktree.** Sessions that span tasks accumulate irrelevant
  context that you pay for on every turn.
* **Start fresh rather than compacting.** Compaction is lossy, and a session that needed it has
  already degraded. Write findings to a file, start clean, read the file.
* **Write intermediate state to disk.** Plans, findings, API notes, reports. Disk is free and
  doesn't degrade. Context is neither.
* **Different sessions for different modes.** Investigation with a read-only role. Implementation
  with edits. Harness changes in a session started explicitly for that.

---

## 10. Anti-patterns to watch for in yourself

**Prompting around a missing gate.** You keep adding "and make sure you run the tests" to every
prompt. Build the gate.

**Approving without reading.** The volume got high, the diffs looked fine, and now you're
signing off reflexively. Reduce scope per task until reading is tractable again.

**Loosening under deadline.** The gate is in the way and the deadline is real. This is the exact
moment the system breaks, because the loosening never gets reversed. Delete the control on
purpose, with a reason, or leave it alone.

**Treating agent output as free.** It isn't, and the cost isn't the tokens. It's the review
attention and the rework, both of which are yours.

**Hoarding conventions in your head.** If it isn't in a skill, it doesn't apply to the agent, to
your teammates, or to you in six months.

---

## The mindset shift

You're moving from *reviewer* to *specifier and systems owner*. The reviewer role scales
linearly with output and caps hard at your attention. The specifier role scales with the quality
of your specifications and controls, and that's a thing that improves over time.

The uncomfortable part is that this front-loads effort into work that feels like overhead.
Writing the acceptance criterion, naming the sibling file, building the gate. All of it feels
like a delay before the real work starts.

It is the real work. Generation was never the bottleneck.
