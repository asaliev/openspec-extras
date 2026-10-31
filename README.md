# OpenSpec Extras

Additional skills for [OpenSpec](https://github.com/Fission-AI/OpenSpec).

## Skills

### `openspec-align-specs`

Generate or align OpenSpec specs from an existing codebase.

Use it when you want an agent to:

- generate first-time baseline specs under `openspec/specs/`
- align existing specs with implemented behavior
- create a schema-shaped OpenSpec alignment change
- report spec/code drift through the active schema's supported artifacts

The skill treats implemented behavior as the source of truth. If no specs exist, it writes baseline capability specs directly. If specs already exist, it creates an alignment change using the configured schema and the local OpenSpec-generated workflow guidance, then stores drift evidence only in artifacts that schema supports.

### `openspec-apply-change-parallel`

Wrap `openspec-apply-change` so independent tasks run concurrently.

Use it when you want an agent to:

- implement a change faster by working several tasks at once
- keep dependent tasks strictly ordered while parallelizing the rest
- cap how many subagents the implementation may use

The skill keeps `openspec-apply-change`'s selection, context loading, and reporting, and replaces only the implementation loop. It maps each remaining task to a concrete file set, groups tasks into waves whose file sets are disjoint, and runs each wave — one subagent per task, up to 4 by default. A wave of one task runs inline with no subagent, and a change whose tasks form a single dependency chain runs exactly like the plain skill.

The orchestrator owns `tasks.md`, the diff review, and the build: subagents never tick checkboxes, never touch files outside their assigned set, and never run the test suite while siblings are working in the same tree.

### `openspec-brainstorm`

Brainstorm an idea into an approved design, then hand it to OpenSpec.

A replacement for `openspec-explore` in flows that want structure before a change exists. The dialogue is borrowed from superpowers' brainstorming skill:

- the request is **classified** out loud as a spike, a bounded change, or architectural work, and the user can override
- intent is written back as a short note the user can correct before it becomes the brief
- clarifying questions come **one at a time**, multiple choice where possible
- architectural work gets 2–3 approaches with a recommendation, then a design presented **in sections** with approval after each
- a hard gate: no artifact is written and no code is touched until the presented design is approved

What is not borrowed is the output. The skill writes no design document and no implementation plan of its own. The approved design is written back as a brief in chat, and OpenSpec captures it through its normal artifact flow — `openspec new change`, then each artifact from `openspec instructions` — or by updating an existing change's artifacts. The brief is content; the schema is form. The skill stops at the artifacts and never applies, implements, or archives.

Requires the `openspec` CLI. Works alongside the upstream `openspec-propose`, `openspec-new-change`, and `openspec-continue-change` skills when they are installed, and runs the same CLI flow inline when they are not.

### `openspec-orchestrate-change`

Orchestrate an OpenSpec change implementation.

The session acts as an orchestrator and never writes product code itself. For each task group (a `## N.` section of `tasks.md`):

- a fresh **implementer** subagent works from a brief, writes tests for the spec scenarios the group satisfies, and commits
- a separate **reviewer** gates the group's commits on spec compliance and code quality
- a bounded **fix loop** resumes the implementer, escalates to a stronger model on the last round, and ends in a scoped re-review

After the last group, one whole-branch review on the most capable model verifies the change against its artifacts. One fix wave and one scoped re-review follow it.

The task group, not each checkbox, is the unit of dispatch. That keeps a typical change to roughly two agents per group rather than two per checkbox. Every model is chosen explicitly by role. A git-ignored ledger under `.openspec-extras/orchestrate/<change>/` lets an interrupted run resume without redoing finished groups.

The orchestrator rules on implementation and design questions without stopping, and records every ruling for the final report. It may correct `tasks.md` and `design.md` in place so the archived change stays truthful. It never rules on specified behaviour: anything that would add, narrow, or drop a spec requirement or scenario stops the run and asks you. Commits land on a feature branch, and the skill asks before committing to the default branch. The change is never archived, synced, or pushed.

Requires a harness that can spawn subagents with an explicit model. This skill sets `disable-model-invocation: true`, so it runs only when you invoke it.

### `openspec-review-cycle`

Run an automated review-and-fix loop on a single OpenSpec change.

Each cycle is two agents with separate jobs:

- a **reviewer** runs a code review and `openspec-verify-change` in one read-only pass
- a **subagent** applies the findings with the `apply-code-review` skill

The reviewer is Codex by default. Ask for Claude instead and the review passes run as read-only subagents inside the same session, while everything else stays the same. Naming an effort level switches those passes to a `claude -p` subprocess, since a subagent takes a model but not an effort level.

The loop repeats until the review comes back clean, stops converging, or the cycle budget runs out (3 by default). Whenever the last apply pass changed the tree, one closing review always follows it — read-only, with no fixes after it — so the final report says whether the fixes actually landed instead of only that they were attempted. Codex runs under `--sandbox read-only`, so the reviewer can never edit the code it reviews. A Claude subagent reviewer runs under a read-only agent type; the `claude -p` fallback runs in plan mode with the write tools denied and nobody to approve a prompt, and the loop checks the tree after each of its passes. Spec verification outranks code review whenever the two disagree.

Requires the Codex CLI — or, for a Claude reviewer at a named effort level, the Claude CLI — plus a code-review skill and `openspec-verify-change` on the reviewer's side.

This skill sets `disable-model-invocation: true`: a loop of Codex reviews and subagent fix passes is too
expensive to start on its own guess of your intent, so it runs only when you invoke it explicitly. Remove
that line from the frontmatter if you want the model to reach for it autonomously.

## Install

Install from this repository with the open agent `skills` CLI. Swap `--skill` for whichever skill you want, or pass it twice to install both.

Project-local install:

```bash
npx skills add asaliev/openspec-extras --skill openspec-align-specs
npx skills add asaliev/openspec-extras --skill openspec-apply-change-parallel
npx skills add asaliev/openspec-extras --skill openspec-review-cycle
npx skills add asaliev/openspec-extras --skill openspec-orchestrate-change
npx skills add asaliev/openspec-extras --skill openspec-brainstorm
```

Global install:

```bash
npx skills add asaliev/openspec-extras --skill openspec-align-specs --global
```

Local checkout install:

```bash
npx skills add . --skill openspec-align-specs
```

Install for specific agents:

```bash
npx skills add asaliev/openspec-extras --skill openspec-align-specs --agent codex
npx skills add asaliev/openspec-extras --skill openspec-align-specs --agent claude-code
npx skills add asaliev/openspec-extras --skill openspec-align-specs --agent cursor
npx skills add asaliev/openspec-extras --skill openspec-align-specs --agent github-copilot
```

## Usage

### `openspec-align-specs`

Example prompts:

```text
Use openspec-align-specs to generate baseline specs for this codebase.
```

```text
Use openspec-align-specs to align the existing OpenSpec specs with the current implementation.
```

```text
Use openspec-align-specs to regenerate specs for the API and CLI capabilities only.
```

Expected behavior:

- If `openspec/specs/` has no capability specs, the agent generates baseline specs directly.
- If specs already exist and the user requests alignment, the agent creates a new change using the schema configured in `openspec/config.yaml`.
- The alignment change includes source-code evidence for detected drift in schema-supported artifacts.
- OpenSpec workspace mode is read-only in v1 because upstream workspace support is still under active development.

### `openspec-apply-change-parallel`

Example prompts:

```text
Use openspec-apply-change-parallel on assemble-session-evidence.
```

```text
Apply add-session-result-read in parallel, max 3 subagents.
```

Expected behavior:

- The change name resolves the same way `openspec-apply-change` resolves it; the agent asks rather than guessing.
- Before implementing anything, the agent prints the wave plan — which tasks run sequentially, which run together, and at what width — then starts.
- Concurrency defaults to 4 and is capped there unless you ask for more.
- Tasks whose file set cannot be determined from the artifacts and the code are run sequentially, not guessed at.
- Shared types, migrations, dependency manifests, wiring and registration tasks, and whole-repo passes always run sequentially.
- After each wave the agent reads the diff, runs the project's build and tests once, and ticks only the tasks that verified.
- If nothing can safely run in parallel, it says so and runs the plain sequential flow with no subagents.
- Pauses happen at wave boundaries; failed tasks are repaired inline rather than re-delegated, and the change is never archived, synced, or committed.

### `openspec-brainstorm`

Example prompts:

```text
Use openspec-brainstorm: I want to add rate limiting to the public API.
```

```text
/openspec-brainstorm add-auth-system — the OAuth integration is more complex than expected
```

```text
Brainstorm with me whether we can stream exports instead of batching them.
```

Expected behavior:

- The agent states the path it chose — spike, bounded, or architectural — before asking anything, so you can override it. When in doubt it takes the heavier path, and it never downgrades mid-task.
- It runs `openspec list --json` first, and reads an active change's artifacts when one is named or clearly relevant.
- Questions arrive one per message. Architectural work gets 2–3 approaches with a recommendation, then a design in sections with a check after each.
- Nothing is written until you approve the presented design. Read-only exploration of the code and the change is allowed throughout.
- On approval the agent writes back a design brief and asks how to hand off: create the change and generate all artifacts, create it and step through artifacts one at a time, update the existing change, or stop.
- Artifacts take their shape from `openspec instructions`, not from the brief. Anything the schema needs that the brainstorm did not settle becomes a question, not an invention.
- A spike ends in a recommendation, not a change. Anything built during it is labelled throwaway.
- The skill stops at the artifacts and points at `openspec-apply-change`, `openspec-apply-change-parallel`, or `openspec-orchestrate-change` for implementation.

### `openspec-review-cycle`

Invoke it explicitly — the model will not start this one for you:

```text
/openspec-review-cycle trace-context-over-rabbit
```

```text
/openspec-review-cycle trace-context-over-rabbit, unstaged changes only, max 2 cycles
```

```text
/openspec-review-cycle add-session-result-read with gpt-5.6-sol at high effort
```

```text
/openspec-review-cycle add-session-result-read, reviewer claude with opus
```

```text
/openspec-review-cycle add-session-result-read, reviewer claude with opus at xhigh effort
```

Expected behavior:

- The change name is resolved from `openspec list --json`; the agent asks rather than guessing when it is ambiguous.
- Scope works the way the `code-review` skill's does: say "unstaged changes", "staged", "the last two commits", "vs develop", or a SHA and it uses exactly that.
- With no scope stated, it defaults to the merge base with the branch's actual base — resolved from the open PR or by asking, never assumed to be `main` — so stacked feature branches don't drag the base branch's commits into the review.
- Spec verification always covers the whole change regardless of how the code review is scoped; findings from outside the reviewed range are marked as such in the report.
- The reviewer is Codex unless you ask for Claude; either way it runs read-only, takes the model you name, stays the same for every pass of the run, and ends its report with `REVIEW_STATUS: CLEAN` or `REVIEW_STATUS: FINDINGS`.
- Findings go to a subagent running `apply-code-review`, which reports Applied / Declined / Deferred.
- Findings say what is wrong and how to tell when it is resolved; the applying subagent decides how to fix it.
- Later cycles carry the previous decisions into the review prompt, so the reviewer re-judges declines instead of repeating them.
- The loop stops early when the review is clean, nothing was applied, findings stop moving, or the build breaks.
- An apply pass is never the last step: budget-exhausted, stuck, and cosmetic-only exits are followed by a mandatory closing review, which reports what the fixes left unresolved and what they broke. Nothing is applied after it.
- The report ends with a single `CYCLE_RESULT: CLEAN | OPEN_FINDINGS | HALTED` line so an orchestrating agent doesn't have to parse the prose.
- Prompts, reviews, and decision ledgers are written to a run directory outside the repo, and the change is never archived, synced, or committed.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
