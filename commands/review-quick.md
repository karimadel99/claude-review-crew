---
description: 'Fast single-pass review for small diffs (<200 lines). Usage: /review-quick [path | PR# | staged]'
argument-hint: "[path | PR# | staged]"
---
Perform a quick single-agent review of: $ARGUMENTS (default: current uncommitted diff).

Do not fan out to specialists. Load the `claude-review-crew:review-rubric` skill. Resolve the stack the same way /review does (`stack` in `review.config.json`, otherwise detect `react`, `next`, `vue`, or `node` from the nearest `package.json`) and keep that framework's common failure modes in mind. Review the diff yourself for correctness, obvious security issues, and missing tests, and print findings in rubric format, most severe first. Max 8 findings. End with APPROVE / APPROVE WITH NITS / REQUEST CHANGES.
