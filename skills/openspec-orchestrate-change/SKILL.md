---
name: openspec-orchestrate-change
description: Implement an OpenSpec change as an orchestrator — one fresh implementer subagent per task group, a spec-compliance and code-quality review gate after each group, a bounded fix loop with model escalation, and one whole-branch review against the change artifacts at the end. Use when asked to orchestrate, drive, or implement an OpenSpec change with subagents and review gates.
disable-model-invocation: true
---

# OpenSpec Orchestrate Change

This session is the **orchestrator**: it never writes product code. For every task group:

1. **Dispatch** a fresh implementer subagent with a brief, on an explicitly chosen model.
2. **Review** the group's commits with a separate reviewer: spec compliance, then code quality.
3. **Fix loop** until the review is clean, or the round cap trips and the orchestrator adjudicates.
4. **Tick** the group's checkboxes, ledger the result, move on without checking in.

After the last group, one **final review** of the whole branch against the change artifacts, one fix wave, one scoped re-review.

The orchestrator owns `tasks.md`, the ledger, and every ruling. Implementers never review, reviewers never write, and nobody but the orchestrator ticks a checkbox.

## The unit of work is the task group

A **task group** is a `## <N>. <title>` section of the change's tasks file — not a single checkbox. Each group is a coherent chunk of work with its own tests and its own commit. Its checkboxes are the steps inside that chunk.

Dispatching per checkbox makes a 30-task change cost 60+ sequential agents. Dispatching per group makes it roughly two agents per group plus the final review. Never dispatch per checkbox.

Adjust groups only at preflight, and ledger each adjustment as a ruling:

- **No group headings** (a schema that tracks a flat list): form groups yourself from consecutive related tasks, 3–6 each.
- **An oversized group** — more than ~8 tasks, or tasks spanning unrelated areas: split it into consecutive sub-groups `4a`, `4b`.
- **A trivial group** — one small mechanical task, such as a doc touch-up: let it ride with the adjacent group in one dispatch.

## Preconditions

Check these before starting; stop and report if one fails.

- The harness can spawn subagents with an explicit model (Claude Code's **Agent** tool). Resuming a live subagent (Claude Code's **SendMessage**) is preferred but optional — without it, fix rounds dispatch a fresh implementer with the report file as memory. With no subagent tool at all, stop and recommend `openspec-apply-change`; this skill without subagents is just a slower apply.
- `openspec --version` succeeds, and the working directory is a git repo with an OpenSpec root, or the user named a store.
- `bash` is available for the helper scripts in `scripts/`. Run them by their path relative to this skill's directory, with `bash <path>`.

## Inputs

Parse these from the user's request. Ask only about the change name, and only when it cannot be resolved.

