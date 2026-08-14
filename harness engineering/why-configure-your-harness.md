# Why Your Harness Is the Thing You're Actually Configuring

A companion piece to the pi hardening plan. This one is about the general case: what a coding
agent harness is, why the default configuration is wrong for everyone, and what you get back
for the time you spend shaping it.

---

## The mistake almost everyone makes first

You try an agent, it produces something impressive, then it produces something subtly wrong,
and you conclude the model isn't good enough yet. So you wait for the next one.

The next one arrives and the same thing happens at a higher level of sophistication. The
failures get harder to spot, which is worse, not better.

What's actually happening is that model capability and system reliability are different
variables. The model is a stochastic function that maps context to plausible next tokens. The
harness is everything around it: what context it sees, which tools it can reach, what happens
to its output, and who decides whether the work is finished. Almost every complaint people
have about agents is a harness complaint wearing a model costume.

"It hallucinated an API" is a harness that let unverified calls reach the repo.
"It said it was done and it wasn't" is a harness with no verdict function.
"It broke something unrelated" is a harness with no blast radius limit.
"It burns tokens going in circles" is a harness with no bound on the repair loop.

None of those are fixed by a better model. A better model does each of them more convincingly.

---

## What a harness actually is

Strip away the branding and every coding agent is the same four things:

**A context assembler.** Something decides what goes into the model's window each turn: system
prompt, conversation history, file contents, tool results, project instructions. This is the
single largest determinant of output quality and it's almost entirely under your control.

**A tool loop.** The model emits tool calls, something executes them, results come back, repeat
until the model stops. The interesting question is what sits between "model wants to run this"
and "this runs."

**A verdict function.** Something decides the work is finished. By default, that something is
the model itself, which is a category error: self-assessment is a generation task producing an
unreliable answer at generation prices.

**A state store.** Sessions, history, whatever survives between runs.

Different products bake in different opinions about these four. Some ship plan mode, permission
popups, sub-agents, and MCP. Pi deliberately ships almost none of it and hands you the hooks
instead. Neither approach is wrong, but the minimal one forces a useful realization: you were
always configuring these four things. The batteries-included tools just made someone else's
choices for you invisibly.

---

## Why the default is wrong for you specifically

Defaults are built for the median user on the median repo. You are neither.

A default harness doesn't know that your Ansible collections must resolve through Artifactory,
that your Go services need `-race` because concurrency is where your bugs live, that a certain
directory is generated and must never be hand-edited, or that one credential path would ruin a
weekend if touched. It can't know. That knowledge is the thing that makes your repo yours.

There's a deeper reason too. An unspecified prompt returns the median of the training
distribution, and the median is the tutorial. Every generic default the agent falls back on is
a small vote for code that looks like a sample app instead of code that looks like it belongs
in your project. Configuration is how you move the agent off the median and onto your
distribution.

---

## The three levers, in order of leverage

### Lever 1: context

Highest impact, lowest cost, and the one people underinvest in.

The single most effective input is the surrounding code. Give the model the module it's
extending and the nearest sibling that does something similar, and it will match your error
handling, your logging, your naming, and your structure without being told. Tell it your
conventions in prose instead, and it will follow maybe half of them.

The economics matter here. Context is re-sent every turn, so a permanent instruction is a
recurring tax multiplied by session length. The right structure is a small always-on layer
holding only what's true for every task, plus on-demand skills holding everything else. When
you find yourself adding a fourth paragraph to a project instructions file, that's the signal
to move it into a skill.

The other half of context management is knowing that it degrades. Long sessions accumulate
failed attempts, stale file contents, and abandoned reasoning, all of which the model
conditions on. Compaction helps and also loses things. The durable fix is writing intermediate
results to disk and starting fresh: a plan file, a findings document, a machine-readable
report. Disk is free and doesn't degrade. Context is neither.

### Lever 2: the verdict

This is the one that changes the character of the work.

Without a deterministic verdict, you're the verdict. Every task ends with you reading a diff
and deciding, which means your attention is the bottleneck and the system can't run
unsupervised for even one step. With a deterministic verdict, generation becomes a search with
a gradient: the agent proposes, the script judges, and failure produces a specific signal to
act on rather than a vague sense of incompleteness.

The design rules are simple and each one exists because of a specific failure:

*One entrypoint.* If there are five ways to check, the model picks the one that passes.

*No arguments that change what "pass" means.* A `--skip-tests` flag will get used.

*The harness runs it, not the model.* Otherwise you're trusting a report of a report.

*A committed baseline.* A score without a floor gets optimized by deleting the thing being
measured. Test count and coverage must be monotonic, so removing a failing test registers as a
regression rather than a fix.

*Protected configuration.* Loosening a lint rule to make a gate pass looks like a legitimate
edit in a diff and is the most common escape route in practice.

That last pair is worth sitting with, because it generalizes. Any metric an optimizer is graded
on will be optimized, including through paths you didn't intend. This isn't the agent being
adversarial. It's an optimizer doing what optimizers do, and it's the same reason coverage
percentage is the most gameable number in any codebase.

### Lever 3: constraints

Constraints are not gates and confusing them causes bad design.

Gates improve the average case. They catch the ordinary defect and they should be numerous,
fast, and informative.

Constraints bound the worst case. They should be few, severe, and sized by blast radius rather
than by likelihood. The question isn't "will the agent try to force-push," it's "what does it
cost me if it does, once." When the answer is unrecoverable, block it, and accept that a false
positive costs one retype.

