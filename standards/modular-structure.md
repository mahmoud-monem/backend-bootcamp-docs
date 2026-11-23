# Modular Structure Rule Catalog (`MA-###`)

Applies to directory layout, module boundaries, Dependency Injection wiring, shared utilities,
persistence ownership, and application bootstrap under `src/**` (and inside each service's
`src/**` in a multi-service monorepo).

Read [`README.md`](./README.md) for severity meanings, the verify-before-report protocol,
and the false-positive rules.

- **Intra-module layering** (controller → service → repository) lives in [`clean-code.md`](./clean-code.md) (`CC-2xx`).
- **TypeScript import syntax & path aliases** live in [`typescript.md`](./typescript.md) (`TS-6xx`).
- **Cross-process service boundaries** (gRPC, RabbitMQ, `apps/` vs `libs/` in Project 3) live in [`../microservices-nestjs-guide.md`](../microservices-nestjs-guide.md) (`MS-###`).

**Bands**

| Band | Topic |
| --- | --- |
| `MA-0xx` | Top-level codebase topology and vertical slicing |
| `MA-1xx` | Inside-module layout, cohesion, and file roles |
| `MA-2xx` | Module encapsulation and public API boundaries |
| `MA-3xx` | Cross-module communication and dependency graph |
| `MA-4xx` | Module data and persistence ownership |
| `MA-5xx` | Shared kernel (`common/` / `shared/`) and infrastructure isolation |
| `MA-6xx` | Composition root, bootstrap, and lifecycle wiring |

---

## Reference Module Blueprints

### Stack A — Express + TypeScript + Mongoose (Project 1)

```text
src/
├── app.ts                       # Express app factory: middleware, mounts, error handler (no .listen())
├── server.ts                    # Composition root: env validation, DB connect, app.listen(), SIGTERM
├── config/
│   └── env.ts                   # Fail-fast Zod/Joi schema validated at boot (CC-701, TS-504)
├── common/                      # Cross-cutting primitives only — zero domain logic (MA-501)
│   ├── errors/
│   ├── middlewares/
│   └── logger/
├── modules/
│   ├── auth/
│   │   ├── index.ts             # Curated public API: authRouter, requireAuth, authFacade
│   │   ├── auth.routes.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.repository.ts
│   │   ├── auth.validation.ts
│   │   ├── auth.types.ts
│   │   └── auth.service.spec.ts
│   └── quiz/
│       └── ...
└── routes.ts                    # Top-level /api/v1 router assembly
```

### Stack B — NestJS + TypeScript + PostgreSQL Modular Monolith (Project 2 & per-service inside Project 3)

```text
src/
├── main.ts                      # Bootstraps Nest app, global pipes/filters, shutdown hooks
├── app.module.ts                # Composition root wiring ConfigModule, DB, EventEmitter, feature modules
├── config/
│   └── env.validation.ts
├── common/                      # Cross-cutting guards, filters, interceptors, pipes, base errors
├── infrastructure/              # Shared technical adapters (outbox worker, mailer, redis, storage)
└── modules/
    ├── booking/
    │   ├── index.ts             # Public exports: BookingModule, BookingFacade, public DTOs/events
    │   ├── booking.module.ts    # Declares internal providers; exports ONLY public facade/ports
    │   ├── booking.controller.ts
    │   ├── booking.facade.ts    # Narrow public contract for sibling modules
    │   ├── booking.service.ts   # Or application/use-cases/*.use-case.ts
    │   ├── dto/                 # Request/response boundary DTOs
    │   ├── events/              # Owned domain events emitted by booking (MA-303)
    │   ├── entities/            # Booking-owned ORM entities (MA-401)
    │   ├── repositories/        # Booking-owned repositories (private to booking)
    │   └── policies/            # Pure domain rules (e.g. cancellation-policy.ts + .spec.ts)
    └── availability/
        └── ...
```

### Allowed Dependency Matrix

```text
  app.module.ts / app.ts / server.ts   (Composition Root)
         │
         ▼
  modules/<A>  ──(public facade / domain events)──▶  modules/<B>
         │
         ▼
  common/ , config/ , infrastructure/  (never imports back from modules/*)
```

