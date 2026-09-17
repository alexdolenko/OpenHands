# CLAUDE.md — Agent Canvas (OpenHands frontend)

`AGENTS.md` is the detailed source of truth for architecture, API-access rules,
testing rules, and PR requirements. Read it before changing code. This file adds
the plan-build-ship workflow configuration and a contribution-model summary.

## Contribution model (summary — see AGENTS.md for detail)

**This repo (`OpenHands/OpenHands`) is the Agent Canvas React/TypeScript frontend
only.** It is one of a multi-repo system. Put work where it belongs:

| Repo | Owns | Contribute here when |
|------|------|----------------------|
| `OpenHands/OpenHands` (this) | UI, frontend state, backend selection, `src/api/` services that *consume* backend APIs, local-stack orchestration | UI/frontend, or how the frontend *calls* an existing endpoint |
| `OpenHands/software-agent-sdk` | Python SDK, Agent Server, agents/tools, conversations, events, workspaces, **new API endpoints** | new server endpoint, agent/tool logic |
| `OpenHands/typescript-client` (`@openhands/typescript-client`) | typed client mirroring the Agent Server API | client-side access to an endpoint (typed method + types) |
| `OpenHands/extensions` | public skills, automations, integrations | new skill/automation/MCP |
| `OpenHands/automation` | scheduling, webhooks, run history, dispatch | automation lifecycle |

Cross-repo feature order: implement the contract in `software-agent-sdk` → expose
in `typescript-client` → consume in Canvas. Coordinate automation lifecycle in `automation`.

**Hard rules (CI-enforced — break them and CI goes red):**

1. **No raw HTTP to agent-server.** Use `@openhands/typescript-client` client
   classes; assemble options via `src/api/agent-server-client-options.ts`. No
   `axios`/`fetch`/legacy `openHands` instance. Cloud calls go through
   `callCloudProxy()` (`src/api/cloud/proxy.ts`). Guard:
   `src/api/no-direct-agent-server-calls.test.ts`. Narrow allowlist only.
2. **No magic strings.** User-facing text → `t(I18nKey.X)` (declared in
   `src/i18n/translation.json`, regenerated via `npm run make-i18n`). Program
   strings (storage/query/event keys, paths, headers) → named constants. Union
   tags → string-literal types. `i18next/no-literal-string` = error.
3. **TDD tests for behavioral changes** — AAA, mock the service not the hook,
   extend existing test files. See `docs/4-unit-tests/TESTING.md`.
4. **PR `HUMAN:` section is human-only** — never add/edit/move/remove it. If PR
   CI fails on it, stop and ask the human; do not fill it in.
5. Follow `.agents/skills/custom-codereview-guide.md` on every PR. Reviews are
   APPROVE or COMMENT only (never REQUEST_CHANGES).

## Workflow configuration

**Any change that alters what the project does — its behaviour or logic — or its
product output (the text and surfaces it presents to users) runs through the
`plan-build-ship` plugin, always, not Claude's built-in behaviour.** The verb does
not matter — feature, fix, copy, tweak, refactor all count when they touch
behaviour or product output. A change sits outside PBS only when it is
behaviour-neutral: explanatory prose *about* the project, a test that only pins
existing behaviour, or editor/CI/tooling config. A change that mixes behavioural
or product-output work with otherwise-exempt work still counts; when unsure
whether it touches behaviour or product output, it counts. Plan, build, and ship
by invoking the plugin skill (`plan-build-ship:plan` / `:build` / `:ship`). Do not
use built-in plan mode for these changes: the plugin skill is named `plan`, so a
bare `/plan` can silently resolve to built-in plan mode, skipping the Codex review
gate, the change-field sections, and the plan-lint. If unsure which ran, confirm
before continuing; when in doubt invoke the fully-qualified `plan-build-ship:plan`.

- **Architecture document**: docs/architecture.md
- **Plans**: docs/1-plans/F_<slug>.plan.md — slug names, never version numbers
- **Changelogs**: docs/2-changelog/ · **Code reviews**: docs/3-code-review/
- **Version file**: package.json
- **Version scheme**: SemVer
- **Main branch**: main
- **Gate commands** (all four blocking):
  - build: `npm run build`
  - test: `npm test`
  - typecheck: `npm run typecheck`
  - lint: `npm run lint`
- **Gate baseline**: 0 errors, 0 warnings (main is green).
- **Tutorials**: off
- **Hard stops** — explicit approval every time (inherited by every plan's Autonomy envelope):
  - Changes to agent/benchmark behavior paths — prompts, tool selection,
    conversation payloads, terminal behavior, planning, memory, eval (per
    custom-codereview-guide; needs human review + lightweight evals).
  - Telemetry / analytics / consent changes (`src/services/telemetry.ts`, PostHog config).
  - Cross-repo contract changes, or adding a new API endpoint access here instead of `typescript-client`.
  - Editing the PR `HUMAN:` section.
  - Adding/removing npm dependencies or `overrides`; publishing the npm package.

### Review checklist additions

(None earned yet. Add project-specific invariants here as they are paid for —
the reviewer reads them alongside the shipped checklist.)
