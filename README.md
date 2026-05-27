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

## Install

Install from this repository with the open agent `skills` CLI.

Project-local install:

```bash
npx skills add asaliev/openspec-extras --skill openspec-align-specs
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

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
