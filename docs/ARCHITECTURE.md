# RoktoSetu — Architecture & Design Patterns

This document says **how the code is organised** and **which design patterns are used where, and why**.
Every pattern here solves a concrete problem in this project. If a pattern stops paying for itself, remove it
and record that in `docs/DECISIONS.md`.

---

## 1. Big picture

```
┌──────────────────────── Browser (React + Vite) ────────────────────────┐
│ pages → hooks (TanStack Query) → api client → fetch('/api/...')         │
│ shared Zod schemas for form validation                                  │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTPS, session cookie
┌───────────────────────────────▼──────── Node process (Render) ──────────┐
│  Express                                                                │
│   middleware chain: helmet → json → session → rateLimit → auth → routes │
│   routes (controllers)  — HTTP only                                     │
│        │                                                                │
│        ▼                                                                │
│   services (application layer) — use cases, transactions, events        │
│        │                    │                    │                      │
│        ▼                    ▼                    ▼                      │
│   domain (pure)        repositories         EventBus ─▶ listeners       │
│   eligibility specs    (Prisma impl)        (audit, workflow triggers)  │
│   matching engine           │                                           │
│   state machines            ▼                                           │
│                        PostgreSQL (Neon)                                │
│                                                                         │
│   Workflow worker (setInterval 3 s) — claims Job rows, runs steps       │
└─────────────────────────────────────────────────────────────────────────┘
```

One process runs the API, serves the built React app, and runs the background worker. This keeps
deployment to a single Render service. The worker is stateless; all workflow state lives in the database.

---

## 2. Layers and dependency rule

| Layer | Folder | May depend on | Must NOT depend on |
|---|---|---|---|
| **Domain** | `server/src/domain/` | `shared/` types only | Prisma, Express, env, clock, I/O |
| **Repositories** | `server/src/repositories/` | domain types, Prisma | Express, services |
| **Services** | `server/src/services/` | domain, repository interfaces, EventBus | Express (`req`/`res`) |
| **Workflow** | `server/src/workflow/` | domain, services, repositories | Express |
| **HTTP** | `server/src/http/` (routes, middleware) | services | Prisma directly, domain internals |
| **Composition root** | `server/src/container.ts` | everything | — (it wires things up) |

Rule of thumb: arrows point inward to the domain. The domain knows nothing about the database or the web.

---

## 3. Folder structure

```
roktosetu/
├─ CLAUDE.md
├─ package.json                  # npm workspaces: client, server, shared
├─ docker-compose.yml            # Postgres 16 for dev/test
├─ render.yaml                   # Render blueprint
├─ shared/
│  └─ src/{schemas.ts, enums.ts, types.ts, index.ts}   # Zod schemas + enums used by client and server
├─ server/
│  ├─ prisma/{schema.prisma, migrations/, seed.ts, data/{districts.json, hospitals.json}}
│  ├─ src/
│  │  ├─ domain/
│  │  │  ├─ blood/compatibility.ts
│  │  │  ├─ eligibility/{EligibilitySpec.ts, specs/*.ts, evaluateEligibility.ts}
│  │  │  ├─ state/{StateMachine.ts, requestMachine.ts, pledgeMachine.ts}
│  │  │  ├─ matching/                                     # ← A02
│  │  │  │  ├─ MatchingEngine.ts
│  │  │  │  ├─ scoring/{ScoringStrategy.ts, GroupScore.ts, ProximityScore.ts,
│  │  │  │  │           ReliabilityScore.ts, FreshnessScore.ts, CompositeScorer.ts}
│  │  │  │  ├─ relaxation/{Relaxation.ts, ExpandRadius.ts, AllowCompatible.ts, ExtendDeadline.ts}
│  │  │  │  ├─ wavePlanner.ts
│  │  │  │  └─ geo.ts                                       # haversine
│  │  │  └─ errors.ts                                      # DomainError, IllegalTransitionError, ...
│  │  ├─ repositories/{interfaces.ts, prisma/*.ts}
│  │  ├─ services/{AuthService, DonorService, RequestService, PledgeService,
│  │  │            ModerationService, AnalyticsService, AuditService, RulesService}.ts
│  │  ├─ events/{EventBus.ts, events.ts, listeners/*.ts}
│  │  ├─ notifications/{NotificationChannel.ts, InAppChannel.ts, FaultInjectingChannel.ts}
│  │  ├─ workflow/                                         # ← A07
│  │  │  ├─ WorkflowEngine.ts
│  │  │  ├─ StepRunner.ts
│  │  │  ├─ StepRegistry.ts
│  │  │  ├─ steps/{MatchStep, NotifyWaveStep, WaitWaveStep, EvaluateStep,
│  │  │  │         CloseStep, CompensateStep}.ts
│  │  │  ├─ JobQueue.ts                                    # SKIP LOCKED claim, stale lock recovery
│  │  │  └─ Worker.ts
│  │  ├─ http/{app.ts, routes/*.ts, middleware/{session, requireAuth, requireRole,
│  │  │        requireOwnership, rateLimit, validate, errorHandler}.ts}
│  │  ├─ db/{prisma.ts, unitOfWork.ts}
│  │  ├─ config/env.ts                                     # Zod-validated env
│  │  ├─ container.ts                                      # composition root
│  │  └─ index.ts
│  └─ tests/{unit/, integration/, helpers/}
├─ client/
│  └─ src/{main.tsx, router.tsx, api/, hooks/, pages/, components/{ui/, request/, donor/, workflow/}, lib/}
├─ docs/{ARCHITECTURE.md, DECISIONS.md, architecture.png}
└─ private/                    # git-ignored local notes (never pushed)
```

