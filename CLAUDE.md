# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository actually is

This is not a normal feature-development repo. It's an MBA course assignment: the deliverable is a **package of design documents** (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-*.md`, `docs/TRACKER.md`) for a hypothetical **Order Webhooks Notification System** feature that a team decided on in a meeting but never implemented.

- `TRANSCRICAO.md` — literal transcript (in Portuguese) of the ~55-minute technical meeting where the feature was decided. This is the primary source of truth for requirements, decisions, alternatives discussed/discarded, and open questions.
- `src/`, `prisma/`, `tests/` — the **existing, working** Order Management System (OMS). It has no webhooks, events, or queues today — that gap is intentional and is exactly what the documents must design for.
- `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/*.md` — currently placeholder stubs to be filled in.
- The full assignment brief (in Portuguese, with grading checklist) currently lives in `README.md`. It is expected to eventually be replaced by process documentation, so don't assume `README.md`'s current content is permanent product documentation.

**Hard constraint: do not modify `src/`, `prisma/`, `tests/`, or other app config.** The task is purely documentational — the code is read-only context/reference. Every claim in the docs must be traceable to either `TRANSCRICAO.md` (with a `[hh:mm] Nome` timestamp) or a real file path in the code; do not invent requirements, decisions, or file references.

When asked to work on this repo, the real task is almost always: read `TRANSCRICAO.md` + relevant `src/`/`prisma/` files, then draft/refine one of the design docs, keeping strict traceability back to those two sources (this is what `docs/TRACKER.md` exists to enforce).

## Commands

```bash
npm run dev          # tsx watch, loads .env, starts src/server.ts
npm run build        # tsc -p tsconfig.build.json
npm start            # run compiled dist/server.js

npm run db:migrate   # prisma migrate dev
npm run db:reset     # prisma migrate reset --force
npm run db:seed      # tsx prisma/seed.ts

npm test             # vitest run (all tests)
npm run test:watch   # vitest watch mode
npx vitest run tests/orders.test.ts   # single test file
npx vitest run -t "some test name"    # single test by name

npm run lint         # eslint . --ext .ts
npm run format       # prettier --write .
```

Requires MySQL (`docker-compose.yml` spins up a local instance) and a `.env` populated from `.env.example` (`DATABASE_URL`, `SHADOW_DATABASE_URL`, `JWT_SECRET`, etc.). Tests run against a real database (`tests/setup.ts` connects Prisma and truncates all tables in `beforeEach`); vitest is configured with `singleFork: true` / `fileParallelism: false` because tests share that one database.

## Architecture of the existing OMS (reference material for the docs)

Layered, module-per-domain structure under `src/modules/{auth,users,customers,products,orders}/`, each following `*.routes.ts → *.controller.ts → *.service.ts → *.repository.ts` (+ `*.schemas.ts` for Zod validation). `src/app.ts` wires everything up manually (no DI container): `buildControllers()` instantiates repository → service → controller per module, `buildApiRouter()` (`src/routes/index.ts`) mounts each module's router under `/api/v1/*`.

Key pieces likely to be referenced/extended by the webhooks design docs:

- **Order state machine** — `src/modules/orders/order.status.ts` defines the allowed `OrderStatus` transitions (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, with `CANCELLED` branches) plus `shouldDebitStock` / `shouldReplenishStock` helpers.
- **`OrderService.changeStatus`** (`src/modules/orders/order.service.ts`) — the transactional core: validates the transition, debits/replenishes `Product.stockQuantity`, updates `Order.status`, and inserts an audit row into `OrderStatusHistory`, all inside one `prisma.$transaction`. This is the natural hook point for outbox-event creation (must happen in the *same* transaction to stay consistent, per the meeting's outbox decision).
- **Error model** — `src/shared/errors/app-error.ts` + `http-errors.ts` define an `AppError` hierarchy (`BadRequestError`, `ValidationError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`, etc.), each carrying an HTTP status, a string `errorCode`, and optional `details`. `src/middlewares/error.middleware.ts` is the single place that turns any thrown error (`AppError`, `ZodError`, known Prisma errors, or unknown) into a JSON `{ error: { code, message, details? } }` response. New `WEBHOOK_*` error codes should follow this same pattern.
- **Auth** — `src/middlewares/auth.middleware.ts` exposes `authenticate` (verifies JWT, populates `req.user`) and `requireRole(...roles)` (`ADMIN` | `OPERATOR`), used per-route in each module's `*.routes.ts`.
- **Logging** — `src/shared/logger` wraps Pino; `src/middlewares/request-logger.middleware.ts` attaches a per-request id/logger via `pino-http`.
- **Data model** — `prisma/schema.prisma`: `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`. No outbox/event/webhook tables exist yet — those would be new models proposed by the docs.
- **Config** — `src/config/env.ts` validates process env via Zod at startup and exits the process on invalid config; `src/config/database.ts` exports the shared `PrismaClient`.

## Document package conventions (for whoever is writing the docs)

- Each document operates at a different altitude — don't duplicate content across them. PRD = product/business "why & what"; RFC = concise (2–4 pages) architecture proposal with alternatives and open questions; ADRs = one closed decision each; FDD = deep implementation detail (endpoints, error matrix, integration points); TRACKER = cross-reference of every item back to `TRANSCRICAO` (`[hh:mm] Nome`) or `CODIGO` (file path).
- ADRs live in `docs/adrs/ADR-NNN-kebab-case-title.md`, MADR-style (Status/Contexto/Decisão/Alternativas Consideradas/Consequências).
- FDD error codes must use the `WEBHOOK_*` prefix, consistent with the existing `AppError` `errorCode` convention.
- The six core decisions from the meeting that the ADR set must cover: outbox pattern on MySQL, retry policy with backoff + DLQ, HMAC-SHA256 auth with per-endpoint secret, at-least-once delivery via `X-Event-Id`, separate polling worker process, reuse of existing project patterns.
