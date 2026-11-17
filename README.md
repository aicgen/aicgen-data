# aicgen Guidelines Repository

This repository contains coding guidelines and best practices that power aicgen configurations.

## Release 2.0.0 Stats

| Metric | Count |
|--------|-------|
| Guidelines | 116 |
| SDLC workflows | 6 |
| Categories | 12 |
| Languages | 12 |
| Architectures | 10 |
| Datasources | 2 |
| Profile levels | `basic`, `standard`, `full` |

## Directory Structure

```
aicgen-data/
├── guideline-mappings.yml    # Maps guideline IDs to files and filters
├── api/                      # API design patterns
├── architecture/             # Architecture patterns (clean, DDD, etc.)
├── database/                 # Database guidelines
├── devops/                   # CI/CD and deployment
├── error-handling/           # Error handling strategies
├── language/                 # Language-specific (typescript, python)
├── patterns/                 # Design patterns
├── performance/              # Performance optimization
├── practices/                # Best practices
├── security/                 # Security guidelines
├── style/                    # Code style guidelines
├── agentic/                  # Assistant capability matrix and profile limits
├── templates/                # Reusable templates
├── workflows/                # SDLC lifecycle command definitions
└── testing/                  # Testing strategies
```

## Agentic Profiles

The `agentic/` directory describes how guideline content maps into assistant-specific surfaces for Claude Code, GitHub Copilot, Antigravity, and Codex.

Gemini CLI generation is removed from active targets. Use Antigravity for Google-side agentic coding profiles.

Profile levels gate capability risk:

- `basic`: stable instructions and rules.
- `standard`: reusable workflows, prompt files, and stable Codex SDLC skills.
- `full`: focused agents, skills, safe lifecycle guardrail hooks, plugin packaging, and advanced documentation templates such as MCP.

Test-suite execution is handled by the `/check` lifecycle command, not by generated hooks. Hook templates must not run full test suites automatically.

## Context Budgets

Instruction files are loaded into the model's context window, so aicgen keeps always-on files small and delivers each guideline only when it is relevant.