| From \ To | Own internals | Sibling `modules/<B>` public API | Sibling `modules/<B>` internals / repo / entity | `common/` & `config/` | `app` / `main` |
| --- | :---: | :---: | :---: | :---: | :---: |
| **`modules/<A>`** | ✅ | ✅ (acyclic) | ❌ (`MA-201`, `MA-401`) | ✅ | ❌ |
| **`common/` / `infrastructure/`** | ✅ | ❌ (`MA-502`) | ❌ (`MA-502`) | ✅ | ❌ |
| **Composition root (`app.ts`, `app.module.ts`)** | ✅ | ✅ | ⚠️ wire only | ✅ | — |

---

## MA-0xx — Top-level codebase topology & vertical slicing

### MA-001 — Layer-first (horizontal) folder layout instead of feature-first (vertical) modules
**Severity:** major · **Applies to:** `src/**`
**Detect:** Top-level folders grouped purely by technical layer (`src/controllers/`, `src/services/`, `src/repositories/`, `src/models/`, `src/routes/`) instead of vertical domain slices (`src/modules/<feature>/`).

```text
// Bad — horizontal slicing scatters every feature across 6 top-level folders
src/
├── controllers/ (auth.controller.ts, booking.controller.ts, doctor.controller.ts)
├── services/    (auth.service.ts, booking.service.ts, doctor.service.ts)
├── models/      (user.model.ts, booking.model.ts, doctor.model.ts)
└── routes/      (auth.routes.ts, booking.routes.ts, doctor.routes.ts)

// Good — vertical feature modules keep cohesive code together and make boundaries visible
src/
├── common/
├── config/
└── modules/
    ├── auth/
    ├── booking/
    └── doctor/
```

**Why:** Layer-first trees force every feature change to jump across half a dozen directories, hide cross-domain coupling, and make extracting or owning a capability painful.
**Not a violation when:** A single-purpose microservice with only one bounded context uses top-level clean-architecture folders (`domain/`, `application/`, `infrastructure/`, `interface/`) inside that one service.

---

### MA-002 — Feature module placed outside the modules root or inconsistently nested
**Severity:** minor · **Applies to:** `src/**`
**Detect:** Some domain modules placed directly under `src/` (`src/cancellation/`, `src/auth/`) while sibling modules live under `src/modules/` (`src/modules/booking/`, `src/modules/doctors/`), or arbitrary multi-level nesting with no sub-domain boundary.

```text
// Bad — half the features live under src/modules/, others sit at src/ root
src/
├── cancellation/
├── modules/
│   ├── booking/
│   └── catalog/

// Good — every domain feature lives consistently under src/modules/
src/
└── modules/
    ├── booking/
    ├── cancellation/
    └── catalog/
```

**Why:** Inconsistent module placement breaks navigation, path aliases (`@modules/*`), and automated boundary lint rules (`MA-304`).
**Not a violation when:** The project uses a single agreed top-level feature root (all features under `src/modules/` or all directly under `src/` in a small service) consistently.

---

### MA-003 — Technical or cross-cutting folders polluting `src/modules/`
**Severity:** minor · **Applies to:** `src/modules/**`
**Detect:** Non-domain folders such as `src/modules/middlewares/`, `src/modules/utils/`, `src/modules/database/`, `src/modules/types/`, or `src/modules/helpers/` sitting alongside business modules.

```text
// Bad
src/modules/
├── auth/
├── booking/
├── middlewares/      # cross-cutting, not a business module
└── utils/            # cross-cutting, not a business module

// Good
src/
├── common/           # middlewares, errors, pipes, shared helpers
├── infrastructure/   # database, redis, mailer
└── modules/          # auth, booking, catalog
```

**Why:** Mixing infrastructure plumbing into `src/modules/` blurs the distinction between domain capabilities and cross-cutting support code.
**Not a violation when:** A dedicated cross-cutting NestJS infrastructure module (such as `HealthModule` or `PrismaModule`) lives explicitly in `src/infrastructure/` or `src/health/`.

---

