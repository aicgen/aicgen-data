# Agentic Capability Profiles

This directory documents how AICGEN maps reusable guideline content into modern AI coding tool surfaces.

Gemini CLI is no longer an active generated target. Use Antigravity for Google-side agentic coding profiles.

## Profile Levels

- `basic`: main repository instructions and stable rule files only.
- `standard`: basic output plus reusable workflows, commands, and prompt files.
- `full`: standard output plus focused agents, skills, guardrail hooks, plugin packaging, and advanced integration templates such as MCP.

Validation stays in the lifecycle commands: `/check` is responsible for running or asking for the relevant test suite. Generated hooks should never invoke full test suites automatically.

## Risk Levels

- `passive`: instructions or rules only.
- `guided`: reusable prompts, commands, or workflows that users invoke.
- `agentic`: subagents, skills, or plugin-packaged behavior selected by the assistant.
- `side-effecting`: hooks, MCP, setup scripts, or executable tool configuration.

Side-effecting capabilities must stay local, deterministic, and explicitly reviewed before use.