---

## 4. Pattern map

| # | Pattern | Where | Problem it solves | What to say to a judge |
|---|---|---|---|---|
| 1 | **Layered architecture + Dependency Inversion** | whole server | Keeps rules independent from DB/web so they can be tested and explained | "The domain is pure; services depend on repository *interfaces*, Prisma is just one implementation." |
| 2 | **Repository** | `repositories/` | Hides Prisma queries behind interfaces; services stay readable; in-memory fakes for unit tests | "Services never write SQL; swapping the DB layer wouldn't touch business rules." |
| 3 | **Unit of Work** | `db/unitOfWork.ts` | Several repository writes must commit or roll back together (pledge + counter, close request + donor cooldowns) | "`uow.run(tx => ...)` gives every repository the same transaction." |
| 4 | **Specification** | `domain/eligibility/specs/` | Each eligibility rule (age, weight, cooldown, availability, compatibility, distance, self, committed, declined) is a small object; they are combined and **every** failure is collected for explanations | "Adding a rule is one new class; the engine reports all reasons a donor was excluded, not just the first." |
| 5 | **Strategy + Composite** | `domain/matching/scoring/` | Ranking is a weighted sum of independent factors; weights come from config | "Each factor is a strategy; `CompositeScorer` combines them with weights an admin can tune." |
| 6 | **Strategy (relaxers)** | `domain/matching/relaxation/` | For infeasible requests, try each relaxation and rank the smallest change that works | "We literally re-run the pure engine per relaxation, so suggestions are guaranteed to be correct." |
| 7 | **Table-driven State Machine** | `domain/state/` | Request and pledge lifecycles with guarded transitions; illegal moves raise `IllegalTransitionError` | "We chose a transition table over the GoF State class-per-state because the states have little behaviour and the table is easier to audit." |
| 8 | **Command** | `workflow/steps/*` | Each workflow step is an object with `execute(ctx)` and optional `compensate(ctx)` | "Steps are commands, so the engine can run, retry, resume or undo them uniformly." |
| 9 | **Template Method** | `workflow/StepRunner.ts` | Every step needs the same wrapper: mark RUNNING → execute → save output → SUCCEEDED, or record error → schedule retry with backoff → NEEDS_ATTENTION | "Retry and persistence logic is written once; steps only contain their own logic." |
| 10 | **Registry / Factory** | `workflow/StepRegistry.ts` | Map a step name stored in the DB back to the step object when resuming | "Resume reads the failed step's name from the DB and asks the registry for it." |
| 11 | **Observer (domain events)** | `events/EventBus.ts`, `listeners/` | Decouple side effects: `PledgeAccepted` → enqueue EVALUATE; `RequestCancelled` → enqueue COMPENSATE; any sensitive action → AuditLog | "Services publish events; the workflow and audit log subscribe. Services don't know who listens." |
| 12 | **Adapter** | `notifications/NotificationChannel.ts` | One interface for in-app notifications now and SMS/email later | "SMS would be a new adapter; nothing else changes." |
| 13 | **Decorator** | `notifications/FaultInjectingChannel.ts` | Wraps any channel to fail after N sends for the A07 demo, without touching real code | "Fault injection is a decorator turned on by the admin, so production code paths are unchanged." |
| 14 | **Chain of Responsibility** | Express middleware | session → requireAuth → requireRole → requireOwnership → validate → handler | "Each middleware either passes the request on or stops it with 401/403/400." |
| 15 | **Composition Root (manual DI)** | `container.ts` | Build all objects in one place; tests build a container with fakes or a test DB | "No DI framework — just constructors, wired in one file." |