### MA-004 — Technical-noun module instead of a business capability
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** Modules named after technical mechanisms (`src/modules/crons/`, `src/modules/webhooks/`, `src/modules/emails/`, `src/modules/sockets/`) that contain business rules belonging to `booking`, `payment`, or `notifications`.

```text
// Bad — crons/ reaches into booking, hold, and payment tables to run business rules
src/modules/crons/
├── expire-holds.cron.ts
├── cancel-unpaid.cron.ts
└── reminder-emails.cron.ts

// Good — each capability owns its own scheduled jobs and webhooks
src/modules/booking/jobs/expire-holds.job.ts
src/modules/payment/webhooks/stripe-webhook.controller.ts
src/modules/notifications/jobs/appointment-reminder.job.ts
```

**Why:** Grouping by technical trigger ("all crons here") forces one module to depend on every domain's repositories and splits business invariants away from the module that owns them.
**Not a violation when:** A purely generic infrastructure mechanism (e.g., `src/infrastructure/outbox/` relay worker or `src/infrastructure/scheduler/` lock wrapper) contains zero domain logic.

---

## MA-1xx — Inside-module layout, cohesion, and file roles

### MA-101 — Inconsistent internal structure across sibling modules
**Severity:** minor · **Applies to:** `src/modules/**`
**Detect:** Module A uses `<feature>.{controller,service,repository,routes,validation}.ts`, Module B uses nested `controllers/`, `services/`, `dal/`, and Module C dumps everything into `index.ts`.

```text
// Bad
src/modules/auth/auth.service.ts
src/modules/quiz/services/quizService.ts
src/modules/attempt/logic.ts

// Good — predictable internal layout across all sibling modules
src/modules/auth/auth.{routes,controller,service,repository,validation}.ts
src/modules/quiz/quiz.{routes,controller,service,repository,validation}.ts
src/modules/attempt/attempt.{routes,controller,service,repository,validation}.ts
```

**Why:** Predictable internal structure lets any engineer open any module and find routes, validation, business logic, and persistence in seconds.
**Not a violation when:** A particularly rich domain module uses clean-architecture subfolders (`domain/`, `application/`, `infrastructure/`, `interface/`) while simpler CRUD modules keep flat role-suffixed files — provided the convention is documented.

---

### MA-102 — God module aggregating unrelated sub-domains
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** A catch-all module (e.g., `src/modules/core/`, `src/modules/app/`, `src/modules/admin/`, `src/modules/users/`) that houses 15+ unrelated endpoints, 4+ unrelated aggregates, or > 1,500 lines of mixed concerns.

```text
// Bad — "admin" module owns quizzes, users, reports, payouts, and audit logs
src/modules/admin/
├── admin.controller.ts   (900 lines)
└── admin.service.ts      (1,200 lines touching 6 collections)

// Good — split by domain capability; admin routes are either a sub-router or dedicated policy inside each domain
src/modules/quiz/
src/modules/user/
src/modules/reporting/
```

**Why:** God modules become merge-conflict magnets, accumulate circular dependencies (`MA-301`), and defeat vertical slicing.
**Not a violation when:** A read-only aggregation module (such as `dashboard/` or `reporting/`) composes public facades from multiple domains solely to serve dashboard read queries.

---

### MA-103 — Anemic pass-through module (over-fragmentation)
**Severity:** minor · **Applies to:** `src/modules/**`
**Detect:** Micro-modules containing a single 15-line service that only forwards calls 1:1 to another module's repository or service with zero independent state, invariants, or boundary value.

```text
// Bad — separate `slot-status/` module that only wraps a single repo call in `availability/`
// slot-status.service.ts
async isFree(slotId: string) { return this.availabilityRepo.isFree(slotId); }

// Good — keep cohesive sub-capabilities inside the owning module (`availability/`)
```

**Why:** Over-fragmentation creates boilerplate modules, redundant DI wiring, and artificial cross-module hops without increasing decoupling.
**Not a violation when:** The module owns a distinct lifecycle, external adapter, or bounded context even if its initial implementation is small (e.g., `health/`, `audit/`).

---

