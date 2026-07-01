# Debug Fix Subagent Prompt Template

Use this template when regression-guard dispatches a debug subagent on
verification failure. Fill all placeholders before dispatching.

```
Subagent (general-purpose):
  description: "Debug: <case.id> — <case.name> (round <round_n>)"
  model: [REQUIRED — choose per SKILL.md Model Selection; an omitted model
         silently inherits the session's most expensive one]
  prompt: |
    A regression case failed verification. Your job: find the root cause
    in the implementation code and fix it.

    **BEFORE you start debugging:** Load the `superpowers:systematic-debugging`
    skill. Follow its methodology — reproduce the failure, trace root cause,
    fix, verify the fix, prevent regression. Do not skip steps.

    Read your debug brief first: <BRIEF_FILE>
    It contains the failure report, the full case definition
    (command, toggle, execution_flow, both expect blocks), and the
    fix contract.

    ## Fix Contract
    After fixing, re-run ONLY the failed CLI command (both TOGGLE=0 and
    TOGGLE=1) and confirm both match expectations. Do NOT run unrelated
    tests or wander into other cases — this is a focused fix.

    ## Before Reporting Back: Self-Review

    Review your fix with fresh eyes:
    - Did I identify the root cause (not just patch a symptom)?
    - Does the fix handle the edge case the case definition describes?
    - Did I avoid changing code unrelated to this failure?
    - Did I verify BOTH toggle states?

    ## Report Format

    Write your full report to <REPORT_FILE>. Include:
    - Root cause (what was broken and why)
    - What you changed (files, line ranges, rationale)
    - Fix verification: exact commands run (with toggle values), output
      observed, exit codes for both ON and OFF states
    - Any concerns or deferred issues

    Then return ONLY (under 15 lines — the detail lives in the report file):
    - **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED
    - Commits created (short SHA + subject)
    - Fix verification summary (both toggle states confirmed)
    - Your concerns, if any
    - The report file path

    **Use DONE_WITH_CONCERNS** if the fix works but you suspect fragility,
    or the root cause may have other manifestations you didn't investigate.

    **Use BLOCKED** if:
    - The root cause is unclear after investigation
    - The fix requires touching code you don't understand
    - You suspect the case expectations may be wrong (not the code)

    If BLOCKED, put the specifics in your return message itself — the
    controller acts on it directly. Describe what you investigated, what
    you found, and what kind of help you need.

    Never silently produce a fix you're unsure about. Bad work is worse
    than no work.
```
