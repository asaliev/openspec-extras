---
name: openspec-align-specs
description: Generate, regenerate, or align OpenSpec specifications from an existing codebase. Use when asked to create specs for brownfield code, infer capabilities from implemented behavior, update specs to match current implementation, produce an OpenSpec drift report, or reconcile existing `openspec/specs/` with source code.
---

# OpenSpec Align Specs

Use this skill to recover behavior-first OpenSpec specs from existing code or to align existing specs with implemented behavior. Treat the current implementation as the source of truth unless the user explicitly asks for desired-state planning.

## Operating Modes

Start by detecting the OpenSpec state:

- **No existing capabilities**: If `openspec/` is absent, `openspec/specs/` is absent, or `openspec/specs/` contains no capability specs, generate baseline specs directly under `openspec/specs/<capability>/spec.md` without prompting. Do not create config, agent files, or change artifacts unless the user asks for full OpenSpec initialization.
- **Existing capabilities**: If specs already exist, only write changes when the user explicitly asks to align, update, regenerate, or repair specs. If the request is vague, prompt before modifying.
- **Workspace**: If the current folder is an OpenSpec workspace rather than a repo-local project, perform best-effort read-only analysis and report recommendations in chat. Do not write workspace specs or linked-repo files in v1.

User instructions override these defaults. If the user asks for no questions, draft-only output, or a narrower scope, follow that and record assumptions.

## Discovery Workflow

Analyze deeply before writing:

1. Read `openspec/config.yaml`, `openspec/project.md`, existing specs, active changes, and repository agent instructions.
2. Identify public behavior from routes, commands, UI flows, public APIs, jobs, tests, schemas, config, docs, and integration boundaries.
3. Group behavior into user-visible capabilities, not source files. Prefer names such as `authentication`, `billing`, `data-export`, `cli-commands`, or domain workflow names.
4. Distinguish externally observable behavior from implementation detail. Keep libraries, functions, class structure, and execution mechanics out of requirements unless they are part of the public contract.
5. Ask only blocking questions when code evidence cannot support a materially correct spec.

Use existing tests and examples as strong evidence, but do not require test coverage before documenting implemented behavior.

## Baseline Generation

Use this path when no capabilities exist under `openspec/specs/`.

- Write one `openspec/specs/<capability>/spec.md` per user-visible capability.
- Do not create `openspec/changes/`, `drift.md`, or persistent evidence files.
- Keep each spec concise enough to be maintainable, but complete enough to guide future changes.
- Mention important assumptions in the final response, not inside `spec.md`, unless the assumption is part of the behavioral contract.

Spec format:

```markdown
# <Capability Name> Specification

## Purpose

<One concise paragraph describing the capability.>

## Requirements

### Requirement: <Short behavior name>
The system SHALL <externally observable behavior>.

#### Scenario: <Concrete scenario>
- **WHEN** <trigger or condition>
- **THEN** <expected outcome>
```

Every requirement must include at least one `#### Scenario:` block.

## Alignment Changes

Use this path when existing specs are present and the user asks to align, update, regenerate, or repair them.

1. Create a new change under `openspec/changes/<alignment-id>/`.
2. Read the configured schema from `openspec/config.yaml`:
   - If `schema: <name>` is present, follow that schema's artifact names, dependency order, templates, and instructions.
   - If no schema is configured, use the default OpenSpec `spec-driven` flow.
3. Add `drift.md` inside the change directory.
4. Generate schema-required artifacts for the alignment change. Do not hardcode `proposal.md`, `tasks.md`, or spec delta structure when a custom schema is configured.
5. For spec deltas, update requirements to match current implemented behavior. Include complete modified requirements, not partial diffs.

`drift.md` should include:

- Summary of analyzed code areas and existing specs.
- Capability coverage table: implemented capability, existing spec status, action needed.
- Requirement-level drift: missing, stale, renamed, over-specified, or unsupported-by-code.
- Source evidence references by file path and symbol, route, command, test, or UI flow.
- Open questions that could not be resolved from code.

Keep `spec.md` files focused on requirements and scenarios. Put implementation evidence in `drift.md`, not inline in specs.

## Validation

Before finishing:

- Run `openspec validate --all` when the CLI is available.
- If validation cannot run, state why.
- Self-check that each capability is user-visible, every requirement has at least one scenario, scenario headings use exactly `#### Scenario:`, and requirements use SHALL/MUST language.
- For alignment mode, check that `drift.md` explains why each changed or missing requirement was identified.

Report the files created or changed, validation results, and any remaining uncertainty.