### MA-104 — NestJS `@Module` over-exporting internals or re-registering another module's providers
**Severity:** major · **Applies to:** `src/**/*.module.ts` in NestJS codebases
**Detect:** `@Module({ exports: [BookingRepository, TypeOrmModule, InternalHelper, BookingService] })` exporting repositories/ORM entities to the world, or Module B adding `BookingService` / `TypeOrmModule.forFeature([Booking])` to its own `providers` array instead of importing `BookingModule`.

```ts
// Bad — exports internal repository AND consumer re-instantiates sibling providers
@Module({
  imports: [TypeOrmModule.forFeature([Booking, SlotHold])],
  providers: [BookingService, BookingRepository],
  exports: [BookingService, BookingRepository, TypeOrmModule], // leaks persistence internals!
})
export class BookingModule {}

// Good — export only the narrow public facade/service; import the owning module cleanly
@Module({
  imports: [TypeOrmModule.forFeature([Booking, SlotHold])],
  providers: [BookingService, BookingFacade, BookingRepository],
  exports: [BookingFacade],
})
export class BookingModule {}
```

**Why:** Exporting internal repositories breaks encapsulation (`MA-201`), while listing another module's provider in your own `providers` array silently creates duplicate instances and broken transaction/state scope.
**Not a violation when:** A dedicated persistence module exists solely to wrap database connection configuration (e.g., `DatabaseModule` exporting configured `PrismaService` or `DataSource`).

---

## MA-2xx — Module encapsulation and public API boundaries

### MA-201 — Deep import into another module's internals
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** A file in `modules/A` importing from `../B/repositories/...`, `../B/entities/...`, `../B/models/...`, `../B/helpers/...`, or `@modules/B/internal/...` instead of `B`'s root public API (`../B` or `../B/B.facade`). Pairs with `CC-204` and `TS-607`.
Grep hint: `from ['"](\.\./|@modules/)[a-z-]+/(repositories|entities|models|schemas|internal|helpers)/`.

```ts
// Bad — payment module reaches directly into booking's internal repository and entity
import { BookingRepository } from "../booking/repositories/booking.repository";
import { BookingEntity } from "../booking/entities/booking.entity";

// Good — depend exclusively on booking's public facade or domain events
import { BookingFacade, type BookingPaymentSummary } from "../booking";
```

**Why:** Deep imports freeze another module's internal folder structure, bypass its domain invariants, and make safe refactoring inside module B impossible.
**Not a violation when:** Importing a shared type/event from an explicitly documented public subpath (e.g., `../booking/events`) when no `index.ts` facade exists yet.

---

### MA-202 — Missing or overly permissive module public API
**Severity:** minor · **Applies to:** `src/modules/*/index.ts`, `src/modules/*.facade.ts`
**Detect:** A multi-file module consumed by other modules with either no `index.ts` public entrypoint, or an `index.ts` containing `export * from "./..."` for internal repositories, ORM schemas, and helpers (pairs with `TS-606`).

```ts
// Bad — src/modules/availability/index.ts exposes everything unconditionally
export * from "./availability.entity";
export * from "./availability.repository";
export * from "./availability.service";

// Good — curated public surface only
export { AvailabilityModule } from "./availability.module";
export { AvailabilityFacade } from "./availability.facade";
export type { HoldSlotCommand, SlotHoldView } from "./availability.contracts";
```

**Why:** An explicit public surface tells every teammate (and linter) what is safe to depend on and what remains private to the module.
**Not a violation when:** A leaf module is consumed exclusively via HTTP routes and has zero in-process module consumers.

---

### MA-203 — Internal domain entity or persistence schema leaked across a module boundary
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** A module's public facade/service accepting or returning a Mongoose `Document`, TypeORM `@Entity()` class, or Prisma model type to another module instead of a plain contract DTO/view interface (pairs with `TS-304`, `CC-801`).

```ts
// Bad — returning TypeORM entity lets callers mutate relations or rely on lazy-loaded fields
export class DoctorFacade {
  async findDoctor(id: string): Promise<DoctorEntity> { ... }
}

// Good — return an immutable read-only contract view owned by the boundary
export interface DoctorPublicProfile {
  readonly id: string;
  readonly fullName: string;
  readonly consultationFeePiastres: number;
}
export class DoctorFacade {
  async findDoctor(id: string): Promise<DoctorPublicProfile | null> { ... }
}
```

