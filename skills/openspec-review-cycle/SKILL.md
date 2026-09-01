---
name: openspec-review-cycle
description: Run an automated review-and-fix loop on an OpenSpec change — Codex reviews the code and verifies the implementation against the change artifacts, then a subagent applies the findings — repeating until the review comes back clean or the cycle budget is spent, then always closing with one more review pass so the last round of fixes is verified rather than assumed. Use when asked to review, verify, and fix an OpenSpec change end to end, run a Codex review loop, or hand a code review to Codex and the fixes to Claude.
disable-model-invocation: true
---

# OpenSpec Review Cycle

Two-agent loop over one OpenSpec change:

1. **Codex reviews** (read-only) — code review plus spec verification in a single pass.
2. **A subagent applies** the findings with the `apply-code-review` skill.
3. Repeat until the review is clean, the loop stops converging, or the cycle budget runs out.
4. **Closing review** (read-only, no fixes) whenever the last apply pass changed the tree.

The invariant behind step 4: **an apply pass is never the last thing the loop does.** A cycle that ends in edits ends with an unverified claim — "fixes were applied" is not "the fixes worked". The closing review turns that claim into a verdict the user or an orchestrating agent can act on.

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
| Max cycles | `3` | One cycle = one review pass plus one apply pass. A closing review follows the last apply pass that changed the tree, so `N` cycles means up to `N + 1` review passes. `1` = one review, one apply, one closing review. Use review-only mode to skip applying entirely. |
| Codex model | unset | Omit `-m` so Codex uses the user's `~/.codex/config.toml` default. Pass whatever model string the user names through verbatim. |
| Reasoning effort | unset | When the user asks for more or less thinking, add `-c model_reasoning_effort=<none\|minimal\|low\|medium\|high\|xhigh>`. |
| Review scope | this branch's work vs its base | If the user states a scope in any form, use it. Only when they say nothing about scope, fall back to the base-branch default. See [Resolving the review scope](#resolving-the-review-scope). |
| Review-only | off | "just review", "no fixes", "dry run" → run cycle 1's review, report, stop before applying. No apply pass runs, so that review is already terminal — do not follow it with a redundant closing review. |

Echo the resolved settings in one line before the first cycle so the user can interrupt: `Change: X · max 3 cycles + closing review · model: config default · scope: git diff <resolved range>`.

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

If the harness provides a session scratchpad directory, use that instead. Every artifact of the loop — prompts, reviews, decision ledgers — goes there, named `cycle-<n>-prompt.md`, `cycle-<n>-review.md`, `cycle-<n>-decisions.md`, plus `closing-prompt.md` and `closing-review.md` for the closing pass. Report the path at the end.

Record the starting point once: `git rev-parse HEAD` and `git status --porcelain`. The final report compares against it.

Record the same two values again immediately **before each apply pass** — that is the pre-apply snapshot. Whether the closing review runs is decided by comparing the tree against that snapshot, not by trusting the ledger's Applied count: an apply pass can edit files while mis-reporting what it did.

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

Read whichever agent instruction files this repo has at its root — AGENTS.md, CLAUDE.md, or
whatever equivalent it uses — plus every guideline file they point to for the languages in
scope, and report violations of them as findings. If the repo has none, say so.

Scope: <the concrete range from "Resolving the review scope", plus any untracked files>.
Do not review code outside that range.

<the "How to write a finding" block below, verbatim>

You are read-only. Do not edit files, do not tick tasks, do not archive the change.
Report findings only.

Finish your final message with exactly one of these lines, alone on the last line:
REVIEW_STATUS: CLEAN
REVIEW_STATUS: FINDINGS
```

If `$code-review` does not resolve in this environment, replace that token with a brief: review the diff for readability and overengineering — vague naming, speculative abstractions, defensive checks against impossible states, duplicated helpers that already exist, error-handling theatre, single-call wrappers, scope creep — grouped under `## Must fix` / `## Should fix` / `## Consider`, one issue per bullet with `file:line`.

#### How to write a finding

Include this block verbatim in every review prompt — cycle 1, cycles 2+, and the closing review.

A review that names a defect without prescribing its repair costs a whole extra cycle, because the applying agent has to guess and the next review disagrees with the guess. A review that fixes one instance of a rule violation and misses its three siblings costs another. Most cycles past the first are spent on those two failures rather than on the code.

