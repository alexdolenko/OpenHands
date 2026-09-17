# Architecture Documentation Rules

[architecture.md](architecture.md) is the record of the Agent Canvas architecture.
After every task — a feature, a refactor, a bug fix — decide whether it now needs
an update.

## When to Update

Update whenever a change touches any of:

- the project's structure (directories added, files moved)
- the technology stack (a new dependency, a version bump)
- **System boundaries** — what Canvas is / is not responsible for, or a repo-ownership shift
- **Runtime services** — Agent Server, ingress, automation, or Cloud integration
- **Frontend modules** — routes, services in `src/api/`, state stores, feature modules
- **Runtime modes** — dev/static/mock/partial-stack launch behavior
- **Packaging and distribution** — standalone app vs library entrypoints, Electron, npm binary
- **Quality gates** — build/test/typecheck/lint commands or CI guards
- **Security posture** — auth, secrets handling, telemetry/consent
- how data flows, or how the frontend talks to the Agent Server / Cloud

## How to Update by Change Type

### Major Feature / Refactor

Review: System boundaries, Frontend modules, Runtime services, Data flow.

### Minor Feature / Enhancement

Update: the specific section(s) touched (usually Frontend modules or Runtime modes).

### Bug Fix

Normally nothing to update — unless it exposes or repairs an architectural flaw.

### Dependency Changes

Update: Technology Stack (in `architecture.md`), plus any section it touches.

## Guidelines

- Stay precise and factual — describe the codebase as it actually is.
- Respect repo ownership (AGENTS.md): endpoints → `software-agent-sdk`, client access → `typescript-client`, skills/automations → `extensions`/`automation`. Do not document backend behavior as if it lived here.
- Stay concise — enough to understand, without implementation minutiae.
- Redraw diagrams when data flow shifts; point at real file paths.