**Why:** Passing ORM entities across module boundaries leaks database schema changes to every consumer and encourages cross-module mutations outside the owning module's transactions.
**Not a violation when:** The return value is strictly inside the same module (e.g., repository → service within `modules/doctor`).

---

## MA-3xx — Cross-module communication and dependency graph

### MA-301 — Circular dependency between feature modules
**Severity:** critical · **Applies to:** `src/modules/**`
**Detect:** Module A imports Module B and Module B imports Module A (directly or transitively A → B → C → A), or NestJS `forwardRef(() => ...)` used to paper over a cycle between two domain modules. Grep hint: `forwardRef\(`. Pairs with `TS-604`.

```ts
// Bad — BookingModule and PaymentModule import each other via forwardRef()
@Module({ imports: [forwardRef(() => PaymentModule)] })
export class BookingModule {}

@Module({ imports: [forwardRef(() => BookingModule)] })
export class PaymentModule {}

// Good — invert dependency with domain events or extract a mediating workflow/port
// PaymentModule emits `payment.succeeded`; BookingModule listens and confirms the booking.
```

**Why:** Circular module graphs create unpredictable startup initialization (`undefined` providers), prevent independent testing, and tightly lock modules together so neither can evolve or extract cleanly.
**Not a violation when:** Never between domain modules — break the cycle using domain events, a shared port interface, or a higher-level orchestrator.

---

### MA-302 — Synchronous cross-module fan-out for reactive side effects
**Severity:** major · **Applies to:** `src/modules/**`
**Detect:** A core module synchronously injecting 3+ sibling modules (`NotificationFacade`, `AuditFacade`, `AnalyticsFacade`, `LoyaltyFacade`) to fire side effects after a state change instead of emitting a domain event (`CC-807`).

```ts
// Bad — BookingService knows about 4 downstream modules
constructor(
  private readonly bookingRepo: BookingRepository,
  private readonly notifications: NotificationsService,
  private readonly analytics: AnalyticsService,
  private readonly prescriptions: PrescriptionsService,
) {}

// Good — emit a single domain event; downstream modules react independently
await this.bookingRepo.markCompleted(bookingId);
await this.outbox.enqueue("booking.completed", { bookingId, patientId, doctorId });
```

**Why:** Synchronous fan-out turns one core module into a hub coupled to every downstream consumer and amplifies failure blast radius.
**Not a violation when:** The synchronous call is required to uphold an immediate pre-condition or invariant before the state change can succeed (e.g., reserving a hold or quoting an authoritative price).

---

### MA-303 — Cross-module event contract unowned or defined in the consumer
**Severity:** major · **Applies to:** `src/modules/**/events/**`, event listeners
**Detect:** Event payload interfaces defined separately inside both producer and consumer, duplicated inline in `@OnEvent(...)` handlers, or owned inside the consuming module instead of the producing module (or a lightweight `events/` contract module).

```ts
// Bad — notifications/ invents its own copy of BookingConfirmedPayload
// notifications/booking-confirmed.listener.ts
@OnEvent("booking.confirmed")
handle(payload: { bookingId: string; phone: string }) { ... } // drifts if producer renames `phone`

// Good — producer module owns and exports the typed event contract
// modules/booking/events/booking-confirmed.event.ts
export const BOOKING_CONFIRMED_EVENT = "booking.confirmed" as const;
export interface BookingConfirmedEvent { readonly eventId: string; readonly bookingId: string; readonly patientId: string; }
```

**Why:** Unowned event payloads drift silently between producer and consumers with zero compile-time warning.
**Not a violation when:** Event schemas live in a dedicated contracts folder/package (`libs/contracts` in a monorepo) explicitly versioned for cross-boundary consumers.

---