```
## How to write each finding — mandatory format

A separate agent, not you, will apply your findings. It cannot ask you questions, and it
applies what you write literally. Every finding must therefore be directly actionable.

For each finding give, in this order:
1. **Anchor** — `file:line` (exact path and line range).
2. **Problem** — one sentence saying what is wrong and which rule, requirement, or defect it
   is. Cite the agent-instruction clause or the spec requirement verbatim when the finding
   comes from one.
3. **Prescribed fix** — the exact change to make, as a fenced code block or unified diff, not
   a description of what should change. A rename gives the old and new identifier; a deletion
   names exactly what to delete.
4. **Done when** — a one-line check the applying agent can use to confirm the fix landed (a
   test name, a build outcome, an observable behavior).

**Report the class, not the instance.** Before you write a finding that cites a rule, search
the whole scope for every other place the same rule is broken and name them all in that one
finding. A rule finding that lists one of four occurrences guarantees another review cycle for
the remaining three. If you cannot sweep exhaustively, say so in the finding and give the
search you would run.

**Prescribe the whole edit.** Your diff must leave the symbol, member group, or test it
touches in a consistent state. If removing a member would break a caller or a test elsewhere
in scope, the diff removes or updates that too — do not hand over a fix that only compiles
after the applier improvises.

**Comment findings are delete-only.** You may say a comment is factually wrong, forbidden by
a named rule, or duplicates rationale that belongs elsewhere, and prescribe its deletion.
Never prescribe replacement prose: a paragraph you author this cycle is a paragraph the next
review argues with. Wording you merely dislike is not a finding.

**Style findings need a named rule.** Cite the clause, name an existing file that shows the
accepted pattern, and give the literal replacement text. A style observation that no rule in
this repo's instruction files supports is not a finding. Your own taste is not the standard.

If you cannot state the prescribed fix concretely, the finding is not ready — drop it, or put
it under `## Consider` and mark it explicitly as a judgement call the applier may decline.

Group findings under `## Must fix`, `## Should fix`, `## Consider`.

**A clean verdict is a valid, expected result.** Do not manufacture a finding to justify the
pass. If the scope conforms to the repo's rules and the change's requirements, say so and
report CLEAN.
```

The last paragraph is load-bearing. Some code-review skills append a footer telling the reviewer that every finding is in scope and that declining needs a stated reason; read alone, that biases the reviewer toward always emitting something. The explicit permission to return CLEAN is what stops a converged loop from grinding on for two more cycles of comment polish.

Cycles 2+ append the previous ledger so Codex reviews the *current* state with memory of what was decided. Keep the same `How to write a finding` block, and add:

```
This is review cycle <n> of <max>. The previous review's findings were actioned as follows:

<contents of cycle-<n-1>-decisions.md>

Re-review the current state of the change:
- Verify each applied fix is correct and introduced no new problems.
- For each declined finding, judge whether the stated reason holds. If it does, drop the finding. If it does not, raise it again with a direct rebuttal of the reason.
- Do not repeat deferred findings that were tracked somewhere.
- Raise anything new you now see — across the whole scope, not only near the lines the last pass edited. Verifying the previous fix is one step, not the frame for this review; a finding four lines from the last edit usually means the earlier pass under-read that region, so read each touched file whole before reporting.
- Text the previous cycle's review told the applier to write is your own output, not the author's work. Do not raise a finding against it unless it is factually wrong about the code — and then prescribe deleting it, never rewriting it.
- Any code you cite this cycle that was already present at cycle 1 means the earlier passes under-read that file. Read every file the previous fixes touched from top to bottom before reporting, so its siblings surface now rather than next cycle.
```

From cycle 3 on, add:

```
Cycles so far have spent findings on comment and documentation wording. Do not continue that.
Report a comment-level, naming-level, or wording-level finding only when it states something
factually wrong about the code or breaks a named clause in this repo's instruction files.
Prioritise defects in executable behaviour and conformance with the change's requirements.
If the change is now in good shape, report CLEAN — that is a valid result, not a failure.
```

### Step 2 — Read the verdict

Read `cycle-<n>-review.md`.

- `REVIEW_STATUS: CLEAN` → stop, success.
- `REVIEW_STATUS: FINDINGS` → continue to the apply pass.
- Status line missing → judge from the content. A review with no findings under any severity heading counts as clean. Note the missing line in the final report.

Before applying, resolve conflicts between the two halves of the review using the [priority rules](#when-the-two-reviews-disagree).

### Step 3 — Apply pass (subagent)

Take the pre-apply snapshot (see [Setup](#setup)), then spawn one subagent per cycle. Give it the review file path, not the review text pasted inline:

```
Use the apply-code-review skill.

The review is at <RUN_DIR>/cycle-<n>-review.md. It contains both code-review findings and
OpenSpec verification findings for the change "<change-name>". Read it and apply it to the
working tree.

Each finding carries a prescribed fix and a "Done when" check. Apply the prescribed fix as
written unless it is factually wrong against the current code; when you deviate, say exactly
how and why in the ledger.

