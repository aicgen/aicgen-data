# Code Review Practices

## Rules

- Review for correctness, security, tests, error handling and design; leave style to linters and formatters.
- Keep pull requests small and focused, with a description of what changed, why, and how it was tested.
- Give specific, kind feedback: explain the reason, suggest an alternative, prefix minor points with `nit:`.
- Review promptly (within a working day) and approve with minor comments rather than blocking.
- Authors self-review the diff and run the checks before requesting review.

## Review Checklist

- [ ] Code follows project style guidelines
- [ ] No obvious bugs or logic errors
- [ ] Error handling is appropriate
- [ ] Tests cover new functionality
- [ ] No security vulnerabilities introduced
- [ ] Performance implications considered
- [ ] Documentation updated if needed

## Giving Feedback

**Good:**
```
Consider using `Array.find()` here instead of `filter()[0]` -
it's more readable and stops at the first match.
```

**Bad:**
```
This is wrong.
```

## PR Description Template

```markdown
## Summary
Brief description of changes

## Changes
- Added X feature
- Fixed Y bug
- Refactored Z

## Testing
- [ ] Unit tests added
- [ ] Manual testing performed

## Screenshots (if UI changes)
```

## Best Practices

- Review promptly (within 24 hours)
- Focus on logic and design, not style (use linters)
- Ask questions rather than make demands
- Praise good solutions
- Keep PRs small and focused
- Use "nitpick:" prefix for minor suggestions
- Approve with minor comments when appropriate