| Tier | What | Examples |
|------|------|----------|
| Always-on | Role, commands, and an index of guideline names. Never guideline bodies. | `CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `.agents/rules/aicgen-instructions.md` |
| Scoped | One file per language, style, error-handling and testing guideline, loaded when matching files are touched | `.claude/rules/*.md` (`paths:`), `.github/instructions/*.instructions.md` (`applyTo`), `.agents/rules/aicgen-*.md` (`trigger: glob`) |
| On-demand | One skill per remaining guideline (security, architecture, API, database, DevOps, performance...). Only name and description are always visible. | `.claude/skills/`, `.github/skills/`, `.agents/skills/`, Antigravity `trigger: model_decision` rules |

Limits aicgen checks when generating (`aicgen init` prints a summary):

- Claude Code: `CLAUDE.md` under ~200 lines.
- Codex: `AGENTS.md` under 32 KiB (hard limit).
- Antigravity: rule files under 24,000 bytes (oversized guidelines are split into parts); every rule needs `trigger` frontmatter.
- Agent Skills: `SKILL.md` body under 5,000 tokens and 500 lines; skill names up to 64 characters.
- Language-agnostic guidelines are localized at generation time: fenced examples in a programming language other than the project's (and sections titled `Rust: ...`, `Python`, etc.) are removed. Language-neutral blocks (sql, bash, yaml, json, pseudocode) are kept, so tag code fences accurately.
- Always-on token budget by profile level: basic ~1,500, standard ~2,500, full ~4,000 (estimated at 4 characters per token).

When writing guidelines:

- Keep each file focused: aim for under ~2,500 tokens. `scripts/validate-data` fails above 3,000.
- Start with a short `## Rules` list of imperative, checkable statements; put examples after it.
- Use code only where it shows something prose cannot, and prefer one good example over several variants.
- Give every guideline a distinct title and a specific `description` (see Mapping Fields). Overlapping guidelines should link to each other rather than repeat content.

## Contributing Guidelines

### Step 1: Create Your Guideline File

Create a markdown file in the appropriate category folder:

```bash
# Example: Adding a new TypeScript guideline
aicgen-data/language/typescript/my-guideline.md
```

**Guideline Format:**

```markdown
# Guideline Title

Brief description of what this guideline covers.

## Section 1

Clear, actionable instructions with code examples:

\`\`\`typescript
// Good example
function goodExample(): string {
  return "well-structured code";
}

// Bad example - explain why
function badExample() {  // Missing return type
  return "unclear code";
}
\`\`\`

## Section 2

More sections as needed...
```

**Writing Tips:**
- Be concise and actionable
- Include both good and bad code examples
- Use language-appropriate code blocks
- Focus on the "why" not just the "what"

### Step 2: Add to Mappings

Add your guideline to `guideline-mappings.yml`:

```yaml
my-guideline-id:
  path: language/typescript/my-guideline.md
  description: "What the guideline prescribes. Use when <the situations that should load it>."
  category: Language
  languages:
    - typescript
  levels:
    - standard
    - full
  tags:
    - typescript
    - relevant-tag
```

### Mapping Fields

| Field | Required | Description |
|-------|----------|-------------|
| `path` | Yes | Relative path to the guideline file |
| `description` | Yes | One line, 40-280 characters, saying what the guideline covers and ending with `Use when ...`. Assistants read only this to decide whether to load the guideline, so name the concrete situations and keywords. No double quotes or backslashes. |
| `category` | Yes | Display category (Language, Architecture, Testing, etc.) |
| `languages` | No | Which languages this applies to. Omit for all languages |
| `levels` | No | Instruction levels: `basic`, `standard`, `full`. Omit for all |
| `architectures` | No | Architecture types. Omit for all |
| `tags` | No | Search/organization tags |

### Available Values

**Languages:**
- `typescript`, `javascript`, `python`, `go`, `rust`, `java`, `csharp`, `ruby`, `dart`, `swift`, `kotlin`, `php`

**Levels:**
- `basic` - Essential guidelines only
- `standard` - Common best practices
- `full` - Comprehensive coverage with advanced agentic surfaces

**Architectures:**
- `layered`, `clean-architecture`, `hexagonal`, `ddd`, `microservices`
- `modular-monolith`, `event-driven`, `serverless`, `monorepo`
- `bounded-contexts`, `component-based`

**Categories:**
- `Language`, `Architecture`, `Testing`, `Security`, `Performance`
- `Database`, `API Design`, `Code Style`, `Error Handling`, `DevOps`
- `Best Practices`, `Design Patterns`

## Examples

### Language-Specific Guideline

```yaml
typescript-decorators:
  path: language/typescript/decorators.md
  category: Language
  languages:
    - typescript
  levels:
    - full
  tags:
    - typescript
    - decorators
    - metadata
```

### Universal Guideline (All Languages)

```yaml
solid-principles:
  path: architecture/solid/principles.md
  category: Architecture
  levels:
    - standard
    - full
  tags:
    - solid
    - design-principles
    - oop
```

### Architecture-Specific Guideline

```yaml
ddd-aggregates:
  path: architecture/ddd/aggregates.md
  category: Architecture
  architectures:
    - ddd
    - clean-architecture
  levels:
    - full
  tags:
    - ddd
    - aggregates
    - domain-model
```

## Testing Your Changes

After adding a guideline:

1. **Validate the data repo:**
   ```bash
   scripts/validate-data
   ```

2. **Rebuild aicgen:**
   ```bash
   AICGEN_DATA_DIR=/path/to/aicgen-data bun run embed
   bun run build
   ```

3. **Check it's loaded:**
   ```bash
   bun run start stats
   ```

4. **Test generation:**
   ```bash
   bun run start init --force
   ```

## Guideline Quality Checklist

- [ ] Distinct, descriptive title and a `description` ending in "Use when ..."
- [ ] Rules first, examples second; under ~2,500 tokens
- [ ] Actionable instructions
- [ ] Code examples (good and bad)
- [ ] Appropriate language/level targeting
- [ ] Meaningful category and tags
- [ ] No duplicate content with existing guidelines

## License

MIT License - See LICENSE file
