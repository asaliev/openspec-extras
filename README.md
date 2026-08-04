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

### `openspec-review-cycle`

Run an automated review-and-fix loop on a single OpenSpec change.

Each cycle is two agents with separate jobs:

- **Codex** runs a code review and `openspec-verify-change` in one read-only pass
- a **subagent** applies the findings with the `apply-code-review` skill

The loop repeats until the review comes back clean, stops converging, or the cycle budget runs out (3 by default). Codex runs under `--sandbox read-only`, so the reviewer can never edit the code it reviews. Spec verification outranks code review whenever the two disagree.

Requires the Codex CLI, plus a code-review skill and `openspec-verify-change` on the Codex side.

## Install

Install from this repository with the open agent `skills` CLI. Swap `--skill` for whichever skill you want, or pass it twice to install both.

Project-local install:

```bash
npx skills add asaliev/openspec-extras --skill openspec-align-specs
npx skills add asaliev/openspec-extras --skill openspec-review-cycle
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

### `openspec-review-cycle`

Example prompts:

```text
Use openspec-review-cycle on trace-context-over-rabbit.
```

```text
Run the review cycle on trace-context-over-rabbit, unstaged changes only, max 2 cycles.
```

```text
Review-cycle add-identomat-session-result-read with gpt-5.6-sol at high effort.
```

Expected behavior:

- The change name is resolved from `openspec list --json`; the agent asks rather than guessing when it is ambiguous.
- Scope works the way the `code-review` skill's does: say "unstaged changes", "staged", "the last two commits", "vs develop", or a SHA and it uses exactly that.
- With no scope stated, it defaults to the merge base with the branch's actual base — resolved from the open PR or by asking, never assumed to be `main` — so stacked feature branches don't drag the base branch's commits into the review.
- Spec verification always covers the whole change regardless of how the code review is scoped; findings from outside the reviewed range are marked as such in the report.
- Codex reviews under `--sandbox read-only` and ends its report with `REVIEW_STATUS: CLEAN` or `REVIEW_STATUS: FINDINGS`.
- Findings go to a subagent running `apply-code-review`, which reports Applied / Declined / Deferred.
- Later cycles carry the previous decisions into the review prompt, so Codex re-judges declines instead of repeating them.
- The loop stops early when the review is clean, nothing was applied, findings stop moving, or the build breaks.
- Prompts, reviews, and decision ledgers are written to a run directory outside the repo, and the change is never archived, synced, or committed.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
