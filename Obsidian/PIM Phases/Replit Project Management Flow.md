# Replit Project Management Flow

A dual-agent development strategy for building large, complex applications in Replit using an external AI as a project manager.

---

## Overview

This workflow uses **two AI agents** in distinct roles to manage complexity, maintain quality, and produce thorough documentation as a natural byproduct of development.

| Role | Tool | Responsibility |
|---|---|---|
| **Project Manager** | External AI (Cursor, Claude Code, etc.) | Planning, prioritization, documentation, review |
| **Developer** | Replit Agent | Implementation, self-review, phase summaries |

The key insight is that **no single AI context window can hold an entire large project**. By splitting planning from implementation and feeding work through in controlled, bite-sized phases, you get reliable results without overwhelming either agent.

---

## The Flow

### Phase 0 — Requirements & Planning

#### 1. Collect Requirements

Gather and document requirements from stakeholders. Write them down in plain language — user stories, feature lists, constraints, timelines. This is your source of truth.

#### 2. Define MVP & Prioritize

Work with the Project Manager AI to:

- Sort requirements into **MVP** vs. future phases based on timeline and dependencies.
- Identify what must ship first and what can wait.
- Produce a prioritized requirements document.

#### 3. Build an Execution Plan

Zoom into a single phase (starting with MVP) and define:

- The execution order of tasks within that phase.
- Dependencies between tasks.
- Enough detail that each task is unambiguous.

This becomes your **Execution Plan** — a living document the Project Manager maintains.

#### 4. Write Implementation Instructions

Break each task in the execution plan into a **standalone `.md` file** — a single, bite-sized instruction set that Replit can process in one pass.

Each file should include:

- **Goal** — what this phase accomplishes.
- **Context** — relevant existing state, schema, or architecture decisions.
- **Detailed instructions** — step-by-step implementation guidance.
- **Acceptance criteria** — how to verify the work is complete.

> Keep each file focused. One concern per phase. If a task feels too large, split it.

---

### Phase N — Implementation Loop

Repeat the following cycle for each phase file:

#### 5. Add the Phase File to Replit

Drop the `.md` file into a `phases/` folder in the Replit project.

#### 6. Execute the Phase

Ask Replit to implement the phase:

> *"Please implement phases/N.md"*

Let it work through the instructions.

#### 7. Self-Review

Once Replit reports completion, have it double-check its own work:

> *"Please re-review phases/N.md to ensure implementation is complete."*

Replit will typically catch things it missed on the first pass — missing edge cases, incomplete migrations, forgotten UI states. This step is cheap and consistently valuable.

#### 8. Cross-Agent Review

Have Replit write a **phase implementation summary** — a detailed account of everything it did, any deviations from the plan, and any decisions it made.

Send that summary to the Project Manager AI for review. Three things happen here:

- **Replit improved on the plan.** Sometimes Replit finds a cleaner implementation than what was specified. The Project Manager updates its documentation and adjusts future phases to align with the actual architecture.

- **The Project Manager has feedback.** Clarifying questions, missed requirements, or concerns about the approach. Send these back to Replit to address.

- **Iterate until both agents agree.** Go back and forth between the two agents until the implementation is confirmed complete and correct.

#### 9. Advance to the Next Phase

Move to the next phase file and repeat steps 5–8. As phases progress, the Project Manager is responsible for:

- Keeping documentation current.
- Updating future phase files to reflect work already completed.
- Adjusting the execution plan if the project's direction has shifted.

---

## What You End Up With

By the time implementation is complete, you have:

- **A full set of phase instruction files** — the exact specifications that were implemented.
- **Implementation summaries for every phase** — a detailed record of what was actually built.
- **Living documentation** — maintained throughout development, not bolted on after the fact.

This documentation is precise because it was the mechanism of development, not an afterthought.

---

## Principles

**Small phases, tight feedback loops.** The smaller each phase file, the more reliable the output. Large instructions lead to large misses.

**Two perspectives catch more errors.** The Project Manager thinks in terms of requirements and architecture. Replit thinks in terms of code. Reviewing from both angles surfaces different classes of issues.

**Documentation is a side effect, not a task.** Every phase file and summary is documentation. You never have to stop and "write docs" — they accumulate naturally.

**The Project Manager holds the big picture.** Replit operates within a single phase. The Project Manager is the one tracking how phases connect, what's changed, and what's coming next.

**Let each agent play to its strengths.** The external AI is better at planning, prioritization, and cross-referencing large amounts of context. Replit is better at writing and running code in its environment. Don't ask either to do the other's job.

---

## Quick Reference

**Planning (once per phase group):**

1. **Requirements** → 2. **MVP Prioritization** → 3. **Execution Plan** → 4. **Phase Files**

**Implementation (repeat per phase):**

| Step | Action | Agent |
|------|--------|-------|
| 5 | Add phase `.md` to Replit | You |
| 6 | Implement the phase | Replit |
| 7 | Self-review against the phase file | Replit |
| 8a | Write implementation summary | Replit |
| 8b | Review summary, provide feedback | Project Manager |
| 8c | Address feedback *(repeat 8a–8c until aligned)* | Both |
| 9 | Advance to next phase | You |
