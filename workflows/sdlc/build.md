# /build [phase]

**Purpose:** Execute the next (or a specified) phase of the current implementation plan, pausing between phases for review.

## When to use

After `/plan`. Run once per phase until all phases are complete. Can also be used to re-execute a specific phase by passing its number.

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `phase` | No | Phase number to execute. If omitted, executes the next incomplete phase. |

## Pre-condition

An active plan must exist in `docs/plans/`. If none is found, the assistant will prompt you to run `/plan` first and stop.

## Steps

1. Read the active plan from `docs/plans/`
2. Determine the next incomplete phase (or the specified phase)
3. Announce which phase is being executed and what it covers
4. Implement step by step, following existing codebase patterns and conventions
5. Summarise what was changed and created
6. Mark the phase complete in the plan file
7. Run relevant tests and report results
8. Ask: "Phase {n} complete. Continue to phase {n+1}?" before proceeding

## Behaviour

- The assistant pauses after each phase and waits for your go-ahead
- If tests fail at the end of a phase, the assistant reports failures before asking to continue
- The plan file is updated in place as phases are completed — it acts as a progress tracker

## Next step

After all phases are complete, run [`/check`](check.md) to verify the full implementation against the spec.
