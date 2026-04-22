# EasyHarness Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create three skills (easyharness-plan, easyharness-develop, easyharness-evaluator) that form an automated development harness — from requirements to verified implementation.

**Architecture:** Three skills form a pipeline: easyharness-plan produces a plan + contract, easyharness-develop orchestrates a RALPH loop executing tasks one at a time with evaluator feedback, easyharness-evaluator provides 4-layer automated verification (hard checks → stub detection → contract compliance → quality scoring). The agent running easyharness-develop IS the generator (no subagent dispatch for implementation); only the evaluator is dispatched as a read-only subagent.

**Tech Stack:** Markdown skills with YAML frontmatter (agentskills.io spec). Prompt templates for evaluator dispatch. Few-shot calibration examples for evaluator.

**Key Sources:**
- superpowers:brainstorming — question-asking flow for requirements exploration
- superpowers:writing-plans — task decomposition, bite-sized granularity, file structure mapping
- superpowers:subagent-driven-development — subagent dispatch patterns, prompt templates, status protocol (DONE/BLOCKED/NEEDS_CONTEXT)
- superpowers:verification-before-completion — evidence-before-claims iron law
- superpowers:test-driven-development — TDD cycle integration
- Anthropic harness article — contract negotiation, calibrated evaluator, few-shot calibration, anti-self-persuasion, RALPH loop mechanics

---

## Prerequisites: Dependency Skills Installation

EasyHarness skills depend on skills from two open-source projects. Users must install these before using easyharness-develop or easyharness-evaluator.

### Required from Superpowers ([github.com/obra/superpowers](https://github.com/obra/superpowers))

Skills referenced:
- `superpowers:test-driven-development` — TDD cycle in easyharness-develop's ACT phase
- `superpowers:verification-before-completion` — Final verification in easyharness-develop
- `superpowers:brainstorming` — Requirements exploration pattern in easyharness-plan
- `superpowers:writing-plans` — Task decomposition pattern in easyharness-plan

**Install (by platform):**

| Platform | Command |
|----------|---------|
| Claude Code (Official) | `/plugin install superpowers@claude-plugins-official` |
| Claude Code (Marketplace) | `/plugin marketplace add obra/superpowers-marketplace` then `/plugin install superpowers@superpowers-marketplace` |
| OpenAI Codex CLI | `/plugins` → search "superpowers" → Install |
| Cursor | `/add-plugin superpowers` |
| OpenCode | Tell your agent: `Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md` |

### Required from gstack ([github.com/garrytan/gstack](https://github.com/garrytan/gstack)) — Optional, for Browser QA

Only needed if easyharness-evaluator will run browser-based QA (UI projects).

Skills referenced:
- `gstack:browse` — Headless browser for UI acceptance criteria verification

**Install:**

```bash
# Claude Code (default)
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup

# Other agents — use --host flag:
# Codex:   ./setup --host codex
# OpenCode: ./setup --host opencode
# Cursor:  ./setup --host cursor
# Hermes:  ./setup --host hermes
```

### Verification

After installation, confirm skills are available:
```
# Ask your agent:
"List available skills matching 'test-driven' and 'browse'"
# Should show superpowers:test-driven-development and gstack:browse
```

---

## File Structure

```
/root/Github/easyharness/
├── skills/
│   ├── easyharness-plan/
│   │   └── SKILL.md                    # Requirements → design → contract generation
│   ├── easyharness-develop/
│   │   ├── SKILL.md                    # RALPH loop orchestration
│   │   └── evaluator-dispatch-prompt.md  # Template for dispatching easyharness-evaluator
│   └── easyharness-evaluator/
│       ├── SKILL.md                    # 4-layer evaluation engine
│       └── few-shot-examples.md        # Calibration examples (PASS/FAIL)
├── docs/
│   └── plans/
│       └── 2026-04-22-easyharness-skills.md  # This plan
└── README.md                           # (not created — user can add later)
```

---

### Task 1: easyharness-evaluator SKILL.md — The 4-Layer Evaluation Engine

**Why first:** easyharness-evaluator is a dependency of easyharness-develop. Writing it first means easyharness-develop can reference its exact interface and output format.

**Files:**
- Create: `skills/easyharness-evaluator/SKILL.md`

- [ ] **Step 1: Write the YAML frontmatter**

```yaml
---
name: easyharness-evaluator
description: Use when verifying implementation quality against a contract — dispatched as read-only evaluator in the easyharness-develop RALPH loop, or standalone for post-implementation audits
---
```

- [ ] **Step 2: Write the Overview section**

