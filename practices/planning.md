# Planning Best Practices

## Plan Before Implementation

**ALWAYS design and plan before writing code:**

1. **Understand Requirements**
   - Clarify the goal and scope
   - Identify constraints and dependencies
   - Ask questions about ambiguous requirements

2. **Break Down Into Phases**
   - Divide work into logical phases
   - Define deliverables for each phase
   - Prioritize phases by value and dependencies

3. **Design First**
   - Sketch architecture and data flow
   - Identify components and interfaces
   - Consider edge cases and error scenarios

4. **Get User Approval**
   - Present the plan to stakeholders
   - Explain trade-offs and alternatives
   - Wait for approval before implementation

## Never Make Assumptions

**CRITICAL: When in doubt, ASK:**

```typescript
// ❌ BAD: Assuming what user wants
async function processOrder(orderId: string) {
  // Assuming we should send email, but maybe not?
  await sendConfirmationEmail(orderId);
  // Assuming payment is already captured?
  await fulfillOrder(orderId);
}

// ✅ GOOD: Clarify requirements first
// Q: Should we send confirmation email at this stage?
// Q: Is payment already captured or should we capture it here?
// Q: What happens if fulfillment fails?
```

**Ask about:**
- Expected behavior in edge cases
- Error handling strategy
- Performance requirements
- Security considerations
- User experience preferences

## Plan in Phases

**Structure work into clear phases:**

### Phase 1: Foundation
- Set up project structure
- Configure tooling and dependencies
- Create basic types and interfaces

### Phase 2: Core Implementation
- Implement main business logic
- Add error handling
- Write unit tests

### Phase 3: Integration
- Connect components
- Add integration tests
- Handle edge cases

### Phase 4: Polish
- Performance optimization
- Documentation
- Final review

**Checkpoint after each phase:**
- Demo functionality
- Get feedback
- Adjust plan if needed

## Planning Template

```markdown
## Goal
[What are we building and why?]

## Requirements
- [ ] Requirement 1
- [ ] Requirement 2
- [ ] Requirement 3

## Questions for Clarification
1. [Question about requirement X]
2. [Question about edge case Y]
3. [Question about preferred approach for Z]

## Proposed Approach
[Describe the solution]

## Phases
1. **Phase 1**: [Description]
   - Task 1
   - Task 2

2. **Phase 2**: [Description]
   - Task 1
   - Task 2

## Risks & Mitigation
- **Risk**: [Description]
  **Mitigation**: [How to handle]

## Alternatives Considered
- **Option A**: [Pros/Cons]
- **Option B**: [Pros/Cons]
- **Chosen**: Option A because [reason]
```

## Communication Principles

1. **Ask Early**: Don't wait until you're stuck
2. **Be Specific**: "Should error X retry or fail immediately?"
3. **Propose Options**: "Would you prefer A or B?"
4. **Explain Trade-offs**: "Fast but risky vs. Slow but safe"
5. **Document Decisions**: Record what was decided and why

## Anti-Patterns

❌ **Don't:**
- Start coding without understanding requirements
- Assume you know what the user wants
- Skip the planning phase to "save time"
- Make architectural decisions without discussion
- Proceed with unclear requirements

✅ **Do:**
- Ask questions when requirements are vague
- Create a plan and get it approved
- Break work into reviewable phases
- Document decisions and reasoning
- Communicate early and often
