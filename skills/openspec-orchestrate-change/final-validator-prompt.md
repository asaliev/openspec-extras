# Final Validator Prompt

Use this template once, after the final review returns findings. It decides which findings hold up. Every finding that holds is fixed before the run ends, whatever its severity; every finding that does not is closed. Nothing is deferred. Dispatch it on the most capable tier.

```
Agent (read-only type where available):
  description: "Validate final review of <change-name>"
  model: <most capable tier — REQUIRED>
  prompt: |
    You are validating the final review of the OpenSpec change "<change-name>". A reviewer
    has reported findings against the whole branch. You decide which of them hold up.
    Every finding you confirm will be fixed before the change is done, and every finding
    you reject is closed. Nothing is deferred, so your verdict is the decision.

    ## Inputs

    - Final review: <FINAL_REVIEW_FILE>
    - Branch diff: <PACKAGE_FILE> (<MERGE_BASE>..<HEAD>). Read it once; do not re-run git
      commands for the same range.
    - Change artifacts — read all of them: <every contextFiles path>
    - Orchestration ledger: <LEDGER_FILE>. Its `Ruling:` lines are decisions already taken;
      its `minor (deferred)` and `parked` lines are the source of the review's ledger items.
    - The repo's agent instruction files — AGENTS.md, CLAUDE.md, or equivalent — and the
      guideline files they point to for the languages in the diff.

    You are read-only. Do not change the working tree, the index, HEAD, or any branch.
    Do not spawn subagents.

    ## What you validate

    Every finding in the review, at every severity: Spec verification CRITICAL, WARNING,
    and SUGGESTION; Branch review Critical, Important, and Minor; every ledger item the
    review reported as still present; every Ruling it challenged. Merge duplicates first —
    two findings about the same defect are one.

    ## How to judge

    The reviewer's descriptions, severities, and reasoning are claims. Check each against
    the code at HEAD yourself: open the file, read the lines, follow the call that matters.
    A verdict that does not cite a file:line you read is not a verdict.

    A finding HOLDS when all four are true:
    1. **It is in the code.** What it describes exists at HEAD — not a misreading, not
       already handled elsewhere. If it is handled elsewhere, cite where.
    2. **It is a defect, not a preference.** Wrong or fragile behaviour; a requirement or
       scenario missed or mis-implemented; a testable scenario with no test; dead code,
       duplication, or noise the branch introduced; a breach of a named clause in the
       repo's instruction files. A style point with no named rule is a preference.
    3. **It belongs to this change.** Code the branch added or changed, or behaviour the
       change's specs describe.
    4. **Fixing it agrees with the artifacts.** Authority order: spec deltas > design >
       tasks > reviewer opinion.

    Size is not validity. Do not reject a finding because it is small, because the fix is
    tedious, or because the code works well enough as is — a small real defect HOLDS.
    Severity is not validity either: a Critical that misreads the code is REJECTED.
    Rejecting a real finding ships a defect; confirming a false one sends an implementer to
    change correct code. Neither error is the safe default.

    A challenged Ruling is judged the same way: it HOLDS when the ruling is wrong and the
    code has to change; it is REJECTED when the ruling stands.

    ## Verdicts

    - **HOLDS** — all four conditions are true.
    - **REJECTED** — condition 1 or 2 fails. Quote the code that refutes it.
    - **ARTIFACT CONFLICT** — the defect is real, but condition 4 fails: a spec, design, or
      task clause mandates what the finding objects to, or the fix would add, narrow, or
      drop specified behaviour. Name the artifact and clause. The orchestrator rules on it.
    - **OUTSIDE CHANGE** — the defect is real, but condition 3 fails: pre-existing code the
      branch did not touch, behaviour no requirement of this change covers.

    ## Writing up what holds

    An implementer who has not seen the review will fix each held finding from your text
    alone, and a re-reviewer will check it against your "Done when". For each:
    1. **Anchor** — file:line. For a finding that breaks a rule or repeats a defect, sweep
       the diff for every other instance and list them all in this one finding; an
       instance left out is an instance left unfixed.
    2. **Problem** — one sentence: what is wrong, and the requirement, scenario, or
       instruction clause it breaks.
    3. **Done when** — one observable check: a named test passes, a behaviour holds, a
       search comes back empty.

    Do not write the fix — no patches, no replacement code. The implementer owns the how.
    A rename is the exception: give the old and the new name.

    ## Output

    Start directly with the first heading — no preamble.

    ### Holds
    Numbered, most severe first. Each: Anchor / Problem / Done when, and the review
    finding(s) it came from.

    ### Rejected
    One line each: the finding, and the file:line evidence that refutes it.

    ### Artifact conflicts
    Each: the finding, the artifact clause, and what each side requires.

    ### Outside change
    One line each: the finding, where it lives, and why it is outside the change.

    Finish with exactly one line, alone on the last line:
    VALIDATION: <H> hold · <R> rejected · <C> conflict · <O> outside
```

## Placeholders

- `<FINAL_REVIEW_FILE>` — `$WS/final-review.md`
- `<PACKAGE_FILE>` — the same package the final reviewer read
- `<LEDGER_FILE>` — `$WS/progress.md`
