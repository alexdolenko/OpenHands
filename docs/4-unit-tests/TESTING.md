# Testing Guidelines

## Test Framework

- **Unit / component**: Vitest (jsdom + node environments) with `@testing-library/react` and MSW for HTTP/WebSocket mocking.
- **E2E**: Playwright — live, mock-LLM, and mock-LLM-docker configs (`playwright*.config.ts`).
- **Mutation**: Stryker (`stryker.config.mjs`) over first-party `src/`.

## Running Tests

```bash
npm test                 # make-i18n then vitest run  (blocking gate)
npm run test:mutation    # full Stryker run (expensive)
npm run test:mutation:diff -- origin/main   # mutate only changed production files
# Targeted example:
npm run test -- __tests__/api/agent-server-config.test.ts
```

E2E frameworks and debugging: see `AGENTS.md` (Live/Mock-LLM E2E sections) and `docs/TESTING_MATRIX.md`.

## Test Organization

- Unit/component tests live under `__tests__/`, mirroring `src/` paths.
- E2E specs under `tests/e2e/`.
- CI guard tests (e.g. `src/api/no-direct-agent-server-calls.test.ts`) enforce architecture rules.

## Writing Tests (project rules, from AGENTS.md `<TESTING_RULES>`)

- AAA structure (Arrange, Act, Assert), one clear focus per test.
- Do **not** mock the hook — mock the underlying service the hook depends on.
- Extend existing test files; create new ones only when none fit.
- No duplicated cases; do not assert the same condition twice.
- Avoid brittle visual-presentation assertions; functional CSS contracts (style scoping, selector transforms) may be tested directly.
- Keep case count minimal while fully covering the change.
- Test fixtures (`__tests__/`, `tests/e2e/`) may use inline string literals — exempt from the no-magic-strings rule.

## Coverage Requirements

Not defined as a hard threshold. Mutation scores are report-only (`reports/mutation.html`) until a baseline is established; track hard-to-cover paths in `COVERAGE-DEBT.md`.
