---
name: parallel-worktree-agents
description: Orchestrate parallel sub-agents across isolated git worktrees using only built-in tools (Agent, Bash, Write, Read). Consumes a `tasks.md` (produce one with the parallel-task-decomposition skill if absent). Use when the user wants to fan out multi-hour independent tasks that will later converge via merge — not for quick research fan-outs.
---

# Parallel Worktree Agents

A skill for an orchestrator to spawn parallel sub-agents, each isolated in its own git worktree, coordinated through the filesystem. The pattern is reproducible with `git worktree`, the `Agent` tool, and plain files — no external tooling required.

## When to use

- Multiple independent tasks in one repo that will each take more than a few minutes (feature + refactor + tests).
- Tasks touch different files or can be made to, and will eventually merge back to one branch.
- The user wants persistent, inspectable parallel work, not a quick research fan-out.

## When NOT to use

- Read-only research or survey questions → use a plain `Agent` call, no worktree needed.
- Tasks that mutate the same files → serialize them instead; isolation won't save you from semantic conflicts.
- One-shot jobs under ~15 minutes → the worktree setup overhead isn't worth it.
- A non-git directory, or a repo with uncommitted chaos in the working tree.

## Prerequisite: a `tasks.md` with a sound decomposition

**The input to this skill is a `tasks.md` file (format below) whose task blocks satisfy the criteria in this section. If the decomposition is wrong, nothing downstream saves you — not isolation, not verification, not merge order.** Decomposition is the orchestrator's hardest and least-delegable job; if no `tasks.md` exists yet, invoke the `parallel-task-decomposition` skill first (or hand-write one to the same schema).

A task block is ready when it is:

- **File-disjoint.** Its `Files` line names a path set that does not overlap with any other parallel task's `Files`. If you can't state the set, the task is not parallelizable — serialize it.
- **Semantically independent.** It does not depend on behavior another task is in the middle of changing. Shared module, different function is often not independent enough. Ask: "If these ran in either order, would both still be correct?" If no, serialize.
- **Self-contained.** A sub-agent can finish from the preamble + block alone, without asking a question. If a judgment call is needed, make it in the block — not after, by interpreting a "blocked" broadcast.
- **Right-sized.** Too small and overhead dominates; too large and context balloons and drifts. Aim for a focused agent finishing in one session.

Frameworks like CrewAI and LangGraph hide bad decomposition behind coordination layers — agents negotiate, retry, replan. This pattern has no such cushion. A bad split surfaces immediately as a merge conflict, a semantic bug, or a sub-agent stuck asking questions it can't ask. That's a feature: **the pattern is honest about where the intelligence has to live, and it has to live in you.**

If any task fails the checks above, stop and re-decompose before spawning. Do not invoke this skill and hope — that is not what this pattern offers. If you hit merge conflicts or re-spawn blocked agents mid-run, the problem is almost never the sub-agents; it's the split. Stop, re-decompose, re-spawn. Do not tune prompts to paper over a structural mistake.

## The five constraints (non-negotiable)

These are the reasons the pattern works, assuming the decomposition above is sound. Violating any of them causes the system to degrade into manual conflict-resolution.

1. **Context is the unit of work.** Each sub-agent gets *exactly* what it needs: shared preamble + its own task block + read-only shared context. No repo dump, no "just in case" history.
2. **Isolation is the enabling constraint.** One worktree, one branch, one sub-agent. Never share a working directory or branch between two live agents.
3. **Asymmetric knowledge.** The orchestrator knows the system exists. Sub-agents do not. If sub-agents learn there are peers, they try to coordinate, and their coordination is worse than yours.
4. **Filesystem is the coordination layer.** Shared context files, append-only broadcast files, git diffs. No queues, no databases, no custom protocols.
5. **Broadcasts are claims; diffs are evidence.** A sub-agent reporting "done" is a claim. The orchestrator verifies via `git diff` before trusting it.

## Directory layout

Set this up in the main repo before spawning anything:

```
<repo-root>/
├── .shared/
│   ├── context.md              # read-only preamble — all sub-agents see this
│   └── broadcasts/
│       └── TASK-<id>.md        # append-only, one file per sub-agent
├── .worktrees/                 # gitignored, holds worktrees
│   ├── TASK-001/
│   │   └── wt-task.md          # this sub-agent's task block
│   └── TASK-002/
│       └── wt-task.md
└── tasks.md                    # source of truth: preamble + task blocks
```