Establish the core principle: evaluator's job is to find problems, not validate success. Include the anti-self-persuasion framing from Anthropic's article.

Key content:
- Purpose: Automated acceptance testing against a contract
- Core principle: "Your job is to find problems. If you lean toward PASS, think again — the generator may be cutting corners."
- 4 layers: automated hard checks → stub/laziness detection → contract compliance → quality scoring
- Evaluator permissions: terminal (run tests/lint/build), file read, browser (if UI project) — NO file write/edit
- Output: structured JSON verdict

- [ ] **Step 3: Write Layer 1 — Automated Hard Checks**

This layer runs without LLM judgment. Pure exit-code checks.

```markdown
## Layer 1: Automated Hard Checks (No LLM)

Run each check. ANY failure → immediate FAIL verdict, skip remaining layers.

1. **Tests**: Find and run the project's test command
   - Look for: `package.json` scripts (test), `Makefile` targets, `pytest.ini`, `cargo test`
   - Run the full test suite, not just changed files
   - Capture exit code + output

2. **Lint** (if configured):
   - Look for: eslint, ruff, clippy, golangci-lint configs
   - Run linter on changed files
   - Capture exit code + output

3. **Build** (if applicable):
   - Look for: tsc, cargo build, go build, make build
   - Run full build
   - Capture exit code + output

4. **Regression check**:
   - Run full test suite (not just new tests)
   - Compare against pre-implementation test count — no tests should have been deleted
```

- [ ] **Step 4: Write Layer 2 — Stub/Laziness Detection**

Hybrid layer: grep scan + LLM review of suspicious code.