### MA-304 — Unenforced module boundaries in multi-module codebases
**Severity:** minor · **Applies to:** `package.json`, `eslint.config.*`, `.dependency-cruiser.*`, CI config
**Detect:** A multi-module project (5+ feature modules) with zero static boundary or cycle check (`dependency-cruiser`, `eslint-plugin-boundaries`, `@nx/enforce-module-boundaries`, or `madge --circular`) in scripts or CI.

```jsonc
// Bad — module boundaries rely purely on human memory during PR review
"scripts": { "lint": "eslint src", "test": "jest" }

// Good — automated dependency-rule and circular-import check enforced in CI
"scripts": {
  "lint": "eslint src",
  "arch:check": "depcruise src --config .dependency-cruiser.cjs && madge --circular --extensions ts src"
}
```

**Why:** Architectural boundaries without automated verification erode under sprint deadline pressure; one hurried `import` creates a cycle nobody notices until months later.
**Not a violation when:** A small single-service project with ≤ 3 modules enforces boundaries explicitly via code review and has a tracked task for static checks.

---

## MA-4xx — Module data and persistence ownership

### MA-401 — Cross-module direct table or collection write
**Severity:** critical · **Applies to:** `src/modules/**`
**Detect:** Module A's service or repository directly inserting, updating, or deleting records in a table/collection owned by Module B (e.g., `PaymentService` executing `UPDATE slot_holds SET status = 'BOOKED'` or `BookingService` mutating `UserModel`).

```ts
// Bad — payment module updates booking and slot_hold tables directly, bypassing booking invariants
await this.dataSource.getRepository(SlotHoldEntity).update({ id: holdId }, { status: "CONFIRMED" });

// Good — payment emits/returns charge result; booking module owns writes to bookings and holds
await this.bookingFacade.confirmFromPayment(holdId, chargeResult);
```

**Why:** Two writers to one table split validation, locking (`CC-507`), and outbox guarantees (`CC-811`) across modules; one module inevitably skips a state guard or lock.
**Not a violation when:** Never for domain writes. Read-only projections fed by domain events inside a separate reporting module own their own projection table.

---

### MA-402 — Cross-module ORM relations, eager joins, or cascading deletes across bounded contexts
**Severity:** major · **Applies to:** `src/modules/**/*.entity.ts`, `src/modules/**/*.schema.ts`, `prisma/schema.prisma`
**Detect:** TypeORM `@ManyToOne` / `@OneToMany` with `eager: true` or `cascade: true` pointing directly to an entity class in a sibling feature module, or Mongoose `.populate()` crossing module ownership boundaries instead of storing the foreign ID and querying via facade or local read snapshot.

```ts
// Bad — BookingEntity tightly binds TypeORM entity graph and cascade rules to DoctorEntity
@ManyToOne(() => DoctorEntity, { eager: true, onDelete: "CASCADE" })
doctor: DoctorEntity;

// Good — reference sibling aggregate by scalar ID; compose read views via facade or snapshot
@Column("uuid")
doctorId: string;
```

**Why:** Cross-module ORM object graphs pull the entire database schema into memory, cause accidental N+1 / cascade deletions across domains, and make future module extraction into a separate service vastly harder.
**Not a violation when:** Relational foreign-key constraints (`REFERENCES`) are used purely for referential integrity at the SQL schema level while keeping ORM entity classes decoupled, OR both entities genuinely belong to the same bounded context (e.g., `Quiz` and `Question`).

---

### MA-403 — Central `src/models/` or `src/entities/` dump shared by all modules
**Severity:** major · **Applies to:** `src/models/**`, `src/entities/**`, `src/schemas/**`
**Detect:** All Mongoose schemas or TypeORM entities placed in one global `src/models/` folder instead of living inside the feature module that owns them (`src/modules/<feature>/entities/`).

```text
// Bad — central entity/model folder lets any service import and mutate any table
src/
├── entities/ (user.entity.ts, doctor.entity.ts, booking.entity.ts, payment.entity.ts)
└── modules/

// Good — persistence models are co-located inside their owning module
src/modules/
├── booking/entities/booking.entity.ts
└── payment/entities/payment.entity.ts
```

