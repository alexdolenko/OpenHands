# Resilient Conversation List Parse (#17497) Implementation Plan

## Intent

One corrupt/malformed conversation record must not blank the entire conversation
history list. Desired outcome: the list renders every well-formed conversation
and silently skips the broken one(s), so a user with a single bad record on disk
keeps a working history instead of being forced to delete all local conversation
data. Unacceptable outcome: a well-formed conversation disappears from the list,
or a genuinely broken *single-conversation* fetch (opening one conversation) is
masked as "not found"/"no permission" instead of surfacing its real parse error.
Tension: resilience_vs_silent_data_loss — dropping is the point for the list, but
each drop must leave a console breadcrumb naming the offending record, and the
single-open path must stay strict so corruption is still diagnosable there.

## Boundaries

- Inherit project non-negotiables (AGENTS.md API-access rules, no-magic-strings,
  TDD). No new dependency, no schema change, no UI/i18n change in v1.
- **Only the search/list path becomes tolerant.** The single-conversation and
  batch paths (`requireDirectConversationInfo`, `requireDirectConversationItems`,
  `batchGetAppConversations`) stay strict and keep throwing on a malformed
  record — opening one bad conversation must still surface the error, not
  resolve to `null`. (`requireConversationSearchPage` and the service's use of
  `toConversationPage` were the old strict search path and are being replaced,
  not retained.)
- Tolerance must cover the **whole per-record pipeline** for the list — both
  validation (`requireDirectConversationInfo`) and conversion
  (`toAppConversation`, which can throw via `combineUsageMetrics`) — so a
  valid-ID record with malformed `stats` cannot blank the list either.
- A kept item still satisfies the same invariants as before (non-empty string
  `id`, planner-parent conversations still filtered out of the list).
- Non-array/non-record page shape is still a real protocol error and still
  throws. Only per-item failures are tolerated.
- No behavior change to the cloud path (`searchCloudConversations` /
  `batchGetCloudConversations`).

## Unknowns & Assumptions

```yaml
- claim: The single-conversation open path routes through batchGetAppConversations, so making batch tolerant would mask a corrupt single conversation as null.
  state: verified
  source: src/hooks/query/use-user-conversation.ts:57 (`batchGetAppConversations([cid])`); updateConversationTitle (agent-server-conversation-service.api.ts:963-967) then calls requireAppConversation which THROWS on null (line 347-355)
  checked: 2026-09-17
- claim: toAppConversation is NOT total — a valid-ID record with malformed stats can throw during conversion, downstream of item validation.
  state: verified
  source: normalizeStats shallow-casts (agent-server-conversation-service.api.ts:150-153); combineUsageMetrics iterates Object.values(stats.usage_to_metrics) and dereferences each entry (src/utils/conversation-metrics.ts:28); toConversationPage maps unguarded (agent-server-adapter.ts:439-441)
  checked: 2026-09-17
- claim: The service file can host the resilient builder without a circular import — it defines requireDirectConversationInfo locally, already imports toAppConversation from the adapter, and already imports from #/utils/plan-file where the planner tag constant lives.
  state: verified
  source: agent-server-conversation-service.api.ts imports toAppConversation from ../agent-server-adapter (line 47-48) and isPlannerConversationOf from #/utils/plan-file (line 66); LOCAL_PLANNER_PARENT_TAG_KEY is defined and exported in src/utils/plan-file.ts:6
  checked: 2026-09-17
- claim: The list planner-parent filter must be preserved in the tolerant path.
  state: verified
  source: toConversationPage filters `!item.tags?.[LOCAL_PLANNER_PARENT_TAG_KEY]` (agent-server-adapter.ts:440)
  checked: 2026-09-17
```

## Decisions

