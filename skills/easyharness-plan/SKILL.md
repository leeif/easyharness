---
name: easyharness-plan
description: Use when starting a new feature or project — explores requirements, designs architecture, decomposes into tasks, and generates verifiable contracts for each subtask
---

# EasyHarness-Plan: Requirements to Contract Pipeline

## Overview
- Purpose: Turn vague requirements into a plan + contract that easyharness-develop can execute autonomously
- Outputs: plan.md (implementation plan) + contract.md (per-task acceptance criteria)
- Core principle: "Every acceptance criterion must be verifiable without human judgment"
- Flow: explore → design → decompose → contract → SDD spec → self-review

## Prerequisites
- Recommended: superpowers (for brainstorming and writing-plans patterns)
- Install: https://github.com/obra/superpowers
- easyharness-plan works standalone but benefits from these methodologies

## Process Flow
```dot
digraph G {
    rankdir=TB;
    node [shape=box, style=rounded];

    Start [shape=circle, label="Start"];
    Explore [label="Explore project context"];
    Clarify [label="Ask clarifying questions"];
    Approaches [label="Propose 2-3 approaches"];
    Design [label="Present design sections"];
    UserApproveDesign [shape=diamond, label="User approves?"];
    Decompose [label="Decompose into tasks"];
    GenerateContract [label="Generate contract"];
    SelfReview [label="Contract self-review"];
    SelfReviewPass [shape=diamond, label="Self-review passes?"];
    UserReviewPlan [label="User reviews plan+contract"];
    Approved [shape=diamond, label="Approved?"];
    SaveFiles [label="Save files"];
    Ready [shape=doublecircle, label="Ready for easyharness-develop"];

    Start -> Explore;
    Explore -> Clarify;
    Clarify -> Approaches;
    Approaches -> Design;
    Design -> UserApproveDesign;
    UserApproveDesign -> Clarify [label="no"];
    UserApproveDesign -> Decompose [label="yes"];
    Decompose -> GenerateContract;
    GenerateContract -> SelfReview;
    SelfReview -> SelfReviewPass;
    SelfReviewPass -> GenerateContract [label="no (fix)"];
    SelfReviewPass -> UserReviewPlan [label="yes"];
    UserReviewPlan -> Approved;
    Approved -> Decompose [label="no (request changes)"];
    Approved -> SaveFiles [label="yes"];
    SaveFiles -> Ready;
}
```

## Phase 1: Requirements Exploration
- Check project context: files, docs, commits, patterns
- Scope assessment: decompose if multiple independent subsystems
- Ask clarifying questions: one per message, prefer multiple choice, focus on purpose/constraints/success criteria/edge cases, ask about testing strategy early
- Propose 2-3 approaches with trade-offs and recommendation
- Key difference: every question pushes toward verifiable outcomes
- Example: Not "What should the UI feel like?" but "Should the form validate on blur or on submit?"

## Phase 2: Task Decomposition
- File structure first: map files to create/modify and responsibilities
- Bite-sized tasks: each = one focused unit (2-5 min for an agent)
- Task format:
  ```
  ### Task N: [Component Name]
  Files: Create/Modify/Test with exact paths
  Complexity: simple | medium | complex
  Dependencies: task numbers or "none"
  Description: 2-3 sentences
  ```
- Complexity classification:
  - simple: single file, clear spec, no integration
  - medium: 2-3 files, some integration, moderate logic
  - complex: multi-file, architecture decisions, complex state

## Phase 3: Contract Generation
For each task, generate machine-verifiable contract.

Contract format per task:
```
## Task N: [name]
### Acceptance Criteria
- [ ] AC-N.1: [Specific verifiable behavior]
### Architectural Constraints (optional but recommended)
- [Structural rules the implementation must follow]
### Test Requirements
- What tests must exist, edge cases, integration points
### Regression Guard
- Existing functionality that must NOT break
### Complexity: simple | medium | complex
```

**Architectural Constraints Writing Rules:**

Inspired by OpenAI's harness practice: enforce invariants, not implementations. Constraints describe WHERE code goes and WHAT boundaries it respects — not HOW to write it.

Good constraints:
- "New service must live in `src/services/`, not in `src/routes/` or `src/components/`"
- "Must not import from UI layer (`src/components/`) into service layer"
- "All new public functions must have JSDoc with @param and @returns"
- "Must reuse existing `src/utils/validate.ts` validators — do not create new validation helpers"

Bad constraints (too vague or too prescriptive):
- "Follow good architecture" — what does "good" mean?
- "Use the Strategy pattern" — prescribes implementation, not boundary
- "Keep it clean" — subjective

**When to include Architectural Constraints:**
- Task creates new files → constrain where they go
- Task modifies shared code → constrain what can depend on it
- Project has ARCHITECTURE.md or layering rules → reference them
- Task is `complex` → almost always needs constraints
- Task is `simple` with one file → usually skip

**AC Writing Rules:**
1. Every AC verifiable by reading code + running tests (no subjective judgment)
2. Concrete values: "returns 400 status" not "returns an error"
3. Include edge cases: "returns empty array when no results"
4. Specify error behavior: "throws InvalidInputError when email is empty"
5. One behavior per AC — if AC contains "and", split it

**Bad ACs (examples):**
- "Works correctly" — what does "correctly" mean?
- "Handles edge cases" — which ones?
- "Good error messages" — what makes them "good"?
- "Responsive design" — at what breakpoints?

