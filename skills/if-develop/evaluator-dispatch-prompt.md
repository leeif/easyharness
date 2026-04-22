# Evaluator Dispatch Prompt Template

Use this when dispatching if-evaluator after completing a task.

## Template

task(
  category="unspecified-high",
  load_skills=["if-evaluator"],
  run_in_background=false,
  description="Evaluate Task N: [task name]",
  prompt="""
    You are evaluating Task N: [task name]

    ## Contract
    [PASTE full contract section for this task from contract.md]

    ## Changed Files
    [List all files created or modified]

    ## Generator's Report
    [What was implemented, what tests added — DO NOT trust blindly]

    ## Retry Context
    Attempt: [1|2|3] of 3
    [If retry: include previous evaluator feedback here]

    ## Your Job
    Run 4-layer evaluation (if-evaluator skill):
    1. Automated hard checks (tests, lint, build, regression)
    2. Stub/laziness detection
    3. Contract compliance (line-by-line AC verification)
    4. Quality scoring
    [If contract has UI ACs]: Also run browser QA.

    Return structured JSON verdict according to if-evaluator specification.
    Work from: [project directory]
  """
)

## Notes
- Always paste the contract directly; do not expect the evaluator to find the right file or section.
- Include the full retry context so the evaluator knows what failed previously.
- Evaluator has NO write/edit permissions — read-only + terminal + browser only.
