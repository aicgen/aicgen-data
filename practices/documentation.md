# Documentation Organization

## Keep Root Clean

**RULE: Documentation must NOT clutter the project root.**

```
❌ BAD: Root folder mess
project/
├── README.md
├── ARCHITECTURE.md
├── API_DOCS.md
├── DEPLOYMENT.md
├── TROUBLESHOOTING.md
├── USER_GUIDE.md
├── DATABASE_SCHEMA.md
├── TESTING_GUIDE.md
└── ... (20 more .md files)

✅ GOOD: Organized structure
project/
├── README.md              (overview only)
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── deployment/
│   └── guides/
└── src/
```

## Documentation Structure

**Standard documentation folder:**

```
docs/
├── architecture/
│   ├── overview.md
│   ├── decisions/         # Architecture Decision Records
│   │   ├── 001-database-choice.md
│   │   └── 002-api-design.md
│   └── diagrams/
│
├── api/
│   ├── endpoints.md
│   ├── authentication.md
│   └── examples/
│
├── guides/
│   ├── getting-started.md
│   ├── development.md
│   ├── deployment.md
│   └── troubleshooting.md
│
├── features/              # Organize by feature
│   ├── user-auth/
│   │   ├── overview.md
│   │   ├── implementation.md
│   │   └── testing.md
│   ├── payments/
│   └── notifications/
│
└── planning/              # Active work planning
    ├── memory-lane.md     # Context preservation
    ├── current-phase.md   # Active work
    └── next-steps.md      # Backlog
```

## Memory Lane Document

**CRITICAL: Maintain context across sessions**

### Purpose
When AI context limit is reached, reload from memory lane to restore working context.

### Structure

```markdown
# Memory Lane - Project Context

## Last Updated
2024-12-10 15:30

## Current Objective
Implementing user authentication system with OAuth2 support

## Recent Progress
- ✅ Set up database schema (2024-12-08)
- ✅ Implemented user registration (2024-12-09)
- 🔄 Working on: OAuth2 integration (2024-12-10)
- ⏳ Next: Session management

## Key Decisions
1. **Database**: PostgreSQL chosen for ACID compliance
2. **Auth Strategy**: OAuth2 + JWT tokens
3. **Session Store**: Redis for performance

## Important Files
- `src/auth/oauth.ts` - OAuth2 implementation (IN PROGRESS)
- `src/models/user.ts` - User model and validation
- `docs/architecture/decisions/003-auth-system.md` - Full context

## Active Questions
1. Should we support refresh tokens? (Pending user decision)
2. Token expiry: 1h or 24h? (Pending user decision)

## Technical Context
- Using Passport.js for OAuth
- Google and GitHub providers configured
- Callback URLs: /auth/google/callback, /auth/github/callback

## Known Issues
- OAuth redirect not working in development (investigating)
- Need to add rate limiting to prevent abuse

## Next Session
1. Fix OAuth redirect issue
2. Implement refresh token rotation
3. Add comprehensive auth tests
```

### Update Frequency
- Update after each significant milestone
- Update before context limit is reached
- Update when switching between features

## Context Reload Strategy

**For AI Tools with Hooks:**

Create a hook to reload memory lane on startup:

```json
{
  "hooks": {
    "startup": {
      "command": "cat docs/planning/memory-lane.md"
    }
  }
}
```

**For AI Tools with Agents:**

Create a context restoration agent:

```markdown
# Context Restoration Agent

Task: Read and summarize current project state

Sources:
1. docs/planning/memory-lane.md
2. docs/architecture/decisions/ (recent ADRs)
3. git log --oneline -10 (recent commits)

Output: Concise summary of where we are and what's next
```

## Feature Documentation

**Organize by feature/scope, not by type:**

```
❌ BAD: Organized by document type
docs/
├── specifications/
│   ├── auth.md
│   ├── payments.md
│   └── notifications.md
├── implementations/
│   ├── auth.md
│   ├── payments.md
│   └── notifications.md
└── tests/
    ├── auth.md
    └── payments.md

✅ GOOD: Organized by feature
docs/features/
├── auth/
│   ├── specification.md
│   ├── implementation.md
│   ├── api.md
│   └── testing.md
├── payments/
│   ├── specification.md
│   ├── implementation.md
│   └── providers.md
└── notifications/
    ├── specification.md
    └── channels.md
```

**Benefits:**
- All related docs in one place
- Easy to find feature-specific information
- Natural scope boundaries
- Easier to maintain

## Planning Documents

**Active planning should be in docs/planning/:**

```
docs/planning/
├── memory-lane.md         # Context preservation
├── current-sprint.md      # Active work
├── backlog.md             # Future work
└── spike-results/         # Research findings
    ├── database-options.md
    └── auth-libraries.md
```

## Documentation Principles

1. **Separate folder**: All docs in `docs/` directory
2. **Organize by scope**: Group by feature, not document type
3. **Keep root clean**: Only README.md in project root
4. **Maintain memory lane**: Update regularly for context preservation
5. **Link related docs**: Use relative links between related documents

## README Guidelines

**Root README should be concise:**

```markdown
# Project Name

Brief description

## Quick Start
[Link to docs/guides/getting-started.md]

## Documentation
- [Architecture](docs/architecture/overview.md)
- [API Docs](docs/api/endpoints.md)
- [Development Guide](docs/guides/development.md)

## Contributing
[Link to CONTRIBUTING.md or docs/guides/contributing.md]
```

**Keep it short, link to detailed docs.**

## Anti-Patterns

❌ **Don't:**
- Put 10+ markdown files in project root
- Mix documentation types in same folder
- Forget to update memory lane before context expires
- Create documentation without clear organization
- Duplicate information across multiple docs

✅ **Do:**
- Use `docs/` directory for all documentation
- Organize by feature/scope
- Maintain memory lane for context preservation
- Link related documents together
- Update docs as code evolves
