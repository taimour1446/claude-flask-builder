# Contributing to claude-flask-builder

Thanks for your interest. This plugin is opinionated and audit-driven —
~334 patterns are locked in via the reviewer agent.

## Reporting issues
- For a defect in the rules themselves, open an issue with a code sample
  showing the false-positive or missed violation.
- For a missing pattern from a real-world Flask codebase, open an issue
  describing the pattern + a link to a public example.

## Submitting a change
1. Fork + branch off `main`
2. Keep diffs surgical — don't expand scope
3. If you add a rule, add it to `coding-standards.md` with a numeric R-id,
   AND describe how the reviewer should detect it
4. Run a real `flask-scaffolder` invocation on a throwaway folder to test
5. PR with a description of what audit-gap or defect this addresses

## What we won't accept
- New libraries outside the Locked Stack (see SKILL.md) without compelling justification
- Loosening of any security rule in `security.md`
- Removing the @staticmethod / explicit-field-assignment / structured-logging rules
