# /ship

**Purpose:** Pre-flight wrap-up — verify everything is ready, then draft a PR description referencing the spec and plan.

## When to use

After `/check` reports all acceptance criteria met and no regressions. This is the final step of the SDLC workflow.

## Steps

1. Run the full test suite — stops and reports if any tests fail
2. Read the active spec and plan
3. Verify `docs/specs/{name}.md` and `docs/plans/{name}.md` are up to date
4. List all uncommitted changes
5. Draft a PR description:
   ```
   ## Summary
   {goal from spec}

   ## Changes
   {summary of completed phases}

   ## Spec
   docs/specs/{name}.md

   ## Plan
   docs/plans/{name}.md

   ## Test plan
   {acceptance criteria as a checklist}
   ```
6. Present the PR description to the user for review
7. Ask: "Ready to commit and push?" — if yes, stage all changes and create a conventional commit

## Output

A complete PR description ready to paste or submit. If the user confirms, a commit is created with a conventional commit message.

## Tips

- Do not run `/ship` if `/check` found unresolved issues
- The PR description links to `docs/specs/` and `docs/plans/` — reviewers can follow the full decision trail
- The commit message follows conventional commits format (e.g. `feat: add {feature-name}`)
