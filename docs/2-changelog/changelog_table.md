# Changelog Table

| Version   | Commit Message                                                                 |
| --------- | ------------------------------------------------------------------------------ |
| `1.20.1`  | fix(conversation): tolerate malformed records in the conversation list (#17497) |

## Changelog Summary

- **1.20.1** — Conversation search/list parsing tolerates individual malformed records (drop+warn), so one corrupt conversation no longer blanks the whole history list (#17497); single/batch paths stay strict; systemic backend errors rethrown.
