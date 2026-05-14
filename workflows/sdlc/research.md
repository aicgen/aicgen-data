# /research

**Purpose:** Analyze the active spec with internal codebase scanning and external web research, and prompt for infrastructure preference before planning begins.

## When to use

After `/spec`, before `/plan`. This step surfaces codebase patterns, risks, and reference architectures that should inform the implementation plan.

## Pre-condition

An active spec must exist in `docs/specs/`. If none is found, the assistant will tell you to run `/spec` first and stop.

## Steps

1. Read the most recently modified spec from `docs/specs/`
2. **Internal scan** — search codebase for related code, patterns, dependencies, conflicts
3. **Infrastructure prompt** — ask the user:
   > "Does this feature require infrastructure decisions?"
   > - Cost-optimised / serverless (pay-per-use: Cloud Run, Cloud Functions, AWS Lambda, Fargate, etc.)
   > - Fixed / dedicated (predictable load: Kubernetes, EC2, GKE, dedicated VMs, etc.)
   > - No infrastructure involved
4. **Web research** — search for architecture patterns, best practices, reference implementations, and cost comparisons relevant to the spec. Results are biased toward the chosen infrastructure model.
5. Surface: recommended approaches, trade-offs, cost implications, reference links
6. Suggest improvements or clarifications to the spec
7. Append `## Research Findings` to the active spec file
8. Prompt the user to run `/plan`

## Infrastructure preference

The infrastructure prompt shapes the direction of web research:

| Choice | Research focus |
|--------|---------------|
| Serverless | Event-driven patterns, cold start mitigation, pay-per-use cost modelling, Cloud Run / Lambda / Fargate comparisons |
| Fixed / dedicated | Kubernetes patterns, resource sizing, HA configuration, long-running process management |
| No infrastructure | Pure code architecture, library selection, algorithm trade-offs |

## Output

Appends a `## Research Findings` section to the active spec file:

```markdown
## Research Findings

### Internal codebase
{relevant existing code, patterns, conflicts found}

### Web research
{architecture recommendations, reference links, cost comparisons}

### Infrastructure recommendation
{recommendation based on chosen preference}

### Suggested spec improvements
{clarifications or additions to the spec}
```

## Next step

Run [`/plan`](plan.md) to turn the refined spec into a phased implementation plan.