**Why:** A shared global `models/` directory signals "every table belongs to everyone", eliminating ownership and inviting cross-module reach-arounds (`MA-401`, `CC-204`).
**Not a violation when:** Using Prisma's single `schema.prisma` file — in that case, encapsulate `PrismaService` reads/writes behind each module's own `*.repository.ts` so raw `prisma.<otherModel>` calls never cross module lines (`CC-202`).

---

## MA-5xx — Shared kernel (`common/` / `shared/`) and infrastructure isolation

### MA-501 — Business domain logic placed in `common/` or `shared/`
**Severity:** major · **Applies to:** `src/common/**`, `src/shared/**`, `libs/**`
**Detect:** Domain rules (pricing, refund tiers, grading formulas, slot duration policies, role-specific business workflows) or domain entities placed under `src/common/` or `src/shared/` instead of the owning feature module.

```text
// Bad — domain cancellation fee policy hidden inside global common/
src/common/utils/calculate-refund.ts

// Good — domain policies live in their owning module; common/ holds only technical cross-cutting primitives
src/modules/booking/policies/cancellation-policy.ts
src/common/errors/app-error.ts
```

**Why:** Putting domain logic in `common/` creates a hidden coupling hub where unrelated modules implicitly share volatile business rules (see also `MS-051` for monorepos).
**Not a violation when:** Truly universal, domain-agnostic value objects or technical helpers (e.g., `Money` minor-unit math, `PaginationQuery`, `AppError`, `DateRange`).

---

### MA-502 — `common/` or `infrastructure/` importing from a feature module
**Severity:** critical · **Applies to:** `src/common/**`, `src/shared/**`, `src/infrastructure/**`, `src/config/**`
**Detect:** Any import path from `common/`, `shared/`, `infrastructure/`, or `config/` pointing upward into `src/modules/**` (e.g., `common/guards/auth.guard.ts` importing `UserRepository` directly instead of a narrow auth token-verifier port).

```ts
// Bad — common/ guard depends directly on a feature module's concrete repository
// src/common/guards/verified-user.guard.ts
import { UserRepository } from "../../modules/user/user.repository";

// Good — common/ defines a narrow port interface; composition root or AuthModule provides the adapter
// src/common/ports/session-verifier.port.ts
export const SESSION_VERIFIER = Symbol("SESSION_VERIFIER");
export interface SessionVerifier { verify(token: string): Promise<AuthenticatedUser>; }
```

**Why:** Lower technical layers depending on higher feature modules invert the dependency rule, guaranteeing circular imports (`TS-604`, `MA-301`) and making `common/` impossible to reuse or unit-test standalone.
**Not a violation when:** An infrastructure adapter inside a feature module (`modules/booking/infrastructure/...`) implements a port owned by that same module.

---

### MA-503 — `@Global()` module abuse on feature modules in NestJS
**Severity:** major · **Applies to:** `src/modules/**/*.module.ts` in NestJS codebases
**Detect:** Feature modules (`@Module(...)`) decorated with `@Global()` (`UserModule`, `BookingModule`, `DoctorModule`) so their services are implicitly injectable everywhere without declaring `imports: [...]`. Grep hint: `@Global\(\)`.

```ts
// Bad — @Global() hides real dependencies between domain modules
@Global()
@Module({ providers: [DoctorService], exports: [DoctorService] })
export class DoctorModule {}

// Good — declare explicit module dependencies in `imports: [...]`
@Module({ providers: [DoctorFacade, DoctorRepository], exports: [DoctorFacade] })
export class DoctorModule {}
```

**Why:** `@Global()` on domain modules destroys visibility of the module dependency graph, encourages sloppy cross-module coupling, and masks circular dependencies until runtime.
**Not a violation when:** Applied strictly to genuine cross-cutting infrastructure modules (`ConfigModule`, `LoggerModule`, `DatabaseModule`, `ClsModule`) registered once at the root.

---

### MA-504 — Unbounded `utils/` grab-bag directory
**Severity:** minor · **Applies to:** `src/utils/**`, `src/helpers/**`, `src/common/utils/**`
**Detect:** Catch-all files (`utils.ts`, `helpers.ts`, `common.ts`) mixing unrelated functions (crypto, JWT, date formatting, pagination, email templating, S3 uploads) with no cohesive sub-domain grouping. Pairs with `CC-110`.