```yaml
- date: 2026-09-17
  fork: v1 surfacing when a list record is dropped
  choice: console.warn including the item index, the attempted id (if readable, else "<unknown>"), and the caught error's message; drop the record; no user-facing UI notice in v1
  overrides: null
- date: 2026-09-17
  fork: batchGetAppConversations behavior on a corrupt requested id
  choice: keep batch STRICT (throws) — REVERSED from the initial "also drop" decision after Codex P1 showed the single-open path (use-user-conversation) and requireAppConversation callers depend on batch throwing; only the search/list path becomes tolerant
  overrides: null
- date: 2026-09-17
  fork: scope of the tolerant catch — validation-only vs whole per-record pipeline
  choice: wrap the whole per-record pipeline (validate + convert) for the search path, after Codex P1 showed toAppConversation can throw via combineUsageMetrics; a validation-only catch would still blank the list on malformed stats
  overrides: null
- date: 2026-09-17
  fork: user-facing "N couldn't load" notice
  choice: deferred to backlog v2 (list UI + i18n); v1 is console-only drop+warn
  overrides: null
```

## Autonomy

```yaml
autonomy:
  proceed_freely: [source edit in agent-server-conversation-service.api.ts, test edits/additions, running gate commands, worktrees, docs]
  stop_and_ask: [changing any batch/single-conversation path, touching the adapter or cloud path, new dependency, UI/i18n changes, project hard-stops]
  forbidden: [deploy, production data, credential changes, editing the PR HUMAN: section]
```

## Graph

```graph
# trace: acceptance
# id                 scope                                                                                                                          deps  accept                          must-not
fix-resilient-parse  src/api/conversation-service/agent-server-conversation-service.api.ts __tests__/api/agent-server-conversation-service-regressions.test.ts  -     AC-1,AC-2,AC-3,AC-4,AC-5        -
```

## Acceptance

- AC-1: `searchConversations` drops a malformed (missing/empty-`id`) list record and returns its well-formed siblings without throwing; an all-bad page resolves to empty items.
- AC-2: `searchConversations` drops a valid-`id` record whose malformed `stats` makes conversion (`combineUsageMetrics`) throw, keeping siblings — proving the catch spans validation and conversion, not validation only.
- AC-3: the single-conversation and batch paths stay strict — a corrupt record fetched via `batchGetAppConversations` / single-open still throws, not silently dropped.
- AC-4: each dropped record emits a `console.warn` containing the item index, the attempted id (`<unknown>` when unreadable), and the caught error message.
- AC-5: planner-parent records are still filtered from the list (not counted as drops), and a non-array/non-record page shape still throws.

## Overview

