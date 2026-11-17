# Version Control Patterns

## Rules

- Prefer trunk-based development or GitHub Flow with short-lived branches; use Git Flow only for scheduled, versioned releases.
- Make commits small and atomic, with a clear subject (imperative, under about 72 characters) and a body that explains why; use Conventional Commits prefixes if the project does.
- Rebase or merge main before pushing; never force-push shared branches.
- Land changes through pull requests with required checks; delete merged branches.
- Tag releases with semantic versions.

## Branching Strategies

### GitHub Flow
Simple: main + feature branches.

```
main ─────●─────●─────●─────●─────
           \         /
feature     ●───●───●
```

### Git Flow
For scheduled releases: main, develop, feature, release, hotfix.

```
main    ─────●─────────────●─────
              \           /
release        ●─────────●
                \       /
develop  ●───●───●───●───●───●───
          \     /
feature    ●───●
```

## Commit Messages

```
feat: add user authentication

- Implement JWT-based auth
- Add login/logout endpoints
- Include password hashing

Closes #123
```

**Prefixes:**
- `feat:` - New feature
- `fix:` - Bug fix
- `refactor:` - Code change that doesn't fix bug or add feature
- `docs:` - Documentation only
- `test:` - Adding tests
- `chore:` - Maintenance tasks

## Best Practices

- Keep commits atomic and focused
- Write descriptive commit messages
- Pull/rebase before pushing
- Never force push to shared branches
- Use pull requests for code review
- Delete merged branches
- Tag releases with semantic versions
