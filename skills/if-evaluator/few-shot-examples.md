# Few-Shot Examples: IF-Evaluator Calibration

## Example 1: PASS — Clean Implementation
**Scenario**: Implementing a `validateEmail` function with proper regex and unit tests.
**Contract ACs**:
1. Support standard TLDs.
2. Reject missing @.
3. Reject multiple @.

**JSON Output**:
```json
{
  "verdict": "PASS",
  "layer_results": {
    "automated_checks": {
      "tests": { "status": "passed", "count": 12 },
      "lint": { "status": "clean" },
      "build": { "status": "success" },
      "regression": { "status": "clean" }
    },
    "stub_detection": { "status": "clean", "findings": [] },
    "contract_compliance": {
      "AC-1": { "status": "PASS", "evidence": "Implementation in utils.js:12 handles .com, .net, .org; verified in tests/validate.test.js:4" },
      "AC-2": { "status": "PASS", "evidence": "Regex correctly checks for exactly one '@' character; verified in tests/validate.test.js:15" },
      "AC-3": { "status": "PASS", "evidence": "Tested with 'user@@example.com', correctly returns false; implementation in utils.js:14" }
    },
    "quality_scores": {
      "functionality": 10,
      "regression": 10,
      "code_quality": 9,
      "test_coverage": "PASS"
    }
  },
  "feedback": "Implementation is clean, well-tested, and meets all criteria.",
  "blocking_issues": []
}
```
**Commentary**: All automated checks passed. No stubs were found. Each AC was verified with direct code inspection and test evidence.

---

## Example 2: FAIL — Subtle Shortcuts
**Scenario**: Implementing a `processPayment` function.
**Contract ACs**:
1. Fetch live exchange rate from ExternalAPI.
2. Handle network failures gracefully.

**JSON Output**:
```json
{
  "verdict": "FAIL",
  "layer_results": {
    "automated_checks": {
      "tests": { "status": "passed", "count": 5 },
      "lint": { "status": "clean" },
      "build": { "status": "success" },
      "regression": { "status": "clean" }
    },
    "stub_detection": { "status": "found", "findings": ["payment.js:25 - hardcoded exchange rate (const RATE = 1.1)"] },
    "contract_compliance": {
      "AC-1": { "status": "FAIL", "reason": "Implementation uses hardcoded value instead of live API call. AC-1 explicitly required ExternalAPI integration." },
      "AC-2": { "status": "FAIL", "reason": "No try/catch or failure handling found in payment.js around API call block. Tests only verify mock success." }
    },
    "quality_scores": {
      "functionality": 4,
      "regression": 9,
      "code_quality": 5,
      "test_coverage": "FAIL"
    }
  },
  "feedback": "Replace hardcoded RATE in payment.js:25 with call to ExternalAPI. Implement try/catch block to handle network errors as required by AC-2.",
  "blocking_issues": ["Hardcoded exchange rate", "Missing network failure handling"]
}
```
**Commentary**: Layer 2 caught a hardcoded value that violated the "fetch live rate" requirement. Layer 3 identified that while tests passed (on the mock), the actual requirement was not implemented.

---

## Example 3: FAIL — Over-Engineering
**Scenario**: Contract asked for a simple file logger.
**Contract ACs**:
1. Log strings to /logs/app.log.
2. Append-only mode.

**JSON Output**:
```json
{
  "verdict": "FAIL",
  "layer_results": {
    "automated_checks": {
      "tests": { "status": "passed", "count": 20 },
      "lint": { "status": "clean" },
      "build": { "status": "success" },
      "regression": { "status": "clean" }
    },
    "stub_detection": { "status": "clean", "findings": [] },
    "contract_compliance": {
      "AC-1": { "status": "PASS", "evidence": "logger.js:5 writes to specified path." },
      "AC-2": { "status": "PASS", "evidence": "fs.appendFileSync used in logger.js:8." }
    },
    "quality_scores": {
      "functionality": 9,
      "regression": 10,
      "code_quality": 5,
      "test_coverage": "PASS"
    }
  },
  "feedback": "Remove unnecessary log rotation (logger.js:20-50), compression logic (logger.js:55), and remote shipping (logger.js:70-90). These were not requested and add maintenance burden.",
  "blocking_issues": ["Excessive complexity", "Unrequested feature additions"]
}
```
**Commentary**: ACs were technically met, but the generator added significant unrequested functionality (rotation, compression, remote shipping). This violates the "minimalist implementation" principle and flags Layer 3's over-engineering check.
