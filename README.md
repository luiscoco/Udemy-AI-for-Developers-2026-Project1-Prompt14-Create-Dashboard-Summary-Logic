# Create Dashboard Summary Logic

This README documents, step by step, what was done to fulfill the latest prompt:

> Create `apps/backend/src/domain/dashboard.ts` with `buildDashboardSummary(workOrders)`. Return `totalOpen`, `criticalOpen`, `unassigned`, and `byState`. Write unit tests with a small inline fixture set. Do not import Fastify or the repository into this module.

## 1. Understood the existing domain conventions

Before writing any code, the codebase was explored to match its existing style:

- [`apps/backend/src/domain/workOrderLifecycle.ts`](apps/backend/src/domain/workOrderLifecycle.ts) and its test file were read to see how domain logic in this project is written: plain functions, no framework imports, typed against the shared contract package.
- [`packages/contract/src/index.ts`](packages/contract/src/index.ts) and [`packages/contract/src/types.gen.ts`](packages/contract/src/types.gen.ts) were read to find the generated types this module needed:
  - `WorkOrder` (with `state`, `priority`, `technicianId`)
  - `WorkOrderState` (the enum: `reported`, `triaged`, `scheduled`, `in_progress`, `completed`, `cancelled`)
  - `DashboardSummary` (the exact return shape already defined in the OpenAPI contract: `totalOpen`, `criticalOpen`, `unassigned`, `byState`)
  - `STATES`, the runtime array kept in sync with `WorkOrderState` via a compile-time check

Reusing these existing types instead of redefining new ones keeps the domain layer consistent with the rest of the backend and the OpenAPI contract.

## 2. Implemented `buildDashboardSummary`

Created [`apps/backend/src/domain/dashboard.ts`](apps/backend/src/domain/dashboard.ts):

- **`totalOpen`** — work orders whose `state` is not `completed` or `cancelled`.
- **`criticalOpen`** — open work orders (per the definition above) with `priority === "critical"`.
- **`unassigned`** — open work orders with `technicianId === null`.
- **`byState`** — a count for *every* `WorkOrderState`, seeded to `0` from `STATES` first so states with zero matches still appear in the result (not just the states present in the input).

The module only imports types/constants from `@equipment-hub/contract`. It does **not** import Fastify or the repository, keeping it a pure, framework-free domain function that's easy to unit test and easy to reuse from any route handler later.

## 3. Wrote unit tests

Created [`apps/backend/src/domain/dashboard.test.ts`](apps/backend/src/domain/dashboard.test.ts), following the same `vitest` style as `workOrderLifecycle.test.ts`:

- A small `makeWorkOrder(overrides)` helper builds a fully-valid `WorkOrder` with sensible defaults, so each fixture only needs to specify the fields relevant to the test.
- An inline fixture array of 6 work orders covers one of each state, with a mix of critical/non-critical priorities and assigned/unassigned technicians.
- Test cases check `totalOpen`, `criticalOpen`, `unassigned`, and `byState` individually, plus:
  - an empty-list case (all counts zeroed, `byState` still lists every state at `0`)
  - a case proving that a *completed* work order (even if critical and unassigned) is correctly excluded from all three "open" counts

## 4. Verified the change

- Ran `npm install` at the repo root (the workspace `node_modules` weren't present yet), which links the local `@equipment-hub/contract` package.
- Ran `npx vitest run` inside `apps/backend` — all test files pass, including the 6 new dashboard tests, alongside the existing `reference.test.ts` and `workOrderLifecycle.test.ts` suites.
- Ran `npx tsc --noEmit` and confirmed `dashboard.ts` type-checks cleanly (fixed a strict-indexing (`noUncheckedIndexedAccess`) warning by using `byState[state] ?? 0` when incrementing counts). Unrelated pre-existing errors in `repository.ts` (Node type/module resolution) were left untouched since they're outside the scope of this prompt.

## Running the project (Windows terminal)

At this stage of the course, the project doesn't have a runnable server or frontend yet — `apps/backend` only contains domain logic and its unit tests (no Fastify entrypoint), and `apps/frontend` has no source files yet. The only thing to "run" right now is the test suite.

From the repository root, in PowerShell or Command Prompt:

```powershell
npm install
npm test
```

`npm install` installs and links the workspace packages (only needs to be run once, or after pulling changes that touch dependencies). `npm test` runs the backend's `vitest` suite, including [`dashboard.test.ts`](apps/backend/src/domain/dashboard.test.ts).

To run just the backend tests directly:

```powershell
cd apps\backend
npx vitest run
```

A `npm run dev` command to actually start the app will become meaningful once a Fastify server (backend) and a Vite dev server (frontend) are scaffolded in a later prompt — right now that script is just a placeholder in the root `package.json`.

## Key takeaway for students

This workflow — **read existing conventions → reuse shared types → write the pure function → write tests against inline fixtures → run tests and the type checker** — is a repeatable pattern for adding domain logic to this codebase. Keeping domain functions free of framework (Fastify) and infrastructure (repository) imports is what makes them trivial to unit test without spinning up a server or a database.