Make the conversation **search/list** path tolerant of individual malformed
records while leaving the single-conversation and batch paths strict. A new
resilient page builder validates and converts each raw item inside a try/catch,
logs and drops a bad record, and returns the rest, so the history list renders
instead of blanking when one record is corrupt (#17497).

## Problem Statement

`searchConversations` (src/api/conversation-service/agent-server-conversation-service.api.ts:912)
builds the list via `toConversationPage(requireConversationSearchPage(data))`.
Two throw seams live on that path, each all-or-nothing:

1. `requireDirectConversationItems` (line 297-304) does
   `items.map(requireDirectConversationInfo)`; a record that is not an object or
   lacks a non-empty string `id` throws (line 266-267), rejecting the whole page.
2. `toConversationPage` (agent-server-adapter.ts:439-441) maps every item through
   `toAppConversation`, which can throw during metric conversion
   (`combineUsageMetrics` dereferences each `usage_to_metrics` entry) for a
   valid-ID record with malformed `stats`.

Either throw blanks the entire list. Users cannot tell which record is bad and
resort to deleting all local conversation data.

## Solution Architecture

Add a **search-only** resilient page builder in the service file (it owns
`requireDirectConversationInfo`, already imports `toAppConversation` from the
adapter, and can import `LOCAL_PLANNER_PARENT_TAG_KEY` from `#/utils/plan-file`
by extending its existing `isPlannerConversationOf` import, so no circular
dependency and no adapter change). It:

1. Parses the page **shape** only (array, or `{ items, next_page_id }`); a
   non-array/non-record page still throws `invalidConversationResponse()`.
2. Iterates raw items with their index; for each, inside `try/catch`:
   validate (`requireDirectConversationInfo`) → skip planner-parent
   conversations (a filter, not an error) → convert (`toAppConversation`) → keep.
   On a **per-record** throw: `console.warn(index, attempted id, error message)`
   and drop. **Exception:** a systemic `NoBackendAvailableError` (the active
   backend was removed/switched mid-search, so `toAppConversation` fails for
   every row) is rethrown, not swallowed — otherwise every valid row would be
   reported as malformed and the query would resolve with a silently empty page.
3. Returns `{ items: kept, next_page_id }`.

`searchConversations` (local branch) calls this builder instead of
`toConversationPage(requireConversationSearchPage(data))`. Everything else is
untouched, so batch and single-conversation loads keep throwing on corruption.

## Implementation Details

### 1. Resilient search page builder + wire-up

**File**: `src/api/conversation-service/agent-server-conversation-service.api.ts`

**Current state**: `searchConversations` local branch returns
`toConversationPage(requireConversationSearchPage(data))`. Item validation
(`requireDirectConversationItems`) and conversion (`toConversationPage`) both
throw on a single bad record.

**Modifications**:

- Add a module-private helper (e.g. `toResilientConversationPage(page: unknown): AppConversationPage`)
  implementing the three-step design above. Reuse the existing `isRecord` guard
  and `invalidConversationResponse()` for the page-shape check; reuse the local
  `requireDirectConversationInfo` and the imported `toAppConversation`.
- Import `LOCAL_PLANNER_PARENT_TAG_KEY` from `#/utils/plan-file` (add it to the
  existing `isPlannerConversationOf` import at line 66) to preserve the
  planner-parent list filter. Use the raw tag key, not `isPlannerConversationOf`
  — the list filter matches on tag *presence*, while `isPlannerConversationOf`
  matches a specific parent id.
- In the per-record `catch`, rethrow systemic backend failures via
  `isNoBackendAvailableError(error)` (imported from `../agent-server-client-options`)
  before the warn+drop, so a mid-search backend removal surfaces the real error
  instead of an empty page.
- In `searchConversations`, replace the local-branch return with
  `return toResilientConversationPage(data);`. Cloud branch unchanged.
- The warn string is a developer diagnostic (console only), not user-facing copy
  (i18n Rule 1 does not apply) and not a reused program identifier (Rule 2 does
  not apply) — keep it a plain inline string.
- Remove `requireConversationSearchPage` (it becomes unused — only
  `searchConversations` called it) and drop the now-unused `toConversationPage`
  import from the service (adapter line 48 import). The exported adapter
  `toConversationPage` itself stays (adapter untouched; still covered by
  `agent-server-adapter.test.ts`). Keep `requireDirectConversationItems` (still
  used by `batchGetAppConversations`).

No batch, single-conversation, adapter, or cloud changes.

## Project-Mandated Impact Section

This project (frontend) mandates no tenancy/isolation section. Repository
ownership (AGENTS.md): frontend parsing of an existing agent-server response —
stays in `OpenHands/OpenHands`; no endpoint, client, or contract change belongs
elsewhere. CI guards (`no-direct-agent-server-calls`, i18n) unaffected. None.

## Technical Considerations

- **API Integration / Frontend modules** (architecture.md §Frontend modules,
  §Security posture): change confined to the service's response-normalization
  layer; no raw HTTP added, so `no-direct-agent-server-calls.test.ts` is
  unaffected.
- **Server-authoritative vs client-trusted**: raw items are
  server-provided-but-untrusted (the issue proves the shape is not guaranteed);
  the tolerant list path treats them accordingly while the strict single/batch
  paths still assert the contract where an error must surface.
- **Edge cases**: empty array → empty page (unchanged). All items bad → empty
  page, list shows its normal empty state. Non-array/non-record page → still
  throws. Planner-parent records → still filtered out (not counted as drops).

## Files to Modify/Create

1. `src/api/conversation-service/agent-server-conversation-service.api.ts` (modify)
   — add `toResilientConversationPage`, import `LOCAL_PLANNER_PARENT_TAG_KEY` from
   `#/utils/plan-file`, wire `searchConversations` to it, remove the unused
   `requireConversationSearchPage` and the unused `toConversationPage` import.
2. `__tests__/api/agent-server-conversation-service-regressions.test.ts` (modify)
   — add #17497 regression tests (see Test Impact). New test file only if this
   one is an unsuitable home.

## Type Definitions

None. Builder returns the existing `AppConversationPage`.

## Performance & Cost Impact

Negligible — same O(n) pass over a ~20-item page; try/catch per item. No new I/O.

## Backward Compatibility

Behavioral change on the list path only: previously-throwing list pages with a
bad record now return the good records. Well-formed pages produce identical
output. Batch/single paths unchanged. No migration.

## Test Impact

- **Affected existing tests**: grep the three
  `agent-server-conversation-service*.test.ts` files for any assertion that a
  malformed **list** item makes `searchConversations` reject; update it to the
  new drop contract. Assertions about batch/single rejection must remain
  green (they still throw) — if any turn red, the change wrongly loosened a
  strict path.
- **New logic needing tests** (regression for #17497), AAA, mocking the
  underlying `ConversationClient.searchConversations` (the service the code
  depends on), not the hook:
  - list drops a no-`id` record: page `[good, bad-no-id, good]` →
    `searchConversations` resolves with 2 items, does not throw.
  - list drops a **valid-ID malformed-stats** record (the P1 #2 case): a record
    whose `stats.usage_to_metrics` holds a malformed entry that makes
    `combineUsageMetrics` throw → dropped, siblings kept, no throw. This proves
    the catch spans conversion, not just validation.
  - warn content (P2): assert `console.warn` was called with a message
    containing the index, the attempted id (and `<unknown>` for the id-less
    case), and the error — not merely that it was called. Cover both a
    known-id and an unknown-id drop.
  - all-bad page → resolves to empty items, no throw.
  - non-array/non-record page → still rejects (shape strictness preserved).
  - planner-parent record → filtered out, not logged as a drop.
  - **strictness preserved**: a single-conversation / batch fetch of a corrupt
    record still throws (guards against loosening the strict paths).
  - **systemic error rethrown** (code-review Major): backend removed mid-search
    (`setRegisteredBackends([])` inside the search mock) → `searchConversations`
    rejects with `NoBackendAvailableError` and warns nothing, proving a systemic
    failure is not swallowed as per-record corruption. Shown red on the pre-fix
    broad catch, green after the rethrow (9 regression tests total).
- **Integration/E2E**: not required for the unit layer — pure parsing logic,
  unit-covered through the real `searchConversations` production path. The repo
  guide's real-app before/after screenshot/video for this user-visible fix is a
  human PR-submission artifact (owner decision), attached at PR time.

## To-dos

- [x] Add `toResilientConversationPage` (shape-parse; per-item try/catch validate→filter→convert; warn index+id+error; drop) and import `LOCAL_PLANNER_PARENT_TAG_KEY`.
- [x] Wire `searchConversations` local branch to the resilient builder; remove `requireConversationSearchPage` if unused.
- [x] Add #17497 regression tests (list drop no-id, list drop malformed-stats, warn-content known+unknown id, all-bad, non-array still throws, planner-parent filtered, single/batch still strict, systemic-error rethrown).
- [x] Rethrow systemic `NoBackendAvailableError` in the per-record catch (code-review Major).
- [x] Run gate: `npm run typecheck`, `npm test` (targeted file first), `npm run lint`, `npm run build`.
- [x] Backlog v2: user-facing "N conversations couldn't be loaded" notice (list UI + i18n) — deferred.
