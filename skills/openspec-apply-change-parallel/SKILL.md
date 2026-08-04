---
name: openspec-apply-change-parallel
description: Implement an OpenSpec change by grouping its tasks into waves and running the independent tasks of each wave in parallel subagents, while dependent tasks stay sequential. Use when asked to apply, implement, or work through an OpenSpec change faster, in parallel, or with multiple agents/subagents.
---

# OpenSpec Apply Change (Parallel)

A wrapper around `openspec-apply-change`. Selection, context loading, and reporting are unchanged; only the implementation loop is replaced:

1. Read the tasks and work out which ones are genuinely independent.
2. Group them into **waves**: a wave is a set of tasks that can run at the same time without touching each other's files.
3. Run each wave — a wave of one runs inline in this thread, a wave of several runs as one subagent per task.
4. After every wave: inspect the diff, run the project's checks once, tick the boxes that passed, then start the next wave.

**Parallelism is an optimization, not the goal.** A change whose tasks are a single dependency chain runs exactly like `openspec-apply-change` and uses zero subagents. Never invent parallelism to justify the skill.

## Preconditions

- The harness can spawn subagents (Claude Code's Agent tool or equivalent). Without it, run the plain `openspec-apply-change` flow sequentially and say why.
- `openspec --version` succeeds, and the working directory is a git repo with an `openspec/` root (or the user named a store).
- The working tree is clean enough to read a diff against. If `git status --porcelain` shows unrelated edits, record them now so post-wave diffs can be separated from them.

## Inputs

| Input | Default | Notes |
| --- | --- | --- |
| Change name | none — must resolve | Same rule as `openspec-apply-change`: use the named change, infer from context, auto-select a lone active change, otherwise **AskUserQuestion** over `openspec list --json`. Never guess. |
| Max concurrency | `4` | Honor a lower number the user names. Treat 4 as the ceiling unless the user explicitly asks for more — beyond that, review of the merged result gets unreliable. |
| Store | none | If the work lives in an OpenSpec store, pass `--store <id>` on `status`, `instructions`, `list`, `show`, `validate` as `openspec-apply-change` describes. |
| Sequential-only | off | "one at a time", "no subagents" → run the plain flow. |

## Steps 1–5 — unchanged

Follow `openspec-apply-change` verbatim: select the change and announce it, run `openspec status --change "<name>" --json`, run `openspec instructions apply --change "<name>" --json`, handle `blocked` / `all_done`, read **every** file under `contextFiles`, and show current progress.

Do not skip the context read. The wave plan is only as good as your knowledge of the design artifact — that is where task-to-file mapping actually lives.

## Step 6 — Build the wave plan

Work from the remaining (unchecked) tasks only.

### 6a. Establish each task's file set

For every remaining task, determine the concrete files it will create or modify — from the task text, the design/spec artifacts, and actual lookups (`Glob`, `Grep`, reading the neighbours of the file it names). Cheap and worth it: a plan built on guessed paths is what produces two agents editing one file.

**A task whose file set you cannot determine is sequential.** No exceptions — that unknown is exactly where the collision hides.

### 6b. Force these sequential

Regardless of how independent they look:

- **Producers of shared code** — types, interfaces, schemas, contracts, base classes, shared fixtures. Everything that imports them waits.
- **Migrations, codegen, and generators** — anything that rewrites files it does not name.
- **Dependency manifests and lockfiles** — `package.json`, `go.mod`, `Cargo.toml`, `pyproject.toml`, and any install step.
- **Wiring and registration** — "register in the container", "add the route", "export from the barrel file", "add to the config". These converge on one file by nature.
- **Tasks that reference another task's output** — "using the client from 1.2", "extend the base adapter", "reuse the helper added above".
- **Whole-repo passes** — formatters, lint autofix, doc regeneration, final verification tasks.
- **Anything the change's design artifact marks as ordered.**

### 6c. Parallel-safe candidates

Tasks may share a wave only when all of these hold:

- Their file sets are **disjoint** — no shared file, in either direction, including test files and snapshots.
- Neither depends on the other's output, and both depend only on work already merged from earlier waves.
- Each is self-contained enough to hand to an agent that has not seen the other tasks.

Typical shapes that qualify: sibling implementations of an interface that already exists, per-module or per-endpoint work, an adapter per provider, tests for modules already implemented, independent docs pages.

### 6d. Assemble waves

- Order waves so every dependency lands before its dependants.
- Cap each wave at the concurrency limit. Split an oversized group across consecutive waves rather than raising the cap.
- **A wave of one is implemented inline, in this thread.** Never spawn a subagent to run a single task — the delegation costs more than it saves and loses your context.
- If no wave holds two or more tasks, say so plainly and run the plain sequential flow.

### 6e. Show the plan, then start

```
## Plan: <change-name> (schema: <schema-name>)

Wave 1 — sequential: 1.1 Define the SessionEvidence type
Wave 2 — parallel (3): 2.1 collector · 2.2 redactor · 2.3 exporter
Wave 3 — sequential: 3.1 Wire collectors into the session pipeline
Wave 4 — parallel (2): 4.1 collector tests · 4.2 exporter tests

4 waves · 7 tasks · max 3 concurrent
```

Start immediately after printing it; the user can interrupt. Ask first only when a grouping rests on a file-overlap call you could not settle from the artifacts and the code — then use **AskUserQuestion** with the ambiguous pair.

## Step 7 — Run the waves

### Sequential wave

Implement inline exactly as `openspec-apply-change` does. Tick the box on completion.

### Parallel wave

Spawn all of the wave's subagents in **one message** so they actually run concurrently. One task per subagent — never bundle two tasks into one agent to "save" an agent.

Each subagent prompt carries:

```
Implement one task from the OpenSpec change "<change-name>".

Task: <verbatim task text, including its number>

Context — read these before you start:
- <every contextFiles path: proposal, specs, design, tasks>
- <the specific spec delta or design section this task implements>

Files you own — create or modify only these:
- <explicit list from the file-set analysis>

Rules:
- Do NOT edit any file outside that list. If the task turns out to need one — including a
  shared type, a config, a barrel export, or a dependency manifest — STOP and report what you
  need instead of editing it. Another agent may be in that file right now.
- Do NOT edit tasks.md or tick any checkbox. The orchestrator owns it.
- Do NOT run git commands that change state (no add, commit, stash, checkout) and do not
  archive or sync the change.
- Do NOT run the full build, test suite, or formatter — other agents are working in this same
  tree and a whole-repo pass will collide with them. Narrow checks on your own files only.
- Match the conventions of the surrounding code. Keep the change minimal and scoped to this task.

Report back:
1. Files created and files modified, with paths.
2. Whether the task is fully complete, or what is missing.
3. Any deviation from the design artifact, and why.
4. Anything you discovered that affects other tasks in this change (a needed shared change, a
   wrong assumption in the design, a task that is already done).
```

Do not use worktree isolation. These tasks are partitioned by file, so they compose in the shared tree; separate worktrees would only add a merge step.

### After every wave — the orchestrator's job

Do all of this yourself. It is what keeps parallel work honest.

1. **Read the actual diff** of the wave's files (`git diff --stat` then the relevant hunks). Verify against the reports rather than trusting them.
2. **Check for overlap.** If two agents touched the same file despite the plan, the plan was wrong: inspect that file closely for a lost or duplicated edit before continuing, and say so in the final report.
3. **Run the project's build, typecheck, and tests once** — the command from `AGENTS.md`, `CLAUDE.md`, or the repo's tooling. Once per wave, in this thread, never inside the agents.
4. **Tick only what verified.** `- [ ]` → `- [x]` for tasks that are complete and passing. A task whose agent reported a gap stays unchecked.
5. **Repair inline.** Fix a failing or incomplete task yourself instead of re-spawning an agent for it — you now have the whole picture and the agent does not.
6. **Re-plan if needed.** An agent's discovery ("this needs a change to the shared type") can invalidate later waves. Rebuild the plan from the current state and show the revision.

## Pause rules

Pause at the **wave boundary**, not mid-wave — let running agents finish, then stop. Work already done stays; do not roll back a wave because one task in it failed.

Pause when:

- A task is unclear, or an agent reports a design issue → surface it, suggest updating artifacts, wait.
- Build or tests break and the inline repair does not fix it → stop with the tree as-is rather than stacking another wave on a broken base.
- Two agents collided on a file and the merged result is ambiguous → stop and show the file.
- An agent returns nothing or dies → do not silently re-run it. Report and either implement inline or wait.

## Final report

Use `openspec-apply-change`'s completion/pause format, plus:

- The wave plan as executed, with which waves ran parallel and at what width.
- Tasks completed this session and overall `N/M`.
- Build and test status from the last wave.
- Any file the plan let two agents touch, and how you resolved it.
- Anything an agent flagged that the user should decide on.

Never `openspec archive`, never `openspec sync`, never commit. The change stays open.

## Failure modes

- **Over-eager parallelism.** Two tasks look independent, both need one shared file, and the second agent's write clobbers the first. The fix is upstream, in 6a: unknown file set → sequential.
- **A subagent ticks its own checkbox.** Concurrent writes to `tasks.md` lose ticks. Only the orchestrator writes that file, and only after verification.
- **Agents run the test suite in parallel.** Port clashes, shared build caches, and formatter races produce failures that look like code bugs. Checks belong to the orchestrator.
- **Design drift across a wide wave.** Four agents interpret the same design artifact four ways. Reading the diff after each wave, not just the reports, is what catches it.
- **A wave that is really a chain.** If wave N's agents keep reporting "I needed the thing from the other task", the grouping was wrong. Fall back to sequential for the rest of the change rather than re-planning around it repeatedly.