**Good ACs (examples):**
- "POST /api/users returns 201 with { id, email, createdAt } when valid email"
- "POST /api/users returns 400 with { error: 'Email required' } when email is empty"
- "User list renders max 20 items per page with 'Load more' button when total > 20"

## Phase 3.5: SDD Spec Generation (Specification-Driven Development)

After generating the contract, create an SDD spec file with concrete test scenarios for each task. These scenarios bridge the gap between abstract ACs and real-world verification.

### SDD Spec Format

For each task, write one or more scenarios in Given/When/Then format:

```
## Task N: [name]

### Scenario N.1: [descriptive scenario name]
- **Given**: [precondition — system state, data setup, environment]
- **When**: [action — API call, user interaction, function invocation with specific inputs]
- **Then**: [expected outcome — exact response, state change, side effect]
- **Verification method**: [unit test | integration test | API test | browser QA | manual check]

### Scenario N.2: [edge case or error scenario]
- **Given**: ...
- **When**: ...
- **Then**: ...
- **Verification method**: ...
```

### SDD Spec Writing Rules

1. **One scenario per behavior** — don't combine happy path and error case in one scenario
2. **Concrete values** — use real example data, not placeholders: `email: "test@example.com"` not `email: "<valid email>"`
3. **Cover the AC** — every AC in the contract MUST have at least one corresponding scenario
4. **Include failure scenarios** — for every happy path, write at least one failure/edge case scenario
5. **Specify verification method** — how will the evaluator check this? Unit test, integration test, browser QA, or API call?
6. **Browser QA scenarios** — if a scenario requires browser verification, specify: URL path, interaction steps, and expected visual state

### User Confirmation (MANDATORY)

Present the SDD spec to the user for review BEFORE saving. Format:

> "Here are the test scenarios for each task. Please review:
>
> **Task 1: [name]**
> - Scenario 1.1: [one-line summary] — [verification method]
> - Scenario 1.2: [one-line summary] — [verification method]
>
> **Task 2: [name]**
> - ...
>
> Any scenarios to add, remove, or modify?"

Only save after user confirms. The user may:
- Add scenarios the agent missed
- Remove scenarios they consider unnecessary
- Modify expected outcomes
- Change verification methods

### Output

Save as: `docs/plans/YYYY-MM-DD-<name>-sdd-spec.md`

## Phase 4: Contract Self-Review
Before presenting to user:
1. Verifiability: Can each AC be checked by code + test? Rewrite if requires human judgment
2. Completeness: Do ACs cover the task description? Any missing behavior?
3. Non-overlap: Any duplicate ACs? Merge them.
4. Regression coverage: For tasks modifying existing code, is there a regression guard?
5. Testability: Can you imagine the test for each AC? If not, AC is too vague.

## Output
Save three files:
- `docs/plans/YYYY-MM-DD-<name>.md` — Implementation plan
- `docs/plans/YYYY-MM-DD-<name>-contract.md` — Contract
- `docs/plans/YYYY-MM-DD-<name>-sdd-spec.md` — SDD test scenarios

### AGENTS.md Verification (MANDATORY)
After saving plan and contract, check the project root for `AGENTS.md`. If it does not exist, copy from the example template at `@AGENTS.md.example` and place it at the project root. If it exists, verify it contains all three rules below — append any missing ones.

Required rules:

1. **Strict Plan Adherence** — Every task in `plan.md` MUST be implemented exactly as specified. After each task passes evaluation, mark it `✅ DONE` in `plan.md` and commit. TodoWrite is session-ephemeral — `plan.md` is the durable record.
2. **Contract-Driven Verification** — Every task MUST be verified against its acceptance criteria in `contract.md`. After each AC passes, mark it `[x]` in `contract.md`. After all ACs for a task pass, mark the task `✅ DONE` in both `plan.md` and `contract.md`, then commit. Do not self-assess — always dispatch `easyharness-evaluator`.
3. **Mandatory Skill Loading & Recovery** — Always load `easyharness-develop` and `easyharness-evaluator` skills. If context compression causes skill loss, immediately re-invoke via `skill` tool before continuing.

**This file is system-level injected and survives context compression — it is the last line of defense for process integrity.**

After saving, prompt user:
> "Plan, contract, and SDD spec saved. Please review all files before we proceed:
> - Plan: `<path>`
> - Contract: `<path>`
> - SDD Spec: `<path>`
> - AGENTS.md: verified ✓
> Ready to start implementation with easyharness-develop?"

## Common Mistakes
- Writing vague ACs ("works correctly", "handles errors")
- Missing regression guards for modified code
- Over-decomposing simple tasks (one function doesn't need 5 subtasks)
- Under-decomposing complex tasks (multi-file changes in single task)
- Forgetting to ask about testing strategy during exploration

## Quick Reference
| Phase | Output | Key Principle |
|-------|--------|---------------|
| Exploration | Validated requirements | One question at a time, prefer multiple choice |
| Decomposition | Task list with file paths | Bite-sized, TDD-friendly, complexity-tagged |
| Contract | Per-task ACs | Machine-verifiable, no subjective judgment |
| SDD Spec | Per-task test scenarios | Given/When/Then with concrete values, user-confirmed |
| Self-Review | Verified contract + spec | Every AC must be testable, every AC has a scenario |
