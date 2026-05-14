# /spec [name]

**Purpose:** Capture the full specification for a feature or task before any code is written.

## When to use

Run this at the very start of any non-trivial feature or task. It ensures the AI assistant and developer share a common understanding of what is being built before any planning or implementation begins.

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `name` | No | Feature name used as the filename. If omitted, the assistant will ask. |

## Steps

1. Ask for a feature name if not provided
2. Gather: goal, user stories, acceptance criteria, constraints, and what is explicitly out of scope
3. Create `docs/specs/` if it does not exist
4. Save to `docs/specs/{name}.md`
5. Confirm the file was saved and prompt the user to run `/research`

## Output

Creates `docs/specs/{name}.md` with this structure:

```markdown
# Spec: {name}

## Goal
{goal}

## User Stories
{user stories}

## Acceptance Criteria
{acceptance criteria}

## Constraints
{constraints}

## Out of Scope
{out of scope}
```

## Tips

- Be specific in acceptance criteria — vague criteria lead to vague implementations
- Out-of-scope is as important as in-scope; it prevents scope creep
- The more detail here, the better `/research` and `/plan` will be

## Next step

Run [`/research`](research.md) to analyze the spec against the codebase and find reference solutions.
