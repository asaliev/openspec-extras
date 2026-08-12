---
name: openspec-review-cycle
description: Run an automated review-and-fix loop on an OpenSpec change — Codex reviews the code and verifies the implementation against the change artifacts, then a subagent applies the findings — repeating until the review comes back clean or the cycle budget is spent. Use when asked to review, verify, and fix an OpenSpec change end to end, run a Codex review loop, or hand a code review to Codex and the fixes to Claude.
---

# OpenSpec Review Cycle

Two-agent loop over one OpenSpec change:

1. **Codex reviews** (read-only) — code review plus spec verification in a single pass.
2. **A subagent applies** the findings with the `apply-code-review` skill.
3. Repeat until the review is clean, the loop stops converging, or the cycle budget runs out.

Codex never writes. The applying subagent never reviews. Keeping those roles separate is the point of the skill — do not collapse them.

## Preconditions

Check these before starting; stop and report if one fails.

- `codex --version` succeeds. If not: the loop cannot run, tell the user to install the Codex CLI.
- The working directory is a git repo with an `openspec/` root (or the user named an OpenSpec store).
- `openspec --version` succeeds. If not, skip every `openspec` command below and let Codex's skill handle discovery.
- The Codex side has a code-review skill and an `openspec-verify-change` skill available. If `$code-review` does not resolve in this environment, fall back to the inline review brief in [Review prompt](#review-prompt).
- The `apply-code-review` skill is available to subagents. If not, inline its decision procedure (Applied / Declined with reason / Deferred with reason) into the subagent prompt.

## Inputs

Parse these from the user's request. Ask only about the change name, and only when it cannot be resolved.

| Input | Default | Notes |
| --- | --- | --- |
| Change name | none — must resolve | Run `openspec list --json`; if the request names one change unambiguously use it, otherwise use **AskUserQuestion** to let the user pick. Never guess. |
| Max cycles | `3` | One cycle = one review pass plus one apply pass. `1` still applies findings; use review-only mode to skip applying. |
| Codex model | unset | Omit `-m` so Codex uses the user's `~/.codex/config.toml` default. Pass whatever model string the user names through verbatim. |
| Reasoning effort | unset | When the user asks for more or less thinking, add `-c model_reasoning_effort=<none\|minimal\|low\|medium\|high\|xhigh>`. |
| Review scope | this branch's work vs its base | If the user states a scope in any form, use it. Only when they say nothing about scope, fall back to the base-branch default. See [Resolving the review scope](#resolving-the-review-scope). |
| Review-only | off | "just review", "no fixes", "dry run" → run cycle 1's review, report, stop before applying. |

Echo the resolved settings in one line before the first cycle so the user can interrupt: `Change: X · max 3 cycles · model: config default · scope: git diff <resolved range>`.

### Resolving the review scope

**A stated scope always wins.** Accept the same range of references the code-review skill accepts, in the user's own words, and translate it to a concrete git command without asking anything:

| The user says | Scope command |
| --- | --- |
| "unstaged changes", "working tree", "what I haven't committed" | `git diff` |
| "staged", "what's staged", "the index" | `git diff --staged` |
| "everything uncommitted" | `git diff HEAD` |
| "the last N commits" | `git diff HEAD~N` |
| "this commit", a SHA | `git show <sha>` |
| "vs develop", "against feature/foo" | `git diff $(git merge-base <ref> HEAD)` |
| a commit range | `git diff <range>` as given |

Do not resolve a base branch, consult `gh`, or ask a question in any of these cases. "Unstaged changes" means unstaged changes.

**Only when the user says nothing about scope** does the base-branch default apply. Feature branches are often cut from another feature branch, not from `main`, and diffing against the wrong base pulls the base branch's commits into the review — the loop then reviews, and tries to "fix", code that is not part of this change. Resolve the base in this order and stop at the first that answers:

1. **An open PR for the current branch.** `gh pr view --json baseRefName -q .baseRefName` when `gh` is available. This is the most reliable signal.
2. **A single plausible candidate.** List local branches by recency with `git for-each-ref --sort=-committerdate --format='%(refname:short)' refs/heads` and check which ones contain `HEAD`'s fork point. If exactly one non-current branch is an ancestor of `HEAD`, propose it.
3. **Ask.** Use **AskUserQuestion** with the recent-branch list. Do not fall back to `main` silently.

Then diff from the merge base, which covers committed and uncommitted work in one range:

```bash
BASE_REF="$(git merge-base <base-branch> HEAD)"
git diff --stat "$BASE_REF"          # sanity-check the size before reviewing
```

If that stat touches files that obviously belong to the base branch's work rather than this change, the base is wrong — re-resolve it.

**In every case**, run `git status --porcelain` and name any untracked files explicitly in the review prompt. No `git diff` form shows them, and new files are exactly where a fresh change lives.

Put the result in the review prompt as a concrete command, not as prose:

```
Scope: everything in `<scope command>`, plus these untracked files: <list, or "none">.
Do not review code outside that range.
```

### Scope applies to the code review, not to spec verification

`openspec-verify-change` is scoped by the change artifacts, not by a diff — it checks the whole change against its specs and tasks however you narrow the code review. That is intentional: a narrow scope should not let a half-implemented requirement pass unnoticed.

The consequence to expect: when you scope to something small like unstaged changes, verification findings can point at code outside that scope. Apply them anyway — specs outrank the diff boundary — but say so in the final report, marking which findings came from outside the reviewed range so the user is not surprised by edits to files they did not ask about.

## Setup

Create a run directory outside the repo so nothing lands in the user's tree:

```bash
RUN_DIR="${TMPDIR:-/tmp}/openspec-review-cycle/<change>-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RUN_DIR"
```

If the harness provides a session scratchpad directory, use that instead. Every artifact of the loop — prompts, reviews, decision ledgers — goes there, named `cycle-<n>-prompt.md`, `cycle-<n>-review.md`, `cycle-<n>-decisions.md`. Report the path at the end.

Record the starting point once: `git rev-parse HEAD` and `git status --porcelain`. The final report compares against it.

## The cycle

### Step 1 — Review pass (Codex)

Write the prompt to `$RUN_DIR/cycle-<n>-prompt.md`, then run Codex read-only, feeding the prompt on stdin so nothing has to be shell-quoted:

```bash
codex exec \
  --sandbox read-only \
  --cd "$REPO_ROOT" \
  -o "$RUN_DIR/cycle-<n>-review.md" \
  - < "$RUN_DIR/cycle-<n>-prompt.md"
```

Add `-m <model>` and `-c model_reasoning_effort=<level>` only when the user asked for them.

`--sandbox read-only` is not optional. It is the guarantee that the reviewer cannot edit the code it is reviewing.

Run this in the background — a thorough review routinely exceeds ten minutes — and wait for it to finish rather than polling on a short timer. If Codex exits non-zero or `cycle-<n>-review.md` is empty, do not retry blindly: report the exit code and the tail of stdout, and stop.

#### Review prompt

Cycle 1:

```
$code-review $openspec-verify-change <change-name>

Code review should not contradict spec verification. Specs have higher priority.

Scope: <the concrete range from "Resolving the review scope", plus any untracked files>.

You are read-only. Do not edit files, do not tick tasks, do not archive the change.
Report findings only.

Finish your final message with exactly one of these lines, alone on the last line:
REVIEW_STATUS: CLEAN
REVIEW_STATUS: FINDINGS
```

If `$code-review` does not resolve in this environment, replace that token with a brief: review the diff for readability and overengineering — vague naming, speculative abstractions, defensive checks against impossible states, duplicated helpers that already exist, error-handling theatre, single-call wrappers, scope creep — grouped under `## Must fix` / `## Should fix` / `## Consider`, one issue per bullet with `file:line`.

Cycles 2+ append the previous ledger so Codex reviews the *current* state with memory of what was decided:

```
This is review cycle <n> of <max>. The previous review's findings were actioned as follows:

<contents of cycle-<n-1>-decisions.md>

Re-review the current state of the change:
- Verify each applied fix is correct and introduced no new problems.
- For each declined finding, judge whether the stated reason holds. If it does, drop the finding. If it does not, raise it again with a direct rebuttal of the reason.
- Do not repeat deferred findings that were tracked somewhere.
- Raise anything new you now see — across the whole scope, not only near the lines the last pass edited. Verifying the previous fix is one step, not the frame for this review; a finding four lines from the last edit usually means the earlier pass under-read that region, so read each touched file whole before reporting.
```

### Step 2 — Read the verdict

Read `cycle-<n>-review.md`.

- `REVIEW_STATUS: CLEAN` → stop, success.
- `REVIEW_STATUS: FINDINGS` → continue to the apply pass.
- Status line missing → judge from the content. A review with no findings under any severity heading counts as clean. Note the missing line in the final report.

Before applying, resolve conflicts between the two halves of the review using the [priority rules](#when-the-two-reviews-disagree).

### Step 3 — Apply pass (subagent)

Spawn one subagent per cycle. Give it the review file path, not the review text pasted inline:

```
Use the apply-code-review skill.

The review is at <RUN_DIR>/cycle-<n>-review.md. It contains both code-review findings and
OpenSpec verification findings for the change "<change-name>". Read it and apply it to the
working tree.

Priority rule: the change artifacts under openspec/changes/<change-name>/ are authoritative.
Where a code-review finding contradicts a spec-verification finding, the spec wins — decline
the code-review finding and say which requirement it conflicts with.

You may tick completed items in tasks.md. Do not rewrite spec deltas, the proposal, or design
artifacts, and do not archive the change — if a finding says an artifact itself is wrong,
decline it and flag it for the user instead.

Run the project's standard build and test command if AGENTS.md, CLAUDE.md, or the repo's
tooling defines one, and report the result.

Report in the skill's Applied / Declined / Deferred format and write that report to
<RUN_DIR>/cycle-<n>-decisions.md.
```

If the subagent does not write the ledger file, write it yourself from its response before starting the next cycle — cycle `n+1`'s prompt depends on it.

### Step 4 — Decide whether to loop

Stop after the apply pass when any of these holds:

- The review came back clean.
- `n == max cycles`.
- The apply pass applied **nothing** — everything was declined or deferred. Another review pass would surface the same list; report the standoff instead of burning a cycle.
- This cycle's findings are substantially the same as the previous cycle's and the fixes did not move them. The loop is stuck; stop and say so.
- **No finding this cycle touched executable code or a spec requirement.** When a whole cycle produced only comment wording and identifier renames, the loop has stopped finding defects and started polishing prose — a search with no natural end. Stop and report the cosmetic findings as a list the user can take or leave, rather than spending another review pass and another full build to chase them.
- The build or tests broke and the subagent could not fix it. Stop with the working tree as-is and report — do not stack another round of edits on a broken tree.

Otherwise increment `n` and go back to Step 1.

## When the two reviews disagree

Spec verification outranks code review. Concretely:

- Code review calls an abstraction speculative, but a requirement in the change spec demands it → keep it, decline the finding, cite the requirement.
- Code review wants a validation or error path removed as defensive, but a scenario in the spec covers that path → keep it.
- Verification says behavior is missing and code review says the surrounding code is overbuilt → implement the missing behavior first; re-judge the overbuild next cycle against the finished code.
- A finding implies the *spec* is wrong rather than the code → do not edit the spec inside this loop. Surface it in the final report as a proposal-level question for the user.

## Final report

State, in this order:

1. Outcome — clean, budget exhausted, stuck, or halted — and after how many cycles.
2. Per cycle: findings count by severity, then applied / declined / deferred counts.
3. Every finding still outstanding, with its reason. This is the part the user acts on.
4. Any spec-level questions raised under the conflict rules above.
5. Build and test status from the last apply pass.
6. `git status --porcelain` and the diff stat versus the recorded starting point, plus the `$RUN_DIR` path.

Never run `openspec archive` or `openspec sync`, and never commit. The change stays open for the user to verify.

## Failure modes

- **Codex edits files anyway.** It cannot under `--sandbox read-only`. If the tree changed during a review pass, something else is running — stop and tell the user.
- **Findings ping-pong** — cycle 2 reverses cycle 1. Usually the two reviewers disagree on the same code. Stop the loop and put both positions in the report rather than letting the tree oscillate.
- **Review names a change that does not exist.** Codex picked a different change than intended. Re-resolve the name with `openspec list --json` and restart cycle 1.
- **Findings land on files this change never touched.** The base branch was resolved wrong and the diff swallowed the base's commits. Stop before applying anything, re-resolve the base, and restart cycle 1 — do not let the apply pass edit another branch's work.
- **Empty review output with a zero exit code.** Usually a model or auth problem on the Codex side. Report the tail of stdout; do not silently treat it as clean.
