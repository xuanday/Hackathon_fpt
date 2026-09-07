---
name: architecture-slide
description: Generate a single, projector-legible architecture slide for a hackathon demo. Reads the repository and produces ARCHITECTURE.md containing one Mermaid diagram and four short sections. Use when a team has been nominated as a finalist and needs to present their build to a large audience in a few minutes.
---

# Architecture Slide

Produce **one slide**. Not a document, not a report, not a blueprint.

The audience is ~300 people watching a projector in three cities over a video
bridge. If it is not legible from the back of the room, it has failed.

## Before you write anything

Read the repository to establish the facts. Do not ask the user questions you
can answer yourself:

1. `README.md` — what the app claims to do
2. The test files — these are the truest description of intended behaviour
3. The source tree — actual modules and their real dependencies

For section 3 you also need how the team used Copilot. That is not in the repo,
so ask for it in **one** batched question, then write. Never guess at
architecture or at process and present either as observed fact.

## Output

Write `ARCHITECTURE.md` at the repository root, with exactly these four
sections and nothing else.

### 1. What it does

One sentence. Plain language, no jargon, no framework names. A person who has
never seen the repo must understand the product from this line alone.

### 2. The diagram

One Mermaid `flowchart LR` — left-to-right matches a projector's aspect ratio,
whereas `TD` produces a tall diagram that renders small.

Hard constraints:

- **Maximum 8 nodes.** If the system has more parts, group them. A box labelled
  `Storage` beats three boxes labelled `UserRepo`, `EventRepo`, `AuditRepo`.
- **Node labels are 1–3 words.** No sentences inside boxes.
- **Label every edge** with what flows along it, not how. `activity events`, not
  `calls .write()`.
- **Mark the activity log.** Every team builds it, so make it visibly distinct
  with `style` — judges compare that node across all three finalists.
- No colour beyond one accent. Projectors wash out pastels.

Verify the diagram parses before finishing. Broken Mermaid on stage is the one
failure that cannot be recovered from.

### 3. How we used AI

A markdown table, one row per feature the team actually used, with a concrete
artefact as evidence.

| Feature | What we did | What it changed |
| --- | --- | --- |

Concrete beats vague. "Plan Mode surfaced that two events can share a timestamp,
so we made ordering explicit in the spec" is worth more than "we used Plan Mode
to plan." Do not include features the team did not use — an honest four-row
table beats a padded six-row one.

### 4. What we'd do next

Two bullets. The honest gaps — what the tests do not yet cover, and the decision
you would revisit. Judges reward teams that know their own weaknesses; claiming
completeness invites the question you cannot answer.

## Rules

- **Total length under 60 lines.** If it does not fit on one screen, cut.
- Never describe code that does not exist. Every node maps to something real.
- Do not add sections. Four, in order. That is the format.
- Do not include token counts, credit spend, or team member names.

## Presenting it

`ARCHITECTURE.md` renders with a live diagram on GitHub and in Copilot Chat.
VS Code's built-in Markdown preview does **not** render Mermaid without an
extension — so if presenting from VS Code, verify the render first and keep a
screenshot as a fallback.