```text
// Bad
src/utils/
├── helpers.ts     # 450 lines: otp, pagination, s3Upload, hashPassword, formatCairoDate
└── index.ts

// Good — cohesive capability-focused folders inside common/ or infrastructure/
src/common/pagination/parse-pagination.ts
src/common/time/cairo-time.ts
src/infrastructure/storage/s3-storage.adapter.ts
```

**Why:** Grab-bag `utils/` directories attract orphaned logic from every sprint, gain dozens of unrelated imports, and defeat tree-shaking and test isolation.
**Not a violation when:** A single small helper file exists per cohesive capability (e.g., `src/common/time/cairo-time.ts`).

---

## MA-6xx — Composition root, bootstrap, and lifecycle wiring

### MA-601 — App construction mixed with server listening (`app.ts` vs `server.ts`)
**Severity:** major · **Applies to:** `src/app.ts`, `src/server.ts`, `src/index.ts`, `src/main.ts`
**Detect:** Calling `app.listen(...)` or `mongoose.connect(...)` directly at module top level inside `app.ts` so integration tests (`supertest(app)`) cannot import the Express app or Nest test module without binding a live TCP port.

```ts
// Bad — importing `app` in a test immediately opens a port and connects to DB
// src/app.ts
export const app = express();
mongoose.connect(env.MONGO_URI).then(() => app.listen(env.PORT));

// Good — pure app factory in app.ts; side-effectful boot in server.ts
// src/app.ts
export function createApp(): Express { const app = express(); /* routes + middleware */ return app; }
// src/server.ts
await connectDatabase(env.MONGO_URI);
const server = createApp().listen(env.PORT);
```

**Why:** Separating app assembly from I/O startup lets tests spin up isolated apps in parallel without port collisions (`EADDRINUSE`) or side effects at import time.
**Not a violation when:** NestJS `main.ts` acts exclusively as the process entrypoint while tests build `AppModule` via `Test.createTestingModule({ imports: [AppModule] })`.

---

### MA-602 — Import-time side effects inside feature modules
**Severity:** major · **Applies to:** `src/modules/**`, `src/common/**`
**Detect:** Top-level statements executing when a file is imported: opening Redis/DB connections, registering global cron timers (`setInterval`, `cron.schedule`), seeding records, or instantiating network SDK clients before `env` validation runs. Pairs with `CC-208`.

```ts
// Bad — importing `mailer.ts` immediately opens an SMTP pool using raw process.env
export const transporter = nodemailer.createTransport({ host: process.env.SMTP_HOST });

// Good — construct clients inside an explicit factory or DI provider during bootstrap
export function createMailer(config: MailConfig): Mailer { ... }
```

**Why:** Import-time side effects execute in arbitrary module-resolution order, bypass fail-fast config validation, and break unit tests that merely import a type or helper from the file.
**Not a violation when:** Declaring pure immutable constants (`as const`), Zod schemas, or `Symbol` injection tokens.

---

### MA-603 — Missing graceful shutdown wiring across modules and infrastructure
**Severity:** major · **Applies to:** `src/server.ts`, `src/main.ts`, `src/infrastructure/**`
**Detect:** Production servers lacking `SIGTERM`/`SIGINT` handlers (Express) or `app.enableShutdownHooks()` + `OnModuleDestroy` (NestJS) to drain HTTP servers, close DB pools, quit Redis, and close RabbitMQ/BullMQ workers cleanly.

```ts
// Bad — SIGTERM kills the process mid-transaction and leaves DB/Redis sockets half-open
const app = await NestFactory.create(AppModule);
await app.listen(port);

// Good — enable lifecycle hooks and close external clients in OnModuleDestroy / signal handler
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();
await app.listen(port);
```

**Why:** Container orchestrators (Docker Compose, Kubernetes, Render) send `SIGTERM` on every deploy or scale-down. Without graceful shutdown, in-flight bookings abort mid-write and leaked connections stall restarts.
**Not a violation when:** A short-lived one-off CLI script or database seed command that exits immediately after completion.
