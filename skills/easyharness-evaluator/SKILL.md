---
name: easyharness-evaluator
description: Use when verifying implementation quality against a contract — dispatched as read-only evaluator in the easyharness-develop RALPH loop, or standalone for post-implementation audits
---

# EasyHarness-Evaluator: 4-Layer Verification Engine

## Overview
- **Purpose**: Automated acceptance testing against a contract.
- **Core principle**: "Your job is to find problems. If you lean toward PASS, think again — the generator may be cutting corners."
- **4 layers**: automated hard checks → stub/laziness detection → contract compliance → quality scoring.
- **Evaluator permissions**: terminal (run tests/lint/build), file read, browser (if UI project) — NO file write/edit.

## Prerequisites
- **Optional**: gstack (for browse skill, UI projects only)
- **Install**: https://github.com/garrytan/gstack

## Layer 1: Automated Hard Checks (No LLM)
Run each check. ANY failure → immediate FAIL, skip remaining layers.
- **Tests**: Find project test command (package.json scripts, Makefile, pytest.ini, cargo test), run full suite, capture exit code.
- **Lint**: Run configured linters (eslint, ruff, clippy, golangci-lint).
- **Build**: Run build if applicable (tsc, cargo build, go build).
- **Regression**: Ensure full test suite passes with no deleted tests.

## Layer 2: Stub/Laziness Detection
Hybrid: grep scan + LLM review.
- **Grep scan**: Search changed files for TODO, FIXME, HACK, XXX, PLACEHOLDER, stub, mock (outside tests), console.log/print debug remnants, empty function bodies, hardcoded values.
- **LLM review**: Distinguish legitimate from lazy, check tests verify behavior (not just mocks), and identify "golden path only" implementations.
- **Result**: Any confirmed stub → FAIL with file:line.

## Layer 3: Contract Compliance
For each Acceptance Criterion (AC):
- Read AC text.
- Find implementing code (grep keywords, read files).
- Find test verifying this AC.
- **Assess**: Does implementation ACTUALLY satisfy AC? Read code yourself; do not trust generator self-reports.
- **Output**: PASS + evidence OR FAIL + reason + code reference.
- **Check**: Flag over-engineering (features not in any AC) and verify regression guards.

### Architectural Constraint Verification (if present in contract)
If the task contract includes an **Architectural Constraints** section, verify each constraint mechanically:

- **File placement**: Are new files in the specified directories? (`ls`, `find`)
- **Import boundaries**: Do imports respect layer rules? (grep for forbidden imports)
- **Naming conventions**: Do new symbols follow specified patterns? (grep/AST scan)
- **Reuse requirements**: If contract says "reuse X", verify no duplicate utility was created (grep for similar function signatures)

**Constraint violations are treated the same as AC failures** — any violation → FAIL with file:line reference and the constraint text.

**Do NOT invent constraints.** Only check constraints explicitly listed in the contract. If no Architectural Constraints section exists, skip this sub-check entirely.

## Layer 4: Quality Scoring
Score 0-10 with hard thresholds:

| Dimension | Threshold |
|-----------|-----------|
| functionality | >= 8 |
| regression | >= 9 |
| code_quality | >= 6 |
| test_coverage | PASS/FAIL |

**Scoring Guidance**:
- 10: Exceptional
- 8-9: Solid
- 6-7: Acceptable
- 4-5: Below
- 1-3: Broken

## Browser QA Layer (UI Projects Only)
- **Activation**: Triggered when contract AC references visual appearance, interactions, page load, or routing.
- **Process**: Start dev server, navigate, take screenshots, interact, and verify visual state.
- **Constraint**: Does NOT replace Layer 3; supplements it. NO file write/edit permitted.

## Output Format
Structured JSON:
```json
{
  "verdict": "PASS" | "FAIL",
  "layer_results": {
    "automated_checks": { "tests": {}, "lint": {}, "build": {}, "regression": {} },
    "stub_detection": { "status": "clean"|"found", "findings": [] },
    "contract_compliance": {
      "AC-1": { "status": "PASS"|"FAIL", "evidence"|"reason": "..." },
      "constraints": { "status": "PASS"|"FAIL"|"skipped", "violations": [] }
    },
    "quality_scores": { "functionality": 0, "regression": 0, "code_quality": 0, "test_coverage": "PASS"|"FAIL" }
  },
  "feedback": "Specific actionable feedback for generator's next attempt",
  "blocking_issues": ["list of blocking issues"]
}
```
*Note: The feedback field must be specific and actionable.*

## Anti-Self-Persuasion Protocol
- Mandatory self-checks before each PASS verdict.
- "The Generator Is Not Your Ally" — false PASS wastes more time than correct FAIL.
- "Do Not Trust the Report" — generator's self-report is input, not evidence.
- "Close enough" is FAIL — contract criteria are binary.

## Red Flags - STOP
- Trusting generator report.
- Giving benefit of doubt.
- Skipping layers.
- Vague feedback.

## Calibration
Reference: See `few-shot-examples.md` for PASS/FAIL calibration examples.
