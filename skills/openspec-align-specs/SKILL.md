---
name: openspec-align-specs
description: Generate, regenerate, or align OpenSpec specifications from an existing codebase. Use when asked to create specs for brownfield code, infer capabilities from implemented behavior, update specs to match current implementation, report spec/code drift, or reconcile existing `openspec/specs/` with source code.
---

# OpenSpec Align Specs

Use this skill to recover behavior-first OpenSpec specs from existing code or to align existing specs with implemented behavior. Treat the current implementation as the source of truth unless the user explicitly asks for desired-state planning.

## Operating Modes

Start by detecting the OpenSpec state:

- **No existing capabilities**: If `openspec/` is absent, `openspec/specs/` is absent, or `openspec/specs/` contains no capability specs, generate baseline specs directly under `openspec/specs/<capability>/spec.md`. Also create a minimal `openspec/config.yaml` (see Baseline Generation) so the workspace is valid for `openspec validate --all` and the rest of the CLI. Do not create agent files, slash-command files, or change artifacts unless the user asks for full OpenSpec initialization (that is `openspec init`'s job).
- **Existing capabilities**: If specs already exist, only write changes when the user explicitly asks to align, update, regenerate, or repair specs. If the request is vague, prompt before modifying.
- **Workspace**: If the current folder is an OpenSpec coordination workspace rather than a repo-local project, perform best-effort read-only analysis and report recommendations in chat. Do not write workspace specs or linked-repo files in v1.

User instructions override these defaults. If the user asks for no questions, draft-only output, or a narrower scope, follow that and record assumptions in the final chat summary.

## Discovery Workflow

Analyze deeply before writing:

1. Read `openspec/config.yaml` (including its `context:` block, which is where current OpenSpec keeps project conventions, and its optional `rules:` block, which holds per-artifact rules keyed by artifact id such as `specs:`, `tasks:`, `design:` — respect both when generating or aligning specs), legacy `openspec/project.md` if present, existing specs, active changes under `openspec/changes/`, and repository agent instructions (`AGENTS.md`, `CLAUDE.md`, etc.).
2. Identify public behavior from routes, commands, UI flows, public APIs, jobs, tests, schemas, config, docs, and integration boundaries.
3. Group behavior into focused OpenSpec capabilities, not source files and not broad domains. Prefer flat kebab-case names for a single public behavior contract, such as `cli-init`, `cli-update`, `config-loading`, `rules-injection`, `workspace-open`, or `data-export`.
4. Treat existing `openspec/specs/<capability>/` folders as authoritative when modifying specs. For new specs, choose names that match the smallest independently reviewable contract.
5. Split broad areas when sub-behaviors have distinct lifecycle, users, commands, APIs, validation gates, or likely future-change ownership. Merge behaviors only when they always change together and do not have independently testable contracts.
6. Avoid umbrella capabilities such as `cli-commands`, `api`, `frontend`, or `authentication` when the implementation exposes separately evolving behaviors. Avoid file-by-file specs unless the file is itself the public surface.
7. Distinguish externally observable behavior from implementation detail. Keep libraries, functions, class structure, and execution mechanics out of requirements unless they are part of the public contract.
8. Ask only blocking questions when code evidence cannot support a materially correct spec.

Use existing tests and examples as strong evidence, but do not require test coverage before documenting implemented behavior.

## Baseline Generation

Use this path when no capabilities exist under `openspec/specs/`.

- If `openspec/config.yaml` is missing, create it with the minimum needed for CLI validity:
  ```yaml
  schema: spec-driven
  ```
  Mention in the final chat summary that running `openspec init` later will add AI-tool integrations (skills and slash commands) without touching the specs you wrote.
- Write one `openspec/specs/<capability>/spec.md` per focused capability.
- Generate enough focused specs to cover independently changeable public behavior. Do not collapse unrelated behavior just to minimize spec count.
- Do not create `openspec/changes/` entries during baseline generation. (Changes are for proposed edits to existing specs; recovering specs that never existed is not a change.)
- Do not invent a `design.md` alongside the capability spec. Upstream OpenSpec allows an optional `openspec/specs/<capability>/design.md` for established HOW patterns, but baseline generation should only recover the behavioral contract; leave design docs for the user or a later schema-driven flow.
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

Every requirement must include at least one `#### Scenario:` block. Scenarios may also use `- **GIVEN**` for preconditions and `- **AND**` for additional conditions or outcomes (BDD-style). Example:

```markdown
#### Scenario: Running with --tools all
- **GIVEN** the user is in a fresh project
- **WHEN** `openspec init --tools all` is executed
- **THEN** every supported AI tool is configured
- **AND** no interactive prompts appear
```

## Alignment Changes

Use this path when existing specs are present and the user asks to align, update, regenerate, or repair them.

1. Read `openspec/config.yaml` and resolve the active schema. If a custom schema is active, inspect its schema file (usually `openspec/schemas/<schema>/schema.yaml`) and artifact instructions/templates before writing.
2. Follow the local OpenSpec-generated workflow for creating a change with that schema. Prefer the workflow installed by the user's agent integration — typically the `openspec-propose` skill for the core one-shot flow, or the OpenSpec new/continue change workflow for the expanded flow. Discover the local invocation form (skill, slash command, or other) from the installed integration; do not assume a fixed file path or command name.
3. Create a change using an alignment intent such as "align current specs with implemented behavior; docs-only unless the schema's workflow says otherwise." Let the OpenSpec skill/schema decide artifact names, dependency order, and whether artifacts such as proposal, design, tasks, research, or specs exist.
4. Put drift evidence into the schema-supported artifact that plays that role. Do not create `proposal.md`, `design.md`, `tasks.md`, `drift.md`, or `specs/<capability>/spec.md` unless the active schema or followed OpenSpec skill calls for that artifact.
5. When the schema includes spec deltas, use exact existing capability folder names for modified capabilities. For missing implemented behavior, propose focused capability names in the schema's appropriate rationale/scope artifact.
6. For existing specs that are too broad, record a split recommendation in the schema's appropriate decision/rationale artifact instead of silently rewriting capability boundaries unless the user requested regeneration.
7. Update requirements to match current implemented behavior. Under the default `spec-driven` schema, change-folder spec files use `## ADDED Requirements` / `## MODIFIED Requirements` / `## REMOVED Requirements` / `## RENAMED Requirements` section headers. Place the complete updated requirement under `## MODIFIED Requirements`. Use `## RENAMED Requirements` with explicit `- FROM: \`### Requirement: Old Name\`` / `- TO: \`### Requirement: New Name\`` lines when a requirement name changes; if the content also changes, include the new version under `## MODIFIED Requirements` using the new header. Follow whatever delta style the active schema defines if it differs.

Capture these drift facts somewhere only if the active schema has an appropriate artifact for them:

- analyzed code areas and existing specs
- implemented capability, existing spec status, and action needed
- missing, stale, renamed, over-specified, or unsupported-by-code requirements
- capability boundary recommendations such as split, merge, rename, or add
- source evidence references by file path and symbol, route, command, test, or UI flow
- open questions that code evidence could not resolve

If the schema has no suitable artifact for a drift fact, report it in the final chat summary instead of inventing a file.

## After Writing the Change

The alignment change is not finished when the files land. State the following in chat:

- The change folder exists but is **unmerged** — `openspec/specs/` is unchanged so far.
- Next step for the user: follow the active OpenSpec workflow for that schema, usually the generated apply/sync/archive skill or command. Name the exact next command or skill only after reading the local OpenSpec guidance.
- Do not run `archive` or `sync` automatically — those are user-driven steps that may need verification first.

## Validation

Before finishing:

- Run `openspec validate --all` when the CLI is available.
- If validation cannot run, state why.
- Self-check that each capability is focused, flat kebab-case, and externally observable; every requirement has at least one scenario; scenario headings use exactly `#### Scenario:`; scenarios use the BDD bullets `- **GIVEN**` / `- **WHEN**` / `- **THEN**` / `- **AND**`; and requirements use SHALL/MUST language.
- For alignment mode, check that every artifact you created is required or allowed by the active schema and follows the local OpenSpec-generated skill instructions.
- If the schema includes spec deltas, verify they follow that schema's delta format and do not use baseline spec structure unless the schema says to. Under the default `spec-driven` schema, spec deltas use `## ADDED Requirements` / `## MODIFIED Requirements` / `## REMOVED Requirements` / `## RENAMED Requirements` section headers.

Report the files created or changed, validation results, the unmerged-change handoff note, and any remaining uncertainty.