Sweep for siblings. For every finding that cites a rule — a guideline clause, a convention, a
requirement — search the whole change for other places that break the same rule and fix those
too, whether or not the review anchored them. List the extras you fixed in the ledger under
the finding that prompted the sweep. A rule fixed at one of four call sites costs another
review cycle for the other three.

Report incomplete prescriptions. If a prescribed diff edits part of a symbol, member group, or
test and leaves a related part unmentioned — deleting two operators of six, updating one
overload of three — say so in the ledger rather than silently choosing. State what you did
with the unmentioned part and why. Do not extend a deletion past what the finding names when
extending it would break a passing test the review never mentioned; flag it instead.

Priority rule: the change artifacts under openspec/changes/<change-name>/ are authoritative.
Where a code-review finding contradicts a spec-verification finding, the spec wins — decline
the code-review finding and say which requirement it conflicts with.

You may tick completed items in tasks.md. Do not rewrite spec deltas, the proposal, or design
artifacts, and do not archive the change — if a finding says an artifact itself is wrong,
decline it and flag it for the user instead.

Before editing: read whichever agent instruction files this repo has at its root — AGENTS.md,
CLAUDE.md, or whatever equivalent it uses — plus every guideline file they point to for the
languages in this diff. This is a source-editing task, so any "only when the task requires
editing code" gate is satisfied. List the files you read in your report, or say the repo has
none.

Run the project's standard build and test command if AGENTS.md, CLAUDE.md, or the repo's
tooling defines one, and report the result.

Report in the skill's Applied / Declined / Deferred format and write that report to
<RUN_DIR>/cycle-<n>-decisions.md.
```

If the subagent does not write the ledger file, write it yourself from its response before starting the next cycle — cycle `n+1`'s prompt depends on it.

### Step 4 — Decide whether to loop

Each stop condition routes to one of two places: straight to the final report, or through the closing review in Step 5.

| Stop condition | Next |
| --- | --- |
| The review came back clean. | Final report — that review already describes the current tree. |
| `n == max cycles`. | **Step 5 — closing review.** |
| The apply pass applied **nothing** — everything was declined or deferred. Another review pass would surface the same list; report the standoff instead of burning a cycle. | Final report — the tree is unchanged, so there is nothing new to verify. |
| This cycle's findings are substantially the same as the previous cycle's and the fixes did not move them. The loop is stuck. | **Step 5 — closing review.** |
| **No finding this cycle touched executable code or a spec requirement.** When a whole cycle produced only comment wording and identifier renames, the loop has stopped finding defects and started polishing prose — a search with no natural end. Report the cosmetic findings as a list the user can take or leave. | **Step 5 — closing review.** |
| The build or tests broke and the subagent could not fix it. Stop with the working tree as-is — do not stack another round of edits on a broken tree. | Final report — halted. The build failure is the finding; re-reviewing a broken tree buys nothing. |

The rule in one sentence: **every exit other than clean, standoff, and broken-build halt goes through the closing review** — and if the tree is byte-identical to the pre-apply snapshot, skip the closing review regardless of which exit fired, because nothing changed for it to look at.

Otherwise increment `n` and go back to Step 1.

### Step 5 — Closing review (no apply pass)

Write the prompt below to `$RUN_DIR/closing-prompt.md`, then run the same invocation as Step 1 against its own artifacts:

```bash
codex exec \
  --sandbox read-only \
  --cd "$REPO_ROOT" \
  -o "$RUN_DIR/closing-review.md" \
  - < "$RUN_DIR/closing-prompt.md"
```

Same `-m` / `-c model_reasoning_effort` handling, same background execution, and the same failure handling: if Codex exits non-zero or the output is empty, report the exit code and the tail of stdout, do not retry blindly, and **do not report the change as verified** — an unrun closing review is `HALTED`, not `CLEAN`.

Prompt:

```
$code-review $openspec-verify-change <change-name>

This is the closing review for this change. The fix budget is spent — no fixes will be applied
after this review. Its purpose is to state whether the change is actually in good shape now.

The previous review's findings were actioned as follows:

<contents of cycle-<n>-decisions.md>

Re-review the current state over the full scope: <scope command>, plus these untracked files:
<list, or "none">.

Report under exactly these two headings:

## Unresolved
Findings from the previous review that the applied fixes did not actually resolve, that were
resolved incorrectly, or that the fixes broke in a new way. Say what remains for each.

## New
Anything else you now see in scope, including problems this last apply pass introduced.

Code review should not contradict spec verification. Specs have higher priority.

Read whichever agent instruction files this repo has at its root — AGENTS.md, CLAUDE.md, or
whatever equivalent it uses — plus every guideline file they point to for the languages in
scope, and report violations of them as findings. If the repo has none, say so.