Add `.worktrees/` and `.shared/broadcasts/` to `.gitignore` at the start. `.shared/context.md` can be committed or not depending on whether the preamble is a durable artifact.

## tasks.md format

Schema the skill expects. See the prerequisite section for the decomposition properties each task block must satisfy.

```markdown
## Preamble

[Shared background, architecture constraints, coding conventions, API shapes.
 Every sub-agent sees this verbatim. Keep it precise — context flooding is as
 bad as context starvation.]

---

### TASK-001
**Title**: Add OAuth provider
**Status**: [ ]
**Priority**: P1
**Depends On**: —
**Files** (expected): src/auth/oauth.ts, src/auth/oauth.test.ts

[Concrete objective. What "done" looks like. Edge cases to handle.
 Files it is allowed to touch. Files it must NOT touch.]

---

### TASK-002
**Title**: Refactor session store
**Status**: [ ]
**Priority**: P2
**Depends On**: —
**Files** (expected): src/session/*.ts

[...]
```

The `Files` list is what the skill uses to predict textual merge conflicts before spawning and to verify diffs after. `Status` starts as `[ ]` and flips to `[x]` via the Update procedure once a sub-agent's broadcast is verified against the diff.

## Spawn procedure

For each task to run in parallel:

1. **Create the worktree and branch:**
   ```bash
   git worktree add .worktrees/TASK-001 -b wt/TASK-001
   ```
2. **Write the task file** into the worktree:
   - `.worktrees/TASK-001/wt-task.md` ← preamble + that task's block only
