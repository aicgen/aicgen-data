# SDLC Workflows

aicgen injects a structured 6-command SDLC workflow into every generated assistant configuration. These commands guide the AI assistant through a repeatable, artifact-driven development lifecycle.

## The Flow

```
/spec → /research → /plan → /build → /check → /ship
```

For Codex, aicgen generates a project-local `aicgen-sdlc` plugin and exposes
namespaced commands to avoid conflicts with built-in Codex commands:

```
/aicgen-spec → /aicgen-research → /aicgen-plan → /aicgen-build → /aicgen-check → /aicgen-ship
```

| Step | Command | Output artifact |
|------|---------|----------------|
| 1 | [`/spec [name]`](sdlc/spec.md) | `docs/specs/{name}.md` |
| 2 | [`/research`](sdlc/research.md) | Appends `## Research Findings` to spec |
| 3 | [`/plan`](sdlc/plan.md) | `docs/plans/{name}.md` |
| 4 | [`/build [phase?]`](sdlc/build.md) | Code changes |
| 5 | [`/check`](sdlc/check.md) | Inline verification report |
| 6 | [`/ship`](sdlc/ship.md) | PR description draft |

## Commands

- [/spec](sdlc/spec.md) — Capture feature requirements before writing any code
- [/research](sdlc/research.md) — Codebase scan + web research + infrastructure preference
- [/plan](sdlc/plan.md) — Phased, checkpoint-driven implementation plan
- [/build](sdlc/build.md) — Execute plan phase by phase with review checkpoints
- [/check](sdlc/check.md) — Verify implementation against spec and run tests
- [/ship](sdlc/ship.md) — Pre-flight checks and PR description draft

## Codex plugin commands

When Codex is selected in `aicgen configure` or `aicgen init`, aicgen installs
the `aicgen-sdlc` plugin locally in the generated project. Use the namespaced
commands in Codex:

- `/aicgen-spec` — aicgen `/spec`
- `/aicgen-research` — aicgen `/research`
- `/aicgen-plan` — aicgen `/plan`
- `/aicgen-build` — aicgen `/build`
- `/aicgen-check` — aicgen `/check`
- `/aicgen-ship` — aicgen `/ship`

## Output directory

All spec and plan artifacts are saved to the `docs/` directory in the user's project. This directory is created automatically if it does not exist.

```
docs/
├── specs/
│   └── {feature-name}.md
└── plans/
    └── {feature-name}.md
```

## Guard rails

| Command | Pre-condition |
|---------|--------------|
| `/research` | Active spec in `docs/specs/` — prompts `/spec` if missing |
| `/plan` | Active spec with `## Research Findings` — warns if research was skipped |
| `/build` | Active plan in `docs/plans/` — prompts `/plan` if missing |

## Source

Command definitions are maintained in [`aicgen-data/workflows/sdlc.md`](https://github.com/aicgen/aicgen-data/blob/main/workflows/sdlc.md) and injected into every generated config at `aicgen init` time.