<the "How to write a finding" block, verbatim>

You are read-only. Do not edit files, do not tick tasks, do not archive the change.

Finish your final message with exactly one of these lines, alone on the last line:
REVIEW_STATUS: CLEAN
REVIEW_STATUS: FINDINGS
```

The finding format matters most here. Nothing in this loop will fix what the closing review reports, so each finding has to be repairable by whoever picks it up later, without this run's context.

**The closing review is terminal.** It never triggers an apply pass and never starts another cycle, however severe its findings. A must-fix here goes into the final report for the user or the orchestrating agent to act on — do not quietly fix it, and do not extend the cycle budget on your own.

## When the two reviews disagree

Spec verification outranks code review. Concretely:

- Code review calls an abstraction speculative, but a requirement in the change spec demands it → keep it, decline the finding, cite the requirement.
- Code review wants a validation or error path removed as defensive, but a scenario in the spec covers that path → keep it.
- Verification says behavior is missing and code review says the surrounding code is overbuilt → implement the missing behavior first; re-judge the overbuild next cycle against the finished code.
- A finding implies the *spec* is wrong rather than the code → do not edit the spec inside this loop. Surface it in the final report as a proposal-level question for the user.

## Final report

State, in this order:

1. Outcome — clean, budget exhausted, stuck, or halted — after how many cycles, and the closing review's verdict: "budget exhausted after 3 cycles; closing review CLEAN" or "budget exhausted after 3 cycles; closing review found 2 must-fix". When no closing review ran, say which exception applied (clean exit, nothing applied, broken-build halt, or tree unchanged).
2. Per cycle: findings count by severity, then applied / declined / deferred counts.
3. Every finding still outstanding, with its reason — including the closing review's, kept under its `Unresolved` and `New` split so it is obvious which fixes failed to land and which problems the last apply pass introduced. This is the part the user acts on.
4. Any spec-level questions raised under the conflict rules above.
5. Build and test status from the last apply pass.
6. `git status --porcelain` and the diff stat versus the recorded starting point, plus the `$RUN_DIR` path.

Finish with exactly one machine-readable line, alone on the last line, so an orchestrating agent does not have to parse the prose:

```
CYCLE_RESULT: CLEAN
CYCLE_RESULT: OPEN_FINDINGS
CYCLE_RESULT: HALTED
```

`CLEAN` only when the terminal review — the last cycle review or the closing review, whichever ran last — came back clean. `OPEN_FINDINGS` when anything is outstanding, including declined and deferred findings. `HALTED` for a broken build or a Codex run that failed to produce a review.

Never run `openspec archive` or `openspec sync`, and never commit. The change stays open for the user to verify.

## Failure modes

- **Codex edits files anyway.** It cannot under `--sandbox read-only`. If the tree changed during a review pass, something else is running — stop and tell the user.
- **Findings ping-pong** — cycle 2 reverses cycle 1. Usually the two reviewers disagree on the same code. Stop the loop and put both positions in the report rather than letting the tree oscillate. The exception is a reversal whose fix is a *deletion* of text an earlier cycle prescribed: that settles the disagreement instead of continuing it, so apply it, forbid a third rewrite in the apply prompt, and note both positions in the final report.
- **Each cycle finds siblings of the last cycle's findings** — the same rule broken in a new file, the same class of defect one symbol over. The reviewer is reporting instances where it should report classes. Tighten the sweep instruction in the next review prompt and tell the apply pass to fix the class; do not just keep spending cycles.
- **A finding lands on code that was present at cycle 1 and nothing touched since.** The reviewer is sampling different regions each pass rather than covering the scope, so a clean verdict means "nothing found this pass", not "nothing there". Say that plainly in the final report instead of presenting a late clean review as full coverage.
- **The closing review raises a new must-fix.** Expected, not a bug — it means the change is not done. Report it under `CYCLE_RESULT: OPEN_FINDINGS`; do not spawn cycle `n+1`.
- **The closing review contradicts the last apply pass's own ledger** — the ledger says Applied, the review says unresolved. Trust the review: the ledger is self-reported. Put both positions in the report.
- **Review names a change that does not exist.** Codex picked a different change than intended. Re-resolve the name with `openspec list --json` and restart cycle 1.
- **Findings land on files this change never touched.** The base branch was resolved wrong and the diff swallowed the base's commits. Stop before applying anything, re-resolve the base, and restart cycle 1 — do not let the apply pass edit another branch's work.
- **Empty review output with a zero exit code.** Usually a model or auth problem on the Codex side. Report the tail of stdout; do not silently treat it as clean.