3. **Symlink shared context** (read-only from the sub-agent's perspective):
   ```bash
   ln -s ../../.shared .worktrees/TASK-001/.shared
   ```
4. **Spawn the sub-agent** via the `Agent` tool. The prompt must include:
   - The sub-agent's working directory: `.worktrees/TASK-001/`
   - Its task file path: `wt-task.md`
   - The read-only shared context path: `.shared/context.md`
   - Its broadcast file path: `.shared/broadcasts/TASK-001.md`
   - Instruction: when done, append a structured entry to the broadcast file and stop. Do not attempt to merge, switch branches, or coordinate with other agents.
   - **No mention of other sub-agents, the orchestration pattern, or this skill.** (Asymmetric knowledge.)
5. **Spawn the current wave in parallel.** A task is *ready* when every task in its `Depends On` list has `Status: [x]`. Spawn all ready tasks — one `Agent` call per task, all in a single message. Tasks with unsatisfied deps do NOT spawn yet.

After the wave completes (every spawned sub-agent has broadcast), run the Update procedure to mark done tasks `[x]` and decide what to do with blocked ones. Then compute the next wave and repeat until no ready tasks remain.

## Broadcast format

Sub-agents append one entry to their `TASK-<id>.md` broadcast file when they finish or get stuck:

```markdown
## <timestamp> — done | blocked | need-input
**Branch**: wt/TASK-001
**Summary**: One-sentence what was changed.
**Files**: path/a.ts, path/b.ts
**Verification**: how I know it works (tests run, type-check, manual)
**Blockers**: (if any) specific missing information or decision
```

The orchestrator reads this, then runs `git -C .worktrees/TASK-001 diff wt/TASK-001 <base>` to verify the claim before trusting it.

## Update procedure (after each wave)

Once every sub-agent in the current wave has appended a broadcast, reconcile `tasks.md` before computing the next wave or merging:

1. **Read every new broadcast.** Classify each: done / blocked / need-input.
2. **Verify "done" claims against diffs.** `git -C .worktrees/TASK-xxx diff wt/TASK-xxx <base>` — files touched must match the `Files` line. If the diff includes surprise paths, treat the task as not-done and demote to blocked.
3. **Update `Status`.** Flip verified "done" tasks to `[x]` in `tasks.md`. Leave blocked / need-input tasks as `[ ]` and append a short blocker note in the task block.
4. **Handle blockers.** For each blocked task, choose: respawn with tighter context (same `Depends On`), serialize a prerequisite fix (new task it now depends on), or drop it. Edit `tasks.md` to reflect the choice.
5. **Handle discovered work.** If a sub-agent surfaced a new required task (refactor, latent bug), add a `TASK-xxx` block with correct `Depends On`. This is scope expansion — surface it to the user before committing.
6. **Re-check the decomposition.** If blockers keep recurring or diffs keep straying outside `Files`, the split is wrong. Stop, re-decompose, don't patch.

`tasks.md` is the source of truth for what's been tried, what succeeded, and what's left. Keep it live across waves.

## Convergence (merge) procedure

When `tasks.md` shows all tasks as `[x]` (or explicitly dropped):

1. Topologically sort completed tasks by `Depends On`.
2. Merge in order: `git merge wt/TASK-001`, resolve if needed, repeat. If a conflict fires between tasks that *shouldn't* have overlapped, that's a decomposition bug — note it and fix the `Files` contract next time.
3. Tear down: `git worktree remove .worktrees/TASK-001 && git branch -d wt/TASK-001`.

## State reconciliation

Before every orchestration pass, run a cheap consistency check:

- `git worktree list` vs `.worktrees/` directory listing → flag orphans both ways.
- For each worktree, check whether its branch still exists and has commits beyond base.
- Stale broadcast files (no corresponding worktree) → archive or delete.

Do this before trusting any in-memory state. The filesystem is the source of truth; your memory of what you spawned is not.

## What the sub-agent prompt looks like

Template for the `Agent` tool `prompt` field (subagent_type: `general-purpose`):

```
You are working on a focused task in an isolated git worktree.

Working directory: <absolute path to .worktrees/TASK-001>
Your task file: wt-task.md (read this first, in full)
Read-only shared context: .shared/context.md (read if task file references it)

Constraints:
- Touch ONLY files listed under "Files" in your task file.
- Do not switch branches. Do not run `git merge`. Do not modify other worktrees.
- When you finish (or hit a blocker you cannot resolve), append a single
  structured entry to .shared/broadcasts/TASK-001.md using this format:
  [paste the broadcast format block]
- Then stop. Do not loop, do not start a new task.

Report in under 200 words at the end: what you changed, how you verified it.
```

Notice what is absent: no mention of peer agents, no mention of the orchestrator, no mention of merge. The sub-agent must not know.

## Honest limits

- **Semantic conflicts aren't prevented — and this pattern will never prevent them.** Two worktrees editing different files can still break each other's assumptions. The `Files` contract catches textual conflicts only. Semantic correctness across parallel tasks is purely a function of how well you decomposed, and there is no tooling here that compensates for a bad split.
- **Sub-agents can't ask clarifying questions mid-flight** through this pattern. Design tasks so that reasonable defaults exist, or accept that a "blocked" broadcast means you re-spawn with tighter context.
- **Cost scales linearly.** N parallel agents cost N× the tokens. Reserve this for cases where wall-clock matters more than token spend.
- **`Agent` tool sub-agents don't persist across the orchestrator's turns.** If the orchestrator conversation ends mid-flight, the work on disk survives but the live agent does not. Plan task granularity around one orchestrator turn, or reach for a dedicated tmux-based orchestrator if you need multi-session persistence.

## Decision checklist before spawning

**Decomposition (the load-bearing checks):**
- [ ] Could I explain, in one sentence per task, why each task is independent of every other?
- [ ] Are the `Files` sets strictly disjoint? If not, have I serialized the overlapping tasks?
- [ ] Would the tasks still be correct if they ran in any order? If no, have I encoded the order in `Depends On` and accepted that dependents cannot start yet?
- [ ] Does every task have enough context to finish without asking a question?

**Mechanics:**
- [ ] Do I have a `tasks.md` with preamble and explicit `Files` per task?
- [ ] Is `.shared/context.md` written and precise (not a repo dump)?
- [ ] Is the repo on a clean base branch with no uncommitted changes?
- [ ] Are `.worktrees/` and `.shared/broadcasts/` gitignored?
- [ ] Have I drafted the sub-agent prompt template with no mention of peers?

If any decomposition check is "no," **do not spawn** — re-split the work. The mechanics checks are recoverable; a bad decomposition is not. The setup cost is the whole point — skipping it is how you end up with the cognitive-overload failure mode this pattern was designed to prevent.