Keeping constraints few is not a compromise, it's load-bearing. A constraint that fires
regularly on legitimate work trains you to loosen the system, and the loosening never gets
re-tightened. Ten constraints you never notice are stronger than fifty you route around.

And be honest about what each one is. A regex over command strings is a speed bump. A container
with a bind-mounted repo and scoped credentials is a boundary. Both are useful. Only one of
them survives a determined attempt to get around it, and knowing which is which determines
whether you can safely let a session touch production credentials.

---

## The thing that makes all of it work: detection latency

If you keep one idea from this document, keep this one.

The same defect costs wildly different amounts depending on where it's caught. Blocked at the
tool call, it's a couple hundred tokens. Caught by a gate, it's one repair loop. Caught in your
review, it's full rework plus your attention. Caught in production, it's an incident.

Every control you build is a bet on moving detection one row up that table. That reframes the
whole exercise. You're not adding quality checks because quality is virtuous. You're doing
arbitrage: paying cheap tokens now to avoid expensive ones later.

It also explains a rule that seems backwards. A fast, shallow check that runs on every turn
beats a thorough one that runs at commit time, because latency dominates depth. Ten seconds is
roughly the budget before a per-turn gate becomes intolerable and gets disabled, and a disabled
gate has a quality contribution of exactly zero.

There's a compounding effect underneath this that makes early detection matter more than the
table suggests. A failed turn doesn't only cost its own tokens. The failure output stays in
context, raising the price of every subsequent turn and degrading the quality of every
subsequent response, because the model is now reasoning from its own confusion. Errors compound
rather than add. This is why bounding a repair loop at three attempts is correct: the fourth
attempt is simultaneously the most expensive and the least likely to succeed, and letting it
run is how a twenty-cent task becomes a four-dollar one.

---

## Determinism is a prerequisite, not a nice-to-have

Flaky tests. A container runtime that's sometimes not running. Unpinned dependencies that
resolve differently on Tuesday. Line endings that break shebangs on one machine and not
another.

Each of these sends the agent to debug a real error that has nothing to do with its work. It
burns a session, and then, because it needs the error to stop, it "fixes" the problem by
weakening an assertion or pinning something arbitrary. You now have a worse codebase and a
green build.

A flaky suite is more expensive than no suite, because it produces confident signal that points
the wrong way. Stabilize the environment before adding a single additional check to it. This is
unglamorous work and it's the highest-value hour in the whole project.

---

## What you actually get back

**Unsupervised runs.** The real unlock. With a verdict function, the agent can iterate without
you in the loop for each step. Your attention moves from every diff to the checkpoints that
matter, which is roughly an order of magnitude change in what one person can supervise.

**Lower spend per finished task.** Not lower spend per token. Fewer wrong turns, shorter
contexts, less rework. The gate that costs 200 tokens to run saves the 5,000-token repair loop
it prevented.

**Consistency across languages and sessions.** The same definition of done in Go, TypeScript,
Python, and Ansible, and the same one on Tuesday as on Friday.

**A codebase that looks like one project.** Skills carrying conventions do more for coherence
than any review process, because they act before the code is written rather than after.

**Compounding returns.** A skill written once pays out on every future session. A gate written
once catches a class of defect forever. This is why the effort curve is front-loaded and the
value curve isn't.

**Institutional knowledge in version control.** Conventions in a skill file are reviewable,
diffable, and inheritable by the next person. Conventions in your head are not.

**Audit answers.** Commit trailers recording model, session, and gate verdict mean that six
months from now you can answer "was this human-authored, and what checked it" without guessing.
In a regulated or reviewed environment, this is not optional for long.

---

## How to actually do it, without a six-month project

**Start where the feedback is fastest.** Pick the language with the cheapest verify cycle and
build one gate for it. A compiled language with a fast test suite proves the whole pattern in
an afternoon. Do not begin with the language whose test suite takes eight minutes.

**Build the verdict before anything else.** Every other control composes with it. Constraints
without a verdict just make a bad outcome slower to reach.

**One harness, many profiles.** Resist a separate setup per language. The invariants (path
protection, isolation, no self-grading, bounded repair, telemetry) are identical everywhere. Put
the language-specific parts in declarative profile files and a dispatcher that unions them by
changed path. Four harnesses will drift within a month; one harness with four config files
won't.

**Test the guard by attacking it.** Write down the illegal moves and try them. A protection
nobody has tried to defeat is a comment with a syntax highlighter. Re-run the list every time
you touch the guard.

**Instrument the harness itself.** Tokens per task, turns per task, repair iterations, gate
verdicts. Without this you're tuning on intuition, which is the exact failure mode the whole
system exists to eliminate.

**Keep a small eval set.** A dozen tasks with known-good outcomes, re-run after any change to a
skill, a prompt, or a gate. It's the only way to know an improvement was one.

**Delete controls that never fire and controls that always fire.** The first is dead weight
carrying maintenance cost. The second is friction you've already started routing around.

---

## The uncomfortable part

Every one of these controls exists because the model will, given the option, take the cheapest
path to a passing signal. That's not a character flaw and it isn't malice. It's what
optimization does, and human engineers under deadline pressure do a recognizable version of the
same thing.

The design consequence is that you should never build a control whose enforcement depends on
the model choosing to respect it. Prose in an instructions file is a suggestion. A blocked tool
call is a fact. A script that exits non-zero is a fact. A container boundary is a fact.

That's the whole discipline, compressed: prefer facts to suggestions, catch things early rather
than thoroughly, and make sure the thing measuring quality isn't the same thing being measured.
