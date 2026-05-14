# SDLC Workflows

## /spec [name]
Capture the full specification for a feature or task before any code is written.

**Steps:**
1. Ask for a feature name if not provided as an argument
2. Ask the user to describe: goal, user stories, acceptance criteria, constraints, and what is explicitly out of scope
3. Create `docs/specs/` directory if it does not exist
4. Save the gathered information to `docs/specs/{name}.md` using the template below
5. Confirm the file was saved and prompt the user to run `/research`

**Output template (`docs/specs/{name}.md`):**
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

---

## /research
Analyze the active spec with internal codebase scanning and external web research.

**Pre-condition:** An active spec must exist in `docs/specs/` — if none is found, tell the user to run `/spec` first and stop.

**Steps:**
1. Read the most recently modified spec from `docs/specs/`
2. **Internal scan:** Search the codebase for related code, existing patterns, similar implementations, dependencies, and potential conflicts relevant to the spec
3. **Infrastructure prompt:** Ask the user:
   > "Does this feature require infrastructure decisions?"
   > - Cost-optimised / serverless (pay-per-use: Cloud Run, Cloud Functions, AWS Lambda, Fargate, etc.)
   > - Fixed / dedicated (predictable load: Kubernetes, EC2, GKE, dedicated VMs, etc.)
   > - No infrastructure involved
4. **Web research:** Search for architecture patterns, best practices, reference implementations, and cost comparisons relevant to the spec. Bias results toward the chosen infrastructure model if one was selected.
5. Surface: recommended approaches, trade-offs, cost implications, and links to reference material
6. Suggest any improvements or clarifications to the spec based on findings
7. Append a `## Research Findings` section to the active spec file containing: internal findings, web research summary, infrastructure recommendation (if applicable), and suggested spec improvements
8. Prompt the user to run `/plan`

---

## /plan
Produce a phased, checkpoint-driven implementation plan based on the spec and research findings.

**Pre-condition:** An active spec with a `## Research Findings` section must exist — if research has not been run, warn the user and ask them to confirm they want to skip it before proceeding.

**Steps:**
1. Read the active spec (including research findings) from `docs/specs/`
2. Analyse the codebase to understand existing structure, patterns, and conventions
3. Break the implementation into phases — each phase must be independently verifiable
4. For each phase, list: files to create or modify, key decisions, and how to verify it is complete
5. Identify risks and propose mitigations
6. Create `docs/plans/` directory if it does not exist
7. Save the plan to `docs/plans/{spec-name}.md`
8. Confirm the file was saved and prompt the user to run `/build`

---

## /build [phase]
Execute the next (or a specified) phase of the current implementation plan.

**Pre-condition:** An active plan must exist in `docs/plans/` — if none is found, tell the user to run `/plan` first and stop.

**Steps:**
1. Read the active plan from `docs/plans/`
2. Determine the next incomplete phase (or the phase specified by the argument)
3. Announce which phase is being executed and what it covers
4. Implement the phase step by step, following existing codebase patterns and conventions
5. After completing the phase, summarise what was changed and what was created
6. Mark the phase as complete in the plan file
7. Run any tests relevant to the completed phase and report results
8. Pause and ask: "Phase {n} complete. Continue to phase {n+1}?" before proceeding

---

## /check
Verify the current implementation against the active spec — tests, code review, and regression check.

**Steps:**
1. Read the active spec from `docs/specs/` and the active plan from `docs/plans/`
2. Run the full test suite and report results
3. Review all changed files against the spec's acceptance criteria — flag any gaps
4. Check for regressions by reviewing changed files for unintended side effects
5. Produce a structured report:
   - ✅ Acceptance criteria met
   - ❌ Acceptance criteria not met (with details)
   - ⚠️ Potential regressions (with details)
   - 📋 Suggested fixes
6. If all criteria are met and no regressions are found, prompt the user to run `/ship`

---

## /ship
Pre-flight wrap-up — verify everything is ready, then draft a PR description.

**Steps:**
1. Run the full test suite — stop and report if any tests fail
2. Read the active spec and plan
3. Verify that `docs/specs/{name}.md` and `docs/plans/{name}.md` are up to date
4. Check for uncommitted changes and list them
5. Draft a PR description referencing the spec and plan:
   ```
   ## Summary
   {goal from spec}

   ## Changes
   {summary of phases completed}

   ## Spec
   docs/specs/{name}.md

   ## Plan
   docs/plans/{name}.md

   ## Test plan
   {acceptance criteria from spec as a checklist}
   ```
6. Present the PR description to the user for review
7. Ask: "Ready to commit and push?" — if yes, stage all changes and create a commit with a conventional commit message
