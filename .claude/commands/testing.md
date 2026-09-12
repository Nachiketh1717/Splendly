---
description: Write tests via the test-writer subagent, then run and diagnose them via the test-runner subagent.
---

Run the full testing workflow for Spendly, in this order:

1. Invoke the `test-writer` subagent with the scope given in `$ARGUMENTS` (e.g. a route, file, or feature to cover). If `$ARGUMENTS` is empty, ask what to write tests for before proceeding, or infer sensible scope from recent changes (`git diff`/`git status`) if that's clearly what's intended.
2. Wait for `test-writer` to finish before continuing — do not run it in parallel with step 3, since the next step depends on the tests it just wrote.
3. Invoke the `test-runner` subagent to run the full pytest suite and diagnose any failures.
4. Summarize for the user: what tests were added, the pytest pass/fail results, and any diagnosis of failures from test-runner. Do not apply any fixes yourself — this project requires explicit confirmation before code changes.
