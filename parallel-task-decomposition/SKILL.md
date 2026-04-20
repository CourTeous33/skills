---
name: parallel-task-decomposition
description: Decompose a goal into a `tasks.md` with preamble + file-disjoint, semantically independent task blocks suitable for parallel execution. Produces the input that parallel-worktree-agents consumes. Use when the user wants to fan out work across parallel agents and no tasks.md exists yet.
---

# Parallel Task Decomposition

Produces a `tasks.md` at the repo root: a preamble plus per-task blocks that satisfy the criteria below. The output is the input to `parallel-worktree-agents` (or any equivalent orchestrator).

## When to use

- User wants to fan out work across parallel agents and no `tasks.md` exists yet.
- User says "decompose this," "split this up," "plan this as parallel tasks."
- You're about to invoke `parallel-worktree-agents` and its prerequisite isn't satisfied.

## When NOT to use

- Work is inherently sequential → plan normally.
- Single task → invoke `Agent` directly, no decomposition needed.
- Goal is vague → ask the user to clarify; do not decompose a vague goal.

## Output

A single `tasks.md` at the repo root, matching the schema `parallel-worktree-agents` expects:

```markdown
## Preamble

[Shared background, architecture constraints, coding conventions, API shapes.
 Every sub-agent sees this verbatim. Keep it precise — context flooding is as
 bad as context starvation.]

---

### TASK-001
**Title**: <short>
**Status**: [ ]
**Priority**: P1 | P2 | P3
**Depends On**: — | TASK-xxx, ...
**Files** (expected): path/a.ts, path/b.ts

[Concrete objective. What "done" looks like. Edge cases. Files allowed and
 files forbidden.]

---

### TASK-002
...
```

## Decomposition criteria (every task block must satisfy all)

- **File-disjoint.** `Files` sets do not overlap between parallel tasks. If you can't name the set, the task isn't parallelizable — serialize it.
- **Semantically independent.** Neither task depends on behavior the other is in the middle of changing. Ask: "If they ran in either order, would both still be correct?" If no, serialize via `Depends On`.
- **Self-contained.** A sub-agent can finish from preamble + block alone, without asking a question. If a judgment call is needed, make it in the block.
- **Right-sized.** One focused agent, one session. Too small → overhead dominates. Too large → context balloons and drifts.

Failing any check → serialize with `Depends On`, merge into a neighbor, or re-split. Do not parallelize and hope.

## Procedure

1. **Clarify the goal.** In one sentence, what is the user trying to accomplish? If vague, ask.
2. **Draft candidate tasks.** 2–8 candidates. Err toward fewer, larger tasks first; split later if needed.
3. **Assign files.** For each candidate, name the exact paths or globs it will touch. If you can't, the task isn't well-defined — go back to step 2.
4. **Check file-disjointness.** Lay the `Files` sets side by side. Overlap? Move the file to one task, or serialize with `Depends On`.
5. **Check semantic independence.** For every parallel pair, ask the "either order" question. If no, serialize.
6. **Check self-containedness.** Read each block as the sub-agent would. Missing context? Deferred judgment call? Fix before writing.
7. **Right-size.** Task feels like > ~2 focused hours → split. Task feels like < ~15 min → merge with a neighbor.
8. **Write the preamble.** Shared context every sub-agent needs: architecture, conventions, constraints. Not a repo dump. No task-specific detail.
9. **Write `tasks.md`** to the repo root using the schema above.
10. **Show the task list** (IDs + titles + Files) and get user sign-off before handoff.

## Common failure modes

- **Layer splits instead of vertical slices.** "Types / implementation / tests" almost always creates semantic dependencies. Prefer vertical slices (one feature, end-to-end).
- **Shared helper files.** Two tasks each "just add one thing" to `utils.ts` → textual conflict. Pick one owner, or do the refactor as a prior serial step.
- **Implicit ordering.** Task A's API shape is what Task B consumes. Unless A is already committed, serialize with `Depends On`.
- **Preamble bloat.** Dumping the README into the preamble. Keep it to the minimum shared context.
- **Deferred judgment.** "Pick whichever error-handling strategy fits" → blocked broadcast. Decide now.

## Handoff

Once `tasks.md` is written and the user has signed off:
- If the user asked for end-to-end execution, invoke `parallel-worktree-agents` next.
- Otherwise return control — the user drives from here.

## What this skill does NOT do

- Spawn agents or create worktrees. That's `parallel-worktree-agents`.
- Guarantee semantic correctness. The criteria are honest checks, not tooling.
- Decide whether parallelization is worth it. If in doubt, ask the user.
