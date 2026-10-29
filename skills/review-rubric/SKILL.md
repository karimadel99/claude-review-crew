---
name: review-rubric
description: Shared severity scale, finding format, and verdict rules used by every claude-review-crew agent. Load before writing any review finding.
---
# Review rubric

## Severity
| Level | Meaning | Blocks merge? |
|---|---|---|
| **P0 Blocker** | Data loss, security hole, broken build, crash on a main path | Yes |
| **P1 Major** | Wrong behaviour on a realistic input, contract drift vs API spec, missing auth check | Yes |
| **P2 Minor** | Edge case, perf smell, weak test, a11y gap | No — fix or ticket |
| **P3 Nit** | Naming, style, dead code | No |

## Finding format (one per finding)
```
[P1] <file>:<line> — <one-sentence defect>
Why: <concrete failure scenario: input/state → wrong result>
Fix: <specific change, or a code snippet ≤ 10 lines>
```

## Rules
- Every finding needs a failure scenario. No scenario → it's a nit or it's dropped.
- Report what is wrong, not what is fine. No praise sections.
- Dedupe across agents: same file+line+root cause = one finding, keep the most specific.
- Verdict: any P0/P1 → REQUEST CHANGES; only P2/P3 → APPROVE WITH NITS; none → APPROVE.
- Bilingual mode: Arabic summary (3–5 bullets, Egyptian-professional register) first, then the full English report.

## Footer
End every full report with exactly:
`— MorviLabs review-crew · v0.1.1`
