# /check

**Purpose:** Verify the current implementation against the active spec — tests, code review, and regression check.

## When to use

After `/build` phases are complete. Can also be run at any point during development to check progress. Safe to run repeatedly.

## Steps

1. Read the active spec from `docs/specs/` and plan from `docs/plans/`
2. Run the full test suite and report results
3. Review changed files against the spec's acceptance criteria — flag any gaps
4. Check for regressions — unintended side effects in changed files
5. Produce a structured report
6. If all criteria are met and no regressions, prompt the user to run `/ship`

## Output

Inline structured report:

```
✅ Acceptance criteria met
   - [criterion 1]
   - [criterion 2]

❌ Acceptance criteria not met
   - [criterion] — [details of gap]

⚠️  Potential regressions
   - [file/behaviour] — [details]

📋 Suggested fixes
   - [fix 1]
   - [fix 2]
```

## Tips

- Run `/check` early and often — not just at the end
- Use it after each `/build` phase to catch issues while context is fresh
- A clean `/check` report is the gate for running `/ship`

## Next step

If all criteria are met and no regressions: run [`/ship`](ship.md).
