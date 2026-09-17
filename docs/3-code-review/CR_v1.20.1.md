# Code Review — v1.20.1

**Feature**: Resilient conversation list parse (#17497)
**Plan**: `docs/1-plans/F_resilient-conversation-list-parse.plan.md`
**Reviewer**: Codex (gpt-5.6-sol, xhigh) — multi-round loop, 3 substantive rounds + SHA re-anchor, converged APPROVED
**Reviewed Tree**: 87fefadfe0811959d98e528b0fde380fc2ad5a15 (base 2c9a66c96cb40a0ac4560a344244954be40b77af)

## Summary

Only the conversation search/list path is made tolerant of individual malformed
records: a new `toResilientConversationPage` validates and converts each raw item
inside a try/catch, warns (index + attempted id + error) and drops a bad record,
preserves the planner-parent filter, and still throws on a non-array/non-record
page shape. Single-conversation and batch paths remain strict. Systemic backend
failures (`NoBackendAvailableError`) are rethrown rather than swallowed.

## Findings

### Major Issues

- **Systemic backend errors were swallowed as malformed records** — `src/api/conversation-service/agent-server-conversation-service.api.ts:351`. **Addressed.** `isNoBackendAvailableError` now rethrows lifecycle/configuration failures before the per-record warning and drop path. Regression coverage at `__tests__/api/agent-server-conversation-service-regressions.test.ts:348`.
  - `proof: regression-test`
  - `recurrence-guard: new:__tests__/api/agent-server-conversation-service-regressions.test.ts`

- **Production-facing visual evidence was absent** — `.agents/skills/custom-codereview-guide.md:151`. **Accepted with override.** The owner accepted the base-red/HEAD-green production-function regression evidence and deferred the real-app screenshot/video to PR submission rather than treating it as a code defect.
  - `proof: existing-evidence`
  - `recurrence-guard: none:owner accepted PR-time visual evidence as a procedural follow-up`

### Minor Issues

- **Plan described every thrown error as warn-and-drop** — `docs/1-plans/F_resilient-conversation-list-parse.plan.md`. **Addressed.** The plan now documents the systemic-error exception in Solution Architecture, Implementation Details, Test Impact (ninth regression), and the To-do inventory. A whole-change sweep found no surviving unconditional-drop claim.

### Suggestions

None.

## Checklist

- [x] 1. Functional Requirements — passed
- [x] 2. Code Quality — passed
- [x] 3. Architectural Compliance — passed
- [x] 4–9. not applicable — no project-defined sections
- [x] 10. Error Handling — passed
- [x] 11. Security — passed; no authentication, authorization, secret, or routing changes
- [x] 12. Performance — passed; one linear pass over the existing page
- [x] 13. Test & Gate Integrity — passed; assertions and gate configuration were not weakened

**Blocking gates** (re-run at ship): build clean · 26 tests pass (9 new) · typecheck 0 · lint 0/0.

## Verdict

**APPROVED with observations**

All implementation and documentation findings are resolved. The owner accepted
deferring real-app visual evidence to PR submission; the optional user-facing
dropped-record notice remains tracked as follow-up (`docs/1-plans/BACKLOG.md`).
