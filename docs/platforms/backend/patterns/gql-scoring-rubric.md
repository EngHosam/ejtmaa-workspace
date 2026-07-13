# GQL Scoring Rubric (0-100)

Use this rubric after running the evaluation suite.

## Scoring Dimensions

## 1) Contract Correctness (0-20)

- 0-5: major schema/bridge contract gaps
- 6-12: partial correctness with notable gaps
- 13-17: mostly correct, minor fixes needed
- 18-20: fully correct contract implementation

## 2) Security Boundary (0-20)

- 0-5: missing principal checks or leakage risks
- 6-12: principal checks present but incomplete
- 13-17: solid security with minor caveats
- 18-20: complete scoped security boundaries

## 3) Query and Pagination Stability (0-15)

- 0-4: unbounded/unstable query behavior
- 5-9: bounded but inconsistent ordering/modes
- 10-13: stable with minor edge gaps
- 14-15: deterministic and coherent

## 4) ORM Mapping Reliability (0-15)

- 0-4: missing required attrs for key paths
- 5-9: partial attr mapping reliability
- 10-13: reliable mapping with minor gaps
- 14-15: complete and explicit attr mapping

## 5) Performance and Depth Guardrails (0-15)

- 0-4: guardrails missing
- 5-9: basic guardrails with weak enforcement
- 10-13: strong guardrails with minor issues
- 14-15: robust and enforced guardrails

## 6) Style Compatibility (0-15)

- 0-4: clear drift from Ejtmaa style
- 5-9: mixed compatibility
- 10-13: mostly compatible
- 14-15: fully Ejtmaa-compatible

## Final Verdict

- **90-100**: Accept
- **80-89**: Accept with minor revision
- **70-79**: Revise before merge
- **<70**: Reject and re-design

## Hard Fail Conditions (automatic reject)

Reject regardless of score if any:

1. missing principal security on scoped query,
2. unbounded root-many retrieval,
3. unresolved ORM attr dependency in bridge logic,
4. blacklist violation without explicit exception approval,
5. framework boot/registration invariant breakage.
