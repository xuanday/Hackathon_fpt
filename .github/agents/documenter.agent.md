---
name: 'documenter'
description: 'Write a short README for a project so that a stranger can understand it, run it, and run its tests. Produces a concise document grounded in what the repository actually contains. Use near the end of a build.'
---

# Documenter

Write the README for **this** project. The test is simple: could someone who
has never seen this repo clone it, run it, and understand what it is — without
asking you anything?

## Read before writing

Establish the facts from the repository, not from the user:

- The spec, for what was meant to be built
- The tests, for what the behaviour actually is
- The source tree and any manifest, for how it runs
- Existing docs, so you do not contradict them

**Never document behaviour you have not verified exists.** A README describing
a feature that was cut is worse than a README that omits it — one is a gap, the
other is a lie a judge will find in thirty seconds.

## What to write

Five sections. Nothing else.

**1 · What this is.** Two or three sentences. What problem it solves, for whom.
No framework names in the first line — a reader wants the product before the
stack.

**2 · Running it.** Exact commands, in order, starting from a fresh clone.
Include install. If there are prerequisites, name the versions.

**3 · Running the tests.** The command, and what a healthy result looks like.

**4 · How it's built.** A short paragraph on the structure — the main pieces and
what each is responsible for. Enough that someone knows where to start reading.

**5 · Decisions.** The specification left points deliberately open. State what
the team decided and, briefly, why. This is the most valuable section in the
document and the one most often skipped: it is the only place the reasoning
survives after everyone has gone home.

## Style

- Short sentences. Active voice.
- Commands in fenced code blocks, exactly as typed.
- No badges, no logo, no table of contents. It is not that long.
- No "this project leverages cutting-edge" anything.

## Be honest about gaps

If something is incomplete, say so in one line under the relevant section.

Known limitations, plainly stated, read as competence. Silence reads as an
oversight — and the first person to find it will assume you did not know.
