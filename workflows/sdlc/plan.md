# /plan

**Purpose:** Produce a phased, checkpoint-driven implementation plan based on the spec and research findings.

## When to use

After `/research`. This step breaks the work into independently verifiable phases that `/build` will execute one at a time.

## Pre-condition

An active spec with a `## Research Findings` section must exist in `docs/specs/`. If research was skipped, the assistant will warn you and ask for confirmation before proceeding.

## Steps

1. Read the active spec (including research findings) from `docs/specs/`
2. Analyse the codebase — existing structure, patterns, conventions
3. Break implementation into phases, each independently verifiable
4. For each phase: list files to create or modify, key decisions, and verification steps
5. Identify risks and propose mitigations
6. Create `docs/plans/` if it does not exist
7. Save the plan to `docs/plans/{spec-name}.md`
8. Confirm saved and prompt the user to run `/build`

## Output

Creates `docs/plans/{spec-name}.md` with a phase-by-phase breakdown:

```markdown
# Plan: {name}

## Overview
{summary of approach}

## Phase 1: {name}
**Files:** {files to create or modify}
**Steps:** {implementation steps}
**Verify:** {how to confirm this phase is complete}

## Phase 2: {name}
...

## Risks
{risks and mitigations}
```

## Tips

- Each phase should be completable in one focused work session
- Verification steps should be concrete — runnable commands or observable outcomes
- Order phases so each one builds on a stable foundation from the previous

## Next step

Run [`/build`](build.md) to execute the first phase of the plan.