| Input | Default | Notes |
| --- | --- | --- |
| Change name | none — must resolve | Same rule as `openspec-apply-change`: the named change, inferred from context, or the lone active one; otherwise **AskUserQuestion** over `openspec list --json`. Never guess. |
| Store | none | Pass `--store <id>` on every `openspec` command, as `openspec-apply-change` describes. |
| Max fix rounds | `3` | Per group. Rounds before the last resume the original implementer; the last round is a fresh implementer one model tier up. |
| Model tiers | harness defaults | The user may pin a model per role ("implement with sonnet, review with opus"). Pinned models override [Model selection](#model-selection). |

Echo the resolved settings in one line, then start: `Change: X (schema: spec-driven) · 6 groups, 4 remaining · branch: change/x · max 3 fix rounds`.

## Setup

### 1. Load the change

Follow `openspec-apply-change` steps 1–5 verbatim: select and announce the change, run `openspec status --change "<name>" --json` and `openspec instructions apply --change "<name>" --json`, handle `blocked` and `all_done`, apply `context` and `operationGuidance` as that skill describes, and read **every** file under `contextFiles`.

Also read the repo's agent instruction files — AGENTS.md, CLAUDE.md, or whatever equivalent it uses. You need them to write briefs and to judge reviews, even though you edit no product code.

Record from the apply JSON: `changeDir`, the tasks file (`tasks[].sourcePath`), and whether that file is inside this git repo (`git ls-files --error-unmatch <path>` succeeds). A change kept in an OpenSpec store usually lives in another repository — then `tasks.md` and the artifacts are edited in place but never committed by this run.

### 2. Branch and tree

Every group commits, so where the commits land matters.

- **On the default branch** (`main`, `master`, or whatever `origin/HEAD` points at): **AskUserQuestion** — create `change/<name>` from here (recommended), or continue on the current branch. Never commit to the default branch without that answer.
- **Dirty tree**: implementers stage by path, but unrelated edits still make diffs unreadable. List `git status --porcelain` and **AskUserQuestion** — commit them as a baseline first (recommended when they are this change's own artifacts), or stop. Never stash or discard anything yourself.

Record `MERGE_BASE` — the commit the branch started from (`git merge-base <default-branch> HEAD`), or `HEAD` now if the branch is new.

### 3. Workspace and ledger

```bash
WS=$(bash scripts/workspace "<change-name>")
```

The workspace is git-ignored. Everything this run produces goes there: `progress.md` (the ledger), `group-<N>-brief.md`, `group-<N>-report.md`, `review-<a>..<b>.diff`.

Conversation memory does not survive compaction. Orchestrators that lost their place have re-dispatched whole sequences of finished work — the most expensive failure there is. **The ledger, `tasks.md`, and `git log` are the record; your recollection is not.**

- **No ledger**: create `$WS/progress.md` with the identity line `# Orchestration ledger — change: <name> — changeDir: <changeDir>`.
- **Ledger exists, identity matches**: resume. A group with a `Group <N>: complete` line is done — never re-dispatch it. A group whose last line is a fix round is mid-loop; resume the loop at the next round.
- **Ledger exists, identity names another changeDir**: a same-named change from another root. Stop and ask.

Cross-check with `tasks.md`: a group whose checkboxes are all ticked but has no ledger line was done outside this skill — ledger it as `Group <N>: complete (pre-existing)` and skip it. A partly ticked group is dispatched with a note in its brief that the ticked items exist and need verifying, not redoing.

### 4. Preflight scan

Before the first dispatch, check the artifacts against each other once. Write the result to the ledger as a table, not a verdict:

- One row per group: the requirements and scenarios from the spec deltas it satisfies (by requirement name and spec path), and whether its own task text is self-consistent.
- One row per pair of groups that share a file or an interface: what one produces, what the other consumes, and what you found.
- Every ADDED or MODIFIED requirement maps to at least one group. An unmapped requirement is a finding.

Rule on each conflict before execution, using [Rulings](#rulings), and ledger it. "The scan is clean" without the rows is not a scan you ran.

## Model selection

Always pass a model when dispatching. An omitted model inherits the session's — usually the most expensive — and silently defeats this section.

Work in three tiers and map them to what the harness offers (in Claude Code: `haiku`, `sonnet`, `opus`):

| Role | Tier |
| --- | --- |
| Implementer — task text names exact files, values, and tests | cheap |
| Implementer — prose description, multi-file, or integration work | standard |
| Implementer — design judgment or broad codebase understanding | most capable |
| Group reviewer | standard; most capable for subtle concurrency, security, or data-integrity diffs |
| Scoped re-review of a small fix diff | cheap to standard |
| Last fix round | one tier above the implementer that got stuck |
| Final review | most capable |

Turn count beats token price: the cheapest tier routinely takes 2–3× the turns on multi-step work. Use it only where the work is transcription plus testing.

## The group loop

Run groups in `tasks.md` order. Do not pause between groups to check in or summarise — the user asked for the change to be implemented. Narrate at most one short line between tool calls; the ledger carries the record.

Never run two implementers at once. Groups commit to one branch in one tree, and every review depends on a clean `BASE..HEAD` range.

### 1. Brief and dispatch

```bash
BASE=$(git rev-parse HEAD)
BRIEF=$(bash scripts/group-brief "$WS" "<tasks file>" <N>)
```

For a group you split, merged, or formed at preflight, the script cannot find it — write `$WS/group-<N>-brief.md` yourself with the group's task lines verbatim.

Append an `## Acceptance` section to the brief yourself:

- The spec files and requirement names this group satisfies, from the preflight table — paths and names, not copied text. The scenarios under them are the acceptance criteria.
- The design sections that bind the group, by heading.
- Interfaces and decisions from earlier groups that the brief cannot know, and any ruling from the ledger that touches this group's area.
- Your resolution of any ambiguity you noticed in the task text.
- When the task text is too thin for an agent that has not seen this conversation — no files named, no observable outcome — the concrete file paths and expected behaviour, found by reading the design and the code.

The brief is the single source of requirements. Keep exact values in the brief, never in the dispatch prompt, and never make an implementer read the whole tasks file.

Dispatch from [implementer-prompt.md](implementer-prompt.md). Record `BASE` and the implementer's agent id in the ledger: `Group <N>: dispatched (base <sha7>, agent <id>, model <tier>)`.

A dispatch prompt describes one group, not the session's history. Do not paste summaries of earlier groups — a fresh implementer needs its brief, the interfaces it touches, and the constraints that bind it.

### 2. Handle the status

| Status | Action |
| --- | --- |
| `DONE` | Go to review. |
| `DONE_WITH_CONCERNS` | Read the concerns. Correctness or scope doubts get resolved before review — answer them and resume the implementer. Observations go to review as-is. |
| `NEEDS_CONTEXT` | Answer from the artifacts and the code, then resume. If the answer would change specified behaviour, that is a [stop](#stop-conditions). |
| `BLOCKED` | Something must change before a retry: more context, a more capable model, a split group, or a ruling on a broken task. Never resend the same brief to the same model. |

### 3. Review

```bash
PKG=$(bash scripts/review-package "$WS" "$BASE" HEAD)
```

Dispatch [group-reviewer-prompt.md](group-reviewer-prompt.md) with the brief, the report file, and the package path. Use a read-only agent type where the harness offers one. The package never enters your own context.

Do not pre-judge findings for the reviewer. If your prompt contains "do not flag", "at most Minor", or "the design chose", you are trying to spare yourself a fix loop — remove it.

The review must return both verdicts, spec compliance and quality. A report missing either is not a review; re-dispatch it. For each **⚠️ Cannot verify from diff** item, check it yourself — you hold the cross-group context the reviewer lacks. A confirmed gap counts as a failed spec verdict.

### 4. Fix loop

Before the loop, two kinds of findings leave it:

- **Minor** findings go to the ledger as `Group <N>: minor (deferred): <one-liner>`. The final review triages them.
- A finding that **conflicts with the artifacts** — the reviewer wants something the spec, design, or task text says otherwise — is yours to rule on with [Rulings](#rulings) before anything is dispatched.

Everything else — spec ❌, Critical, Important, confirmed ⚠️ — enters the loop. One round is one fix dispatch plus one scoped re-review:

- **Rounds before the last**: resume the original implementer with the open findings verbatim. Its context is intact. Without a resume tool, dispatch a fresh implementer with the brief, the report file, and the findings.
- **The last round**: a fresh implementer one tier up, framed as "A prior implementer attempted this group <R> times; you own it now. Read the report file for what was tried."
- **Every round**: the implementer fixes, re-runs the covering tests, appends a fix report to the same report file, and commits. Confirm the fix report names the tests, the command, and the output before re-reviewing. Then `PKG=$(bash scripts/review-package "$WS" "$FIX_BASE" HEAD)`, where `FIX_BASE` is the head the previous review saw, and dispatch [re-review-prompt.md](re-review-prompt.md).

Ledger every round: `Group <N>: fix round <R>/<max> (<X> addressed, <Y> open — <one-liners>; commits <a7>..<b7>)`. New Critical or Important breakage in the fix diff joins the open list. Out-of-scope observations go to the ledger as deferred minors and never extend the loop.

Never fix findings yourself. Orchestrator fixes skip review and fill the context you need for coordination.

**The breaker.** When the last round still leaves findings open, stop dispatching and adjudicate each one:

- **The reviewer is wrong, or the point is contestable**: `Group <N>: parked — <finding> — Ruling: <why the code stands>`.
- **Real, but nothing later builds on it**: park it the same way, with a ruling that says it is real and deferred.
- **Real and load-bearing** — a later group builds on it: rule on the smallest change that unblocks the dependent work, ledger it, and carry it into that group's brief. If it is a gap in specified behaviour, that is a [stop](#stop-conditions).

Adjudicate only at the cap. Adjudicating earlier to end a loop is pre-judging under another name.

### 5. Complete the group

When the review is clean, or every open finding is parked with a ruling:

1. Tick the group's checkboxes at their `sourcePath` and `line`, confirming each line still matches its task first. **Do not tick a task whose specified behaviour has a parked gap** — leave it `- [ ]` and name it in the final report.
2. If the tasks file is tracked in this repo, commit it by path: `git commit <tasks-file> -m "<change-name>: complete group <N>"`. If it lives elsewhere, leave the edit uncommitted there.
3. Rerun `openspec instructions apply --change "<name>" --json` and confirm progress moved.
4. Ledger `Group <N>: complete (commits <base7>..<head7>, review clean)` or `… , <K> parked)`.

Never start the next group while Critical or Important findings are neither fixed nor parked with a ruling.

## Final review

After the last group:

```bash
PKG=$(bash scripts/review-package "$WS" "$MERGE_BASE" HEAD)
```

Dispatch [final-reviewer-prompt.md](final-reviewer-prompt.md) on the most capable tier, read-only, with the package, the ledger path, and the change name. It verifies the whole change against its artifacts — what `openspec-verify-change` checks — reviews the branch as code, and triages the ledger's deferred minors and parked findings.

Save its report to `$WS/final-review.md` yourself — a subagent answers to you, not to a file.

If it returns findings, dispatch **one** fix implementer with the complete list — never one per finding. Use [implementer-prompt.md](implementer-prompt.md) with `$WS/final-review.md` as the brief and `$WS/final-fix-report.md` as the report, on the standard tier or higher. Then run exactly one scoped re-review of the fix range. Adjudicate any residuals as in the breaker. There is no second fix wave; residual load-bearing findings go to the user in the final report.

## Rulings

A running change does not wait on a human for things you can decide. Rule, ledger it as `Ruling: <what you decided> — <why> — <what it costs if wrong>`, and continue.

The authority order is fixed: **spec deltas > design > tasks > reviewer opinion**. What you may do depends on which artifact a ruling contradicts:

| The ruling contradicts | What you do |
| --- | --- |
| Nothing — it settles how, not what | Ledger it. |
| `tasks.md` text (wrong file, impossible order, a task already done) | Ledger it, and correct the task text in place so the archive stays true. |
| `design.md` (an approach that does not survive contact with the code) | Ledger it, and amend the affected design section in place, marked `<!-- amended during implementation: <one line> -->`. |
| A spec requirement or scenario — adding, narrowing, deferring, or dropping specified behaviour | **Do not rule.** That is the user's contract. [Stop](#stop-conditions). |

Never edit `proposal.md` or a spec delta during the run.

## Stop conditions

Stop at the group boundary and ask only for these:

- A finding, a `NEEDS_CONTEXT`, or a breaker adjudication that would add, narrow, defer, or drop behaviour a spec delta specifies.
- An irreversible or destructive operation, a security-sensitive action, or a side effect outside this tree that norms say to ask about first — a push, a merge, a publish, a migration against a shared database.
- The build or tests are broken at a group boundary and the fix loop could not recover. Do not stack another group on a broken base.
- The artifacts are so contradictory that every path forward is a guess.

"Should I continue?" is not on the list. Neither is a progress summary between groups.

When you stop, leave the tree and the ledger as they are, and use `openspec-apply-change`'s pause format with the options you see.

## Final report

Before anything else, collect every ledger line containing `Ruling:` — preflight, parked findings, breaker adjudications, artifact amendments — into a **Rulings I made** list, in order, each with what it costs if wrong. The list is exhaustive. It is the only place the decisions you took on the user's behalf reach them.

Then, in `openspec-apply-change`'s completion format:

1. Progress `N/M`, the groups completed this run, and any task left unticked because of a parked gap.
2. Per group: fix rounds used, parked count, commit range.
3. Final review verdict, what the fix wave addressed, and every residual finding.
4. Artifact amendments made to `tasks.md` or `design.md`, with paths.
5. Deferred minors the final review left for later.
6. Build and test status from the last implementer report, and the branch with its commit range from `MERGE_BASE`.

When the result is `COMPLETE`, delete the workspace (`rm -rf "$WS"`) — git history and the report are the record now. Otherwise keep it; it is what a resumed run reads.

Never `openspec archive`, never `openspec sync`, never push. Suggest `openspec-verify-change` or `openspec-review-cycle` before archiving.

Finish with exactly one line, alone on the last line:

```
ORCHESTRATE_RESULT: COMPLETE
ORCHESTRATE_RESULT: PARTIAL
ORCHESTRATE_RESULT: HALTED
```

`COMPLETE` when every task is ticked and the final review left no Critical or Important residual. `PARTIAL` when the run finished but something is unticked, parked as a gap, or residual. `HALTED` when a stop condition ended the run early.

## Failure modes

| Excuse | Reality |
| --- | --- |
| "Each checkbox is a task, so each gets an agent" | That is the 60-agent run. The group is the unit. |
| "I'll fix it myself, dispatching is overhead" | Orchestrator fixes skip review and pollute your context. Resume the implementer. |
| "Close enough on spec compliance" | A spec gap means not done. Fix it, or hit the cap and adjudicate. |
| "One more round will converge" | Past the cap the failure is structural. Adjudicate and route. |
| "The scenario is impractical; I'll rule it out of scope" | Spec behaviour is the user's call. Stop and ask. |
| "This finding is obviously wrong, I'll drop it" | Adjudicate only at the cap, and every ruling is a ledger line. |
| "The fix was small, skip the re-review" | Unreviewed fixes are how regressions land. |
| "I remember which groups are done" | After compaction you do not. Read the ledger and `git log`. |
| "The implementer spawned its own reviewer — extra assurance" | It duplicated a review seat at full cost. Flag it; the group review is the gate. |
| "Two groups look independent; run them together" | Parallel implementers break the `BASE..HEAD` ranges every review depends on. |
