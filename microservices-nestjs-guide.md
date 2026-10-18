# Microservices with NestJS — Engineer's Handbook

> **Audience:** backend engineers on Project 3, ENR (Egyptian National Railways ticket booking)
> **Read time:** ~2.5 hours end to end · Parts 6–8 are the core · Part 15 is what your reviewer uses
> **Companions:** [`standards/README.md`](./standards/README.md) — the `CC`/`TS`/`JT`/`RS` catalogs,
> which still apply inside every service · [`git-github-guide.md`](./git-github-guide.md) — the
> workflow around the code
> **Not covered:** Kubernetes, service meshes, CI/CD pipelines. Everything here runs on Docker Compose.

---

## What this document is for

In DOCURA you built a modular monolith: one process, one database, modules talking through
function calls and an outbox-backed event bus. ENR takes the same ideas and puts a network between
every module. That network is where most of the new bugs come from: timeouts, duplicate messages,
half-finished bookings, stale caches, and events arriving in the wrong order.

This handbook gives you the reasoning first and the NestJS code second, so you can tell when a
pattern applies and when it is ceremony. Every chapter ends with `MS-###` rules your mentor will cite
in review, written in the same format as the [standards catalogs](./standards/README.md).

**Contents**

| Part | Topic | Rules |
| --- | --- | --- |
| [1](#part-1--when-microservices-are-worth-their-cost) | When microservices are worth their cost — and why ENR uses them | `MS-00x` |
| [2](#part-2--finding-boundaries) | Finding boundaries: bounded contexts, data ownership, the ENR service map | `MS-10x` |
| [3](#part-3--nestjs-building-blocks) | NestJS building blocks: monorepo, hybrid apps, clients, patterns | `MS-05x` |
| [4](#part-4--the-api-gateway) | The API gateway: auth at the edge, routing, identity downstream | `MS-20x` |
| [5](#part-5--synchronous-calls-with-grpc) | Synchronous calls with gRPC: contracts, deadlines, errors, retries | `MS-25x` |
| [6](#part-6--asynchronous-events-with-rabbitmq) | Asynchronous events with RabbitMQ: topology, acks, the envelope | `MS-30x` |
| [7](#part-7--reliability-outbox-inbox-and-poison-messages) | Reliability: outbox, inbox, ordering, poison messages | `MS-35x` |
| [8](#part-8--sagas) | Sagas: the ENR booking saga, compensation, timeouts | `MS-40x` |
| [9](#part-9--data-across-services) | Data across services: ownership, replication, read models, joins | `MS-15x` |
| [10](#part-10--caching-with-redis) | Caching with Redis: patterns, invalidation, stampedes, degradation | `MS-50x` |
| [11](#part-11--observability) | Observability: correlation IDs, logs, traces, health | `MS-60x` |
| [12](#part-12--resilience) | Resilience: timeouts, retries, circuit breakers, shutdown | `MS-65x` |
| [13](#part-13--testing-microservices) | Testing: unit, Testcontainers, contract tests, a few end-to-end flows | `MS-70x` |
| [14](#part-14--the-local-stack) | The local stack: Compose, databases, config, seeding | `MS-80x` |
| [15](#part-15--review-checklist-and-rule-catalogue) | Review checklist and the full rule catalogue | all |

## How the `MS` rules work

The rules use the severity ladder from [`standards/README.md`](./standards/README.md#1-severity-ladder):
`critical` blocks the merge, `major` blocks unless an accepted-debt note is written, `minor` is
batched. Escalate one level when the violation sits on an authentication, payment, or refund path.

Bands follow the contents table: `MS-1xx` boundaries and data, `MS-2xx` synchronous calls, `MS-3xx`
messaging and reliability, and so on.

A finding is reported exactly like the other catalogs:

```text
apps/booking/src/application/confirm-booking.use-case.ts:58 — MS-351 (critical): booking.confirmed is
emitted with ClientProxy after the transaction commits, so a crash in between loses the event.
Fix: insert the event into the outbox inside the same transaction; let the relay publish it.
```

## Versions this handbook was checked against

| Package | Version | Note |
| --- | --- | --- |
| `@nestjs/core`, `@nestjs/common`, `@nestjs/microservices` | 11.2.x | NestJS 12 shipped in August 2026 (ESM-only, new gRPC exception classes). **Stay on 11 for ENR**; every API below is v11's. |
| `@nestjs/terminus` · `@nestjs/config` · `@nestjs/schedule` | 11.1.x · 4.0.x · 6.1.x | The versions that pair with Nest 11 |
| `@grpc/grpc-js` · `@grpc/proto-loader` | 1.14.x · 0.8.x | |
| `amqplib` · `amqp-connection-manager` | latest | Both are required by Nest's RabbitMQ transport |
| `ioredis` | 5.x or 6.x | 6.x defaults to RESP3; the commands used here are unchanged |
| `stripe` (Node SDK) | 23.x | Requires Node 20+ |
| `testcontainers` modules | 12.x | Requires Node 22.22+ and an explicit image in every constructor |
| Node.js | 22 LTS or 24 | The floor set by Testcontainers 12 and `nestjs-pino` |

---

# Part 1 — When microservices are worth their cost

## 1.1 What you buy and what you pay

Microservices are a trade. You buy independence and pay for it in network failures and lost
transactions. The bill arrives in every feature, not once.

| You buy | You pay |
| --- | --- |
| **Independent deployment.** Payment ships a webhook fix without redeploying search. | **Partial failure.** Every call can be slow, time out, or fail halfway. "Did it happen?" becomes a real question. |
| **Independent scaling.** Six catalog instances during Eid, one notification instance. | **No cross-service transaction.** Consistency becomes eventual; you write compensations by hand. |
| **Fault isolation.** Notification crashing does not stop people buying tickets. | **Distributed debugging.** A stack trace becomes a trace across five processes and a broker. |
| **Ownership.** The owner of a service decides its schema and release cadence. | **Contract discipline.** A rename across a boundary is a migration, not an IDE refactor. |
| **Smaller codebases.** One service fits in one head. | **Operational weight.** N databases, N configs, N health checks, N sets of credentials. |

## 1.2 The same feature in both architectures

| Operation | Modular monolith (DOCURA) | Microservices (ENR) |
| --- | --- | --- |
| Call another module | Function call; cannot time out | gRPC call; needs a deadline, an error mapping and a retry policy |
| Keep two writes consistent | One `BEGIN … COMMIT` | Saga + outbox + idempotent consumers |
| Show station names on a ticket | `JOIN stations` | A local copy kept fresh by events, or a batch RPC |
| Rename a field | Refactor, compile, done | Add the new field, migrate both sides, remove the old one later |
| "Why did Ahmed get two emails?" | One log file | Correlation ID across four services and a broker |
| Run it locally | One process, one database | Compose with seven apps and six infrastructure containers |

Read the right-hand column as a list of things that will go wrong in your sprint demo if you skip
the corresponding part of this handbook.

## 1.3 When it is the right call

Microservices pay off when most of these are true:

- **Several teams must ship independently.** Your architecture ends up mirroring how your teams
  communicate (Conway's law). One team rarely needs seven deployables.
- **Parts of the system have very different load or risk profiles.** Seat contention at 8 a.m. on
  the first day of Eid sales looks nothing like profile edits.
- **The boundaries are understood.** You have already built the system once, or the domain is
  stable. Drawing service lines through a domain you do not understand yet is expensive to undo.
- **You can automate the operational weight**: deployment, configuration, monitoring.

When those are not true, a **modular monolith** — what you built in DOCURA — is usually the better
architecture. It keeps the module boundaries and loses the network. Many successful systems never
leave that shape, and the ones that do extract one service at a time, when a boundary has proven itself.

## 1.4 Why ENR uses microservices anyway

Be honest about this: ENR at bootcamp scale would run perfectly well as a modular monolith.
**Microservices are a training choice here.** The project exists so you meet partial failure,
duplicate delivery, eventual consistency, sagas and contract evolution in a place where a mistake
costs a grade rather than a customer's money.

There are real ENR pressures that *could* justify the split in production, and they are worth
keeping in mind because they explain where the lines are drawn:

- **Inventory** takes the burst of seat contention when a popular trip opens; it must stay correct
  under concurrency and can scale on its own.
- **Payment** isolates Stripe keys and money movement from everything else.
- **Notification** absorbs bursts (a cancelled trip notifies hundreds of passengers) without
  slowing bookings down.
- **Catalog** search is read-heavy and cacheable; it scales very differently from booking.

You are not starting from zero. Most of DOCURA maps across:

| You built in DOCURA | You build in ENR |
| --- | --- |
| A `modules/booking` folder | An `apps/booking` deployable with its own database |
| Outbox table + `SKIP LOCKED` worker | The same, in `libs/messaging`, one outbox per producing service |
| In-process events or BullMQ | A RabbitMQ topic exchange, one queue per consuming service |
| `availabilityService.holdSlot()` | `inventory.HoldSeats` over gRPC, with a deadline |
| One transaction across modules | A saga with compensations (Part 8) |
| `@nestjs/config` schema, fail fast | The same, per service (Part 14) |

## 1.5 The network is not your friend

The classic *fallacies of distributed computing* — the network is reliable, latency is zero, topology
does not change, there is one administrator — all hold inside one process and all fail in ENR.
`HoldSeats` succeeds in inventory but the response is lost; booking calls catalog 40 times for one
list; a service restarts mid-saga and RabbitMQ redelivers its messages; Stripe, Mailpit and RabbitMQ
each fail on their own schedule. Everything from Part 5 onwards is a defence against one of these.

## Rules — architecture decision

### MS-001 — Service boundary without a written decision
**Severity:** major · **Applies to:** `docs/adr/`, the service map
**Detect:** A service, or a split or merge of services, with no architecture decision record (ADR)
stating the capability it owns, the data it is the single writer of, why it is a separate
deployable, and its owner.
**Why:** A boundary nobody wrote down gets eroded one convenient shortcut at a time — a direct
table read here, a shared entity there — until the system is a distributed monolith (Part 2.5).
**Not a violation when:** The service is one of the seven in the approved ENR map; its ADR is your
design-gate document.

### MS-002 — Contract written after the code that uses it
**Severity:** major · **Applies to:** `libs/contracts/**`
**Detect:** A new `.proto` service or event type that first appears in the same pull request as the
producer and consumer code, with no prior design-gate review; a consumer merged before the event's
schema exists in `libs/contracts`.
**Why:** Contracts are the expensive part to change (Parts 5 and 6). In DOCURA you reviewed the event
catalogue before writing consumers; with five owners and seven deployables that review matters more,
not less.
**Not a violation when:** The change is an additive field on an existing, reviewed contract.

---

# Part 2 — Finding boundaries

## 2.1 Bounded contexts: one word, several meanings

A **bounded context** is a part of the system where every word has exactly one meaning. Draw service
boundaries around contexts, not around nouns. The same word means different things on each side
of a line:

| Word | In catalog | In inventory | In booking |
| --- | --- | --- | --- |
| **Trip** | Train 916 on 24 Oct: ordered stops, times, fares | A grid of coaches × seats × segments that can be occupied | One leg of what a customer bought, with a locked fare |
| **Seat** | Coach 5 layout: seats 1–80 in 2+2 rows | Seat 44 is free Banha→Tanta, held Cairo→Banha | "Coach 5 · Seat 44" printed on a ticket |
| **Passenger** | — | — | A snapshot copied into the ticket at booking time |

Identity owns *saved passengers* (the Passengers tab). Booking owns *the passenger on a ticket*. If
Ahmed edits a saved passenger's name after travelling, his old ticket must not change — they are
different concepts that happen to share a word.

## 2.2 Data ownership: one writer per fact

For every fact in the system, exactly one service is allowed to write it. Everyone else either asks
the owner, or keeps a read-only copy that the owner's events keep fresh.

| Question | Owner | Everyone else |
| --- | --- | --- |
| Is this email registered? Is this PIN right? | identity | Never sees passwords or PINs |
| Which stops does train 916 make? What does Alex→Cairo 1st class cost? | catalog | Booking locks the fare into the booking when the hold is created (R6) |
| Is seat 44 in coach 5 free from Banha to Tanta? | inventory | Catalog *displays* counts; it never decides |
| What did Ahmed book? Can ticket T be cancelled, and for how much? | booking | Gateway shows what booking returns |
| Did the money move? | Stripe, mirrored by payment | Booking reacts to payment's events |
| How many unread notifications does Ahmed have? | notification | Gateway shows the count |

"Who decides" and "who displays" are different questions. Catalog shows "Available Seats: 170" on
the results screen; inventory decides whether the seat is actually free when the hold is created.

## 2.3 The ENR service map

Seven deployables, five owners. Synchronous calls go over gRPC; everything else is an event.

```text
                                mobile app (EN / AR)
                                        │  HTTPS · REST /api/v1 · one envelope
                                        ▼
                               ┌─────────────────┐   Redis: rate limits, token denylist,
                               │     gateway     │          idempotency responses
                               └────────┬────────┘
                                        │  gRPC · user-facing RPCs only · access token relayed
      ┌─────────────┬─────────────┬─────┴───────┬─────────────┬───────────────┐
      ▼             ▼             ▼             ▼             ▼               ▼
 ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
 │ identity │ │ catalog  │ │ inventory │ │ booking  │ │ payment  │ │ notification │
 └──────────┘ └──────────┘ └───────────┘ └──────────┘ └──────────┘ └──────────────┘

 Service-to-service gRPC (booking orchestrates the saga, Part 8):
   booking ──► catalog      price quote for the chosen trip, class and stations
   booking ──► inventory    hold seats · confirm a hold · release a hold
   booking ──► payment      create / cancel a PaymentIntent · request a refund
   Stripe  ──► payment      webhooks over HTTPS, signature-verified, never through the gateway

 Every service behind the gateway owns one PostgreSQL database, publishes through its outbox,
 and consumes from its own queue bound to the RabbitMQ topic exchange "enr.events" (Part 6).
```

## 2.4 Heuristics for drawing a line

1. **What must change atomically stays together.** R7 says both legs of a round trip are held
   together or not at all. That is one database transaction inside inventory — not booking calling
   `HoldSeats` twice and compensating when the second call fails. Never build a saga where a
   transaction would do.
2. **Split by capability, not by table.** "Allocate seats adjacently for a group" (R2) is a
   capability. "Seats" is a table. The service that owns the capability owns the tables it needs.
3. **The decider owns the data.** The refund tiers in R8 live in booking because booking decides
   refunds. Payment executes refunds; it does not decide them.
4. **Follow the change rate.** Fares change weekly, timetables seasonally, seat occupancy every
   second. Data that changes at very different speeds often belongs to different services.

## 2.5 Anti-patterns

| Anti-pattern | What it looks like in ENR | Fix |
| --- | --- | --- |
| **Distributed monolith** | Services must be deployed together; a shared `libs/domain` holds entities; one HTTP request triggers a chain gateway → booking → catalog → inventory → payment | Own your data; replace chains with events or local copies; keep `libs/` cross-cutting (Part 3.6) |
| **Shared database** | Booking runs `SELECT … FROM inventory.seat_segment`; catalog reads booking tables to show "your trips" | Ask the owner, or keep a copy fed by events (Part 9) |
| **Entity services** | A `SeatService` exposing `GetSeat`/`UpdateSeat`, while booking runs the allocation logic and writes the result back | Expose the capability (`HoldSeats`) so the rule and its concurrency guard live with the data |
| **Chatty services** | Booking calls `catalog.GetStation` once per ticket on a 40-ticket list | Batch RPC (`GetStations` with 40 ids) or a local copy |

The entity-service anti-pattern deserves a second look. If booking reads a seat, decides it is free,
and writes it back over the network, you have rebuilt `CC-507` (unprotected read-modify-write) with
a network round trip in the middle. Two customers will get seat 44.

## Rules — boundaries

### MS-101 — Two services write the same table
**Severity:** critical · **Applies to:** entities, migrations, repositories in `apps/**`
**Detect:** The same table name in the migrations or entities of two apps; an entity class imported
by two apps; a migration in one app altering a table created by another.
**Why:** With two writers, no single place enforces the invariants, and a schema change in one
service silently breaks the other.
**Not a violation when:** Two services have *different* tables that happen to share a name inside
their own databases (for example both have an `inbox` table).

### MS-102 — A service reads another service's database
**Severity:** critical · **Applies to:** `apps/*/src/**`, `apps/*/.env*`, Compose files
**Detect:** A connection string for another service's database in an app's env; a second TypeORM
data source; SQL that names another service's schema or database.
**Why:** A cross-database read couples you to the owner's schema, bypasses its rules and locks, and
makes "deploy independently" a lie.
**Not a violation when:** Never, in application code. Ops tooling outside `apps/` that reads
replicas for reporting is out of scope for this project.

### MS-103 — Entity service instead of a capability
**Severity:** major · **Applies to:** `.proto` services, gRPC controllers
**Detect:** RPCs named `Get<Noun>`/`Update<Noun>`/`Create<Noun>` whose caller then applies the business
rule; a service whose only logic is mapping rows to messages.
**Why:** The rule ends up in the caller, far from the data and its concurrency guard (see 2.5).
**Not a violation when:** The RPC is a genuine read the caller only displays (`GetTrip`, `ListStations`).

### MS-104 — Atomic invariant split across services
**Severity:** critical · **Applies to:** service design, sagas
**Detect:** A saga or a sequence of RPCs used to enforce something that must never be observed
half-done: holding the two legs of a round trip (R7), the same National ID on overlapping segments
(R4), one passenger per seat per segment.
**Why:** Between the two calls, other requests see the half-done state, and compensation runs
after the damage. A local transaction with a constraint has no such window.
**Not a violation when:** The business tolerates the intermediate state and a compensation is
acceptable (payment taken, then seats confirmed — Part 8).

### MS-105 — Chatty call: an RPC per item
**Severity:** major · **Applies to:** `apps/*/src/**`
**Detect:** A gRPC call inside a `for`/`map`/`Promise.all` over a collection; more than two
sequential synchronous hops to answer one user request.
**Why:** Latency adds up and availability multiplies down: five dependencies at 99.9% each give you
99.5% at best.
**Not a violation when:** A business rule bounds the collection to two or three items and the owner
offers no batch RPC yet — and an issue asking for one is open.

---

# Part 3 — NestJS building blocks

## 3.1 One repository, many deployables

Each team keeps the whole system in one Nest **monorepo**: one `package.json`, one lockfile, one
`node_modules`, many apps. That gives you atomic refactors of shared libraries and one place to
run every test, while each app still builds and runs as its own process.

```text
enr/
├── apps/
│   ├── gateway/            src/main.ts · REST edge
│   ├── identity/           src/main.ts · gRPC + events
│   ├── catalog/
│   ├── inventory/
│   ├── booking/
│   ├── payment/            + HTTP endpoint for Stripe webhooks
│   └── notification/
├── libs/
│   ├── contracts/          .proto files, generated TS types, event payload schemas
│   ├── messaging/          envelope, outbox relay, inbox, retry topology, consumer helper
│   ├── auth/               access-token verification with the public key (gateway and services)
│   ├── observability/      logger setup, correlation context, gRPC interceptors
│   ├── resilience/         backoff with jitter, circuit-breaker wiring
│   ├── errors/             error-code base types, envelope, gRPC ↔ HTTP mapping
│   ├── cache/              cache-aside helper with single-flight, jitter, metrics
│   └── testing/            Testcontainers helpers, event and token builders
├── infra/                  postgres init scripts, rabbitmq definitions
├── docker-compose.yml
├── nest-cli.json
└── tsconfig.json           path aliases for @enr/*
```

Inside each app, keep the layering you used in DOCURA: `domain/` (pure rules), `application/`
(use cases and ports), `infrastructure/` (TypeORM, gRPC clients, RabbitMQ, Redis, Stripe adapters)
and `interface/` (gRPC controllers, event handlers, HTTP controllers). The dependency rule does not
change because there is a network involved; it matters more.

## 3.2 Monorepo mode

Running `nest generate app <name>` inside a standard project converts it to monorepo mode: apps
move under `apps/`, and `nest-cli.json` gains `"monorepo": true` and a `projects` map.
`nest generate library <name>` creates `libs/<name>` and asks for an import prefix (default `@app`;
choose `@enr`), which it registers in the root `tsconfig.json` `paths`.

```json
{
  "monorepo": true,
  "root": "apps/gateway",
  "sourceRoot": "apps/gateway/src",
  "compilerOptions": { "webpack": true, "tsConfigPath": "apps/gateway/tsconfig.app.json" },
  "projects": {
    "booking": {
      "type": "application",
      "root": "apps/booking",
      "entryFile": "main",
      "sourceRoot": "apps/booking/src",
      "compilerOptions": { "tsConfigPath": "apps/booking/tsconfig.app.json" }
    },
    "messaging": {
      "type": "library",
      "root": "libs/messaging",
      "entryFile": "index",
      "sourceRoot": "libs/messaging/src",
      "compilerOptions": { "tsConfigPath": "libs/messaging/tsconfig.lib.json" }
    }
  }
}
```

`nest start booking --watch` runs one app; `nest build booking` builds one. In monorepo mode the
Nest 11 CLI bundles each app with webpack and keeps `node_modules` external, which is what lets
OpenTelemetry instrument them (Part 11). Webpack does **not** copy `.proto` files, so resolve them
from the repository root at runtime and copy `libs/contracts/proto` into every image:

```ts
// libs/contracts/src/proto-file.ts
import { join } from 'node:path';

export function protoFile(relativePath: string): string {
  return join(process.cwd(), 'libs/contracts/proto', relativePath);
}
```

## 3.3 One process, three doors: hybrid applications

Most ENR services need three entry points: gRPC for synchronous requests, RabbitMQ for events,
and a small HTTP server for health checks (plus Stripe webhooks in payment). Nest runs all three in
one process: create an HTTP application, then attach microservice servers with
`connectMicroservice()`.

```ts
// apps/inventory/src/main.ts
import './tracing'; // must stay the first import — Part 11.4
import { NestFactory } from '@nestjs/core';
import { ConfigService } from '@nestjs/config';
import { Logger } from 'nestjs-pino';
import { AppModule } from './app.module';
import { configureApp } from './configure-app';
import { InventoryEnv } from './config/env.schema';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));
  configureApp(app);
  await app.startAllMicroservices();
  const config = app.get<ConfigService<InventoryEnv, true>>(ConfigService);
  await app.listen(config.get('HTTP_PORT', { infer: true }));
}
void bootstrap();
```

```ts
// apps/inventory/src/configure-app.ts — shared by main.ts and the integration tests (Part 13)
export function configureApp(app: INestApplication): void {
  const config = app.get<ConfigService<InventoryEnv, true>>(ConfigService);
  app.enableShutdownHooks();
  app.connectMicroservice<MicroserviceOptions>(
    {
      transport: Transport.GRPC,
      options: {
        package: 'enr.inventory.v1',
        protoPath: protoFile('enr/inventory/v1/inventory.proto'),
        url: config.get('GRPC_BIND_URL', { infer: true }),
        loader: { longs: Number, enums: String, defaults: true },
        gracefulShutdown: true, // let in-flight calls finish on close (Part 12.6)
      },
    },
    { inheritAppConfig: true },
  );
  app.connectMicroservice<MicroserviceOptions>(
    eventConsumerOptions(
      'inventory.events',
      config.get('RABBITMQ_URL', { infer: true }),
      config.get('RABBITMQ_PREFETCH', { infer: true }),
    ),
    { inheritAppConfig: true },
  );
}
```

Three details carry most of the weight:

- **`inheritAppConfig: true`.** Connected microservices do *not* inherit global pipes, guards,
  interceptors and filters unless you ask. Without it, your correlation-ID interceptor and your
  error filter silently skip every gRPC and RabbitMQ handler.
- **Filters must handle both contexts.** A global filter now receives HTTP *and* RPC exceptions.
  Branch on `host.getType()`, or register RPC-specific filters on gRPC controllers. Nest 11's default
  RPC filter turns any non-`RpcException` into gRPC `UNKNOWN` with "Internal server error"; your
  `libs/errors` filter must map domain errors to real status codes (Part 5.5).
- **`startAllMicroservices()` before `listen()`.** Consumers start before the HTTP server reports
  ready. `enableShutdownHooks()` lets Nest close all of them on `SIGTERM` (Part 12.6).

## 3.4 Clients: `ClientGrpc` and `ClientProxy`

Outbound connections are registered with `ClientsModule` and injected by token. Use
`registerAsync` so URLs come from validated config, never from `process.env`.

```ts
// apps/booking/src/infrastructure/grpc/inventory-client.module.ts (excerpt)
imports: [
  ClientsModule.registerAsync([{
    name: INVENTORY_GRPC,
    inject: [ConfigService],
    useFactory: (config: ConfigService<BookingEnv, true>) => ({
      transport: Transport.GRPC,
      options: {
        package: 'enr.inventory.v1',
        protoPath: protoFile('enr/inventory/v1/inventory.proto'),
        url: config.get('INVENTORY_GRPC_URL', { infer: true }),
        loader: { longs: Number, enums: String, defaults: true },
        channelOptions: { interceptors: [withDefaultDeadline(2_000), withCorrelationId()] },
      },
    }),
  }]),
],
providers: [
  {
    provide: INVENTORY_SERVICE,
    inject: [INVENTORY_GRPC],
    useFactory: (client: ClientGrpc) => client.getService<InventoryServiceClient>(INVENTORY_SERVICE_NAME),
  },
  { provide: SEAT_HOLDS, useClass: GrpcSeatHolds },
],
```

`InventoryServiceClient` and `INVENTORY_SERVICE_NAME` are generated from the `.proto` by `ts-proto`
with `nestJs=true` (Part 5.2). The interceptors are explained in Parts 5.4 and 11.2.

The use case never sees any of this. It depends on a **port** — `SeatHolds`, an interface in
`application/ports` with `hold()` and `confirm()` — and the gRPC client is one adapter behind it:

```ts
// apps/booking/src/infrastructure/grpc/grpc-seat-holds.adapter.ts
@Injectable()
export class GrpcSeatHolds implements SeatHolds {
  constructor(@Inject(INVENTORY_SERVICE) private readonly inventory: InventoryServiceClient) {}

  async hold(request: SeatHoldRequest): Promise<SeatHold> {
    const response = await lastValueFrom(this.inventory.holdSeats(toHoldSeatsRequest(request)));
    return toSeatHold(response); // map the generated type to the application's own model
  }
  // confirm(holdId) follows the same shape
}
```

The payoff: booking's saga is unit-tested against an in-memory `SeatHolds`, and the day inventory
changes transport, one adapter changes.

`ClientProxy` is the equivalent for message transports. In ENR you use it in exactly one place: the
outbox relay publishes events through a RabbitMQ `ClientProxy` (Part 7.2). Remember the difference:
`emit()` is **hot** (it publishes even if nobody subscribes), `send()` is **cold** (nothing happens
until you subscribe), and both return Observables — convert them with `lastValueFrom`.

## 3.5 `@MessagePattern` vs `@EventPattern`

| | `@MessagePattern` | `@EventPattern` |
| --- | --- | --- |
| Shape | Request → response | Fire and forget |
| Caller | Waits for an answer | Does not know who listens |
| ENR transport | gRPC (`@GrpcMethod` is a `@MessagePattern` bound to `Transport.GRPC`) | RabbitMQ |
| Use for | "Hold these seats and tell me which ones" | "Trip 916 on 24 Oct was cancelled" |

ENR uses RabbitMQ for **events only**. Request/response over a broker gives you the latency of a
call with the debugging cost of a queue; when you need an answer, use gRPC.

In a hybrid app an unscoped handler is registered on *every* connected server. Scope event handlers
to the transport they belong to:

```ts
@Controller()
export class TripEventsController {
  constructor(private readonly closeTripInventory: CloseTripInventory) {}

  @EventPattern('trip.cancelled', Transport.RMQ)
  async onTripCancelled(
    @Payload() event: EventEnvelope<TripCancelledV1>,
    @Ctx() context: RmqContext,
  ): Promise<void> {
    await this.closeTripInventory.handle(event, context); // ack / retry / park: Part 6.5
  }
}
```

## 3.6 What belongs in `libs/`

`libs/` is for code every service needs *the same way*. The moment a library knows what a booking
is, every service that imports it is coupled to booking's model and release cycle.

| Belongs in `libs/` | Never in `libs/` |
| --- | --- |
| `.proto` files and generated types; event payload schemas | TypeORM entities, repositories, migrations |
| Event envelope, outbox relay, inbox, retry topology | Refund tiers, fare and fee calculation, seat allocation |
| Logger setup, correlation context, gRPC interceptors | National ID validation (identity's rule) |
| Error-code base types, envelope, gRPC ↔ HTTP mapping | A `BookingStatus` enum shared "for convenience" |
| Config helpers (not the schemas — each app has its own) | Anything with `if (booking...)` in it |
| Testcontainers helpers, builders for envelopes and tokens | Fixtures of another service's data |

Contracts are shared by definition — they are the published language between services. They contain
shapes, never behaviour.

## Rules — codebase layout

### MS-051 — Domain code in `libs/`
**Severity:** major · **Applies to:** `libs/**`
**Detect:** Entities, repositories, migrations, domain services or business constants under `libs/`;
a library importing `typeorm` entities; names like `libs/common/booking-*`. Escalate to **critical**
when the shared code is an entity or repository (that is a shared database by another name).
**Why:** A shared domain library forces lockstep releases and turns seven services into one
distributed monolith.
**Not a violation when:** The code is a contract (generated proto types, event payload schema) or
genuinely cross-cutting (logging, envelope, outbox mechanics).

### MS-052 — One app imports another app's source
**Severity:** critical · **Applies to:** `apps/**`
**Detect:** An import path containing `apps/` from inside a different app, or a relative path that
climbs out of the app (`../../../inventory/src/...`). Grep hint: `from ['"].*apps/`
**Why:** It compiles today and couples two deployables at the source level; the next refactor in one
app breaks the build of the other.
**Not a violation when:** Never. Share through `libs/contracts`.

### MS-053 — Request/response over RabbitMQ, or an unscoped event handler
**Severity:** major · **Applies to:** controllers in `apps/**`
**Detect:** `@MessagePattern` used for RabbitMQ; `ClientProxy.send()` to an RMQ client; an
`@EventPattern` without `Transport.RMQ` in a hybrid app.
**Why:** RPC over a broker hides latency and failures behind a queue; unscoped handlers register on
every transport and surprise you later.
**Not a violation when:** The app has exactly one transport connected.

### MS-054 — Transport types leak into the application layer
**Severity:** major · **Applies to:** `apps/*/src/domain/**`, `apps/*/src/application/**`
**Detect:** `ClientGrpc`, `ClientProxy`, `Observable`, `RmqContext`, `Metadata`, generated proto types
or `Stripe` types imported in domain or application code.
**Why:** The domain should not change when the transport does, and use cases should be unit-testable
without a network (`CC-205`).
**Not a violation when:** The file is an adapter in `infrastructure/` or a handler in `interface/`.

### MS-055 — Connected microservice without the app's pipes, filters and interceptors
**Severity:** major · **Applies to:** `main.ts`, `configure-app.ts`, global filters
**Detect:** `connectMicroservice(...)` without `{ inheritAppConfig: true }` and without its own
filters/interceptors; a global exception filter that calls `switchToHttp()` unconditionally.
**Why:** Validation, correlation IDs and error mapping silently stop applying to gRPC and RabbitMQ
handlers — exactly the handlers that need them most.
**Not a violation when:** The microservice registers its own equivalents explicitly.

---

# Part 4 — The API gateway

## 4.1 What the gateway is for

The mobile app talks to exactly one host. The gateway is that host: it terminates the public REST
API and translates it into gRPC calls to the services behind it.

| The gateway does | The gateway does not |
| --- | --- |
| Verify the access token on every request | Call identity to ask "is this token valid?" |
| Deny by default; enforce coarse roles (`ops` routes) | Decide whether a ticket is refundable |
| Rate-limit per user and per IP, in Redis | Keep per-instance counters in memory |
| Route `/api/v1/...` to the owning service | Join data from two services' databases |
| Aggregate a screen from a few parallel calls | Orchestrate a booking (that is booking's saga) |
| Render one envelope; localize error codes (EN/AR, R13) | Leak gRPC details, stack traces or SQL errors |
| Store `Idempotency-Key` responses for money routes | Replace idempotency inside the services |

A gateway that starts making business decisions becomes a second monolith in front of your
microservices. Keep it thin.

## 4.2 Authentication at the edge

Identity signs short-lived access tokens with an **RS256 private key**. The gateway verifies them
with the **public key** — locally, in microseconds, without a network call. That choice buys you two
things: no extra latency per request, and existing sessions keep working while identity is down.

```text
 sign-in:   app ──► gateway ──gRPC──► identity   signs { sub, roles, jti, iss, aud, exp } with private key
 requests:  app ──► gateway (verifies with public key, checks jti denylist in Redis) ──► services
```

"Verify" means four checks: the algorithm is pinned to `RS256` (accepting whatever the token's
header says invites `alg` confusion attacks); the signature matches identity's public key; `exp`,
`iss` and `aud` are right; and the token's `jti` is not in the Redis denylist, so logout takes effect
before expiry (AUTH-1).

```ts
// apps/gateway/src/auth/access-token.guard.ts — registered as APP_GUARD: deny by default
@Injectable()
export class AccessTokenGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly jwt: JwtService,
    private readonly denylist: TokenDenylist,
    private readonly config: ConfigService<GatewayEnv, true>,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const targets = [context.getHandler(), context.getClass()];
    if (this.reflector.getAllAndOverride<boolean>(IS_PUBLIC, targets)) return true;
    const request = context.switchToHttp().getRequest<GatewayRequest>();
    const token = bearerTokenOf(request); // throws AUTH_TOKEN_MISSING
    const claims = await this.verify(token);
    if (await this.denylist.isRevoked(claims.jti)) throw new UnauthorizedException('AUTH_TOKEN_REVOKED');
    request.caller = { userId: claims.sub, roles: claims.roles, accessToken: token };
    return true;
  }

  private async verify(token: string): Promise<AccessTokenClaims> {
    try {
      const decoded = await this.jwt.verifyAsync<Record<string, unknown>>(token, {
        publicKey: this.config.get('JWT_PUBLIC_KEY', { infer: true }),
        algorithms: ['RS256'],
        issuer: 'enr-identity',
        audience: 'enr-api',
      });
      return accessTokenClaimsSchema.parse(decoded); // validate, then type (TS-104)
    } catch {
      throw new UnauthorizedException('AUTH_TOKEN_INVALID');
    }
  }
}
```

Public routes (sign-in, sign-up, OTP, station search) opt out with a `@Public()` decorator; everything
else needs a token, including routes added next sprint by someone who forgot to think about it.

## 4.3 Passing the caller's identity downstream

Once the gateway knows who is calling, the services behind it need to know too. The tempting
shortcut is a header such as `x-user-id: 42`. Here is why that is dangerous on its own:

1. **The header can come from outside.** If the gateway forwards client headers, any user can send
   `x-user-id: 42` and become user 42. The gateway must strip every inbound identity header.
2. **The service cannot tell who set it.** Any process that can reach booking's gRPC port — a
   compromised container, a debugging port published in Compose, a teammate's script — can set
   any metadata it likes. "Only the gateway calls us" is a network assumption, not a guarantee.

ENR uses **token relay**: the gateway forwards the verified access token in gRPC metadata, and each
service verifies it again locally with the same public key (a shared `libs/auth` helper; cheap and
cross-cutting). The service takes the user from the verified token — never from a request field.

```ts
// gateway — forward the token the guard already verified
export function callerMetadata(request: GatewayRequest): Metadata {
  const metadata = new Metadata();
  metadata.set('authorization', `Bearer ${request.caller.accessToken}`);
  return metadata;
}

// booking — every user-facing RPC starts by proving who is calling
@GrpcMethod('BookingService', 'GetTicket')
async getTicket(request: GetTicketRequest, metadata: Metadata): Promise<TicketView> {
  const caller = await this.callers.fromMetadata(metadata); // RS256 verify; UNAUTHENTICATED if bad
  return this.getTicketForUser.execute(caller.userId, request.ticketId); // 404, not 403 (RS-503)
}
```

Service-to-service commands that no user calls directly (booking confirming a hold in inventory)
are **not routed by the gateway at all**: the gateway has no client stub for them. In production you
would add mutual TLS or service credentials; for ENR, keep internal gRPC ports off the published
Compose ports and keep user-facing and internal RPCs in separate `.proto` services.

## 4.4 Routing and aggregation

Each REST route maps to one gRPC call on the owning service. When one screen needs data from
several services, the gateway may **aggregate**: call them in parallel and degrade gracefully when an
optional part fails.

```ts
// apps/gateway/src/home/home.controller.ts — the Home screen
@Get('home')
async home(@Req() request: GatewayRequest): Promise<HomeView> {
  const metadata = callerMetadata(request);
  const [profile, unread, recent] = await Promise.allSettled([
    lastValueFrom(this.identity.getProfile({}, metadata)),
    lastValueFrom(this.notifications.countUnread({}, metadata)),
    lastValueFrom(this.catalog.listRecentSearches({ limit: 5 }, metadata)),
  ]);
  if (profile.status === 'rejected') throw toHttpError(profile.reason); // required part
  return {
    greetingName: profile.value.fullName,
    unreadCount: unread.status === 'fulfilled' ? unread.value.count : null, // badge hidden, page works
    recentSearches: recent.status === 'fulfilled' ? recent.value.searches : [],
  };
}
```

Aggregation is presentation glue. If you find yourself writing `if` statements about bookings or
fares in the gateway, the logic belongs in a service.

## 4.5 Rate limiting

Rate limits live in **Redis**, because a limit kept in each gateway instance's memory is multiplied
by the number of instances, and resets on every deploy. Key by user for authenticated routes and by
IP for anonymous ones, and give the abuse-prone routes (sign-in, OTP request, OTP verify) their own,
much stricter class.

```ts
// fixed window: one key per (route class, subject, window); the key expires with its window
async consume(routeClass: RouteClass, subject: string): Promise<RateDecision> {
  const { limit, windowMs } = RATE_LIMITS[routeClass];
  const window = Math.floor(Date.now() / windowMs);
  const key = `gateway:rate:v1:${routeClass}:${subject}:${window}`;
  await this.redis.set(key, 0, 'PX', windowMs, 'NX'); // create once, with its TTL
  const count = await this.redis.incr(key);           // INCR keeps the TTL
  return { isAllowed: count <= limit, retryAfterMs: (window + 1) * windowMs - Date.now() };
}
```

Rejected requests get `429` with a `Retry-After` header. `@nestjs/throttler` v6 (TTL in
milliseconds) with the community `@nest-lab/throttler-storage-redis` storage is an acceptable
alternative; what is not acceptable is the default in-memory storage behind several instances.

## 4.6 One envelope, one error catalogue

Every response leaves the gateway in the envelope your standards already require (`RS-401`,
`RS-402`), with the message rendered in the caller's language:

```json
{ "success": false,
  "error": { "code": "SEATS_UNAVAILABLE", "message": "المقاعد المختارة لم تعد متاحة", "correlationId": "4f1c…" } }
```

Services return **stable error codes**, not sentences. The gateway maps each code to an HTTP status
and an EN/AR message from one catalogue, so wording can change without touching a service, and the
app never parses English. Unknown upstream failures become `503 SERVICE_UNAVAILABLE` or
`500 INTERNAL_ERROR` — never the raw gRPC `details` string.

For money-moving routes, the gateway also honours `Idempotency-Key` (`RS-105`): it stores the first
response under `(userId, key)` in Redis for 24 hours and replays it for retries. That protects the
mobile client's retries; the services below still make their own operations idempotent (Part 7).

## Rules — the gateway

### MS-201 — Access token verified remotely or loosely
**Severity:** critical · **Applies to:** `apps/gateway/src/auth/**`, `libs/auth/**`
**Detect:** A gRPC/HTTP call to identity per request to validate a token; `verify` without
`algorithms: ['RS256']`; no `issuer`/`audience`; a symmetric secret for access tokens; no `jti`
denylist check.
**Why:** Remote validation couples every request to identity's availability; loose verification
accepts tokens it should not.
**Not a violation when:** The call is the refresh-token flow, which identity owns by design.

### MS-202 — Route reachable without an explicit access decision
**Severity:** critical · **Applies to:** gateway controllers, global guards
**Detect:** No `APP_GUARD` for authentication; per-controller `@UseGuards(...)` as the only protection;
an `ops` route without a role check. Pairs with `RS-501`.
**Why:** Opt-in security fails on the route someone forgot. Deny by default fails safe.
**Not a violation when:** The route carries an explicit `@Public()` and appears in the reviewed list
of public routes.

### MS-203 — Caller identity trusted from a header or the body
**Severity:** critical · **Applies to:** gateway, gRPC controllers in every service
**Detect:** A service reading `x-user-id` (or similar) from metadata without verifying a token; a
`userId` field in a user-facing request message that the handler trusts; a gateway that forwards
inbound client headers unfiltered. Pairs with `RS-303`.
**Why:** Anyone who can set a header becomes anyone they like.
**Not a violation when:** The RPC is internal (not routed by the gateway) and the identity it carries
was taken from an event or a verified token upstream, such as a saga step acting for a booking's owner.

### MS-204 — Business rule in the gateway
**Severity:** major · **Applies to:** `apps/gateway/src/**`
**Detect:** Domain conditions (fares, refund tiers, seat counts, booking states) in gateway code; the
gateway calling two services in sequence to complete a business operation.
**Why:** The gateway becomes a hidden owner of rules nobody else can see or test.
**Not a violation when:** The logic is presentation: parallel aggregation, field renaming, hiding an
optional badge when its service is down.

### MS-205 — Upstream error shape or detail leaks through the edge
**Severity:** major · **Applies to:** gateway exception filter, aggregation code
**Detect:** gRPC `details`, `metadata` or a service's message returned to the client; more than one
error shape; an upstream failure surfacing as `500` with the raw error. Pairs with `RS-403`.
**Why:** Clients cannot branch on unstable text, and internals help attackers map your system.
**Not a violation when:** Never.

### MS-206 — Rate limit kept per instance or keyed too broadly
**Severity:** major · **Applies to:** gateway rate limiting
**Detect:** `ThrottlerModule` with the default in-memory storage; one global limit for every route;
OTP and sign-in routes on the same limit as search. Pairs with `RS-505`.
**Why:** Per-instance counters multiply the limit by the instance count; one shared bucket lets a
search-happy user lock themselves out of sign-in, or an attacker brute-force OTPs.
**Not a violation when:** The route is a health probe.

---

# Part 5 — Synchronous calls with gRPC

## 5.1 When to call at all

A synchronous call couples the caller's availability to the callee's. Use one only when the caller
**cannot continue without the answer**.

| Situation | Choice | ENR example |
| --- | --- | --- |
| I need a value to make my decision now | gRPC | Booking needs the fare before it can create the hold |
| I need another service to do something and tell me the result now | gRPC | Booking asks inventory to hold seats; the user is waiting |
| Something happened; others may care | Event | Catalog cancelled trip 916; booking, inventory and notification react |
| I need another service's data often | Local copy fed by events | Booking keeps station names (Part 9.3) |

gRPC over REST between services gives you typed contracts generated from one source, HTTP/2
multiplexing, deadlines built into the protocol, and a fixed set of status codes. The price is a
binary protocol (debug with `grpcurl`) and the discipline of `.proto` evolution.

## 5.2 Designing a `.proto` contract

Contracts live in `libs/contracts/proto/enr/<service>/v<N>/` and the package name carries the version.
This is an **illustrative** fragment — your contracts are a design-gate deliverable:

```proto
syntax = "proto3";

package enr.catalog.v1;

service FareService {
  rpc QuoteFare(QuoteFareRequest) returns (QuoteFareResponse);
}

enum SeatClass {
  SEAT_CLASS_UNSPECIFIED = 0;   // proto3 default; treat as "not provided" and reject
  SEAT_CLASS_FIRST = 1;
  SEAT_CLASS_SECOND_AC = 2;
}

message QuoteFareRequest {
  string trip_id = 1;
  string origin_station_id = 2;
  string destination_station_id = 3;
  SeatClass seat_class = 4;
}

message QuoteFareResponse {
  reserved 1;                    // was: double fare_egp — never reuse the number
  reserved "fare_egp";
  int64 fare_piastres = 2;       // integer minor units, unit in the name
  string quote_id = 3;
  string valid_until = 4;        // ISO-8601 UTC, e.g. "2026-10-24T09:30:00Z"
}
```

Conventions that save you later:

- **One request and one response message per RPC**, even when they look alike today. Shared messages
  make one RPC's change a breaking change for another.
- **Units in names** (`fare_piastres`, `hold_ttl_seconds`). Money is `int64` piastres. Load it with
  proto-loader `longs: Number` (and `ts-proto`'s default `forceLong=number`) so it arrives as a
  JavaScript `number` — the same trap as TypeORM's decimal-as-string in DOCURA, one layer down.
- **Enums start with `*_UNSPECIFIED = 0`.** In proto3 an absent field reads as its zero value; make
  "absent" distinguishable from a real choice.
- **Timestamps as ISO-8601 UTC strings** (or `google.protobuf.Timestamp` if the whole team agrees).
  Never local time, never an ambiguous number (`RS-604`).

Generate TypeScript with `ts-proto` so the server controller and client interfaces come from the same
file, for example with `buf generate`:

```yaml
# libs/contracts/buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-ts_proto
    out: src/generated
    opt:
      - nestJs=true
      - addGrpcMetadata=true
```

## 5.3 Evolving a contract

Clients and servers are deployed at different times, so for a while old clients talk to new
servers and the other way round. Every change must survive that.

| Change | Safe within `v1`? |
| --- | --- |
| Add a new field with a new number | Yes — old readers ignore it, new readers see the default when absent |
| Add a new RPC or a new enum value | Yes — but old clients must tolerate unknown enum values |
| Remove a field | Only by `reserved`-ing its number **and** name |
| Rename a field (same number, same type) | Wire-compatible, but breaks generated code and JSON; treat as breaking |
| Change a field's type or number | **No** |
| Reuse a number that used to mean something else | **Never** — old messages decode into the wrong field |
| Change what a field *means* (piastres → pounds) | **No**, even though no tool will notice |

A breaking change becomes a new package, `enr.catalog.v2`, served **next to** `v1` until every caller
has moved. `buf lint` and `buf breaking --against '.git#branch=main'` catch most structural breaks
before review (Part 13.4). Semantic breaks — like the unit change in the last row — only review catches.

## 5.4 Deadlines on every call

A call without a deadline can wait forever. While it waits it holds the user's HTTP request, a saga
step, maybe a database connection. Under load, those waits pile up until the caller falls over too.
**A deadline is the maximum time the caller is willing to wait**, and gRPC sends it to the server, so
the server can stop working on an answer nobody will read.

Budget deadlines from the outside in:

```text
 gateway   ───── 3.0 s for the whole user request ──────────────────────────────►
 booking        └─ 2.5 s for its work ─────────────────────────────────────────►
 inventory           └─ 1.0 s for HoldSeats ─►      catalog └─ 0.5 s for QuoteFare ─►
```

Set a default per client where the client is registered, so no call can forget one:

```ts
// libs/observability/src/grpc/with-default-deadline.ts
import { InterceptingCall, type Interceptor } from '@grpc/grpc-js';

export function withDefaultDeadline(timeoutMs: number): Interceptor {
  return (options, nextCall) =>
    new InterceptingCall(nextCall({ ...options, deadline: options.deadline ?? Date.now() + timeoutMs }));
}
```

Two traps: a gRPC deadline is an **absolute** time (`Date` or epoch milliseconds), never a duration —
`deadline: 1000` means January 1970. And a server that calls another service must not hand out more
time than it has left: read the remaining budget from `call.getDeadline()` on the incoming
`ServerUnaryCall` (the third argument of a `@GrpcMethod` handler).

## 5.5 Errors across the wire

gRPC has a fixed set of status codes. Use them deliberately, and carry your own **stable error code**
in the status message so the gateway can map it to the right HTTP status and localized text.

| gRPC status | Default HTTP | Use it in ENR for |
| --- | --- | --- |
| `INVALID_ARGUMENT` | 400 | Malformed request: unknown station id format, `SEAT_CLASS_UNSPECIFIED` |
| `FAILED_PRECONDITION` | 400 | State does not allow it: cancelling a ticket < 2 h before boarding (R8) |
| `OUT_OF_RANGE` | 400 | Date outside the published timetable |
| `UNAUTHENTICATED` | 401 | Missing or invalid relayed token |
| `PERMISSION_DENIED` | 403 | Authenticated, not allowed (non-ops calling an ops RPC) |
| `NOT_FOUND` | 404 | Unknown trip; another user's booking (404, not 403) |
| `ALREADY_EXISTS` | 409 | Duplicate create with a different payload |
| `ABORTED` | 409 | Concurrency conflict: the seats were taken while you were deciding |
| `RESOURCE_EXHAUSTED` | 429 | A per-user quota |
| `UNAVAILABLE` | 503 | The service or its dependency is down — **the** retryable code |
| `DEADLINE_EXCEEDED` | 504 | The deadline passed |
| `UNIMPLEMENTED` | 501 | RPC not deployed yet (version skew) |
| `INTERNAL` / `UNKNOWN` | 500 | Bugs. Never for an expected outcome |

(The default HTTP column follows `google/rpc/code.proto`; the error catalogue may override it per code —
`SEATS_UNAVAILABLE` is a `409` in ENR whatever its gRPC status.)

On the server, domain code throws a domain error; one filter on the gRPC controllers translates it:

```ts
// libs/errors/src/grpc-domain-error.filter.ts — used via @UseFilters on gRPC controllers
@Catch(DomainError)
export class GrpcDomainErrorFilter implements RpcExceptionFilter<DomainError> {
  catch(error: DomainError): Observable<never> {
    // grpc-js reads `code` as the status and `message` as the status details
    return throwError(() => ({ code: grpcStatusFor(error.code), message: error.code }));
  }
}
```

In Nest 11 an error that is not an `RpcException` (or handled by a filter like this one) reaches the
client as `UNKNOWN` with "Internal server error" — which is why an unmapped domain error shows up as
a `500` at the gateway. On the client side the error is a `ServiceError` from `@grpc/grpc-js` with
`code`, `details` (your error code) and `metadata`; the gateway's filter looks `details` up in the
catalogue and falls back to the table above.

## 5.6 Retries: only for idempotent calls

Retrying is how you survive a blip, and how you charge a customer twice. The difference is
idempotency.

- **Reads are idempotent.** `QuoteFare` may be retried.
- **Writes are idempotent only by design.** If booking retries `HoldSeats` after a lost response,
  inventory must recognise the second call. Give every write a caller-generated key —
  `hold_request_id = bookingId` — and have the server return the original result for a known key.
- **Retry only codes that mean "try again"**: `UNAVAILABLE` always; `DEADLINE_EXCEEDED` only for
  reads (a write may have completed). Never `INVALID_ARGUMENT`, `FAILED_PRECONDITION` or `ABORTED`
  at this level — `ABORTED` means the *caller* must re-decide, not resend.
- **Bounded, with backoff and jitter** (Part 12.3), and at one layer only.

```ts
const quote = await lastValueFrom(
  this.fares.quoteFare(toQuoteFareRequest(selection)).pipe(
    retry({ count: 2, delay: (error: unknown, attempt: number) => retryUnavailable(error, attempt) }),
  ),
);

function retryUnavailable(error: unknown, attempt: number): Observable<number> {
  if (!isServiceError(error) || error.code !== status.UNAVAILABLE) return throwError(() => error);
  return timer(backoffWithJitter(attempt)); // Part 12.3
}
```

Each attempt gets its own deadline from the interceptor, so three attempts can take three deadlines:
budget for that.

## Rules — gRPC

### MS-251 — gRPC call without a deadline
**Severity:** critical · **Applies to:** gRPC client registrations and adapters
**Detect:** A `ClientsModule` gRPC registration without a deadline interceptor; a call with a
relative number as `deadline`; a handler calling downstream with more time than `call.getDeadline()`
leaves.
**Why:** One slow dependency without a deadline exhausts the caller's resources and spreads the outage.
**Not a violation when:** The call is a server-streaming subscription that is meant to stay open and
is cancelled explicitly — ENR has none.

### MS-252 — Breaking change inside a published proto package
**Severity:** critical · **Applies to:** `libs/contracts/proto/**`
**Detect:** A field number reused or changed; a field removed without `reserved` number and name; a
type change; a semantic change (units, meaning) under the same field; `buf breaking` failing.

```proto
// Bad — number 1 used to be double fare_egp; old messages now decode into the wrong field
message QuoteFareResponse { int64 fare_piastres = 1; }

// Good
message QuoteFareResponse { reserved 1; reserved "fare_egp"; int64 fare_piastres = 2; }
```

**Why:** Services deploy at different times. A breaking change inside `v1` corrupts data silently
during the window when old and new versions talk to each other.
**Not a violation when:** The contract has never been merged to `main` or consumed by another service.

### MS-253 — Retry of a non-idempotent call, or an unbounded retry
**Severity:** major · **Applies to:** gRPC adapters
**Detect:** `retry()` on a write RPC whose request has no idempotency key; retry on any error rather
than on `UNAVAILABLE`; `retry()` without a count or without backoff.
**Why:** Retrying a non-idempotent write duplicates holds or payments; unbounded retries turn a blip
into a self-inflicted outage.
**Not a violation when:** The retried write carries a key the server deduplicates, and that
deduplication is covered by a test.

### MS-254 — Expected failure returned as `UNKNOWN` or `INTERNAL`
**Severity:** major · **Applies to:** gRPC controllers, `libs/errors`
**Detect:** Domain errors or `HttpException`s thrown from gRPC handlers with no filter mapping them;
`RpcException('some sentence')`; a status message that is free text instead of a catalogued code.
**Why:** The gateway cannot tell "seats taken" (409, show a message) from "bug" (500, alert), and
clients cannot localize text.
**Not a violation when:** The failure really is unexpected (a bug); `INTERNAL` is correct then.

### MS-255 — Hand-written copy of a contract type
**Severity:** major · **Applies to:** `apps/**`
**Detect:** An `interface` mirroring a proto message or gRPC service by hand; client code typed with
an inline literal instead of the generated `*Client` interface.
**Why:** Hand copies drift from the `.proto`; the compiler then vouches for a contract that no longer
exists (`TS-202`).
**Not a violation when:** The type is the application's own model, mapped from the generated type
in an adapter.

---

# Part 6 — Asynchronous events with RabbitMQ

## 6.1 Why events

An event says "this happened" and does not care who listens. That buys three things a call cannot:

- **Temporal decoupling.** Booking confirms a ticket while notification is down; the email goes out
  when notification comes back.
- **Fan-out without coupling.** When ops cancels trip 916 (R9), catalog publishes one event. It does
  not know, and must not know, that booking will void tickets and request refunds, inventory will
  close the seat map, and catalog's own cache will be invalidated.
- **Load levelling.** A cancelled trip with 600 passengers produces 600 refunds. A queue lets payment
  work through them at the rate Stripe allows, instead of all at once.

```text
                             trip.cancelled (catalog)
                                      │
            ┌─────────────────────────┼──────────────────────────┐
            ▼                         ▼                          ▼
   booking: void tickets,    inventory: close the       catalog: drop cached search
   refund fares + fee (R9)   trip's seat inventory      results for that date
            │
            ▼  ticket.voided, refund.requested (booking)
   payment: Stripe refunds, one by one ──► refund.succeeded ──► booking marks Refunded (R10)
   notification: "Your trip was cancelled, refund on its way" (EN/AR)
```

Notice who notifies passengers: **booking's** events, not catalog's. Catalog does not know who is on
the train. Events flow from the service that knows the facts.

## 6.2 RabbitMQ in five minutes

| Concept | What it is | In ENR |
| --- | --- | --- |
| **Exchange** | Receives every published message and routes it; stores nothing | `enr.events` (topic) for domain events; `enr.retry` (direct) for retries |
| **Queue** | Stores messages until a consumer acknowledges them | One per consuming service: `notification.events`, `booking.events`, … |
| **Binding** | "Send messages with routing key matching X to queue Y" | `notification.events` ← `booking.confirmed`, `ticket.*` |
| **Routing key** | A dotted string on each message | The event type: `trip.cancelled` |
| **Topic wildcards** | `*` matches exactly one word, `#` matches zero or more | `ticket.*` matches `ticket.voided`; `trip.#` matches `trip.delay.reported` |
| **Competing consumers** | Several consumers on one queue share its messages | Three notification instances split the load |

Two rules fall out of that table. **Each consuming service gets its own queue**, so every service
receives its own copy of each event. **Instances of the same service share that queue**, so each
event is processed once per service, not once per instance.

## 6.3 The ENR topology

```text
   outbox relays ──publish──► enr.events (topic, durable)
                                │ booking.confirmed │ trip.cancelled │ ticket.voided …
                ┌───────────────┴──────┬────────────┴───────┬──────────────────┐
                ▼                      ▼                    ▼                  ▼
        booking.events        inventory.events      notification.events     …   (quorum, durable)
                ▼                      ▼                    ▼
          booking ×n             inventory ×n         notification ×n          (competing consumers)
```

Failures need their own topology, because a message that failed should come back **later**, not
immediately, and should stop coming back after a few tries:

```text
 notification.events ── handler fails (attempt n) ──► enr.retry, key "notification.events.retry.n"
        ▲                                                         │
        │                                                         ▼
        │                         notification.events.retry.n   (TTL 5 s · 30 s · 2 min · 10 min)
        │                                                         │ TTL expires → dead-lettered
        └──────────── enr.retry, key "notification.events" ◄──────┘
 after the last tier ──► enr.retry, key "notification.events.parking-lot" ──► the parking lot
 rejected, or over the quorum delivery limit ──► main queue's dead-letter exchange ──► the parking lot
```

**Retry queues** have no consumers: a message waits until its TTL expires and is dead-lettered back
to the main queue. One queue per delay tier means every message in it has the same TTL, so none
waits behind a longer-lived one. **The parking lot** has no consumers either; someone fixes the cause
and replays (Part 7.6). **The main queue's own dead-letter exchange** catches what the handler never
saw: messages Nest rejects for lack of a handler, and poison messages that crash the process until
the quorum queue's delivery limit (default 20 since RabbitMQ 4.0) is reached.

```ts
// libs/messaging/src/topology.ts
export const EVENTS_EXCHANGE = 'enr.events';
export const RETRY_EXCHANGE = 'enr.retry';
export const RETRY_DELAYS_MS = [5_000, 30_000, 120_000, 600_000] as const;

export const retryQueue = (queue: string, tier: number): string => `${queue}.retry.${tier}`;
export const parkingLot = (queue: string): string => `${queue}.parking-lot`;

// Nest asserts the main queue with these exact options; declare them in one place only.
export function mainQueueOptions(queue: string): Options.AssertQueue {
  return {
    durable: true,
    deadLetterExchange: RETRY_EXCHANGE,
    deadLetterRoutingKey: parkingLot(queue),
    arguments: { 'x-queue-type': 'quorum' },
  };
}

// Run once at boot, before app.startAllMicroservices()
export async function declareTopology(channel: Channel, queue: string): Promise<void> {
  await channel.assertExchange(EVENTS_EXCHANGE, 'topic', { durable: true });
  await channel.assertExchange(RETRY_EXCHANGE, 'direct', { durable: true });
  await channel.assertQueue(queue, mainQueueOptions(queue));
  await channel.bindQueue(queue, RETRY_EXCHANGE, queue);
  for (const [tier, delayMs] of RETRY_DELAYS_MS.entries()) {
    const name = retryQueue(queue, tier);
    await channel.assertQueue(name, {
      durable: true, messageTtl: delayMs, deadLetterExchange: RETRY_EXCHANGE, deadLetterRoutingKey: queue,
    });
    await channel.bindQueue(name, RETRY_EXCHANGE, name);
  }
  await channel.assertQueue(parkingLot(queue), { durable: true });
  await channel.bindQueue(parkingLot(queue), RETRY_EXCHANGE, parkingLot(queue));
}
```

`Channel` and `Options` come from `amqplib`. RabbitMQ refuses to re-declare a queue with different
arguments (`PRECONDITION_FAILED`), which is why `mainQueueOptions` is the single source for both this
function and Nest.

## 6.4 Nest's RabbitMQ transport, configured for ENR

Nest 11's built-in RMQ transport can bind one queue to a topic exchange with one binding per
`@EventPattern`, and it hands you the raw channel for manual acknowledgement. Its defaults, however,
are wrong for ENR, so every option below is set on purpose:

```ts
// libs/messaging/src/event-consumer-options.ts
export function eventConsumerOptions(queue: string, url: string, prefetchCount: number): RmqOptions {
  return {
    transport: Transport.RMQ,
    options: {
      urls: [url],
      queue,
      queueOptions: mainQueueOptions(queue),
      exchange: EVENTS_EXCHANGE,
      exchangeType: 'topic',
      wildcards: true,   // bind the queue once per @EventPattern of this app
      noAck: false,      // Nest's default is true (auto-ack): a crash would lose the message
      prefetchCount,     // Nest's default is 0, which means unlimited (6.6)
    },
  };
}
```

Things to know about the transport:

- **The message format is Nest's.** The body is `{ "pattern": "trip.cancelled", "data": <envelope> }`
  and Nest dispatches on `pattern` from the body, not on the routing key. Publish through Nest's
  `ClientProxy` (Part 7.2) so producers and consumers agree; a non-Nest producer must use that shape.
- **A throwing handler is neither acked nor nacked.** With `noAck: false`, Nest leaves the message
  unacknowledged until the channel closes. Every handler must end in ack, retry or park — which is
  what the consumer helper in 6.5 guarantees.
- **A message with no matching handler is rejected without requeue**, so it lands in the parking lot
  through the main queue's dead-letter exchange. That is a feature: you see routing mistakes.

Why not something else? Raw `amqplib` or `amqp-connection-manager` work, but you would rebuild handler
discovery and DI wiring. `@golevelup/nestjs-rabbitmq` is a good library too, but its default error
behaviour requeues failed messages immediately — a hot loop unless you change it. Pick one approach
per team; ENR's reference is the built-in transport plus `libs/messaging`.

## 6.5 Consuming: acknowledge after the work, never before

An acknowledgement tells RabbitMQ "you can delete this". Send it **after** your database transaction
commits. Ack before the work and a crash loses the event; ack after and a crash means the event is
delivered again — which the inbox makes harmless (Part 7.3).

```ts
// libs/messaging/src/event-consumer.ts
@Injectable()
export class EventConsumer {
  constructor(
    @Inject(CONSUMER_QUEUE) private readonly queue: string,
    private readonly inbox: Inbox,
    private readonly retries: RetryScheduler,
    private readonly logger: PinoLogger,
  ) {}

  async handle<TPayload>(
    context: RmqContext,
    envelope: EventEnvelope<unknown>,
    schema: ZodType<TPayload>,
    work: (payload: TPayload, tx: EntityManager) => Promise<void>,
  ): Promise<void> {
    const channel = channelOf(context);
    const message = messageOf(context);
    const parsed = schema.safeParse(envelope.payload);
    if (!parsed.success) {
      return this.retries.park(channel, message, this.queue); // retrying will never fix a bad payload
    }
    try {
      await withEventContext(envelope, () =>
        this.inbox.processOnce(envelope.eventId, (tx) => work(parsed.data, tx)));
      channel.ack(message);
    } catch (error) {
      this.logger.warn({ err: error, eventId: envelope.eventId, type: envelope.type }, 'event failed');
      await this.retries.retryOrPark(channel, message, this.queue);
    }
  }
}

// The one place Nest's loosely typed RmqContext is narrowed.
export const channelOf = (context: RmqContext): ConfirmChannel => context.getChannelRef();
export const messageOf = (context: RmqContext): ConsumeMessage => context.getMessage() as ConsumeMessage;
```

```ts
// libs/messaging/src/retry-scheduler.ts
async retryOrPark(channel: ConfirmChannel, message: ConsumeMessage, queue: string): Promise<void> {
  const attempt = Number(message.properties.headers?.['x-retry-attempt'] ?? 0);
  const target = attempt < RETRY_DELAYS_MS.length ? retryQueue(queue, attempt) : parkingLot(queue);
  channel.publish(RETRY_EXCHANGE, target, message.content, {
    persistent: true,
    messageId: message.properties.messageId,
    contentType: message.properties.contentType,
    headers: { ...message.properties.headers, 'x-retry-attempt': attempt + 1 },
  });
  await channel.waitForConfirms(); // the copy is safely stored…
  channel.ack(message);            // …before the original is released
}
```

`withEventContext` restores the correlation ID from the envelope for logging (Part 11.2). An app's
handler then reads like a sentence:

```ts
@EventPattern('booking.confirmed', Transport.RMQ)
async onBookingConfirmed(
  @Payload() event: EventEnvelope<unknown>,
  @Ctx() context: RmqContext,
): Promise<void> {
  await this.consumer.handle(context, event, bookingConfirmedV1, (payload, tx) =>
    this.sendBookingConfirmation.execute(payload, tx));
}
```

## 6.6 Prefetch, concurrency and durability

**Prefetch** is how many unacknowledged messages RabbitMQ will push to one consumer. Nest's default is
`0` — unlimited — so one instance can pull the whole backlog into memory and starve its siblings.
Nest processes prefetched messages **concurrently**, so prefetch is also your concurrency limit: size
it against the database pool (the `pg` driver's pool defaults to 10 connections), and keep it low
for handlers that call Stripe.

A message survives a broker restart only when **all four** hold:

| Requirement | Who sets it | Default in Nest 11 |
| --- | --- | --- |
| Durable exchange | Nest asserts `enr.events` as durable | Yes |
| Durable queue | `queueOptions.durable` | Set it explicitly (`mainQueueOptions`) |
| Persistent message | `persistent: true` on the client, or per record | **No** — `persistent` defaults to `false` |
| Publisher confirm awaited | `amqp-connection-manager` uses confirm channels; the publish settles after the broker's ack | Yes, if you wait for it |

## 6.7 The event envelope

Every event has the same outer shape, whatever its payload. Consumers, logs, the inbox and the
replay tooling all rely on it.

```ts
// libs/messaging/src/event-envelope.ts
export interface EventEnvelope<TPayload> {
  readonly eventId: string;       // UUID; the inbox deduplicates on it
  readonly type: string;          // 'trip.cancelled' — also the routing key
  readonly version: number;       // payload schema version: 1, 2, …
  readonly occurredAt: string;    // ISO-8601 UTC, when the fact happened (not when it was sent)
  readonly correlationId: string; // the user request or job that started the whole chain
  readonly causationId: string;   // the request or eventId that directly caused this event
  readonly producer: string;      // 'catalog'
  readonly payload: TPayload;
}
```

`correlationId` answers "what else happened because of this click?"; `causationId` answers "what
directly caused this?" Together they let you rebuild the chain: request → `trip.cancelled` →
`ticket.voided` → `refund.requested` → `refund.succeeded`.

## 6.8 Naming, versioning, and what goes inside

**Name events as past-tense business facts**: `<aggregate>.<past participle>`.

| Good | Bad | Why it is bad |
| --- | --- | --- |
| `booking.confirmed` | `confirmBooking` | A command; the sender expects a specific receiver to act |
| `hold.expired` | `seat.updated` | "Updated" tells the consumer nothing; it must diff to find out |
| `trip.delay.reported` | `sendDelayEmail` | Names one consumer's reaction, not the fact |
| `refund.succeeded` | `stripe.webhook.received` | Leaks a provider detail into everyone's vocabulary |

**Version payloads, not envelopes.** Adding an optional field keeps `version: 1`. Removing or
renaming a field, or changing its meaning, is `version: 2`: publish both versions side by side until
every consumer has moved, then stop publishing `v1`. Consumers are **tolerant readers**: they ignore
fields they do not know and validate only what they use (the zod schema in 6.5).

**Carry what consumers need to act, not the whole database.** `trip.cancelled` carries the trip id
and reason; booking already knows who holds tickets. Keep personal data to the minimum the consumer
needs, and never put tokens, passwords or card data in an event. The OTP email is the one deliberate
exception in ENR: the code must travel from identity to notification. It goes in the payload (never
in headers), has a short expiry, and is redacted from every log (Part 11.3).

## Rules — RabbitMQ

### MS-301 — Message acknowledged before the work commits, or never
**Severity:** critical · **Applies to:** RMQ consumer options, event handlers, `libs/messaging`
**Detect:** `noAck` missing or `true`; `channel.ack(...)` before the transaction resolves; a handler
path that returns or throws without ack, retry or park.
**Why:** Auto-ack loses the event on a crash; a missing ack leaves the message stuck as unacked,
then redelivered to someone else on reconnect.
**Not a violation when:** Never.

### MS-302 — Unbounded or untuned prefetch
**Severity:** major · **Applies to:** RMQ consumer options
**Detect:** `prefetchCount` absent (Nest default `0` = unlimited) or larger than the handler's
database pool can serve.
**Why:** Unlimited prefetch hoards the backlog in one instance and runs that many handlers at once.
**Not a violation when:** Never in ENR; every consumer sets it.

### MS-303 — Message can be lost by the broker
**Severity:** critical · **Applies to:** RMQ clients, topology, outbox relay
**Detect:** A non-durable queue or exchange; a client without `persistent: true`; the outbox relay
marking a row published without awaiting the publish (and so the broker confirm).
**Why:** A broker restart silently deletes non-persistent messages; an unconfirmed publish may never
have arrived.
**Not a violation when:** Never for domain events.

### MS-304 — Queue shared across services, or consumed by a non-owner
**Severity:** major · **Applies to:** topology, consumer options
**Detect:** Two different apps configured with the same `queue`; an app consuming a queue named after
another service; one queue per event type shared by all consumers.
**Why:** Competing consumers from different services split events between them, so each service sees
only some of them.
**Not a violation when:** The consumers are instances of the same service.

### MS-305 — Message without the standard envelope
**Severity:** major · **Applies to:** producers, outbox writes
**Detect:** An event published as a bare payload; missing `eventId`, `version`, `occurredAt`,
`correlationId`, `causationId` or `producer`; `occurredAt` set at publish time by the relay.
**Why:** Without `eventId` there is no deduplication; without correlation there is no debugging;
without `version` there is no evolution.
**Not a violation when:** Never.

### MS-306 — Event named as a command or a CRUD operation
**Severity:** minor · **Applies to:** event types in `libs/contracts`
**Detect:** Imperative names (`sendEmail`, `releaseSeats`), `*.updated`/`*.changed` without a
business meaning, provider names (`stripe.*`) in domain event types.
**Why:** Commands couple the producer to one consumer's job; vague names push the diffing work onto
every consumer.
**Not a violation when:** The event deliberately replicates an aggregate's display data for copies
(event-carried state transfer), such as `station.updated` carrying the full bilingual station record.

### MS-307 — Breaking event change under the same version
**Severity:** major · **Applies to:** event schemas in `libs/contracts`
**Detect:** A field removed, renamed, retyped or re-meant without bumping `version`; consumers that
reject unknown fields; a `v2` producer deployed with no `v1` still published.
**Why:** Consumers deploy on their own schedule; a breaking change parks every message they receive
until they catch up.
**Not a violation when:** No consumer of that event has been merged yet.

### MS-308 — Failed message requeued in a hot loop, or dropped
**Severity:** critical · **Applies to:** event handlers, `libs/messaging`
**Detect:** `channel.nack(message, false, true)`; `@golevelup` subscriptions with the default
`REQUEUE` behaviour; a `catch` that acks and logs; no parking lot; schema-invalid messages retried.
**Why:** Immediate requeue spins a CPU and floods logs while blocking the queue; ack-and-log loses the
event with no trace of what was lost.
**Not a violation when:** Never.

### MS-309 — Secret or unnecessary personal data in an event
**Severity:** critical · **Applies to:** event schemas, producers
**Detect:** Tokens, passwords, PINs, card data or full National IDs in payloads or headers; whole
user profiles where an id would do.
**Why:** Messages are copied to retry queues, parking lots and logs, and are read by every
subscriber — present and future.
**Not a violation when:** The field is the OTP code in the event that asks notification to send it,
it lives only in the payload, and it is covered by log redaction.

---

# Part 7 — Reliability: outbox, inbox and poison messages

## 7.1 The dual-write problem, again

You met this in DOCURA, and the reviews showed how often it slipped: two of the four teams published
without a working outbox. In ENR the cost is higher, because other *services* act on the event.

```text
 booking:  BEGIN; confirm booking; issue tickets; COMMIT         — succeeds
           publish booking.confirmed                            — fails (broker restarting,
                                                                  process killed, network blip)
 result:   tickets exist, no confirmation email, inventory never converts the hold to a sale,
           the seats are released at hold expiry and resold — to someone else.
```

Publishing first is no better: the event goes out, the transaction rolls back, and notification
emails a confirmation for a booking that does not exist. You cannot make a database and a broker
commit atomically. You can make **one** of them the source of truth for both.

## 7.2 The transactional outbox

Write the event into an `outbox` table **in the same transaction** as the state change. A separate
relay publishes rows from that table and marks them published. If the transaction rolls back, the
event never existed; if it commits, the relay will eventually publish it.

```ts
// apps/booking/src/application/confirm-booking.use-case.ts (excerpt)
await this.unitOfWork.run(async (tx) => {
  await this.bookings.markConfirmed(tx, booking);
  await this.tickets.issue(tx, booking);
  await this.outbox.add(tx, bookingConfirmedEvent(booking, currentContext()));
});
```

The outbox table is platform infrastructure, identical in every producing service:

```sql
CREATE TABLE outbox (
  id            bigserial   PRIMARY KEY,
  event_id      uuid        NOT NULL UNIQUE,
  envelope      jsonb       NOT NULL,
  trace_context jsonb       NOT NULL DEFAULT '{}',   -- W3C traceparent captured at write time (Part 11.4)
  created_at    timestamptz NOT NULL DEFAULT now(),
  claimed_until timestamptz,
  published_at  timestamptz
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

The relay **claims** a batch with `FOR UPDATE SKIP LOCKED`, so several relay instances never grab
the same rows, and sets a short lease. It then publishes **outside** any database transaction —
holding row locks while waiting on the network is `CC-506` — and marks each row after the broker
confirms it.

```sql
-- claimBatch: one short transaction
UPDATE outbox SET claimed_until = now() + interval '30 seconds'
WHERE id IN (
  SELECT id FROM outbox
  WHERE published_at IS NULL AND (claimed_until IS NULL OR claimed_until < now())
  ORDER BY id
  LIMIT 100
  FOR UPDATE SKIP LOCKED
)
RETURNING id, envelope, trace_context;
```

```ts
// libs/messaging/src/outbox-relay.ts
@Injectable()
export class OutboxRelay implements OnModuleDestroy {
  private isDraining = false;
  private isStopping = false;

  constructor(
    private readonly outbox: OutboxStore,
    @Inject(EVENTS_CLIENT) private readonly events: ClientProxy,
  ) {}

  @Interval(500)
  async drain(): Promise<void> {
    if (this.isDraining || this.isStopping) return;
    this.isDraining = true;
    try {
      const batch = await this.outbox.claimBatch(100); // sorted by id in code
      for (const row of batch) {                       // sequential on purpose: keeps producer order
        await this.publish(row);
        await this.outbox.markPublished(row.id);
      }
    } finally {
      this.isDraining = false;
    }
  }

  onModuleDestroy(): void {
    this.isStopping = true; // no new batches during shutdown (Part 12.6)
  }

  private async publish(row: OutboxRow): Promise<void> {
    const record = new RmqRecordBuilder(row.envelope)
      .setOptions({
        messageId: row.envelope.eventId,
        contentType: 'application/json',
        persistent: true,
        headers: { 'x-correlation-id': row.envelope.correlationId },
      })
      .build();
    const parentContext = propagation.extract(ROOT_CONTEXT, row.traceContext);
    await context.with(parentContext, () =>
      lastValueFrom(this.events.emit(row.envelope.type, record).pipe(timeout(5_000)), {
        defaultValue: undefined,
      }),
    );
  }
}
```

`EVENTS_CLIENT` is a RabbitMQ `ClientProxy` registered with `exchange: 'enr.events'`,
`exchangeType: 'topic'`, `wildcards: true` and `persistent: true`, so `emit(type, …)` publishes to the
topic exchange with the event type as routing key. `@Interval` comes from `@nestjs/schedule`.

If a publish fails or times out, the loop stops; the rest of the batch is re-claimed when the lease
expires, still in order. If the relay crashes **after** publishing but **before** marking, the row
is published again. That is not a bug in the outbox — it is the definition of at-least-once, and the
inbox handles it. Delete published rows after a few days with a scheduled cleanup.

## 7.3 The inbox: idempotent consumers

Every consumer will, sooner or later, receive the same event twice: the relay re-published it, the
consumer crashed after committing but before acking, or someone replayed a parking lot. The inbox
records each processed `eventId` **in the same transaction as the consumer's own writes**:

```ts
// libs/messaging/src/inbox.ts
async processOnce(
  eventId: string,
  work: (tx: EntityManager) => Promise<void>,
): Promise<'processed' | 'duplicate'> {
  return this.dataSource.transaction(async (tx) => {
    const inserted: Array<{ event_id: string }> = await tx.query(
      'INSERT INTO inbox (event_id) VALUES ($1) ON CONFLICT DO NOTHING RETURNING event_id',
      [eventId],
    );
    if (inserted.length === 0) return 'duplicate'; // already done: ack and move on
    await work(tx);
    return 'processed';
  });
}
```

If `work` throws, the inbox row rolls back with everything else, so the retry will run the work
again. If two copies arrive at once, the second `INSERT` waits on the primary key and then sees the
first one's row — exactly one wins.

The inbox protects **database** effects. Effects outside your database need their own key:

| Side effect | How to make it idempotent |
| --- | --- |
| Stripe refund for ticket T | Store the refund row first (`PENDING`); call Stripe with `idempotencyKey: 'refund:' + refundId`; never refund a row that already has a Stripe id |
| PaymentIntent for booking B | `idempotencyKey: 'payment-intent:' + bookingId` |
| Confirmation email | Record the delivery in the same transaction, send after commit, mark sent. A crash between send and mark sends one duplicate; document that you accept it |

Stripe keeps idempotency keys for at least 24 hours, so your own `PENDING` row is the long-term
guard; the key protects the minutes around a retry.

## 7.4 At-least-once is the contract

| Guarantee | What it means | Who offers it |
| --- | --- | --- |
| At-most-once | Never duplicated, sometimes lost | Auto-ack consumers (Nest's default) |
| At-least-once | Never lost, sometimes duplicated | Outbox + confirms + manual ack |
| Effectively-once | Duplicates arrive but change nothing | At-least-once **plus** inbox and idempotency keys |

End-to-end exactly-once delivery across a broker, two databases and Stripe is not on offer. Build
effectively-once from the parts you control.

## 7.5 Ordering, and designing so you do not need it

RabbitMQ keeps order only for one publisher on one channel to one queue with one consumer. ENR
breaks every part of that: several relay instances, retries that come back thirty seconds later,
competing consumers, and `prefetchCount` messages handled concurrently. Assume events arrive in any
order, and make the order not matter:

- **Guard with state.** A refund that is already `REFUNDED` ignores a late `refund.pending`. The
  booking saga (Part 8) only accepts events valid in its current state.
- **Newer wins.** Put an increasing `version` (or the owner's `occurredAt`) in replicated data and
  update only when the incoming one is newer: two delay reports for trip 916 (30 min, then 75 min)
  arriving swapped must leave 75.
- **Prefer facts that commute.** "Seat 44 held for segment Banha→Tanta by hold H" can be applied in
  any order with "hold H released"; "available count is now 169" cannot.
- **When confused, ask the owner.** A consumer that sees an event for an unknown booking can park it
  and let the retry find the booking later, or fetch current state over gRPC.

## 7.6 Poison messages and replay

A **poison message** fails every time: a payload that does not parse, a bug in the handler, data
the handler never expected. Without a limit it blocks or spins forever. In ENR it ends up in the
parking lot by one of three roads:

| Road | Mechanism |
| --- | --- |
| Payload fails schema validation | Parked immediately; retrying cannot help |
| Handler keeps throwing | Retried through each delay tier, then parked |
| Handler crashes the process | Redelivered on reconnect until the quorum queue's delivery limit dead-letters it |

Every parking lot needs an owner, an alert on its depth, and a **replay procedure**. Fix the cause,
deploy, then move the messages back:

```ts
// apps/notification/src/scripts/replay-parking-lot.ts — run by hand after the fix is deployed
const channel = await connection.createConfirmChannel();
for (let replayed = 0; replayed < limit; replayed += 1) {
  const message = await channel.get(parkingLot(QUEUE), { noAck: false });
  if (message === false) break; // parking lot empty
  channel.publish(RETRY_EXCHANGE, QUEUE, message.content, {
    persistent: true,
    messageId: message.properties.messageId,
    contentType: message.properties.contentType,
    headers: { ...message.properties.headers, 'x-retry-attempt': 0 },
  });
  await channel.waitForConfirms();
  channel.ack(message);
}
```

Replay is safe because of the inbox: anything that had in fact been processed is skipped.

## Rules — reliability

### MS-351 — State change published without the outbox
**Severity:** critical · **Applies to:** every producing service
**Detect:** `ClientProxy.emit(...)` or `channel.publish(...)` in a use case, controller or
repository; an outbox insert outside the transaction that changes state; publishing inside the
transaction callback. Grep hint: `\.emit\(` outside `libs/messaging`.

```ts
// Bad — a crash between the two lines loses the event
await this.unitOfWork.run((tx) => this.bookings.markConfirmed(tx, booking));
this.events.emit('booking.confirmed', payload);

// Good — one transaction; the relay publishes
await this.unitOfWork.run(async (tx) => {
  await this.bookings.markConfirmed(tx, booking);
  await this.outbox.add(tx, bookingConfirmedEvent(booking, currentContext()));
});
```

**Why:** The dual write fails silently and partially — the worst way to fail (`CC-807` covered the
in-process version).
**Not a violation when:** The publish is the relay itself, or the message is not a domain event
(retry copies, replay tooling).

### MS-352 — Outbox relay that holds a transaction across the publish, or cannot run twice
**Severity:** major · **Applies to:** `libs/messaging` relay
**Detect:** `emit`/`publish` awaited inside the claiming transaction; claims without
`FOR UPDATE SKIP LOCKED`; rows marked published before the publish settles; no lease, so a crashed
relay's rows are never retried.
**Why:** Held locks starve the pool (`CC-506`); without `SKIP LOCKED` two instances publish the same
rows in lockstep; without a lease, rows claimed by a dead relay are stuck forever.
**Not a violation when:** The relay is single-instance by deployment *and* publishes outside the
transaction.

### MS-353 — Consumer without an inbox
**Severity:** critical · **Applies to:** event handlers
**Detect:** A handler that writes to the database without recording `eventId` in the same
transaction; an inbox check in one transaction and the work in another. Pairs with `CC-808`, `JT-706`.
**Why:** At-least-once delivery guarantees duplicates; without an inbox they become double refunds
and double emails.
**Not a violation when:** The handler is naturally idempotent — an unconditional cache delete, or an
upsert guarded by version (`MS-152`).

### MS-354 — Consumer depends on delivery order
**Severity:** major · **Applies to:** event handlers, replicated data
**Detect:** A handler that assumes a previous event was processed ("booking must exist"), overwrites
replicated data without a version check, or decrements counters from events.
**Why:** Retries and competing consumers reorder events routinely; the bug appears only under load.
**Not a violation when:** The handler checks state and parks or retries when its precondition is
missing.

### MS-355 — External side effect without an idempotency key
**Severity:** critical · **Applies to:** payment adapters, notification senders, saga steps
**Detect:** `stripe.paymentIntents.create`, `stripe.refunds.create` without `{ idempotencyKey }`; a
random key generated per attempt; refunds created without first persisting a refund record.
**Why:** A retried handler or saga step charges or refunds twice. Stripe deduplicates only what you
give the same key.
**Not a violation when:** The call is a read (`retrieve`, `list`).

### MS-356 — Parking lot without an owner or a replay path
**Severity:** major · **Applies to:** each consuming service's README and runbook
**Detect:** No alert or dashboard on parking-lot depth; no replay script; replay done by publishing
hand-edited JSON.
**Why:** A parking lot nobody watches is a slower way to drop messages.
**Not a violation when:** Never.

---

# Part 8 — Sagas

## 8.1 A business transaction without a database transaction

Buying a ticket touches four services: catalog prices it, inventory holds the seats, payment takes
the money, booking issues the tickets. There is no `BEGIN … COMMIT` across four databases and
Stripe. A **saga** replaces the one big transaction with a sequence of local transactions, each
with a **compensation** that semantically undoes it if a later step fails.

Compensation is not rollback. A rollback makes it as if nothing happened. A compensation is a new
action visible to everyone: the hold existed for twelve minutes and was then released; the money was
taken and then refunded. Design the user-facing story of each compensation, not just the code.

## 8.2 Orchestration or choreography

| | Orchestration | Choreography |
| --- | --- | --- |
| Shape | One service owns the process and tells others what to do | Each service reacts to events and emits its own |
| Where the flow lives | One state machine you can read top to bottom | Spread across consumers |
| "Where is booking B stuck?" | One row tells you | Correlate events across services |
| Good for | Many steps, timeouts, compensations, nasty edge cases | Simple fan-out where each reaction is independent |
| ENR | **The booking saga** — booking orchestrates | **`trip.cancelled`** — each service reacts (Part 6.1) |

A trip cancelled by ops *after* confirmation is not a compensation of the booking saga. It is a new
business process (R9), choreographed through `trip.cancelled`.

Orchestration does not mean "booking calls everything synchronously". The orchestrator issues
commands (gRPC) and waits for outcomes (gRPC responses *or* events such as `payment.succeeded`),
persisting its state between each.

## 8.3 The ENR booking saga, step by step

```text
 app ──POST /bookings──► gateway ──gRPC──► booking   creates the booking (CREATED)
                                           booking ──QuoteFare──────────────► catalog     fare locked (R6)
                                           booking ──HoldSeats──────────────► inventory   both legs, 15 min
                                           booking ──CreatePaymentIntent────► payment ──► Stripe
 app ◄── review: seats, total, countdown, clientSecret ◄──────────────────── (via gateway)
 app ──confirm card + 3-D Secure with Stripe's SDK─────────────────────────────────────► Stripe
                                                       payment ◄──webhook──────────────── Stripe
                                                       payment: verify, dedupe event.id, outbox
                                           booking ◄═══ payment.succeeded ═══ payment
                                           booking ──ConfirmHold────────────► inventory   hold → sold
                                           booking: tickets + outbox booking.confirmed (one transaction)
                                           booking ═══ booking.confirmed ═══► notification (email, bell)
 app ◄── Payment Success; the tickets appear in the Tickets tab
```

| Step | Owner | How | Idempotency key | If it fails |
| --- | --- | --- | --- | --- |
| 1. Quote | catalog | gRPC, read | — (retry safe) | Booking `FAILED`; nothing to undo |
| 2. Hold | inventory | gRPC, write | `bookingId` | `SEATS_UNAVAILABLE` (409); nothing to undo |
| 3. PaymentIntent | payment → Stripe | gRPC, write | `payment-intent:{bookingId}` | Keep the hold; the user may retry until the deadline |
| 4. Customer pays | app ↔ Stripe | Stripe SDK, 3-D Secure | Stripe's | Card declined: stay in `AWAITING_PAYMENT`, user may retry |
| 5. Payment outcome | payment | Webhook → outbox → `payment.succeeded` | Stripe `event.id` in payment's inbox | Stripe redelivers webhooks; payment dedupes |
| 6. Confirm hold | inventory | gRPC, write | `holdId` | Transient: retry. `HOLD_EXPIRED`: late-payment path (8.7) |
| 7. Issue tickets | booking | Local transaction + outbox | — | Same transaction as `CONFIRMED`; cannot half-happen |

R3's 3-D Secure extension fits the same model: when payment reports that the PaymentIntent requires
customer action, booking asks inventory to extend the hold once by five minutes and moves its own
deadline with it.

## 8.4 Step 5 up close: Stripe's webhook

The customer pays inside the app, with Stripe's SDK, so the backend learns the outcome from Stripe's
**webhook**: an HTTPS `POST` signed with your endpoint's secret. Three rules shape the handler.

1. **Verify the signature against the raw bytes.** Stripe signs the exact body it sent. If Nest parses
   the JSON and you re-serialize it, the bytes change and verification fails. Ask Nest to keep the raw
   body (this needs the built-in body parser, so do not pass `bodyParser: false`).
2. **Return 2xx quickly.** Verify, record, acknowledge. The saga work happens afterwards, through
   the outbox — never inside the webhook request.
3. **Expect duplicates and any order.** Stripe can deliver the same event twice and does not
   guarantee order. Deduplicate on `event.id`; let the saga's state decide what an event means.

```ts
// apps/payment/src/main.ts
const app = await NestFactory.create(AppModule, { rawBody: true, bufferLogs: true });

// apps/payment/src/interface/http/stripe-webhook.controller.ts — reached directly, not via the gateway
@Controller('webhooks/stripe')
export class StripeWebhookController {
  constructor(
    @Inject(STRIPE_CLIENT) private readonly stripe: Stripe,
    private readonly config: ConfigService<PaymentEnv, true>,
    private readonly recordStripeEvent: RecordStripeEvent,
  ) {}

  @Post()
  @HttpCode(HttpStatus.OK)
  async receive(
    @Req() request: RawBodyRequest<Request>,
    @Headers('stripe-signature') signature: string | undefined,
  ): Promise<void> {
    const event = this.verify(request.rawBody, signature);
    await this.recordStripeEvent.execute(event); // one transaction: dedupe on event.id + outbox write
  }

  private verify(rawBody: Buffer | undefined, signature: string | undefined): Stripe.Event {
    if (!rawBody || !signature) throw new BadRequestException('STRIPE_SIGNATURE_MISSING');
    try {
      const secret = this.config.get('STRIPE_WEBHOOK_SECRET', { infer: true });
      return this.stripe.webhooks.constructEvent(rawBody, signature, secret); // default tolerance: 5 min
    } catch {
      throw new BadRequestException('STRIPE_SIGNATURE_INVALID');
    }
  }
}
```

`RawBodyRequest` comes from `@nestjs/common` and `Request` from `express`. The webhook bypasses the
gateway's JWT guard on purpose: Stripe authenticates with its signature, not with a user token. In
development, `stripe listen --forward-to` delivers to payment's HTTP port (Part 14.3).

Do not rely on redelivery alone. In test mode Stripe retries a failed delivery only three times over
a few hours (live mode: up to three days), and a forward missed by `stripe listen` while payment was
down may never come back. Before booking expires a saga in `AWAITING_PAYMENT`, it asks payment to
check the PaymentIntent's status with Stripe — the same reconciliation PLAT-4 runs on a schedule.

## 8.5 A persisted state machine

The saga's state is a column on the booking, not a variable in memory. If booking restarts in the
middle, the next instance reads the state and carries on.

```text
 happy path
   CREATED ─► QUOTED ─► SEATS_HELD ─► AWAITING_PAYMENT ─► PAID ─► CONFIRMED

 failure and recovery
   CREATED, QUOTED ─────────────── quote or hold failed ─────────► FAILED
   SEATS_HELD, AWAITING_PAYMENT ── deadline passed ──────────────► EXPIRED
   EXPIRED ─────────────────────── payment succeeded anyway ─────► LATE_PAYMENT
   PAID ────────────────────────── hold lost before confirming ──► LATE_PAYMENT
   LATE_PAYMENT ─ seats re-acquired ─► PAID
   LATE_PAYMENT ─ seats gone ─► REFUNDING ─ refund succeeded ─► REFUNDED
```

Keep the transition logic a **pure function** — no I/O, no clock, no framework — so every
(state, event) pair is one line of unit test:

```ts
// apps/booking/src/domain/booking-saga.ts
export type SagaState =
  | 'CREATED' | 'QUOTED' | 'SEATS_HELD' | 'AWAITING_PAYMENT' | 'PAID' | 'CONFIRMED'
  | 'FAILED' | 'EXPIRED' | 'LATE_PAYMENT' | 'REFUNDING' | 'REFUNDED';

export type SagaEvent =
  | 'QUOTE_SUCCEEDED' | 'QUOTE_FAILED' | 'HOLD_PLACED' | 'HOLD_FAILED' | 'INTENT_CREATED'
  | 'PAYMENT_SUCCEEDED' | 'HOLD_CONFIRMED' | 'HOLD_LOST' | 'DEADLINE_PASSED'
  | 'SEATS_REACQUIRED' | 'REFUND_SUCCEEDED';

const TRANSITIONS: Readonly<Record<SagaState, Partial<Record<SagaEvent, SagaState>>>> = {
  CREATED:          { QUOTE_SUCCEEDED: 'QUOTED', QUOTE_FAILED: 'FAILED' },
  QUOTED:           { HOLD_PLACED: 'SEATS_HELD', HOLD_FAILED: 'FAILED' },
  SEATS_HELD:       { INTENT_CREATED: 'AWAITING_PAYMENT', DEADLINE_PASSED: 'EXPIRED' },
  AWAITING_PAYMENT: { PAYMENT_SUCCEEDED: 'PAID', DEADLINE_PASSED: 'EXPIRED' },
  PAID:             { HOLD_CONFIRMED: 'CONFIRMED', HOLD_LOST: 'LATE_PAYMENT' },
  EXPIRED:          { PAYMENT_SUCCEEDED: 'LATE_PAYMENT' },
  LATE_PAYMENT:     { SEATS_REACQUIRED: 'PAID', HOLD_FAILED: 'REFUNDING' },
  REFUNDING:        { REFUND_SUCCEEDED: 'REFUNDED' },
  CONFIRMED: {}, FAILED: {}, REFUNDED: {},
};

/** Returns the next state, or undefined when the event does not apply in this state. */
export function nextState(current: SagaState, event: SagaEvent): SagaState | undefined {
  return TRANSITIONS[current][event];
}
```

`undefined` is a real answer, not an error: `PAYMENT_SUCCEEDED` arriving in `CONFIRMED` is a
duplicate that was already handled. The caller logs it and acks. What it must never do is guess.

Each step follows the same rhythm, so a crash at any point is recoverable:

1. In one transaction: lock the booking row (`SELECT … FOR UPDATE`), compute `nextState`, save the new
   state with `next_step_at` (when the sweeper should look again), and add any outbox events.
2. After commit, execute the command the new state calls for (`ConfirmHold`, `Refund`) over gRPC.
3. Feed the result back in as an event (`HOLD_CONFIRMED`, `HOLD_LOST`), starting again at 1.

If the process dies between 1 and 2, the state already says what to do next, and the sweeper does it.
That only works because every command is idempotent.

## 8.6 Timeouts and the sweeper

Every state that waits for something must have a deadline, or the saga can wait forever.

| Waiting in | For | Deadline | On deadline |
| --- | --- | --- | --- |
| `SEATS_HELD`, `AWAITING_PAYMENT` | The customer to pay | Hold expiry (15 min, +5 once for 3-D Secure) | Ask payment for the intent's real status (a webhook may be lost); if unpaid, cancel it; inventory releases the seats itself; `EXPIRED` |
| `PAID` | Inventory to confirm the hold | Short, with retries | Keep retrying `ConfirmHold`; `HOLD_EXPIRED` → `LATE_PAYMENT` |
| `REFUNDING` | Stripe's refund outcome | Hours | Alert ops; R10 says a failed refund must be visible |

A **sweeper** is a scheduled job that finds sagas whose `next_step_at` has passed and runs their
next step. Claim rows with `FOR UPDATE SKIP LOCKED` exactly like the outbox relay, so several booking
instances can sweep without stepping on each other. The sweeper is also how PLAT-4's "stuck saga"
report works: anything that has been in a waiting state far beyond its deadline is a bug to look at.

Note the two clocks. Inventory owns the hold and releases expired holds on its own — it does not wait
for booking. Booking owns the saga deadline. Both run from the same `expiresAt` that inventory
returned with the hold, so they agree on when "too late" is.

## 8.7 The nastiest case: payment succeeds after the hold expired

```text
 14:45:00  hold placed, expires 15:00:00
 14:59:58  customer completes 3-D Secure; Stripe captures the payment
 15:00:00  inventory releases the seats; booking's deadline fires: cancel PaymentIntent → Stripe
           refuses, the intent already succeeded
 15:00:03  payment.succeeded arrives; booking is EXPIRED → LATE_PAYMENT
```

Booking now holds the customer's money and no seats. Two ways out, decided in this order:

1. **Re-acquire.** Ask inventory to hold and confirm the same allocation (or an equivalent one that
   still satisfies R2's grouping) for the same booking. If it succeeds, continue as `PAID` → `CONFIRMED`.
   The customer never notices.
2. **Refund automatically.** If the seats are gone, refund the full amount **including the booking
   fee** (the failure was ours, not theirs), move to `REFUNDING`, and tell the customer in their
   language what happened and when the refund will arrive.

The same path handles a slower variant: payment succeeded in time, but `ConfirmHold` kept failing
until the hold expired. Inventory answers `HOLD_EXPIRED`, the saga receives `HOLD_LOST`, and it
moves to `LATE_PAYMENT`. One mechanism, two causes — fewer special cases to test.

## Rules — sagas

### MS-401 — Cross-service process without a persisted state machine
**Severity:** critical · **Applies to:** booking saga, any multi-step cross-service flow
**Detect:** A use case that calls several services in sequence and keeps progress only in local
variables; saga state not persisted between steps; no single row that answers "where is booking B?".
**Why:** A crash or deploy mid-flow leaves money taken and seats held with nothing that knows to
finish or undo the work.
**Not a violation when:** The flow has one remote step and nothing to compensate.

### MS-402 — Saga step that is not idempotent
**Severity:** critical · **Applies to:** saga commands and the services that receive them
**Detect:** A command without a deterministic key (`bookingId`, `holdId`, `refundId`); a server that
creates a second hold or intent for a repeated key; keys generated randomly per attempt.
**Why:** Recovery repeats steps by design — after a crash, a timeout or a sweeper pass.
**Not a violation when:** The step is a pure read.

### MS-403 — Waiting state without a deadline
**Severity:** critical · **Applies to:** saga state machine, sweeper
**Detect:** A state with no `next_step_at`/deadline; no sweeper; a deadline kept in an in-memory
`setTimeout`.
**Why:** Lost webhooks and crashed instances leave sagas waiting forever, with seats held and money in
limbo.
**Not a violation when:** The state is terminal (`CONFIRMED`, `FAILED`, `REFUNDED`).

### MS-404 — Step without a defined compensation
**Severity:** critical · **Applies to:** saga design document and code
**Detect:** A step after which a later failure is possible, with no compensation written down and
implemented (for example: money taken, seats lost, no automatic refund).
**Why:** The undefined path is the one that happens during the demo.
**Not a violation when:** Nothing before the step has a lasting effect.

### MS-405 — Late or out-of-order outcome not handled
**Severity:** critical · **Applies to:** saga transitions
**Detect:** No transition for `PAYMENT_SUCCEEDED` in `EXPIRED`; `HOLD_EXPIRED` from `ConfirmHold`
treated as a generic error; duplicates of terminal events throwing instead of being ignored.
**Why:** Late payments and duplicate webhooks are normal with Stripe and RabbitMQ, not exotic.
**Not a violation when:** Never.

### MS-406 — Transition logic tangled with I/O, or untested
**Severity:** major · **Applies to:** `apps/booking/src/domain/**`
**Detect:** State changes decided inside controllers, consumers or repositories; transitions without a
table-driven unit test per (state, event) pair.
**Why:** The saga's correctness is a property of its transition table; it should be provable without
starting RabbitMQ.
**Not a violation when:** Never.

---

# Part 9 — Data across services

## 9.1 Database per service

Each service owns one database. Nobody else connects to it — not for a quick read, not for a report,
not "just in development". The database is the service's private implementation detail; its gRPC
contract and its events are its public interface.

In ENR's Compose stack all six databases live in **one Postgres container** to save memory, each
with its own role and password, and with `CONNECT` revoked from everyone else (Part 14.2). The
container is shared; the data is not. In production they would be separate instances, and nothing in
your code should need to change when that happens.

What you give up, and what replaces it:

| Monolith habit | Replacement |
| --- | --- |
| `JOIN` across modules | Local copy fed by events, or one batch call (9.5) |
| Foreign key to another module's table | An id plus a validation call or event at write time |
| One transaction across modules | A saga (Part 8) |
| Ad-hoc SQL report across everything | A read model built from events, owned by one service |

## 9.2 Migrations per service

Each service keeps its migrations in its own folder, runs them against its own database, and records
them in its own migrations table. `synchronize: true` stays off everywhere — it rewrites tables from
entity metadata and has deleted more columns than any junior developer.

Because services deploy independently, a schema change must work with **both** the old and the new
version of the code for a while. Use **expand and contract**:

```text
 1 expand    add the new nullable column / table          old code ignores it, new code may write it
 2 migrate   backfill; deploy code that writes both       both versions run side by side
 3 switch    deploy code that reads only the new shape
 4 contract  drop the old column in a later release       nothing reads it any more
```

## 9.3 Replicating reference data through events

Booking prints station names on tickets, in emails, and in the Tickets tab — in English and Arabic.
Calling catalog for every ticket is chatty (`MS-105`) and makes the Tickets tab fail whenever catalog
does. Instead, booking keeps a **read-only local copy** of the station data it needs, kept fresh by
catalog's events:

```ts
// apps/booking/src/interface/events/station-events.controller.ts
@EventPattern('station.updated', Transport.RMQ)
async onStationUpdated(
  @Payload() event: EventEnvelope<unknown>,
  @Ctx() context: RmqContext,
): Promise<void> {
  await this.consumer.handle(context, event, stationUpdatedV1, (station, tx) =>
    tx.query(
      `INSERT INTO station_copy (station_id, name_en, name_ar, version)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (station_id) DO UPDATE
         SET name_en = EXCLUDED.name_en, name_ar = EXCLUDED.name_ar, version = EXCLUDED.version
         WHERE station_copy.version < EXCLUDED.version`, // newer wins (Part 7.5)
      [station.stationId, station.nameEn, station.nameAr, station.version],
    ),
  );
}
```

Three properties make a copy trustworthy:

- **Read-only.** Only the event handler writes it. Booking never "fixes" a station name locally.
- **Ordered by version.** The `WHERE … version <` guard makes late or duplicate events harmless.
- **Bootstrapped.** A new consumer starts with an empty copy. Load a snapshot once (a paginated
  `ListStations` call, or a seed), then let events keep it current. "Snapshot, then subscribe."

## 9.4 Read models: CQRS-lite

Sometimes a service needs a shape of data that no single owner has. The results screen shows trains
(catalog's timetable) with "Available Seats: 170" (inventory's occupancy). Catalog builds a **read
model**: it stores the counts inventory publishes, next to its own timetable, in the form search
needs — a table or a Redis hash.

That read model is **eventually consistent**. Between a seat being held and catalog consuming the
event, the count is off by one. That is fine for *display*, and it is exactly why the read model
must never be used to *decide*:

> **The cache displays, the database decides.** The results screen may say 170. When the customer
> presses Continue, inventory checks its own tables, under its own locks, and may say the seats are
> gone. The 409 is correct; the 170 was a hint.

## 9.5 No distributed transactions, and how to "join"

Two-phase commit across six databases, RabbitMQ and Stripe is not available in this stack, and where
it exists it trades availability for consistency: one slow participant blocks everyone. ENR uses the
tools from Parts 7 and 8 instead: local transactions, the outbox, idempotent consumers and sagas. The
corollary: **never hold a database transaction open across a gRPC call, a publish, or a Stripe call.**

When you need data from two services together:

| Option | Use when | ENR example |
| --- | --- | --- |
| **Batch call** to the owner | You need fresh data, rarely, for a bounded set | Gateway asks catalog for 6 stations at once |
| **Local copy** fed by events | You need it often, and slightly stale is acceptable | Booking's `station_copy` |
| **Snapshot into your record** | The value must not change after the fact | The fare locked at hold time; the passenger name on a ticket |
| **Compose at the edge** | Purely presentational, independent parts | Gateway's Home screen (Part 4.4) |

The snapshot row deserves emphasis. A ticket is a historical record: it shows what was sold — the
passenger's name and National ID as entered, the fare that was charged, the stations as named that
day. If you store only ids and look them up later, history silently rewrites itself.

## 9.6 Money, end to end

All money in ENR is **integer piastres** (R6), at every layer, with no exceptions:

| Layer | Representation | Trap |
| --- | --- | --- |
| PostgreSQL | `integer` column | `numeric`/`decimal`, `bigint` and `SUM(...)` come back from `pg` as **strings** — the DOCURA bug (`CC-806`) |
| TypeORM entity | `farePiastres: number` with `type: 'integer'` | Do not let a `bigint` sneak in without a transformer |
| `.proto` | `int64 fare_piastres` | Load with `longs: Number`, or it arrives as a `Long` object or a string |
| Event payload | `"farePiastres": 15500` | Never `155.00`, never `"155"` |
| Stripe | `amount: 15500, currency: 'egp'` | EGP is a two-decimal currency: Stripe also counts piastres |
| UI | Formatted at the edge only | — |

Rounding happens once, in the service that owns the rule, with integer arithmetic:

```ts
// 7% booking fee, rounded half-up to the piastre (R6) — integers in, integer out
export function bookingFeePiastres(totalFarePiastres: number): number {
  return Math.floor((totalFarePiastres * 7 + 50) / 100);
}
```

A 65 EGP fare is 6,500 piastres; the fee is 455 piastres — the "4.55 EGP" on the Payment screen.

## Rules — data

### MS-151 — Schema changed outside the owning service's migrations
**Severity:** major · **Applies to:** `apps/*/src/**/migrations`, ORM config
**Detect:** `synchronize: true`; migrations of one service touching another's database; a breaking
schema change (drop, rename, `NOT NULL` without default) in the same release as the code change.
**Why:** Independent deploys need schemas that work with two code versions at once.
**Not a violation when:** The table is brand new and nothing reads it yet.

### MS-152 — Replicated data written without an ordering guard
**Severity:** major · **Applies to:** local copies and read models
**Detect:** A copy updated with a plain `UPDATE`/`save()` from an event, with no version or timestamp
comparison; application code other than the event handler writing to the copy.
**Why:** Events arrive late, twice and out of order; without a guard the copy drifts from the owner.
**Not a violation when:** The copy is rebuilt wholesale from a snapshot.

### MS-153 — Transaction held across a remote call
**Severity:** critical · **Applies to:** use cases, repositories, saga steps
**Detect:** `await` on a gRPC client, `ClientProxy`, Stripe or SMTP inside a `transaction(...)`
callback; an attempt at two-phase commit across services. Pairs with `CC-506`.
**Why:** The remote call's latency becomes lock time; when it hangs, the pool drains and unrelated
requests fail.
**Not a violation when:** The call is to the service's own Redis with a short command timeout.

### MS-154 — Money not in integer piastres end to end
**Severity:** critical · **Applies to:** entities, migrations, `.proto`, event schemas, Stripe calls
**Detect:** `float`, `double`, `decimal`, `numeric` money columns; `double`/`string` money fields in
`.proto`; pounds with decimals in events; money fields without a unit in the name.
Grep hint: `type:\s*['"](decimal|numeric|float|double)['"]|double\s+\w*(fare|fee|amount|price)`
**Why:** Floating point and decimal-as-string bugs moved from a module boundary to a network
boundary are harder to find and touch real refunds.
**Not a violation when:** The value is a ratio, not an amount (the 7% fee rate).

### MS-155 — Historical record references mutable data instead of a snapshot
**Severity:** major · **Applies to:** tickets, bookings, refunds
**Detect:** A ticket that stores only `passengerId`/`stationId` and renders names by looking them up;
a booking that recomputes its fare from catalog after the hold.
**Why:** Editing a saved passenger or a fare table must not rewrite tickets already sold (R6 locks
the price at hold time for the same reason).
**Not a violation when:** The referenced data is immutable by rule.

---

# Part 10 — Caching with Redis

## 10.1 Two different jobs for one Redis

ENR uses Redis for two jobs that look alike and behave very differently when Redis fails:

| Job | Examples in ENR | If Redis loses the data |
| --- | --- | --- |
| **Cache** — a disposable copy of data whose truth lives in Postgres | Stations, trip stops, search results, availability counts, unread counts | Slower, then refilled from the source |
| **Store** — Redis *is* the system of record, for short-lived data | Rate-limit counters, logout denylist, idempotency responses, OTP attempt counters | The data is gone; you must decide what that means |

Write down which job each key does. Everything in this part about invalidation and staleness applies
to the cache job; the store job needs the failure decisions in 10.7.

## 10.2 Patterns

| Pattern | How it works | Use in ENR |
| --- | --- | --- |
| **Cache-aside** | Read the cache; on a miss, load from the source and store with a TTL | The default for every cache in ENR |
| Read-through | A cache library loads on a miss for you | Same idea; fine if you still control keys, TTLs and failures |
| Write-through | Every write updates the source and the cache together | Rarely worth it: two writes, two failure modes |
| Write-behind | Write to the cache; flush to the database later | **Never** for anything that matters — the cache becomes the source of truth |

Cache-aside plus **event-driven invalidation** (10.6) plus a **TTL as a safety net** is the
combination ENR uses.

## 10.3 Key design

```text
 <service>:<what>:v<shape>:<identifiers>

 catalog:stations:v2:all                          shared — identical for every user
 catalog:trip-stops:v1:trip_916_20261024          shared
 catalog:search:v3:2026-10-24:g17:ALX:CAI:FIRST   shared — g17 is the date's generation (10.6)
 notification:unread:v1:user:8c1f…                per user — the user id is in the key
 catalog:recent-searches:v1:user:8c1f…            per user
```

- **Namespace by owning service.** Catalog owns `catalog:*`. No other service reads or writes those
  keys — the same rule as database ownership, applied to Redis.
- **Version the shape.** When the cached object's structure changes, bump `v2` → `v3`. Old and new
  code can then run side by side during a deploy without reading each other's entries; the old keys
  expire on their own.
- **Per-user data only under per-user keys.** Search results are the same for everyone, so they are
  shared. Recent searches and unread counts belong to one user and carry that user's id. A shared key
  containing anything user-specific is a data leak waiting for its first cache hit.
- **No personal data in key names.** A user id, yes; an email or National ID, never — keys show up in
  monitoring tools and logs.

## 10.4 TTLs and jitter

Every cache entry gets a TTL, even when events invalidate it: the TTL is what heals the cache when an
invalidation event is lost. Long TTLs (hours) for data that changes rarely and is invalidated by
events — stations, trip stops. Short TTLs (tens of seconds) for data that changes often — search
results that include availability.

Add **jitter**. If 5,000 search keys are written during the morning peak with the same 60-second
TTL, they expire in the same second and the database takes the whole refill at once.

```ts
export function withJitter(ttlSeconds: number): number {
  return Math.round(ttlSeconds * (0.9 + Math.random() * 0.2)); // ±10 %
}
```

## 10.5 Stampede protection

A **stampede** happens when a hot key expires and every concurrent request misses at once, all
running the same expensive query. Two layers of protection:

- **Single-flight inside one instance**: concurrent misses for the same key share one load.
- **A short lock across instances** for expensive loads: one instance loads, the others wait briefly
  and re-read.

```ts
// libs/cache/src/cached-query.ts — one instance per cache, so the in-flight map is typed
export class CachedQuery<T> {
  private readonly inFlight = new Map<string, Promise<T>>();

  constructor(
    private readonly redis: Redis,
    private readonly options: { name: string; ttlSeconds: number; schema: ZodType<T> },
    private readonly metrics: CacheMetrics,
  ) {}

  async get(key: string, load: () => Promise<T>): Promise<T> {
    const cached = await this.readSoft(key);
    this.metrics.record(this.options.name, cached === undefined ? 'miss' : 'hit');
    if (cached !== undefined) return cached;

    const pending = this.inFlight.get(key); // single-flight: share a load already in progress
    if (pending) return pending;
    const loading = this.loadOnce(key, load);
    this.inFlight.set(key, loading);
    return loading;
  }

  private async loadOnce(key: string, load: () => Promise<T>): Promise<T> {
    try {
      const value = await load();
      await this.writeSoft(key, value);
      return value;
    } finally {
      this.inFlight.delete(key);
    }
  }

  private async readSoft(key: string): Promise<T | undefined> {
    try {
      const raw = await this.redis.get(key);
      if (raw === null) return undefined;
      const parsed = this.options.schema.safeParse(JSON.parse(raw)); // an old deploy may have written it
      return parsed.success ? parsed.data : undefined;
    } catch {
      this.metrics.record(this.options.name, 'error'); // Redis down or slow: behave like a miss
      return undefined;
    }
  }

  private async writeSoft(key: string, value: T): Promise<void> {
    try {
      await this.redis.set(key, JSON.stringify(value), 'EX', withJitter(this.options.ttlSeconds));
    } catch {
      this.metrics.record(this.options.name, 'error');
    }
  }
}
```

For loads expensive enough to need the cross-instance lock, wrap the load in a lock that only its
owner can release:

```ts
const RELEASE_IF_OWNER =
  "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

const token = randomUUID();
const isLeader = (await this.redis.set(`${key}:lock`, token, 'PX', 5_000, 'NX')) === 'OK';
if (!isLeader) {
  await sleep(100);                                 // follower: give the leader a moment…
  return (await this.readSoft(key)) ?? load();      // …then read its result, or load once yourself
}
try {
  return await this.loadOnce(key, load);
} finally {
  await this.redis.eval(RELEASE_IF_OWNER, 1, `${key}:lock`, token);
}
```

The lock has a TTL so a crashed leader cannot block the key forever, and the release checks the token
so a slow leader cannot delete a lock that has already passed to someone else.

## 10.6 Invalidation driven by events

The owner of a cache invalidates it, in response to events — including its own:

| Event | Catalog does |
| --- | --- |
| `station.updated` | `DEL catalog:stations:v2:all` |
| Timetable published or changed for a date; fare changed; trip cancelled | Bump that date's search generation |
| Inventory's availability events | Update the counts in the availability read model |

Search results are keyed by date, stations and class, so "every search touching trip 916 on 24 Oct"
is hard to enumerate. A **generation counter** per date avoids enumerating:

```ts
const generation = (await this.redis.get(`catalog:search-gen:v1:${date}`)) ?? '0';
const key = `catalog:search:v3:${date}:g${generation}:${from}:${to}:${seatClass}`;

// on any change affecting that date: every old key becomes unreachable and expires by TTL
await this.redis.incr(`catalog:search-gen:v1:${date}`);
```

Invalidating from an event handler (rather than right after the database commit) also covers the case
where the process died between commit and invalidation: the event is in the outbox and will arrive.

## 10.7 When Redis is down

"Redis down" must mean **slower, not broken**. Configure the client to fail fast instead of queuing
commands behind a dead connection:

```ts
new Redis(config.get('REDIS_URL', { infer: true }), {
  maxRetriesPerRequest: 1,   // default 20: each command would stall while ioredis reconnects
  enableOfflineQueue: false, // fail immediately while disconnected instead of buffering
  commandTimeout: 200,       // ms; there is no default
  connectTimeout: 2_000,
});
```

Then decide, per use, what failure means — and write it in the service README:

| Use | Redis down means | Reason |
| --- | --- | --- |
| Stations, stops, search results | Read from Postgres (cache-aside already does this) | Slower is acceptable |
| Availability counts on results | Hide the count; still show the trains | A hint, not a decision |
| Gateway rate limits | Fall back to a per-instance in-memory limiter | Better than no limit; temporary |
| Logout denylist | Refuse money-moving routes; allow reads until the access token expires | Bounded risk: tokens are short-lived |
| OTP attempt counters | Refuse OTP verification with a retry-later code | Fail closed: unlimited attempts would allow brute force |
| Unread badge | Count from notification's database, or hide the badge | Display only |

## 10.8 Measure it

A cache you cannot measure is a guess. Record every lookup as hit, miss or error, labelled with the
cache's name, using the OpenTelemetry metrics API:

```ts
const lookups = metrics.getMeter('enr-cache').createCounter('cache.lookups', {
  description: 'Cache lookups by cache name and result',
});
lookups.add(1, { cache: 'search', result: 'hit' });
```

A hit ratio near zero means the keys are too specific or the TTL too short; a sudden rise in `error`
means Redis is in trouble before anyone reports slowness.

## Rules — caching

### MS-501 — Decision taken from a cached value
**Severity:** critical · **Applies to:** inventory, booking, any write path
**Detect:** A hold, payment or refund path that reads a cached count, price or status to decide;
`if (cachedAvailable >= passengers)` before calling inventory; a refund computed from a cached fare.

```ts
// Bad — the cache decides
const available = await this.availabilityCache.get(tripId, seatClass);
if (available < passengers.length) throw new DomainError('SEATS_UNAVAILABLE');

// Good — the owner decides, under its own locks; the cache only ever displayed
const hold = await this.seatHolds.hold({ bookingId, legs, passengers });
```

**Why:** "The cache displays, the database decides." A stale count oversells seats or refuses
customers who could have booked.
**Not a violation when:** The value is used only for display or ranking.

### MS-502 — Redis failure fails the request
**Severity:** major · **Applies to:** cache helpers, Redis client config
**Detect:** Unhandled Redis errors on cache reads/writes; default `maxRetriesPerRequest`,
`enableOfflineQueue` and no `commandTimeout`; no documented decision for store-type keys.
**Why:** A cache outage should cost latency, not availability; and for store-type keys someone must
decide on purpose whether to fail open or closed.
**Not a violation when:** The key is a store-type key whose documented decision is to fail closed.

### MS-503 — Unversioned key, or a key outside the owner's namespace
**Severity:** major · **Applies to:** every `redis.get/set/del` call
**Detect:** Keys without a service prefix and a `v<N>` segment; a service touching another service's
prefix; cached JSON read without schema validation.
**Why:** Mixed-version deploys read each other's entries; cross-service keys are a shared database.
**Not a violation when:** The key is a lock or counter that never stores a structured value.

### MS-504 — Per-user data under a shared key
**Severity:** critical · **Applies to:** cache keys and the values stored under them
**Detect:** Shared keys (no user id) whose value includes user-specific fields — saved passengers,
prices with personal discounts, "your booking" flags; user-specific data cached under a route-only key.
**Why:** The first user to fill the key serves their data to everyone after them.
**Not a violation when:** Never.

### MS-505 — No TTL, no jitter, or no stampede protection on a hot key
**Severity:** major · **Applies to:** cache writes
**Detect:** `SET` without `EX`/`PX`; identical TTLs for keys written in bursts; expensive loads with
neither single-flight nor a lock.
**Why:** Entries without a TTL never heal; synchronized expiry and stampedes move load spikes onto
Postgres at the worst moment.
**Not a violation when:** The entry is cheap to compute and rarely read.

### MS-506 — Cache invalidated by someone other than its owner, or not at all
**Severity:** major · **Applies to:** event handlers, use cases
**Detect:** Service A deleting service B's keys; a cached entity with no invalidation path except its
TTL while the UI expects changes to show immediately.
**Why:** Invalidation is part of the owner's contract with its own data.
**Not a violation when:** The documented freshness requirement is satisfied by the TTL alone.

### MS-507 — Cache without hit/miss metrics
**Severity:** minor · **Applies to:** cache helpers
**Detect:** A cache with no hit, miss and error counters labelled by cache name.
**Why:** Without numbers you cannot tell a useful cache from an expensive no-op.
**Not a violation when:** Never.

---

# Part 11 — Observability

## 11.1 The questions you will be asked

During the sprint 5 drill your mentor will kill a service mid-saga and ask questions like these:

- "Ahmed says he paid but has no ticket. Where is his booking?"
- "Why did Mona get two cancellation emails?"
- "Search got slow at 10:40. Which call was slow?"
- "Is anything stuck right now?"

In a monolith you would open one log file. Here the answer is spread over seven processes and a
broker, and you need three tools to put it back together: a **correlation ID** in every log line,
**traces** that show the path and timing of one request, and **health and backlog signals** that
tell you something is wrong before a user does.

## 11.2 One correlation ID, every hop

The gateway mints a correlation ID for each incoming request. Every hop after that carries it, and
every log line includes it. Then "show me everything about this click" is one search.

| Hop | Carrier |
| --- | --- |
| App → gateway | Generated at the gateway; returned in the `x-correlation-id` response header |
| Gateway → service (gRPC) | `x-correlation-id` metadata, set by a client interceptor |
| Service → outbox → RabbitMQ | `correlationId` in the envelope, plus an `x-correlation-id` message header |
| Consumer | Restored from the envelope; `causationId` becomes the consumed `eventId` |
| Every log line | Added by the logger's `mixin` |

Inside a process, keep it in `AsyncLocalStorage`, so you never thread it through function arguments:

```ts
// libs/observability/src/request-context.ts
import { AsyncLocalStorage } from 'node:async_hooks';

export interface RequestContext {
  readonly correlationId: string;
  readonly causationId: string;
}

const storage = new AsyncLocalStorage<RequestContext>();

export const runWithContext = <T>(context: RequestContext, work: () => T): T => storage.run(context, work);
export const currentContext = (): RequestContext | undefined => storage.getStore();

// Takes the two envelope fields it needs, so this library does not depend on libs/messaging
export const withEventContext = <T>(event: { correlationId: string; eventId: string }, work: () => T): T =>
  runWithContext({ correlationId: event.correlationId, causationId: event.eventId }, work);
```

Outgoing gRPC calls pick it up in `withCorrelationId()`, a client interceptor with the same shape as
the deadline interceptor in Part 5.4: its requester's `start(metadata, listener, next)` sets
`x-correlation-id` from `currentContext()` before calling `next`. Incoming gRPC calls restore it in a
Nest interceptor. With `inheritAppConfig: true` that interceptor also runs for RabbitMQ handlers,
whose RPC context is an `RmqContext` rather than `Metadata`, so it checks first; event handlers get
their context from the envelope instead:

```ts
@Injectable()
export class GrpcContextInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const rpcContext: unknown = context.getType() === 'rpc' ? context.switchToRpc().getContext() : null;
    if (!(rpcContext instanceof Metadata)) return next.handle(); // HTTP, or an RmqContext
    const [incoming] = rpcContext.get('x-correlation-id');
    const correlationId = typeof incoming === 'string' ? incoming : randomUUID();
    const requestContext = { correlationId, causationId: correlationId };
    return new Observable((subscriber) =>
      runWithContext(requestContext, () => next.handle().subscribe(subscriber)),
    );
  }
}
```

## 11.3 Structured logs without secrets

Logs are JSON, one object per line, so you can filter by field instead of grepping prose. Use
`nestjs-pino` (create the app with `bufferLogs: true` and call `app.useLogger(app.get(Logger))`, as
in Part 3.3), add the correlation and trace IDs to every line, and redact what must never be written:

```ts
LoggerModule.forRoot({
  pinoHttp: {
    level: 'info',
    mixin: () => ({
      service: 'booking',
      correlationId: currentContext()?.correlationId,
      traceId: trace.getActiveSpan()?.spanContext().traceId,
    }),
    redact: {
      paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.pin',
              '*.otp', '*.nationalId', '*.cardNumber', '*.clientSecret'],
      censor: '[redacted]',
    },
  },
}),
```

Redaction is the safety net, not the plan. The plan is to **log facts, not payloads**:

| Log | Example fields |
| --- | --- |
| Each request (pino-http does it) | method, route, status, duration, correlationId |
| Each event handled | `eventId`, `type`, outcome (`processed`, `duplicate`, `retried`, `parked`), duration |
| Each saga transition | `bookingId`, from, to, event |
| Each external call that fails | dependency, operation, gRPC/HTTP status, attempt |

Never log request bodies, event payloads, tokens, OTP codes, PINs, National IDs, card data, Stripe
client secrets, or full phone numbers (`CC-804`). Most of those are personal data, protected by
Egypt's data protection law as well as by your reviewer.

## 11.4 Tracing with OpenTelemetry and Jaeger

A **trace** is the tree of everything one request caused; each node is a **span** with a start, a
duration and attributes. OpenTelemetry's Node auto-instrumentation creates spans for HTTP, Express,
NestJS, gRPC, `amqplib`, `pg` and `ioredis`, and propagates the W3C `traceparent` through HTTP
headers, gRPC metadata and AMQP message headers for you. Jaeger stores and displays the traces.

The SDK must start **before** any instrumented module is loaded — hence the first line of every
`main.ts` (Part 3.3):

```ts
// apps/booking/src/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

// Service name and endpoint come from OTEL_SERVICE_NAME and OTEL_EXPORTER_OTLP_ENDPOINT (Part 14)
export const tracing = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
});
tracing.start();
```

Call `await tracing.shutdown()` from an `onApplicationShutdown` hook so the last spans are flushed.

One place automatic propagation cannot reach: **the outbox**. The relay publishes later, from its own
timer, where the original request's trace is no longer active. Capture the trace context when you
write the outbox row, and restore it around the publish (Part 7.2 shows the restore):

```ts
// libs/messaging/src/outbox-store.ts — inside add(), in the business transaction
const traceContext: Record<string, string> = {};
propagation.inject(context.active(), traceContext); // writes traceparent (and tracestate)
await tx.query('INSERT INTO outbox (event_id, envelope, trace_context) VALUES ($1, $2, $3)',
  [envelope.eventId, envelope, traceContext]);
```

Open Jaeger at <http://localhost:16686> after a booking and check that one trace contains the
gateway request, booking's gRPC calls, the publish, and notification's consume. If the consumer
starts a new trace instead, the context was lost on the way.

## 11.5 Health: liveness and readiness

Two different questions, two endpoints:

| Probe | Question | Checks | If it fails |
| --- | --- | --- | --- |
| `GET /health/live` | Is the process able to run at all? | Nothing external | Restart the container |
| `GET /health/ready` | Should this instance receive work? | Its **own** database and RabbitMQ | Stop routing to it; do not restart |

Liveness that checks the database turns a database outage into a restart storm. Readiness that
checks *other services* turns one service's outage into all of them being "unready". And Redis is not
in readiness at all: by design (Part 10.7) the service works without it.

```ts
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly database: TypeOrmHealthIndicator,
    private readonly microservice: MicroserviceHealthIndicator,
    private readonly config: ConfigService<BookingEnv, true>,
  ) {}

  @Get('live')
  live(): { status: 'ok' } {
    return { status: 'ok' };
  }

  @Get('ready')
  @HealthCheck()
  ready(): Promise<HealthCheckResult> {
    return this.health.check([
      () => this.database.pingCheck('database', { timeout: 1_000 }),
      () => this.microservice.pingCheck('rabbitmq', {
        transport: Transport.RMQ,
        options: { urls: [this.config.get('RABBITMQ_URL', { infer: true })] },
      }),
    ]);
  }
}
```

`TerminusModule` provides both indicators. For checks it does not ship, implement a small indicator
with `HealthIndicatorService.check(key)` and its `up()` / `down()` results.

## 11.6 A few metrics worth having

You do not need a dashboard for everything. You need the numbers that answer "is anything stuck?":

| Metric | Where it comes from | Worry when |
| --- | --- | --- |
| Request rate, error rate, latency per route and RPC | Traces / OpenTelemetry metrics | Errors or p95 latency jump |
| Queue depth and unacked count per queue | RabbitMQ management UI (<http://localhost:15672>) | Depth grows for minutes |
| Parking-lot depth per service | RabbitMQ management UI | Anything above zero |
| Outbox backlog and age of the oldest unpublished row | `SELECT count(*), min(created_at) … WHERE published_at IS NULL` | Age above a minute |
| Sagas past their deadline | `SELECT count(*) … WHERE next_step_at < now() - interval '5 minutes'` | Anything above zero |
| Cache hit ratio and Redis errors | Part 10.8 counters | Ratio collapses; errors appear |

Jaeger shows traces only. To see OpenTelemetry metrics locally, add the
`@opentelemetry/exporter-prometheus` exporter or log the values periodically; the stack does not
mandate a metrics backend, but it does mandate that these numbers exist.

## Rules — observability

### MS-601 — Correlation ID lost at a hop
**Severity:** major · **Applies to:** gateway middleware, gRPC interceptors, outbox writes, consumers
**Detect:** gRPC clients without the correlation interceptor; envelopes with a fresh `correlationId`
instead of the current one; consumers that do not restore context; log lines without
`correlationId`.
**Why:** One missing hop breaks every investigation that crosses it.
**Not a violation when:** The work genuinely starts there (a scheduled job mints its own ID).

### MS-602 — Unstructured log, or a secret or personal data in a log line
**Severity:** critical · **Applies to:** all logging
**Detect:** `console.log`; string-interpolated messages carrying data; logged request bodies, event
payloads, tokens, OTPs, PINs, National IDs, card data or Stripe client secrets; no `redact` config.
Pairs with `CC-804`, `CC-605`.
**Why:** Logs are copied, shipped and kept far longer than the data they contain deserves.
**Not a violation when:** The value is an opaque identifier (`bookingId`, `eventId`, `userId`).

### MS-603 — Service not traced, or trace broken at the outbox
**Severity:** major · **Applies to:** `tracing.ts`, `main.ts`, outbox store and relay
**Detect:** No `tracing.ts`, or imported after other modules; no service name; outbox rows without
trace context; the relay publishing outside the restored context.
**Why:** Without traces "which call was slow?" becomes guesswork across seven log streams.
**Not a violation when:** Never, from sprint 5 onwards (PLAT-5).

### MS-604 — Liveness checks dependencies, or readiness checks the wrong ones
**Severity:** major · **Applies to:** health controllers, Compose healthchecks
**Detect:** Database or broker checks in liveness; readiness that calls other services or Redis; a
single `/health` used for both.
**Why:** Wrong probes turn a dependency outage into restarts or into every service going unready.
**Not a violation when:** Never.

### MS-605 — Backlogs invisible
**Severity:** major · **Applies to:** each service's README and runbook
**Detect:** No documented way (query, UI, metric) to see outbox backlog, parking-lot depth and
sagas past their deadline.
**Why:** These three numbers are how you learn that something is stuck before a customer tells you.
**Not a violation when:** The service neither publishes nor consumes events and runs no saga.

---

# Part 12 — Resilience

## 12.1 Design for the failure, not around it

Every dependency of every service will be down at some point during the five sprints — usually during
a demo. Resilience is not making failures impossible; it is deciding in advance what each failure
looks like to the user, and keeping one failure from becoming many.

Four tools do most of the work: **timeouts** stop you waiting forever, **retries** survive blips,
**circuit breakers** stop you hammering something that is down, and **bulkheads** stop one failure
from using up resources everything else needs. Graceful shutdown makes your own restarts a non-event.

## 12.2 Timeouts on every network call

| Call | Where the timeout lives | Default if you do nothing |
| --- | --- | --- |
| gRPC | Deadline interceptor per client (Part 5.4) | None — waits forever |
| Stripe | `new Stripe(key, { timeout })` | 80 seconds |
| Redis | `commandTimeout` on the ioredis client (Part 10.7) | None |
| PostgreSQL | `statement_timeout` in the driver options (TypeORM `extra`) | None |
| RabbitMQ publish | `timeout()` around the relay's publish (Part 7.2) | Waits for the connection to return |
| SMTP, push provider | The client library's connection and socket timeouts | Library-specific; look it up |

An 80-second Stripe timeout behind a 3-second gateway budget means the user gave up 77 seconds ago
while your service still holds a connection, a saga step and a slot in the bulkhead.

## 12.3 Retries with exponential backoff and jitter

Retry only what is idempotent and only on transient errors (Part 5.6). Then space the attempts out:
**exponential backoff** gives the dependency room to recover, and **jitter** stops every client from
retrying in the same millisecond.

```ts
// libs/resilience/src/backoff.ts — "full jitter": a random delay up to an exponential ceiling
export function backoffWithJitter(attempt: number, baseMs = 100, capMs = 5_000): number {
  const ceiling = Math.min(capMs, baseMs * 2 ** attempt);
  return Math.floor(Math.random() * ceiling);
}
```

Retry at **one** layer. If the gateway retries 3 times, booking retries 3 times and inventory's
client retries 3 times, one failing call becomes 27. Choose the layer closest to the failing call,
and let the layers above fail fast.

Stripe's SDK retries network failures itself (`maxNetworkRetries`; the runtime default is 2, and it
adds an idempotency key to retried POSTs). Set `maxNetworkRetries` explicitly and do not wrap Stripe
calls in a second retry loop of your own.

## 12.4 A circuit breaker around Stripe

When Stripe is down, every payment attempt waits for a timeout and then fails. A **circuit breaker**
notices the failure rate and, for a while, fails calls immediately instead.

```text
 CLOSED ── failure rate ≥ threshold ──► OPEN (calls fail fast) ── resetTimeout ──► HALF-OPEN
   ▲                                     ▲                                          │
   │                                     └──────────── trial call fails ────────────┤
   └───────────────────────────────────────────────── trial call succeeds ──────────┘
```

```ts
// apps/payment/src/infrastructure/stripe/stripe-payment-intents.adapter.ts
this.createIntent = new CircuitBreaker(
  (params: Stripe.PaymentIntentCreateParams, idempotencyKey: string) =>
    this.stripe.paymentIntents.create(params, { idempotencyKey }),
  {
    timeout: 10_000,              // our ceiling for one call
    errorThresholdPercentage: 50, // open when half the recent calls fail…
    volumeThreshold: 10,          // …but only once there have been 10 calls in the window
    resetTimeout: 30_000,         // try one call again after 30 s
    // a declined card is not an outage: do not count it
    errorFilter: (error: unknown) => error instanceof Stripe.errors.StripeCardError,
  },
);
this.createIntent.on('open', () => this.logger.warn({ dependency: 'stripe' }, 'circuit opened'));
```

`opossum` is the breaker library here; it does not bundle TypeScript types, and `@types/opossum`
trails the latest major, so pin both and check the typings you rely on. When the breaker is open,
`fire()` rejects immediately: translate that into `PAYMENT_PROVIDER_UNAVAILABLE` (gRPC `UNAVAILABLE`,
HTTP 503) so the app can say "payment is temporarily unavailable, your seats are held until 14:48".
`fallback()` exists for degraded answers, but there is no honest fallback for taking money — failing
fast with a clear code is the fallback.

`errorFilter` matters: a declined card is the customer's problem, not Stripe's. Counting declines as
failures would open the circuit for everyone because a few cards bounced.

## 12.5 Bulkheads

A ship's bulkheads keep one flooded compartment from sinking the ship. In ENR:

- **One queue per consuming service** (Part 6.3): a slow notification consumer cannot delay booking.
- **Prefetch per consumer** (Part 6.6) caps how much concurrent work one queue can start.
- **One database and pool per service**: inventory's lock contention cannot exhaust booking's
  connections.
- **Separate paths for bulk and interactive work.** When trip 916 is cancelled, 600 refunds flow
  through payment's event queue with a low prefetch, while new PaymentIntents arrive over gRPC. The
  refund burst cannot starve customers who are trying to pay.

## 12.6 Graceful shutdown

Containers are stopped all the time: every deploy, every `docker compose up` after a rebuild. A
graceful shutdown finishes or safely hands back in-flight work instead of cutting it in half.

With `app.enableShutdownHooks()`, Nest 11 handles `SIGTERM` in this order:

```text
 SIGTERM
   1. onModuleDestroy()            stop starting new work: outbox relay, saga sweeper, schedulers
   2. beforeApplicationShutdown()  wait (bounded) for in-flight work you track yourself
   3. dispose                      HTTP server closes; gRPC server closes (gracefulShutdown: true lets
                                   in-flight calls finish); RabbitMQ consumer channel and connection
                                   close — unacked messages return to the queue for another instance
   4. onApplicationShutdown()      TypeORM destroys its DataSource; flush OpenTelemetry (tracing.shutdown)
```

Consequences worth internalising:

- A message being handled at step 3 is redelivered to another instance. That is safe **only**
  because the handler is idempotent (Part 7.3).
- Set `gracefulShutdown: true` in the gRPC server options, or close aborts in-flight calls.
- In Docker, `SIGTERM` must reach Node: run `node dist/apps/booking/main.js` directly (not through
  `npm start`) or set `init: true` in Compose, and give the container enough `stop_grace_period`
  before Docker sends `SIGKILL`.

## 12.7 When each dependency is down

Write this table for your own service; it is part of the README (`MS-655`) and the sprint 5 drill
checks it. The ENR-wide version looks like this:

| Down | What users see | What the system does |
| --- | --- | --- |
| A service's own Postgres | That service's routes return 503; others work | Readiness fails; consumers stop acking, messages wait in the queue |
| RabbitMQ | Almost nothing at first: bookings still commit | Outboxes fill up; relays retry; consumers reconnect; everything drains on recovery |
| Redis | Slower search; stricter behaviour on OTP and logout (Part 10.7) | Cache-aside falls back to Postgres; local rate limits |
| identity | Cannot sign in or edit profile; **existing sessions keep working** | Gateway verifies tokens locally |
| catalog | Cannot search or start a booking | Bookings in progress continue; the Tickets tab shows stations from booking's copy |
| inventory | Cannot hold seats | Paid sagas wait in `PAID`; the sweeper retries `ConfirmHold`; late-payment path if the hold expires |
| payment | Cannot pay | Stripe retries webhooks (test mode: three times over a few hours); booking's sweeper asks payment to reconcile intent status on restart |
| Stripe | "Payment temporarily unavailable" | Breaker open; holds expire normally; refunds wait in the queue and retry |
| notification / Mailpit | Emails and the bell arrive late | Events wait in the queue, then retry tiers, then the parking lot |
| gateway | Everything | Run two instances; it holds no state of its own except Redis |

## Rules — resilience

### MS-651 — Network call without a timeout
**Severity:** critical · **Applies to:** every client: gRPC, Stripe, Redis, Postgres, SMTP, HTTP
**Detect:** A client constructed with library defaults for timeouts; `fetch` without an
`AbortSignal.timeout`; a Stripe client without `timeout`. Generalizes `MS-251`.
**Why:** Without timeouts, one hung dependency holds connections and requests until the caller dies.
**Not a violation when:** The library's default is short and documented, and you have checked it.

### MS-652 — Retry without backoff, jitter or a cap, or at several layers
**Severity:** major · **Applies to:** retry helpers, client configuration
**Detect:** Fixed-delay or immediate retries; no maximum attempt count; the same failure retried by
the gateway, the service and the client library.
**Why:** Synchronized, multiplied retries turn a recovering dependency back into a failed one.
**Not a violation when:** The retries are RabbitMQ delay tiers (Part 6.3), which already space out.

### MS-653 — Stripe called without a circuit breaker
**Severity:** major · **Applies to:** payment adapters
**Detect:** `stripe.paymentIntents.*` / `stripe.refunds.*` called directly from use cases; a breaker
that counts card declines as failures; no mapping from "breaker open" to a clear error code.
**Why:** Without a breaker, a Stripe outage becomes a pile-up of 10-second waits across every payment
attempt.
**Not a violation when:** The call is made by an offline script (reconciliation run by hand).

### MS-654 — Shutdown drops in-flight work
**Severity:** major · **Applies to:** `main.ts`, relays, sweepers, Dockerfiles, Compose
**Detect:** No `enableShutdownHooks()`; pollers without a stop flag; gRPC server without
`gracefulShutdown: true`; `CMD ["npm", "run", "start:prod"]` without `init: true`.
**Why:** Every deploy otherwise cuts sagas and publishes in half, and tests your idempotency in
production instead of in CI.
**Not a violation when:** Never.

### MS-655 — Undocumented behaviour when a dependency is down
**Severity:** major · **Applies to:** each service's README
**Detect:** No failure-mode table like 12.7 for the service; behaviour in the table never exercised
(the sprint 5 drill or a test with the dependency stopped).
**Why:** Undecided failure behaviour is decided by accident, under pressure.
**Not a violation when:** Never.

---

# Part 13 — Testing microservices

## 13.1 The shape of the suite

The DOCURA reviews counted test files: 79 in the strongest team, 2 in the weakest. In ENR the
interesting bugs live **between** services — duplicates, retries, contract drift, races — so the
suite needs a level that most of you skipped last time.

| Level | Scope | Real infrastructure? | Catches | How many |
| --- | --- | --- | --- | --- |
| **Unit** | One domain function or class | None | Wrong rules: refund tiers, fees, transitions, allocation | Hundreds; milliseconds each |
| **Integration** | One service, booted for real | Postgres, RabbitMQ, Redis via Testcontainers | Wrong SQL, locks, acks, retries, inbox, cache behaviour | Dozens per service |
| **Contract** | One `.proto` or event schema | None | A change that breaks another team's service | One per contract and version |
| **End-to-end** | The whole Compose stack | Everything, Stripe test mode | Wiring and configuration | A handful of flows |

## 13.2 Unit tests: the rules, with their boundaries

Every business rule gets unit tests at its boundaries, exactly as `JT-704` asks. R8's refund tiers are
the textbook case: the boundaries are the bugs.

```ts
// apps/booking/src/domain/refund-policy.spec.ts
describe('refundFor', () => {
  const boardingAt = new Date('2026-10-24T10:00:00Z');
  const ticket = { farePiastres: 15_500, boardingAt };

  it.each([
    ['exactly 24 h before boarding: full fare', '2026-10-23T10:00:00Z', 15_500],
    ['one second inside 24 h: half fare', '2026-10-23T10:00:01Z', 7_750],
    ['exactly 2 h before boarding: half fare', '2026-10-24T08:00:00Z', 7_750],
  ])('refunds %s', (_case, cancelledAt, expectedPiastres) => {
    expect(refundFor(ticket, new Date(cancelledAt)))
      .toEqual({ kind: 'REFUND', amountPiastres: expectedPiastres });
  });

  it('refuses cancellation less than 2 h before boarding', () => {
    expect(refundFor(ticket, new Date('2026-10-24T08:00:01Z')))
      .toEqual({ kind: 'REFUSED', reason: 'TOO_CLOSE_TO_DEPARTURE' });
  });
});
```

The same treatment for: the 7% fee rounding (R6), National ID validation (R4), adjacency allocation
(R2), and **every (state, event) pair of the saga's transition table** (Part 8.5). None of these
need a database; if they do, the rule is in the wrong layer.

## 13.3 Integration tests with Testcontainers

An integration test boots one service the way production does — same `configureApp`, same modules —
against throwaway containers. Other services are replaced at the **port** (an in-memory `SeatHolds`,
a fake `PaymentGateway`), never by mocking your own database or broker.

```ts
// apps/notification/test/booking-confirmed.int-spec.ts
describe('booking.confirmed consumer', () => {
  let postgres: StartedPostgreSqlContainer;
  let rabbit: StartedRabbitMQContainer;
  let app: INestApplication;

  beforeAll(async () => {
    [postgres, rabbit] = await Promise.all([
      new PostgreSqlContainer('postgres:18').start(),
      new RabbitMQContainer('rabbitmq:4-management').start(),
    ]);
    app = await startNotificationApp({
      DATABASE_URL: postgres.getConnectionUri(),
      RABBITMQ_URL: rabbit.getAmqpUrl(),
      RABBITMQ_PREFETCH: '1', // one message at a time: deliveries are handled in publish order
    });
  }, 120_000);

  afterAll(async () => {
    await app.close();
    await Promise.all([postgres.stop(), rabbit.stop()]);
  });

  it('sends one confirmation when the same event arrives twice', async () => {
    const event = aBookingConfirmedEvent();
    const sentinel = aBookingConfirmedEvent();
    await publishEvents(rabbit.getAmqpUrl(), [event, event, sentinel]);

    await waitUntil(async () => (await deliveriesFor(app, sentinel.payload.bookingId)).length === 1);
    expect(await deliveriesFor(app, event.payload.bookingId)).toHaveLength(1);
  });
});
```

The sentinel is how you assert that something did *not* happen without sleeping: once the message
published after the duplicate has been handled, the duplicate has been handled too. `waitUntil` polls
with a timeout; there is no `setTimeout(2000)` anywhere (`JT-503`).

Integration tests every service needs:

| Test | Proves |
| --- | --- |
| Same event delivered twice → one effect | The inbox works (`MS-353`, `JT-706`) |
| Handler throws → message in a retry queue → eventually parked | Retry topology and `x-retry-attempt` counting |
| Schema-invalid payload → parked immediately | Non-retryable errors skip the retries |
| Two relays draining one outbox → each event published once | `FOR UPDATE SKIP LOCKED` and the lease |
| Two concurrent holds for the last seat → exactly one succeeds | The database guard, not luck (`JT-310`) |
| Redis container stopped → reads still served | Graceful degradation (Part 10.7) |

Each test file gets its own containers, or its own database inside one container; tests never share
rows (`JT-604`). Testcontainers 12 needs Node 22.22 or later and an explicit image in every
constructor; `PostgreSqlContainer` can also `snapshot()` a migrated database and `restoreSnapshot()`
it between tests, which is faster than re-running migrations.

## 13.4 Contract tests

Contracts break between teams, so test them where both teams can see the failure.

**gRPC.** Run `buf lint` and `buf breaking --against '.git#branch=main'` on every pull request that
touches `libs/contracts/proto`. That catches removed fields, changed types and renamed elements.
`MS-252`'s semantic breaks (units, meaning) are still a reviewer's job.

**Events.** Each event type and version has a schema (zod) and a **golden fixture** in
`libs/contracts/events`:

```text
libs/contracts/events/booking-confirmed/
├── booking-confirmed.v1.ts          zod schema + inferred type
└── booking-confirmed.v1.sample.json a realistic envelope, reviewed like code
```

- **Producer test** (in booking): the event your code builds passes the schema.
- **Consumer test** (in notification): every fixture of the versions you claim to support is accepted
  by your handler.
- **Compatibility test** (in `libs/contracts`): fixtures written for an older release still pass the
  current schema of the same version. If one fails, you made a breaking change without a new version.

**Stripe.** Do not mock Stripe's webhook format by hand. Build signed test payloads with
`stripe.webhooks.generateTestHeaderString({ payload, secret })` so the real `constructEvent` runs in
your webhook tests.

## 13.5 A few end-to-end flows

End-to-end tests run against the full Compose stack. They are slow and the most fragile, so keep
them to the flows that prove the system is wired together:

1. Sign up → read the OTP from Mailpit's HTTP API → verify → sign in.
2. Search → hold → create a PaymentIntent → confirm it in the test with Stripe's `pm_card_visa`
   test payment method → the webhook arrives through `stripe listen` → tickets appear.
3. Cancel a ticket 30 hours before boarding → refund `PENDING` → `REFUNDED`.
4. Ops cancels a trip → every ticket voided, refunds including the fee, notifications sent.
5. The sprint 5 drill: stop inventory mid-saga, start it again → the booking completes or is refunded.

Poll for outcomes with timeouts; never sleep. Use `stripe trigger` only for webhook plumbing, not for
the booking flow — triggered events do not belong to your bookings.

## 13.6 What not to do

| Don't | Because | Instead |
| --- | --- | --- |
| Mock the broker or the database in integration tests | You test your mock's ack and lock semantics, not RabbitMQ's or Postgres's | Testcontainers |
| Share one test database between services or test files | Order-dependent, flaky tests (`JT-604`) | A database per test file, or snapshots |
| `sleep(2000)` and hope | Slow when it works, flaky when it does not (`JT-503`) | Poll with a timeout; use a sentinel |
| Test business rules end to end | Minutes per case, and the failure points nowhere | Unit tests at the boundaries |
| Spin up all seven services for one service's test | You are testing Compose | Fake the neighbours at the port |
| `expect(controller).toBeDefined()` | Cannot fail (`JT-401`) | Assert a behaviour |

## Rules — testing

### MS-701 — Business rule without boundary unit tests
**Severity:** major · **Applies to:** `apps/*/src/domain/**`
**Detect:** A rule from the brief (R2, R4, R6, R8, R9, the saga transitions) with no unit test, or
tests that skip the exact boundary values. Pairs with `JT-704`, `JT-701`.
**Why:** Boundaries are where money rules break; DOCURA's 24-hour cutoff bug was exactly this.
**Not a violation when:** Never.

### MS-702 — Integration test against mocks of the database or broker
**Severity:** major · **Applies to:** `apps/*/test/**`
**Detect:** `jest.mock` of TypeORM, `amqplib`, `ClientProxy` or ioredis in a test named or placed as
an integration test; a service with no Testcontainers-based tests at all.
**Why:** The behaviours that matter — locks, acks, dead-lettering, unique constraints — exist only
in the real thing.
**Not a violation when:** The mocked collaborator is another service, replaced at its port.

### MS-703 — Contract without a contract test
**Severity:** major · **Applies to:** `libs/contracts/**`, CI or pre-merge scripts
**Detect:** No `buf lint`/`buf breaking` step; event types without a schema and fixture; producers or
consumers not tested against the shared schema.
**Why:** A contract break is invisible to the team that made it and fatal to the team that consumes it.
**Not a violation when:** Never.

### MS-704 — Consumer untested for duplicates and failures
**Severity:** major · **Applies to:** event handlers
**Detect:** A consumer without an integration test for duplicate delivery, and for failure ending in
retry or the parking lot. Pairs with `JT-706`.
**Why:** Duplicates and failures are the normal case for a consumer, not the edge case.
**Not a violation when:** Never.

### MS-705 — End-to-end suite used for rules, or built on sleeps
**Severity:** minor · **Applies to:** `e2e/**`
**Detect:** More than a handful of end-to-end tests; business-rule cases asserted only end to end;
fixed sleeps.
**Why:** A slow, flaky suite stops being run.
**Not a violation when:** Never.

### MS-706 — Tests share state across files or services
**Severity:** major · **Applies to:** test setup
**Detect:** One long-lived database for all test files; tests relying on rows created by another
file; integration tests pointing at the Compose stack's databases. Pairs with `JT-604`, `JT-105`.
**Why:** Shared state makes results depend on execution order and on whoever ran last.
**Not a violation when:** The shared resource is read-only (a container image, a schema snapshot).

---

# Part 14 — The local stack

## 14.1 One command, the whole system

`docker compose up -d --build` from a clean clone must bring all seven services and their
infrastructure to healthy. That is the only infrastructure requirement of the project, and it is
what your mentor runs first.

| Container | Image | Host ports | Purpose |
| --- | --- | --- | --- |
| `postgres` | `postgres:18` | 5432 | One server, six databases, six roles |
| `redis` | `redis:8` | 6379 | Cache and short-lived stores |
| `rabbitmq` | `rabbitmq:4-management` | 5672, **15672** (UI) | Events, retries, parking lots |
| `mailpit` | `axllent/mailpit` | **8025** (UI) | Catches every email; SMTP on 1025 inside the network |
| `stripe-cli` | `stripe/stripe-cli` | — | Forwards test-mode webhooks to payment |
| `jaeger` | `cr.jaegertracing.io/jaegertracing/jaeger:2.22.0` | **16686** (UI), 4318 | Receives traces over OTLP/HTTP on 4318 |
| 7 apps | Built from `apps/*/Dockerfile` | gateway only: 3000 | Internal gRPC ports stay unpublished (Part 4.3) |

Jaeger v1 (`jaegertracing/all-in-one`) reached end of life on 31 December 2025; use v2.

## 14.2 Postgres: one container, separate databases and credentials

An init script creates a database and a role per service, and revokes everyone else's access.
Postgres runs scripts in `/docker-entrypoint-initdb.d` **only when the data directory is empty** —
after changing them, reset with `docker compose down -v`.

```sql
-- infra/postgres/init/01-service-databases.sql (development credentials only)
CREATE ROLE booking LOGIN PASSWORD 'booking_dev';
CREATE DATABASE booking OWNER booking;
REVOKE ALL ON DATABASE booking FROM PUBLIC;

CREATE ROLE inventory LOGIN PASSWORD 'inventory_dev';
CREATE DATABASE inventory OWNER inventory;
REVOKE ALL ON DATABASE inventory FROM PUBLIC;
-- … identity, catalog, payment, notification
```

Booking's env contains booking's connection string and nothing else. If someone writes a query
against inventory's tables from booking, it fails with a permission error on the first run, not in
review three sprints later.

## 14.3 The Compose file

```yaml
# docker-compose.yml (excerpt)
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres   # development only
    ports: ["5432:5432"]            # infrastructure ports are published so apps can run outside Docker
    volumes:
      - pgdata:/var/lib/postgresql  # Postgres 18 images keep data under /var/lib/postgresql
      - ./infra/postgres/init:/docker-entrypoint-initdb.d:ro
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s, timeout: 3s, retries: 10 }

  rabbitmq:
    image: rabbitmq:4-management
    ports: ["5672:5672", "15672:15672"]
    healthcheck: { test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"], interval: 10s, timeout: 5s, retries: 10 }

  redis:
    image: redis:8
    ports: ["6379:6379"]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 3s, retries: 10 }

  mailpit:
    image: axllent/mailpit
    ports: ["8025:8025"]

  jaeger:
    image: cr.jaegertracing.io/jaegertracing/jaeger:2.22.0
    ports: ["16686:16686", "4318:4318"]

  stripe-cli:
    image: stripe/stripe-cli
    command: listen --forward-to http://payment:3005/webhooks/stripe
    environment:
      STRIPE_API_KEY: ${STRIPE_SECRET_KEY}   # from the untracked root .env

  booking:
    build: { context: ., dockerfile: apps/booking/Dockerfile }
    env_file: apps/booking/.env
    environment:
      OTEL_SERVICE_NAME: booking
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4318
    depends_on:
      postgres: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }
      redis: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "node", "-e",
             "fetch('http://localhost:3004/health/ready').then(r => process.exit(r.ok ? 0 : 1), () => process.exit(1))"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    init: true                # forwards SIGTERM to node (Part 12.6)
    stop_grace_period: 30s

volumes:
  pgdata:
```

The app healthcheck uses Node's built-in `fetch`, so the image needs no `curl`.

## 14.4 Start order is a hint, not a guarantee

`depends_on` with `condition: service_healthy` makes Compose wait for a dependency's healthcheck
before starting a service. Use it — and still write services that survive dependencies coming and
going, because they will after startup anyway:

- `@nestjs/typeorm` retries the initial connection (9 attempts, 3 s apart, by default).
- Nest's RabbitMQ transport reconnects through `amqp-connection-manager`, retrying forever by default.
- ioredis reconnects on its own; with the options from Part 10.7, commands fail fast meanwhile.
- The gateway does **not** `depends_on` the services: it must start, and degrade, without them.

## 14.5 Configuration: one schema per service, validated at boot

Every team in Project 1 and three of four in Project 2 shipped services that started without
required configuration and failed later, in one endpoint. In ENR each of the seven services
validates its environment at boot and refuses to start if anything is missing or malformed.

```ts
// apps/booking/src/config/env.schema.ts
import { z } from 'zod';

export const bookingEnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  HTTP_PORT: z.coerce.number().int().positive(),
  GRPC_BIND_URL: z.string().min(1),           // e.g. 0.0.0.0:50054
  DATABASE_URL: z.url(),
  RABBITMQ_URL: z.url(),
  RABBITMQ_PREFETCH: z.coerce.number().int().min(1).max(100),
  REDIS_URL: z.url(),
  CATALOG_GRPC_URL: z.string().min(1),
  INVENTORY_GRPC_URL: z.string().min(1),
  PAYMENT_GRPC_URL: z.string().min(1),
  JWT_PUBLIC_KEY: z.string().startsWith('-----BEGIN PUBLIC KEY-----'),
});

export type BookingEnv = z.infer<typeof bookingEnvSchema>;

export function validateBookingEnv(raw: Record<string, unknown>): BookingEnv {
  const result = bookingEnvSchema.safeParse(raw);
  if (!result.success) {
    throw new Error(`Invalid booking configuration:\n${z.prettifyError(result.error)}`);
  }
  return result.data;
}

// apps/booking/src/app.module.ts
ConfigModule.forRoot({ isGlobal: true, validate: validateBookingEnv }),
```

Inject `ConfigService<BookingEnv, true>` and read values with `config.get('DATABASE_URL', { infer: true })`
— typed, validated, and never `process.env` deep in a module (`CC-701`). Commit an
`apps/<service>/.env.example` with every key and a safe placeholder; never commit the `.env` itself.

The Stripe webhook secret is the one value you get from a tool: run
`docker compose run --rm stripe-cli listen --print-secret` once, and put the `whsec_…` value in
payment's `.env`. The secret printed by `stripe listen` stays the same between restarts.

## 14.6 Seeding

ENR needs a realistic network before anything is testable: bilingual stations, trains, coach
layouts, timetables and fares (CAT-1, CAT-2). Seed **through the owning service**:

- The seed is a script in catalog that boots catalog's own modules
  (`NestFactory.createApplicationContext`) and calls its use cases — the same validation, the same
  outbox.
- It is **idempotent**: running it twice changes nothing (upsert by natural keys such as station
  code and train number).
- Downstream data appears the normal way. Catalog's outbox publishes the timetable; inventory builds
  seat inventory from it (SEAT-1); booking fills its station copy. Nobody inserts rows into another
  service's database "to save time".

Run it as a one-shot Compose service that finishes before dependents start
(`depends_on: { catalog-seed: { condition: service_completed_successfully } }`), or by hand with
`docker compose run --rm catalog node dist/apps/catalog/seed.js`.

## Rules — local stack and configuration

### MS-801 — Service boots with missing or invalid configuration
**Severity:** critical · **Applies to:** `apps/*/src/config/**`, `app.module.ts`
**Detect:** `ConfigModule.forRoot` without `validate` or `validationSchema`; `process.env.X` outside
the config module; `as string`, `!`, `|| ''` or `String(...)` on configuration values. Pairs with
`CC-701`, `TS-504`.
**Why:** Two cohorts in a row shipped this bug. A service that starts half-configured fails later,
somewhere else, under load.
**Not a violation when:** The variables are the standard `OTEL_*` ones, read by the OpenTelemetry SDK
itself in `tracing.ts` before Nest starts.

### MS-802 — The stack does not come up healthy from a clean clone
**Severity:** major · **Applies to:** Compose file, Dockerfiles, connection setup, README
**Detect:** Manual steps beyond copying `.env.example` files and setting the Stripe secret (14.5);
`depends_on` without health conditions; infrastructure without healthchecks; connection retries
disabled; a gateway that cannot start while a service is down.
**Why:** Your mentor, your teammates and the demo machine all start from a clean clone — and Compose
start order does not help when RabbitMQ restarts at 3 p.m.
**Not a violation when:** Never.

### MS-803 — Secret committed, or no `.env.example` per service
**Severity:** critical · **Applies to:** the repository
**Detect:** A committed `.env`, Stripe key, webhook secret or private key; a service without an
`.env.example` listing every key its schema requires. Pairs with `CC-702`.
**Why:** A committed secret is a leaked secret (see the git handbook, Part 10.1); a missing example
means the next person discovers each key by crashing.
**Not a violation when:** The value is an obvious development placeholder in `.env.example`, or a
development-only password in the init script and Compose file.

### MS-804 — Seed that bypasses the owner, or is not idempotent
**Severity:** major · **Applies to:** seed scripts, SQL fixtures
**Detect:** A seed writing into several services' databases; SQL dumps loaded into another service's
database; a seed that duplicates rows when run twice.
**Why:** Seeds that bypass the owner create data that no event announced, so every copy and read
model downstream is wrong from day one.
**Not a violation when:** The data is the owner's own and inserted through its own code.

---

# Part 15 — Review checklist and rule catalogue

## 15.1 How mentors use this part

Review a pull request in the order of [`standards/README.md`](./standards/README.md#2-review-protocol):
scope the diff, read the rules, detect, **verify by reading the code**, deduplicate, rank, report. The
`MS` rules sit beside the `CC`, `TS`, `JT` and `RS` catalogs — a gRPC adapter can break `MS-251` and
`CC-401` at once; report the root cause once and mention the other in the fix.

Start from the checklist for the kind of change in front of you, then look up the rule.

## 15.2 Checklist by kind of change

**Adds or changes a gRPC call**
- [ ] Deadline set by the client's interceptor, within the caller's remaining budget (`MS-251`)
- [ ] Retried only if idempotent, only on `UNAVAILABLE`, bounded with backoff (`MS-253`, `MS-652`)
- [ ] Errors mapped to a status and a catalogued code at both ends (`MS-254`, `MS-205`)
- [ ] Called through a port; no per-item loop (`MS-054`, `MS-105`)

**Changes a `.proto` or an event schema**
- [ ] Additive within the version; numbers and names of removed fields reserved (`MS-252`, `MS-307`)
- [ ] `buf lint`/`buf breaking` pass; event fixture added or updated (`MS-703`)
- [ ] Money in integer piastres, units in names (`MS-154`)

**Publishes an event**
- [ ] Written to the outbox in the same transaction as the state change (`MS-351`)
- [ ] Standard envelope, past-tense name, no secrets (`MS-305`, `MS-306`, `MS-309`)

**Adds or changes a consumer**
- [ ] Manual ack after commit; every path ends in ack, retry or park (`MS-301`, `MS-308`)
- [ ] Inbox in the same transaction; external effects keyed (`MS-353`, `MS-355`)
- [ ] Order-independent; replicated data newer-wins (`MS-354`, `MS-152`)
- [ ] Integration tests for duplicates and for retry → park (`MS-704`)

**Touches the booking saga or payments**
- [ ] State persisted; every waiting state has a deadline; sweeper covers it (`MS-401`, `MS-403`)
- [ ] Steps idempotent; compensations defined; late outcomes handled (`MS-402`, `MS-404`, `MS-405`)
- [ ] Stripe behind a breaker, with idempotency keys and a verified, deduplicated webhook (`MS-653`, `MS-355`)

**Adds or changes a cache**
- [ ] Never used to decide (`MS-501`)
- [ ] Owner-namespaced, versioned key; per-user data under per-user keys (`MS-503`, `MS-504`)
- [ ] Jittered TTL, stampede protection, event invalidation, metrics (`MS-505`, `MS-506`, `MS-507`)
- [ ] Works with Redis stopped (`MS-502`)

**Every pull request**
- [ ] No cross-database access or cross-app import (`MS-102`, `MS-052`)
- [ ] Correlation ID preserved; structured logs without secrets (`MS-601`, `MS-602`)
- [ ] Configuration validated at boot; `.env.example` updated (`MS-801`, `MS-803`)
- [ ] Unit tests at the rule's boundaries (`MS-701`)

## 15.3 The rule catalogue

Severity is the default; escalate one level on authentication, payment and refund paths.

| ID | Rule | Severity | Part |
| --- | --- | --- | --- |
| [`MS-001`](#ms-001--service-boundary-without-a-written-decision) | Every service has an ADR: the capability it owns, the data it alone writes, why it is separate, its owner. | major | 1 |
| [`MS-002`](#ms-002--contract-written-after-the-code-that-uses-it) | Contracts are reviewed in `libs/contracts` before producer or consumer code merges. | major | 1 |
| [`MS-051`](#ms-051--domain-code-in-libs) | `libs/` holds contracts and cross-cutting code only — no entities, repositories or business rules. | major | 3 |
| [`MS-052`](#ms-052--one-app-imports-another-apps-source) | Apps never import another app's source; they share through `libs/contracts`. | **critical** | 3 |
| [`MS-053`](#ms-053--requestresponse-over-rabbitmq-or-an-unscoped-event-handler) | RabbitMQ carries events only, and `@EventPattern` is scoped to `Transport.RMQ`. | major | 3 |
| [`MS-054`](#ms-054--transport-types-leak-into-the-application-layer) | Domain and application code depend on ports, never on `ClientGrpc`, Observables, `RmqContext` or Stripe types. | major | 3 |
| [`MS-055`](#ms-055--connected-microservice-without-the-apps-pipes-filters-and-interceptors) | Connected microservices inherit (or declare) pipes, filters and interceptors; filters handle the rpc context. | major | 3 |
| [`MS-101`](#ms-101--two-services-write-the-same-table) | Each table has exactly one owning service. | **critical** | 2 |
| [`MS-102`](#ms-102--a-service-reads-another-services-database) | No service connects to another service's database. | **critical** | 2 |
| [`MS-103`](#ms-103--entity-service-instead-of-a-capability) | Services expose capabilities, not CRUD over another service's nouns. | major | 2 |
| [`MS-104`](#ms-104--atomic-invariant-split-across-services) | An invariant that needs atomicity lives inside one service's transaction. | **critical** | 2 |
| [`MS-105`](#ms-105--chatty-call-an-rpc-per-item) | No RPC per item: batch the call or keep a local copy. | major | 2 |
| [`MS-151`](#ms-151--schema-changed-outside-the-owning-services-migrations) | Each service migrates only its own database, expand-then-contract, with `synchronize` off. | major | 9 |
| [`MS-152`](#ms-152--replicated-data-written-without-an-ordering-guard) | Local copies are read-only and updated newer-version-wins. | major | 9 |
| [`MS-153`](#ms-153--transaction-held-across-a-remote-call) | No transaction is held open across a remote call; no distributed transactions. | **critical** | 9 |
| [`MS-154`](#ms-154--money-not-in-integer-piastres-end-to-end) | Money is integer piastres in the database, `.proto`, events and Stripe. | **critical** | 9 |
| [`MS-155`](#ms-155--historical-record-references-mutable-data-instead-of-a-snapshot) | Historical records snapshot the facts they sold. | major | 9 |
| [`MS-201`](#ms-201--access-token-verified-remotely-or-loosely) | The gateway verifies RS256 tokens locally, pinning `alg`, `iss`, `aud`, `exp`, and checks the `jti` denylist. | **critical** | 4 |
| [`MS-202`](#ms-202--route-reachable-without-an-explicit-access-decision) | Every route is denied by default; public routes opt out explicitly. | **critical** | 4 |
| [`MS-203`](#ms-203--caller-identity-trusted-from-a-header-or-the-body) | Caller identity comes from a verified token, never from a header or a request field. | **critical** | 4 |
| [`MS-204`](#ms-204--business-rule-in-the-gateway) | The gateway authenticates, routes, aggregates and translates — it holds no business rules. | major | 4 |
| [`MS-205`](#ms-205--upstream-error-shape-or-detail-leaks-through-the-edge) | One envelope; every upstream failure maps to a catalogued code; no internals leak. | major | 4 |
| [`MS-206`](#ms-206--rate-limit-kept-per-instance-or-keyed-too-broadly) | Rate limits live in Redis, keyed per user or IP and per route class. | major | 4 |
| [`MS-251`](#ms-251--grpc-call-without-a-deadline) | Every gRPC call carries an absolute deadline within the caller's remaining budget. | **critical** | 5 |
| [`MS-252`](#ms-252--breaking-change-inside-a-published-proto-package) | Proto changes inside a version are additive; numbers are never reused; removals are reserved. | **critical** | 5 |
| [`MS-253`](#ms-253--retry-of-a-non-idempotent-call-or-an-unbounded-retry) | Only idempotent calls are retried, only on retryable codes, bounded and with backoff. | major | 5 |
| [`MS-254`](#ms-254--expected-failure-returned-as-unknown-or-internal) | Expected failures cross gRPC as a specific status plus a stable error code. | major | 5 |
| [`MS-255`](#ms-255--hand-written-copy-of-a-contract-type) | Contract types are generated from the `.proto`, never copied by hand. | major | 5 |
| [`MS-301`](#ms-301--message-acknowledged-before-the-work-commits-or-never) | `noAck: false`; ack after the commit; every path ends in ack, retry or park. | **critical** | 6 |
| [`MS-302`](#ms-302--unbounded-or-untuned-prefetch) | Every consumer sets a prefetch sized to the resources its handler uses. | major | 6 |
| [`MS-303`](#ms-303--message-can-be-lost-by-the-broker) | Durable exchanges and queues, persistent messages, awaited publisher confirms. | **critical** | 6 |
| [`MS-304`](#ms-304--queue-shared-across-services-or-consumed-by-a-non-owner) | One queue per consuming service; that service's instances compete on it. | major | 6 |
| [`MS-305`](#ms-305--message-without-the-standard-envelope) | Every event travels in the standard envelope. | major | 6 |
| [`MS-306`](#ms-306--event-named-as-a-command-or-a-crud-operation) | Events are named as past-tense business facts. | minor | 6 |
| [`MS-307`](#ms-307--breaking-event-change-under-the-same-version) | Event changes within a version are additive; consumers are tolerant readers. | major | 6 |
| [`MS-308`](#ms-308--failed-message-requeued-in-a-hot-loop-or-dropped) | Failures go through delayed retry tiers, then the parking lot — never a hot requeue loop, never dropped. | **critical** | 6 |
| [`MS-309`](#ms-309--secret-or-unnecessary-personal-data-in-an-event) | Events carry no secrets and only the personal data a consumer needs. | **critical** | 6 |
| [`MS-351`](#ms-351--state-change-published-without-the-outbox) | State changes publish through an outbox written in the same transaction. | **critical** | 7 |
| [`MS-352`](#ms-352--outbox-relay-that-holds-a-transaction-across-the-publish-or-cannot-run-twice) | The relay claims with `SKIP LOCKED` and a lease, publishes outside the transaction, marks after the confirm. | major | 7 |
| [`MS-353`](#ms-353--consumer-without-an-inbox) | Consumers record the `eventId` in an inbox in the same transaction as their writes. | **critical** | 7 |
| [`MS-354`](#ms-354--consumer-depends-on-delivery-order) | Consumers never depend on delivery order. | major | 7 |
| [`MS-355`](#ms-355--external-side-effect-without-an-idempotency-key) | External side effects carry idempotency keys derived from business ids. | **critical** | 7 |
| [`MS-356`](#ms-356--parking-lot-without-an-owner-or-a-replay-path) | Every parking lot has an owner, an alert and a replay script. | major | 7 |
| [`MS-401`](#ms-401--cross-service-process-without-a-persisted-state-machine) | Cross-service processes are persisted state machines with one owner. | **critical** | 8 |
| [`MS-402`](#ms-402--saga-step-that-is-not-idempotent) | Every saga step is idempotent on a deterministic key. | **critical** | 8 |
| [`MS-403`](#ms-403--waiting-state-without-a-deadline) | Every waiting state has a deadline and a sweeper. | **critical** | 8 |
| [`MS-404`](#ms-404--step-without-a-defined-compensation) | Every step that a later failure can strand has a defined compensation. | **critical** | 8 |
| [`MS-405`](#ms-405--late-or-out-of-order-outcome-not-handled) | Late, duplicate and out-of-order outcomes have explicit transitions. | **critical** | 8 |
| [`MS-406`](#ms-406--transition-logic-tangled-with-io-or-untested) | Saga transitions are a pure function with a test per (state, event) pair. | major | 8 |
| [`MS-501`](#ms-501--decision-taken-from-a-cached-value) | The cache displays; the database decides. | **critical** | 10 |
| [`MS-502`](#ms-502--redis-failure-fails-the-request) | A Redis failure degrades to the source; store-type keys have a decided failure mode. | major | 10 |
| [`MS-503`](#ms-503--unversioned-key-or-a-key-outside-the-owners-namespace) | Keys are namespaced by their owner and versioned; cached values are validated on read. | major | 10 |
| [`MS-504`](#ms-504--per-user-data-under-a-shared-key) | Per-user data lives only under per-user keys. | **critical** | 10 |
| [`MS-505`](#ms-505--no-ttl-no-jitter-or-no-stampede-protection-on-a-hot-key) | Every entry has a jittered TTL; hot keys have stampede protection. | major | 10 |
| [`MS-506`](#ms-506--cache-invalidated-by-someone-other-than-its-owner-or-not-at-all) | The owner invalidates its cache from events; the TTL is the safety net. | major | 10 |
| [`MS-507`](#ms-507--cache-without-hitmiss-metrics) | Every cache records hit, miss and error counts. | minor | 10 |
| [`MS-601`](#ms-601--correlation-id-lost-at-a-hop) | The correlation ID crosses every hop and appears in every log line. | major | 11 |
| [`MS-602`](#ms-602--unstructured-log-or-a-secret-or-personal-data-in-a-log-line) | Logs are structured JSON with redaction; no secrets or personal data. | **critical** | 11 |
| [`MS-603`](#ms-603--service-not-traced-or-trace-broken-at-the-outbox) | Every service exports traces, and the outbox preserves trace context. | major | 11 |
| [`MS-604`](#ms-604--liveness-checks-dependencies-or-readiness-checks-the-wrong-ones) | Liveness checks the process; readiness checks only the service's own hard dependencies. | major | 11 |
| [`MS-605`](#ms-605--backlogs-invisible) | Outbox backlog, parking-lot depth and overdue sagas are observable. | major | 11 |
| [`MS-651`](#ms-651--network-call-without-a-timeout) | Every network call has a timeout. | **critical** | 12 |
| [`MS-652`](#ms-652--retry-without-backoff-jitter-or-a-cap-or-at-several-layers) | Retries use capped exponential backoff with jitter, at one layer only. | major | 12 |
| [`MS-653`](#ms-653--stripe-called-without-a-circuit-breaker) | Stripe calls go through a circuit breaker that does not count card declines. | major | 12 |
| [`MS-654`](#ms-654--shutdown-drops-in-flight-work) | Shutdown stops intake, lets in-flight work finish, and relies on redelivery only for the rest. | major | 12 |
| [`MS-655`](#ms-655--undocumented-behaviour-when-a-dependency-is-down) | Each service documents and exercises its behaviour when each dependency is down. | major | 12 |
| [`MS-701`](#ms-701--business-rule-without-boundary-unit-tests) | Every business rule has unit tests at its boundaries. | major | 13 |
| [`MS-702`](#ms-702--integration-test-against-mocks-of-the-database-or-broker) | Integration tests run against real Postgres, RabbitMQ and Redis via Testcontainers. | major | 13 |
| [`MS-703`](#ms-703--contract-without-a-contract-test) | Every `.proto` and event schema has contract tests. | major | 13 |
| [`MS-704`](#ms-704--consumer-untested-for-duplicates-and-failures) | Every consumer is tested for duplicate delivery and for failure → retry → park. | major | 13 |
| [`MS-705`](#ms-705--end-to-end-suite-used-for-rules-or-built-on-sleeps) | End-to-end tests cover a handful of flows and never sleep. | minor | 13 |
| [`MS-706`](#ms-706--tests-share-state-across-files-or-services) | Tests never share state across files or services. | major | 13 |
| [`MS-801`](#ms-801--service-boots-with-missing-or-invalid-configuration) | Configuration is validated by a typed schema at boot; invalid configuration stops the process. | **critical** | 14 |
| [`MS-802`](#ms-802--the-stack-does-not-come-up-healthy-from-a-clean-clone) | A clean clone comes up healthy with one command; clients reconnect on their own. | major | 14 |
| [`MS-803`](#ms-803--secret-committed-or-no-envexample-per-service) | No secrets in the repository; a complete `.env.example` per service. | **critical** | 14 |
| [`MS-804`](#ms-804--seed-that-bypasses-the-owner-or-is-not-idempotent) | Seeds go through the owning service and are idempotent. | major | 14 |

## 15.4 Extending the catalogue

Follow the same discipline as the standards catalogs: never renumber a rule; retire it with
`Status: retired`; give a new rule the next free number in its band; and give every rule a severity,
a `Detect` line, a reason, and a "Not a violation when" clause. A rule nobody can detect, or that has
no exceptions, produces noise in review — and noise is how real findings get ignored.
