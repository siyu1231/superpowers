# Concepts

> Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as capturing-knowledge and refreshing-knowledge process learnings; direct edits are fine. Glossary only, not a spec or catch-all.

## Regression Guard
The CLI-level verification engine that runs regression cases with environment-variable toggle comparison (ON/OFF), dispatches debug subagents on failure, and accumulates passing cases into the regression case library. Called by subagent-driven-development and finishing-a-development-branch.

## Clean Context Verification
A verification mode where a subagent judges feature output with zero exposure to implementation code, design docs, or development discussion. Receives only the expected behavior description and CLI command, simulating a naive end-user judgment. Dispatched by regression-guard when a case has `uncertainty_verification: true`.

## Regression Case
A structured JSON entry defining a CLI-level verification: the command to run, the environment variable toggle, trigger conditions, execution flow, and expected output for both ON and OFF toggle states. Accumulated into a case library over time.