Patterns deliberately **not** used: Singleton (except the Prisma client instance), Abstract Factory, Visitor,
GoF State classes. Each would add ceremony without solving a problem here.

---

## 5. Key designs in detail

### 5.1 Eligibility (Specification)

```ts
export interface EligibilitySpec {
  readonly code: ExclusionReason;           // e.g. 'COOLDOWN'
  check(donor: DonorSnapshot, ctx: MatchContext): SpecResult;
}
export type SpecResult = { ok: true } | { ok: false; reason: ExclusionReason; detail: string };

export function evaluateEligibility(donor, ctx, specs: EligibilitySpec[]) {
  const failures = specs.map(s => s.check(donor, ctx)).filter(r => !r.ok);
  return { eligible: failures.length === 0, failures };
}
```

`MatchContext` contains the request, rules config and `now`. The same specs power the donor's own
"Am I eligible?" card (subset without request-specific specs).

### 5.2 Matching engine (A02)

```ts
class MatchingEngine {
  constructor(private specs: EligibilitySpec[], private scorer: CompositeScorer,
              private relaxers: Relaxation[]) {}
  match(input: MatchInput): MatchResult        // filter → score → sort (deterministic tie-break) → feasibility → conflicts
  suggest(input: MatchInput): Suggestion[]     // for each relaxer (and pairs): re-run match(), keep feasible ones, sort by cost
}
```

Pure: `MatchInput = { request, donors: DonorSnapshot[], hospital, config, now }`. The service loads donors
from the repository (pre-filtered by compatible groups and division to keep it small), then calls the engine.

### 5.3 Workflow engine (A07)

```
Event / Job ──▶ WorkflowEngine.advance(runId)
                   │ reads WorkflowRun + steps
                   │ decides next step (conditional: escalate? matched? unfulfilled?)
                   ▼
               StepRegistry.get(name) ──▶ StepRunner.run(step, ctx)   [Template Method]
                                              ├─ success → save output, SUCCEEDED, advance
                                              ├─ error, attempts < 3 → FAILED, enqueue retry Job (backoff)
                                              └─ error, attempts = 3 → run NEEDS_ATTENTION, AuditLog, alert moderators
Moderator "Resume" ──▶ WorkflowEngine.resume(runId) → reruns first non-SUCCEEDED step only
Cancel ──▶ CompensateStep (release pledges, notify donors, cancel pending jobs)
```

Idempotency is guaranteed by data, not by hope:
- `Notification @@unique([requestId, userId, wave, type])` → `createMany({ skipDuplicates: true })`.
- `Job.idempotencyKey @unique` (e.g. `wait:{runId}:{wave}`, `evaluate:{runId}:{wave}`).
- Jobs claimed with `FOR UPDATE SKIP LOCKED`; stale `RUNNING` jobs (> 2 min) re-queued.

### 5.4 Atomic pledge (M06, not an advanced challenge)

Raw SQL inside the Unit of Work:
```sql
UPDATE "BloodRequest" SET "unitsPledged" = "unitsPledged" + 1, "updatedAt" = now()
WHERE id = $1 AND status IN ('OPEN','PARTIALLY_PLEDGED') AND "unitsPledged" < "unitsNeeded"
RETURNING "unitsPledged", "unitsNeeded";
```
Then update the pledge with `WHERE status = 'NOTIFIED'`. If either affects 0 rows → throw `ConflictError` → 409.

### 5.5 Error model

`DomainError` subclasses: `ValidationError(400)`, `UnauthenticatedError(401)`, `ForbiddenError(403)`,
`NotFoundError(404)`, `ConflictError(409)`, `IllegalTransitionError(409)`, `RateLimitedError(429)`.
The error handler maps them to `{ error: { code, message, fields? } }`; unknown errors → 500 with a generic
message and a request id that is also logged server-side.

---

## 6. Frontend structure

- **Pages** are thin; data comes from hooks (`useRequest(id)`, `useAcceptPledge()`), built on TanStack Query.
- **API client** (`client/src/api/client.ts`) is the only place that calls `fetch`; it parses the error format
  and throws a typed `ApiError` so forms can show field errors.
- **Forms** use `react-hook-form` + `zodResolver` with the shared schemas.
- **Components**: small accessible UI primitives in `components/ui/` (Button, Field, Select, Badge, Toast,
  Dialog, Spinner). Status is always text + icon, never colour alone.
- **Error boundary** per route with a "Try again" button; empty and loading states for every list.