```markdown
## Layer 2: Stub/Laziness Detection

Scan changed files for signs the generator took shortcuts.

**Automated grep scan** (search changed files for):
- `TODO`, `FIXME`, `HACK`, `XXX`, `PLACEHOLDER`
- `stub`, `mock` (outside test files), `fake`, `dummy`
- `console.log`, `print(` (debug remnants, outside logging utilities)
- Empty function bodies: `{}`、`pass`、`return None`、`return undefined`
- Hardcoded values where dynamic logic was specified in contract

**LLM review of suspicious code:**
- For each grep hit, read surrounding context
- Distinguish legitimate patterns (e.g., `// TODO: future enhancement` noted in contract) from laziness
- Check if tests actually test behavior vs. only testing mocks
- Check for "golden path only" implementations that skip error handling

**Verdict:** Any confirmed stub → FAIL with exact file:line references
```

- [ ] **Step 5: Write Layer 3 — Contract Compliance**

Line-by-line AC verification. This is the core evaluator function.

```markdown
## Layer 3: Contract Compliance

For each Acceptance Criterion (AC) in the contract:

1. Read the AC text
2. Find the implementing code (grep for keywords, read relevant files)
3. Find the test that verifies this AC
4. Assess: does the implementation ACTUALLY satisfy the AC?
   - Read the code yourself — do NOT trust the generator's report
   - Check edge cases mentioned in the AC
   - Verify the test actually exercises the AC (not a trivial assertion)

**Output per AC:**
- `PASS` + evidence (test name, code reference)
- `FAIL` + reason (what's missing, what's wrong) + code reference

**Check for over-engineering:**
- Features not in any AC → flag as "unspecified addition"
- Not an automatic FAIL, but noted in report

**Check for regression guards:**
- If contract specifies "must not break X", verify X still works
```

- [ ] **Step 6: Write Layer 4 — Quality Scoring**

LLM-based quality assessment with hard thresholds.

```markdown
## Layer 4: Quality Scoring

Score each dimension 0-10. Hard thresholds determine PASS/FAIL.

| Dimension | Description | Hard Threshold |
|-----------|-------------|----------------|
| functionality | Does it work correctly for all specified behaviors? | >= 8 |
| regression | Does existing functionality remain intact? | >= 9 |
| code_quality | Clean, readable, follows codebase conventions? | >= 6 |
| test_coverage | Do tests cover the specified behaviors meaningfully? | PASS/FAIL |

**Scoring guidance:**
- 10: Exceptional, production-ready, handles edge cases elegantly
- 8-9: Solid, works correctly, minor improvements possible
- 6-7: Acceptable, functional but has rough edges
- 4-5: Below standard, missing important aspects
- 1-3: Fundamentally broken or incomplete

**Any dimension below threshold → overall FAIL**
```

- [ ] **Step 7: Write the Output Format section**

```markdown
## Output Format

Return a structured JSON report:

\```json
{
  "verdict": "PASS" | "FAIL",
  "layer_results": {
    "automated_checks": {
      "tests": { "status": "pass" | "fail", "output_summary": "34/34 pass" },
      "lint": { "status": "pass" | "fail" | "skipped", "output_summary": "..." },
      "build": { "status": "pass" | "fail" | "skipped", "output_summary": "..." },
      "regression": { "status": "pass" | "fail", "tests_before": 34, "tests_after": 38 }
    },
    "stub_detection": {
      "status": "clean" | "found",
      "findings": [
        { "file": "src/auth.ts", "line": 42, "issue": "Empty catch block", "severity": "high" }
      ]
    },
    "contract_compliance": {
      "AC-1": { "status": "PASS", "evidence": "test `should validate email` in auth.test.ts:15" },
      "AC-2": { "status": "FAIL", "reason": "Returns hardcoded value instead of computed result", "file": "src/calc.ts", "line": 28 }
    },
    "quality_scores": {
      "functionality": 9,
      "regression": 10,
      "code_quality": 7,
      "test_coverage": "PASS"
    }
  },
  "feedback": "Specific actionable feedback for the generator's next attempt",
  "blocking_issues": ["AC-2: returns hardcoded value instead of computed result"]
}
\```

**The `feedback` field is critical** — it must contain specific, actionable instructions the generator can use directly in the next RALPH iteration. Not vague ("improve error handling") but precise ("add try/catch in processOrder() at line 45 for the case where inventory API returns 503").
```

- [ ] **Step 8: Write the Anti-Self-Persuasion section**

```markdown
## Anti-Self-Persuasion Protocol

You are an evaluator, not an advocate. Your instinct will be to find reasons things work. Resist it.

**Mandatory self-checks:**
1. Before writing PASS for any AC, ask: "Did I verify this by reading code, or am I trusting the generator's description?"
2. Before giving a score above 7, ask: "Would a skeptical senior engineer agree with this score?"
3. Before writing the overall verdict, ask: "Am I giving PASS because the work is genuinely good, or because I don't want to seem harsh?"

**The Generator Is Not Your Ally:**
- The generator wants to pass. You want accuracy.
- A false PASS wastes more time than a correct FAIL (generator will build on broken foundation)
- "Close enough" is FAIL. Contract criteria are binary.

**Do Not Trust the Report:**
- Generator's self-reported status is input, not evidence
- "DONE" means "claims to be done" — verify independently
- Read the actual code. Run the actual tests. Check the actual output.
```

- [ ] **Step 9: Write the Browser QA section (for UI projects)**

```markdown
## Browser QA Layer (UI Projects Only)

When the contract includes UI acceptance criteria, add browser verification AFTER Layer 3.

**Activation:** Contract contains any AC referencing:
- Visual appearance, layout, responsive behavior
- User interactions (click, type, drag, navigate)
- Page load, routing, client-side state

**Process:**
1. Start the dev server (find start command from package.json/Makefile)
2. Navigate to relevant pages
3. For each UI-related AC:
   - Take screenshot as evidence
   - Interact with elements (click buttons, fill forms, navigate)
   - Verify visual state matches AC description
   - Check responsive behavior if AC mentions it
4. Include screenshots in report as evidence

**Browser QA does NOT replace Layer 3** — it supplements code-level AC verification with runtime visual checks.

**Permissions:** Use browse/playwright tools. Still NO file write/edit.
```

- [ ] **Step 10: Commit easyharness-evaluator SKILL.md**

```bash
cd /root/Github/easyharness
git add skills/easyharness-evaluator/SKILL.md
git commit -m "feat: add easyharness-evaluator skill — 4-layer automated verification engine"
```

---

### Task 2: easyharness-evaluator Few-Shot Calibration Examples

**Files:**
- Create: `skills/easyharness-evaluator/few-shot-examples.md`

- [ ] **Step 1: Write PASS example — Clean implementation**

A complete example showing what a genuine PASS looks like across all 4 layers. The scenario: implementing a `validateEmail` function with proper tests, clean code, all ACs met.

Key characteristics to demonstrate:
- All tests pass including new and existing
- No stubs or shortcuts
- Each AC has clear evidence (test + code reference)
- Quality scores all above threshold
- Feedback is minimal ("No issues found")

- [ ] **Step 2: Write FAIL example — Subtle shortcuts**

A complete example showing a case that LOOKS correct but has hidden issues. The scenario: implementing a `processPayment` function where the generator:
- Tests pass (but one test only checks mock, not real behavior)
- Has a hardcoded exchange rate instead of API call (contract said "fetch live rate")
- Missing error handling for network failures
- Code quality acceptable on surface

Key characteristics to demonstrate:
- Layer 1 passes (tests technically green)
- Layer 2 catches hardcoded value
- Layer 3 catches AC non-compliance (live rate vs hardcoded)
- Feedback is specific and actionable

- [ ] **Step 3: Write FAIL example — Over-engineering**

A case where implementation is technically correct but adds unrequested complexity. The scenario: contract asked for simple file logger, generator built a full logging framework with rotation, compression, and remote shipping.

Key characteristics to demonstrate:
- All ACs technically met
- But "unspecified additions" flagged
- Code quality suffers from unnecessary complexity
- Feedback: "Strip to contract scope"

- [ ] **Step 4: Commit few-shot-examples.md**

```bash
cd /root/Github/easyharness
git add skills/easyharness-evaluator/few-shot-examples.md
git commit -m "feat: add evaluator few-shot calibration examples (PASS + 2 FAIL patterns)"
```

---

### Task 3: easyharness-plan SKILL.md — Requirements to Contract Pipeline

**Files:**
- Create: `skills/easyharness-plan/SKILL.md`

- [ ] **Step 1: Write the YAML frontmatter**

```yaml
---
name: easyharness-plan
description: Use when starting a new feature or project — explores requirements, designs architecture, decomposes into tasks, and generates verifiable contracts for each subtask
---
```

- [ ] **Step 2: Write the Overview section**

Core concept: easyharness-plan bridges user intent to machine-verifiable contracts. It combines brainstorming-style requirement exploration with writing-plans-style task decomposition, then adds a novel contract generation layer.

Key content:
- Purpose: Turn vague requirements into a plan + contract that easyharness-develop can execute autonomously
- Outputs: `plan.md` (implementation plan) + `contract.md` (per-task acceptance criteria)
- Core principle: "Every acceptance criterion must be verifiable without human judgment"
- Flow: explore → design → decompose → contract → self-review

- [ ] **Step 3: Write the Process Flow**

```markdown
## Process Flow

\```dot
digraph if_plan {
    "Explore project context" [shape=box];
    "Ask clarifying questions\n(one at a time, prefer multiple choice)" [shape=box];
    "Propose 2-3 approaches\nwith trade-offs" [shape=box];
    "Present design sections\n(get approval per section)" [shape=box];
    "User approves design?" [shape=diamond];
    "Decompose into tasks\n(bite-sized, TDD-friendly)" [shape=box];
    "Generate contract\n(per-task ACs)" [shape=box];
    "Contract self-review" [shape=box];
    "Self-review passes?" [shape=diamond];
    "User reviews plan + contract" [shape=diamond];
    "Save plan.md + contract.md" [shape=box];
    "Ready for easyharness-develop" [shape=doublecircle];

    "Explore project context" -> "Ask clarifying questions\n(one at a time, prefer multiple choice)";
    "Ask clarifying questions\n(one at a time, prefer multiple choice)" -> "Propose 2-3 approaches\nwith trade-offs";
    "Propose 2-3 approaches\nwith trade-offs" -> "Present design sections\n(get approval per section)";
    "Present design sections\n(get approval per section)" -> "User approves design?";
    "User approves design?" -> "Present design sections\n(get approval per section)" [label="revise"];
    "User approves design?" -> "Decompose into tasks\n(bite-sized, TDD-friendly)" [label="yes"];
    "Decompose into tasks\n(bite-sized, TDD-friendly)" -> "Generate contract\n(per-task ACs)";
    "Generate contract\n(per-task ACs)" -> "Contract self-review";
    "Contract self-review" -> "Self-review passes?";
    "Self-review passes?" -> "Generate contract\n(per-task ACs)" [label="fix issues"];
    "Self-review passes?" -> "User reviews plan + contract" [label="yes"];
    "User reviews plan + contract" -> "Generate contract\n(per-task ACs)" [label="changes"];
    "User reviews plan + contract" -> "Save plan.md + contract.md" [label="approved"];
    "Save plan.md + contract.md" -> "Ready for easyharness-develop";
}
\```
```

- [ ] **Step 4: Write the Requirements Exploration section**

Adapted from brainstorming skill but streamlined (no visual companion, focused on machine-verifiable requirements).

```markdown
## Phase 1: Requirements Exploration

Adapted from superpowers:brainstorming — explore before building.

1. **Check project context**: files, docs, recent commits, existing patterns
2. **Scope assessment**: If request covers multiple independent subsystems, decompose first
3. **Ask clarifying questions**:
   - One question per message
   - Prefer multiple choice (reduces ambiguity)
   - Focus on: purpose, constraints, success criteria, edge cases
   - Ask about testing strategy early ("How will we know this works?")
4. **Propose 2-3 approaches** with trade-offs and your recommendation

**Key difference from brainstorming:** Every question should push toward verifiable outcomes.
- Not: "What should the UI feel like?"
- But: "Should the form validate on blur or on submit? (affects AC wording)"
```

- [ ] **Step 5: Write the Task Decomposition section**

Adapted from writing-plans skill.

```markdown
## Phase 2: Task Decomposition

Follows superpowers:writing-plans conventions.

**File structure first:** Map which files will be created/modified and their responsibilities.

**Bite-sized tasks:** Each task = one focused unit of work (2-5 minutes for an agent).

**Task format:**
\```markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.ext`
- Modify: `exact/path/to/existing.ext`
- Test: `tests/exact/path/to/test.ext`

**Complexity:** simple | medium | complex

**Dependencies:** [Task numbers this depends on, or "none"]

**Description:** [What this task does, in 2-3 sentences]
\```

**Complexity classification** (informs easyharness-develop's retry strategy):
- **simple**: Single file, clear spec, no integration concerns
- **medium**: 2-3 files, some integration, moderate logic
- **complex**: Multi-file, architecture decisions, complex logic or state management
```

- [ ] **Step 6: Write the Contract Generation section**

This is the novel part — not from any existing skill.

```markdown
## Phase 3: Contract Generation

For each task, generate a machine-verifiable contract.

**Contract format (per task):**

\```markdown
## Task N: [name]

### Acceptance Criteria
- [ ] AC-N.1: [Specific verifiable behavior — must be testable without human judgment]
- [ ] AC-N.2: [Another specific behavior]
- [ ] AC-N.3: ...

### Test Requirements
- [What tests must exist]
- [Edge cases that must be covered]
- [Integration points to verify]

### Regression Guard
- [Existing functionality that must NOT break]
- [Existing tests that must continue passing]

### Complexity: simple | medium | complex
\```

**AC Writing Rules:**
1. Every AC must be verifiable by reading code + running tests (no subjective judgment)
2. Use concrete values: "returns 400 status" not "returns an error"
3. Include edge cases: "returns empty array when no results" not just "returns results"
4. Specify error behavior: "throws InvalidInputError when email is empty" not "handles errors"
5. One behavior per AC — if AC contains "and", split it

**Bad ACs (NEVER write these):**
- "Works correctly" — what does "correctly" mean?
- "Handles edge cases" — which ones?
- "Good error messages" — what makes them "good"?
- "Responsive design" — at what breakpoints? What changes?

**Good ACs:**
- "POST /api/users returns 201 with { id, email, createdAt } when valid email provided"
- "POST /api/users returns 400 with { error: 'Email required' } when email is empty string"
- "User list renders max 20 items per page with 'Load more' button when total > 20"
```

- [ ] **Step 7: Write the Contract Self-Review section**

```markdown
## Phase 4: Contract Self-Review

Before presenting to user, review every AC against these checks:

1. **Verifiability**: Can this AC be checked by reading code + running a test? If it requires human judgment ("looks good", "feels responsive"), rewrite it.
2. **Completeness**: Does the set of ACs cover the task description? Any behavior described in the task but missing from ACs?
3. **Non-overlap**: Do any ACs test the same thing? Merge duplicates.
4. **Regression coverage**: For each task that modifies existing code, is there a regression guard?
5. **Testability**: For each AC, can you imagine the test that verifies it? If not, the AC is too vague.

Fix issues inline. Then present plan + contract to user for review.
```

- [ ] **Step 8: Write the Output section**

```markdown
## Output

Save two files:
- `docs/plans/YYYY-MM-DD-<name>.md` — Implementation plan (task decomposition with file paths, code patterns, TDD steps)
- `docs/plans/YYYY-MM-DD-<name>-contract.md` — Contract (per-task ACs, test requirements, regression guards)

After saving, present to user:

> "Plan and contract saved. Please review both files before we proceed:
> - Plan: `<path>`
> - Contract: `<path>`
>
> Ready to start implementation with easyharness-develop?"
```

- [ ] **Step 9: Commit easyharness-plan SKILL.md**

```bash
cd /root/Github/easyharness
git add skills/easyharness-plan/SKILL.md
git commit -m "feat: add easyharness-plan skill — requirements to contract pipeline"
```

---

### Task 4: easyharness-develop SKILL.md — RALPH Loop Orchestration

**Files:**
- Create: `skills/easyharness-develop/SKILL.md`

- [ ] **Step 1: Write the YAML frontmatter**

```yaml
---
name: easyharness-develop
description: Use when executing an implementation plan with a contract — runs a RALPH loop per task with evaluator feedback, retries, and model escalation
---
```

- [ ] **Step 2: Write the Overview section**

```markdown
# EasyHarness-Develop: RALPH Loop Orchestration

Execute a plan + contract by iterating through tasks with automated evaluation and feedback.

**Core principle:** Implement one task at a time. After each task, dispatch an evaluator. If FAIL, incorporate feedback and retry. Never move to the next task until the current one passes.

**You ARE the generator.** You implement the code directly — no subagent dispatch for implementation. Only the evaluator is dispatched as a read-only subagent (using easyharness-evaluator skill).

**RALPH = Reason → Act → Learn → Plan → Hypothesize**
- **Reason**: Read the task, understand what's needed, check dependencies
- **Act**: Implement following TDD (use superpowers:test-driven-development)
- **Learn**: Receive evaluator feedback, understand what went wrong
- **Plan**: Decide what to change in next attempt
- **Hypothesize**: If stuck after 2 failures, consider whether the approach is wrong (not just the implementation)
```

- [ ] **Step 3: Write the Process Flow**

```markdown
## Process Flow

\```dot
digraph if_develop {
    rankdir=TB;

    "Read plan.md + contract.md" [shape=box];
    "Create TodoWrite\n(one todo per task)" [shape=box];
    "Pick next task\nMark in_progress" [shape=box];
    "Extract task contract\n(ACs, test reqs, regression guards)" [shape=box];

    subgraph cluster_ralph {
        label="RALPH Loop (per task)";
        "REASON: Read task,\ncheck deps, plan approach" [shape=box];
        "ACT: Implement with TDD\n(superpowers:test-driven-development)" [shape=box];
        "Dispatch evaluator\n(easyharness-evaluator, read-only)" [shape=box];
        "Evaluator verdict?" [shape=diamond];
        "LEARN: Parse feedback,\nidentify root cause" [shape=box];
        "Retry < 3?" [shape=diamond];
        "PLAN + HYPOTHESIZE:\nAdjust approach" [shape=box];
        "BLOCKED: Report to user" [shape=box, style=filled, fillcolor="#ffcccc"];
    }

    "Mark task complete" [shape=box];
    "More tasks?" [shape=diamond];
    "Final evaluation\n(full contract, all tasks)" [shape=box];
    "Verification\n(superpowers:verification-before-completion)" [shape=box];
    "Done" [shape=doublecircle];

    "Read plan.md + contract.md" -> "Create TodoWrite\n(one todo per task)";
    "Create TodoWrite\n(one todo per task)" -> "Pick next task\nMark in_progress";
    "Pick next task\nMark in_progress" -> "Extract task contract\n(ACs, test reqs, regression guards)";
    "Extract task contract\n(ACs, test reqs, regression guards)" -> "REASON: Read task,\ncheck deps, plan approach";
    "REASON: Read task,\ncheck deps, plan approach" -> "ACT: Implement with TDD\n(superpowers:test-driven-development)";
    "ACT: Implement with TDD\n(superpowers:test-driven-development)" -> "Dispatch evaluator\n(easyharness-evaluator, read-only)";
    "Dispatch evaluator\n(easyharness-evaluator, read-only)" -> "Evaluator verdict?";
    "Evaluator verdict?" -> "Mark task complete" [label="PASS"];
    "Evaluator verdict?" -> "LEARN: Parse feedback,\nidentify root cause" [label="FAIL"];
    "LEARN: Parse feedback,\nidentify root cause" -> "Retry < 3?";
    "Retry < 3?" -> "PLAN + HYPOTHESIZE:\nAdjust approach" [label="yes"];
    "PLAN + HYPOTHESIZE:\nAdjust approach" -> "ACT: Implement with TDD\n(superpowers:test-driven-development)";
    "Retry < 3?" -> "BLOCKED: Report to user" [label="no"];
    "Mark task complete" -> "More tasks?";
    "More tasks?" -> "Pick next task\nMark in_progress" [label="yes"];
    "More tasks?" -> "Final evaluation\n(full contract, all tasks)" [label="no"];
    "Final evaluation\n(full contract, all tasks)" -> "Verification\n(superpowers:verification-before-completion)";
    "Verification\n(superpowers:verification-before-completion)" -> "Done";
}
\```
```

- [ ] **Step 4: Write the Setup section**

```markdown
## Setup

1. Read `plan.md` — extract all tasks, file structure, context
2. Read `contract.md` — extract per-task ACs, test requirements, regression guards
3. Create TodoWrite with one todo per task (include task name + complexity)
4. Verify dependencies: check that required tools exist (test runner, linter, build)
5. Record pre-implementation baseline:
   - Run existing tests, record count + pass/fail
   - Record existing lint status
   - This becomes the regression baseline
```

- [ ] **Step 5: Write the Per-Task RALPH Loop section**

```markdown
## Per-Task RALPH Loop

### REASON (Before Coding)

1. Read the task description from plan
2. Read the task's contract section (ACs, test reqs, regression guards)
3. Check dependencies — are prerequisite tasks complete?
4. If this is a retry: read previous evaluator feedback carefully
5. Plan your approach in 2-3 sentences (for your own clarity)

### ACT (Implementation)

Follow TDD strictly (superpowers:test-driven-development):
1. Write failing test for first AC
2. Verify it fails
3. Write minimal code to pass
4. Verify it passes
5. Repeat for remaining ACs
6. Run full test suite — no regressions
7. Run lint + build if applicable

**If retry:** Focus on the specific feedback from evaluator. Don't rewrite everything — fix the identified issues.

### EVALUATE (Dispatch Evaluator)

Dispatch easyharness-evaluator as a **read-only subagent**:

\```
task(
  category="unspecified-high",
  load_skills=["easyharness-evaluator"],
  run_in_background=false,
  description="Evaluate Task N: [name]",
  prompt=<see evaluator-dispatch-prompt.md template>
)
\```

**Evaluator gets:**
- Task description
- Full contract for this task (ACs, test reqs, regression guards)
- List of changed files
- Current retry count

**Evaluator returns:** JSON verdict (see easyharness-evaluator output format)

### LEARN (On FAIL)

1. Parse evaluator's `blocking_issues` array
2. Parse evaluator's `feedback` string — this contains specific actionable fixes
3. Categorize: is this a code bug, a missing feature, or a wrong approach?
4. If Layer 1 failed (tests/lint/build): fix is usually mechanical
5. If Layer 2 failed (stubs found): you cut a corner, go back and do it properly
6. If Layer 3 failed (AC not met): re-read the AC, understand what "met" means
7. If Layer 4 failed (quality below threshold): refactor, don't rewrite

### PLAN + HYPOTHESIZE (On Retry)

- Retry 1: Apply feedback directly, fix identified issues
- Retry 2: Step back — is the approach wrong? Consider alternative implementation strategy
- Retry 3 (final): If still failing, STOP. Report BLOCKED with full context.

**Record learnings** after each task (success or failure):
- What worked / what didn't
- Patterns discovered in the codebase
- Gotchas for future tasks
- Pass these learnings as context to subsequent tasks
```

- [ ] **Step 6: Write the Retry Strategy section**

```markdown
## Retry Strategy

| Attempt | Strategy | Context Injection |
|---------|----------|-------------------|
| 1 | Direct implementation | Task description + contract |
| 2 | Apply evaluator feedback | + evaluator feedback from attempt 1 |
| 3 | Reconsider approach | + evaluator feedback from attempts 1-2 + "consider alternative approach" |
| 4+ | BLOCKED | Stop, report to user with all feedback history |

**Max retries per task: 3** (configurable in contract metadata)

**Escalation on BLOCKED:**
Report to user with:
- Task description
- All evaluator feedback (each attempt)
- What was tried
- Why it's stuck
- Suggested next steps (human decision needed)
```

- [ ] **Step 7: Write the Final Evaluation section**

```markdown
## Final Evaluation (After All Tasks)

After all tasks pass individually, run a final full evaluation:

1. Dispatch easyharness-evaluator with the COMPLETE contract (all tasks)
2. This catches cross-task integration issues individual evaluations missed
3. Use superpowers:verification-before-completion before claiming done

**If final evaluation fails:**
- Identify which tasks' interactions caused the failure
- Re-run RALPH loop on affected tasks with integration context
- Re-run final evaluation

**Completion report:**
- Tasks completed: N/N
- Total evaluator runs: X
- Retries needed: Y
- Learnings accumulated: [list]
- Time to completion: Z
```

- [ ] **Step 8: Write the Learnings section**

```markdown
## Cross-Task Learnings

After each task (pass or fail), record:

\```markdown
### Task N Learnings
- **Pattern**: [What worked that future tasks should know]
- **Gotcha**: [What tripped us up]
- **Codebase note**: [Conventions, quirks discovered]
\```

Inject relevant learnings into subsequent task context. This prevents repeating mistakes across the plan.
```

- [ ] **Step 9: Write the Integration section**

```markdown
## Prerequisites

Install dependency skills before using easyharness-develop. See the top-level Prerequisites section for full install instructions.

**Required:**
- [superpowers](https://github.com/obra/superpowers) — provides `test-driven-development` and `verification-before-completion`
- **easyharness-evaluator** — must be installed alongside easyharness-develop

**Optional (for UI projects):**
- [gstack](https://github.com/garrytan/gstack) — provides `browse` skill for browser QA in easyharness-evaluator

## Integration

**Required skills:**
- **easyharness-evaluator** — Dispatched as read-only subagent after each task
- **superpowers:test-driven-development** — TDD discipline during ACT phase
- **superpowers:verification-before-completion** — Final verification before claiming done

**Input:**
- `plan.md` — from easyharness-plan
- `contract.md` — from easyharness-plan

**Output:**
- Implemented code (committed per task)
- Learnings log
- Completion report

**Permissions:**
- You (generator): full access — read, write, edit, terminal
- Evaluator subagent: read-only + terminal (run tests/lint/build) + browser (if UI) — NO write/edit
```

- [ ] **Step 10: Commit easyharness-develop SKILL.md**

```bash
cd /root/Github/easyharness
git add skills/easyharness-develop/SKILL.md
git commit -m "feat: add easyharness-develop skill — RALPH loop orchestration with evaluator feedback"
```

---

### Task 5: easyharness-develop Evaluator Dispatch Prompt Template

**Files:**
- Create: `skills/easyharness-develop/evaluator-dispatch-prompt.md`

- [ ] **Step 1: Write the dispatch prompt template**

This template is used by easyharness-develop when dispatching easyharness-evaluator after each task implementation.

```markdown
# Evaluator Dispatch Prompt Template

Use this when dispatching easyharness-evaluator after completing a task.

\```
task(
  category="unspecified-high",
  load_skills=["easyharness-evaluator"],
  run_in_background=false,
  description="Evaluate Task N: [task name]",
  prompt="""
    You are evaluating Task N: [task name]

    ## Contract

    [PASTE the full contract section for this task from contract.md]

    ## Changed Files

    [List all files created or modified]

    ## Generator's Report

    [What was implemented, what tests were added — but DO NOT trust this blindly]

    ## Retry Context

    Attempt: [1|2|3] of 3
    [If retry, include previous evaluator feedback here]

    ## Your Job

    Run the 4-layer evaluation defined in your easyharness-evaluator skill:
    1. Automated hard checks (tests, lint, build, regression)
    2. Stub/laziness detection
    3. Contract compliance (line-by-line AC verification)
    4. Quality scoring

    [If contract has UI ACs]: Also run browser QA verification.

    Return the structured JSON verdict.

    Work from: [project directory]
  """
)
\```
```

- [ ] **Step 2: Commit evaluator-dispatch-prompt.md**

```bash
cd /root/Github/easyharness
git add skills/easyharness-develop/evaluator-dispatch-prompt.md
git commit -m "feat: add evaluator dispatch prompt template for easyharness-develop"
```

---

### Task 6: Final Review and Cross-Skill Consistency Check

- [ ] **Step 1: Cross-reference check**

Verify consistency across all three skills:
- easyharness-plan's contract format matches what easyharness-evaluator expects to parse
- easyharness-develop's evaluator dispatch matches easyharness-evaluator's input expectations
- Status protocol consistent (PASS/FAIL, DONE/BLOCKED/NEEDS_CONTEXT)
- Output format JSON matches between easyharness-develop's expectations and easyharness-evaluator's output
- Skill cross-references use correct names (`easyharness-plan`, `easyharness-develop`, `easyharness-evaluator`)

- [ ] **Step 2: Verify all files exist with correct structure**

```bash
find /root/Github/easyharness/skills -type f -name "*.md" | sort
```

Expected:
```
skills/easyharness-develop/SKILL.md
skills/easyharness-develop/evaluator-dispatch-prompt.md
skills/easyharness-evaluator/SKILL.md
skills/easyharness-evaluator/few-shot-examples.md
skills/easyharness-plan/SKILL.md
```

- [ ] **Step 3: Word count check (token efficiency)**

```bash
wc -w skills/*/SKILL.md skills/*/few-shot-examples.md skills/*/evaluator-dispatch-prompt.md
```

Target: Each SKILL.md under 1500 words (these are complex skills, but token efficiency matters).

- [ ] **Step 4: Final commit**

```bash
cd /root/Github/easyharness
git add -A
git commit -m "feat: complete easyharness skill system (easyharness-plan + easyharness-develop + easyharness-evaluator)"
```
