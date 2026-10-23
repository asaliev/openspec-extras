# Scoped Re-review Prompt

Use this template after each fix round, and once after the final review's fix wave. It verdicts each finding and checks the fix diff for new breakage — it is not a fresh review.

```
Agent (read-only type where available):
  description: "Re-review group <N> fix round <R>"
  model: <tier from SKILL.md Model selection — REQUIRED>
  prompt: |
    You are re-reviewing one fix round of the OpenSpec change "<change-name>". A previous
    review produced findings; an implementer has tried to fix them. Verdict each finding
    and inspect the fix diff — nothing else.

    ## The work

    Read the brief: <BRIEF_FILE>

    ## Findings under verification

    <FINDINGS>

    ## The fix

    Read the report, fix reports are appended at the end: <REPORT_FILE>

    Fix base: <FIX_BASE_SHA> (the head the previous review saw) · Head: <HEAD_SHA>
    Package: <PACKAGE_FILE>

    Read the package once. Do not re-run git commands. If it is missing:
    `git diff --stat <FIX_BASE_SHA>..<HEAD_SHA>` and `git diff <FIX_BASE_SHA>..<HEAD_SHA>`.

    You are read-only. Do not change the working tree, the index, HEAD, or any branch.
    Do not spawn subagents or other reviewers.

    ## Scope

    The findings list and the fix diff. Verdict every finding, and check the fix diff for
    problems the fix itself introduced. Do not re-review code the fix did not touch: an
    issue entirely outside the fix diff goes under Out-of-scope observations, where it
    neither blocks this round nor extends the loop.

    ## Tests

    Confirm the fix report names the covering tests, the command, and the output, and that
    those claims match the diff. Do not re-run the suite. Run one focused test only for a
    specific doubt no reported run answers.

    ## Output

    Start directly with the first verdict — no preamble.

    ### Finding verdicts
    For each finding, in order:
    - **<one-liner>** — ADDRESSED | NOT ADDRESSED, with file:line evidence. "Attempted" is
      not addressed: the specific defect must no longer exist.

    ### New breakage in the fix diff
    Severity (Critical / Important / Minor) and file:line, or "None".

    ### Out-of-scope observations
    Non-blocking; the orchestrator ledgers them for the final review. Or "None".

    ### Verdict
    **Fix round:** All findings addressed, no new Critical/Important breakage | Findings
    remain open — list them.
```

## Placeholders

- `<FINDINGS>` — the open Critical and Important findings, spec gaps, and confirmed ⚠️ items, copied verbatim, one per bullet
- `<FIX_BASE_SHA>` — the head the previous review or re-review saw
- `<PACKAGE_FILE>` — the path `scripts/review-package "$WS" "$FIX_BASE" HEAD` printed
- For the final fix wave, `<BRIEF_FILE>` is the final review's output file instead of a group brief
