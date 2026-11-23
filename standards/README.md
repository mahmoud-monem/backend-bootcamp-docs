# Code Standards — Review Agent Operating Manual

Five rule catalogs live beside this file. Each rule has a stable ID so findings are
comparable across teams, across runs, and over time.

| Catalog | Prefix | Scope |
| --- | --- | --- |
| [`clean-code.md`](./clean-code.md) | `CC-###` | Naming, function design, layering, duplication, error handling, async correctness, resource/timeout hygiene, concurrency, domain events |
| [`typescript.md`](./typescript.md) | `TS-###` | Compiler config, type escapes, type modelling, framework/ORM numeric & DI typing, module conventions |
| [`testing-jest.md`](./testing-jest.md) | `JT-###` | Test structure, mocking policy, assertions, isolation, Vitest/Testcontainers parity, HTTP boundary & event/concurrency testing |
| [`restful-api.md`](./restful-api.md) | `RS-###` | Resource naming, HTTP methods, status codes, request/response contracts, webhook & API security, idempotency, health probes |
| [`modular-structure.md`](./modular-structure.md) | `MA-###` | Feature slicing, module layout, encapsulation, public APIs, dependency graph, data ownership, shared kernel, composition root |

> **Companion for Project 3 (ENR):** Cross-process microservices rules (`MS-###`: gRPC, RabbitMQ, outbox/inbox, sagas, Redis caching, distributed resilience) live in [`../microservices-nestjs-guide.md`](../microservices-nestjs-guide.md). All five `docs/standards/` catalogs continue to apply inside every service.

Target codebases across bootcamp projects:
- **Project 1 (Online Exam):** Node + Express + TypeScript + Mongoose services under `online-exam/team-*`, organised as `src/modules/<feature>/<feature>.{controller,service,repository,routes,validation}.ts`.
- **Project 2 (DOCURA):** Node + NestJS + TypeScript + PostgreSQL (TypeORM/Prisma) + Event-Driven modular monoliths under `health-app/team-*`, organised as vertical slices (`@Module`, `@Controller`, `@Injectable`).
- **Project 3 (ENR):** NestJS monorepo + PostgreSQL + Redis + RabbitMQ + gRPC + Stripe microservices (`apps/*` + `libs/*`), where `CC`/`TS`/`JT`/`RS`/`MA` apply inside each deployable and `MS-###` governs cross-service boundaries.

---

## 1. Severity ladder

| Severity | Meaning | Merge policy |
| --- | --- | --- |
| `critical` | Wrong behaviour, security hole, data loss, or silent failure in production paths. | Block. Fix before merge. |
| `major` | Maintainability or correctness-risk debt: broken layering, untyped boundaries, tests that cannot fail. | Block unless the author writes down an accepted-debt note. |
| `minor` | Style, naming, or local clarity. No behaviour risk. | Batch-fix. Never block a merge alone. |

A rule's listed severity is the default. Escalate one level when the violation sits on
an authentication, authorization, payment, refund, grading, or shared-inventory concurrency path.
Never escalate for taste.

---

## 2. Review protocol

Follow these steps in order. Do not skip step 4.

1. **Scope the diff.** Review only files the task names (changed files, a module, a
   team folder). Never review `node_modules`, `dist`, `coverage`, or lockfiles.
2. **Read the catalogs first**, then the code. Findings must map to an existing rule ID.
3. **Detect.** For each file, walk the catalogs. Each rule carries a `Detect:` line with
   an observable heuristic and, where useful, a grep hint. Heuristics are search aids,
   not verdicts.
4. **Verify every candidate by reading the surrounding code.** A grep hit is a
   hypothesis. Confirm the violation actually holds — check the `Not a violation when:`
   clause on the rule before reporting. Discard anything you cannot confirm by reading.
5. **Deduplicate.** One finding per root cause. If one fat controller method triggers
   CC-110, CC-201, and CC-402, report the strongest rule and mention the others in the
   fix line — do not emit three findings for one block.
6. **Rank** by severity, then by blast radius (shared code before a single handler).
7. **Report** in the format below.

---

## 3. Report format

One line per finding, most severe first:

```
<path>:<line> — <RULE-ID> (<severity>): <what is wrong, one sentence>. Fix: <concrete action>.
```

Example:

```
team-b/src/modules/auth/auth.controller.ts:42 — CC-201 (major): controller validates input, hashes the password, writes to Mongoose, and sends mail. Fix: move the workflow into auth.service.register(dto); leave the controller to map dto -> service -> response.
team-b/src/common/middlewares/authentication.ts:11 — TS-101 (major): req.user typed as any, so every downstream consumer is unchecked. Fix: declare an AuthenticatedUser interface and augment Express.Request with it.
team-b/src/modules/booking/booking.service.ts:19 — MA-201 (major): booking module imports availability repository internals directly instead of its public service facade. Fix: inject the exported AvailabilityFacade from the module root and expose a holdSlot capability.
team-c/src/modules/quiz/quiz.test.ts:88 — JT-203 (critical): the test mocks quizService — the unit under test — so it asserts on the mock, not the code. Fix: mock the repository boundary instead and let the real service run.
```

When a whole file or module is missing something (no tests at all, no validation layer),
anchor the finding at line 1 of the most relevant file and say so explicitly.

If nothing survives verification, say exactly that. Do not pad a report with observations
that are not violations.

---

## 4. False-positive rules

Do not report:

- **Style a formatter owns.** Quotes, semicolons, trailing commas, import order, line
  width. Prettier/ESLint territory. Exception: a rule below that explicitly covers it.
- **Hypotheticals.** "This could break if X" without a path in the code that reaches X.
- **Framework idiom.** Express's `(req, res, next)` signature, NestJS module/DI and parameter decorators (`@Module`, `@Injectable`, `@Inject`, `@Body()`, `@Param()`, `@Req()`), Mongoose's `Schema`/`model` pairing, TypeORM/Prisma entity definitions, and Jest/Vitest/Supertest/Testcontainers idioms (`describe`/`it`, `Test.createTestingModule`) are not violations.
- **Test-only shortcuts inside test files** that a rule in `testing-jest.md` explicitly
  permits (fixtures, builders, `as unknown as` on a mock).
- **Pre-existing code untouched by the diff**, unless the task asks for a full audit.
- **Preference rewrites.** `for` vs `map`, `class` vs factory, named vs default export —
  unless a rule names it.

Do report, even when it looks intentional: swallowed errors, `any` on a public boundary,
tests that cannot fail, business logic in a controller, and secrets or config read
directly from `process.env` deep inside a module.

---

## 5. Prompt to launch a review agent

```
Review <TARGET> against the rule catalogs in docs/standards/:
clean-code.md, typescript.md, testing-jest.md, restful-api.md, modular-structure.md
(plus docs/microservices-nestjs-guide.md for Project 3 microservices).

Follow the review protocol in docs/standards/README.md exactly:
read the catalogs, detect, then VERIFY each candidate by reading the
surrounding code before reporting. Apply the false-positive rules.

Output only the finding lines in the documented format, most severe
first. No summary, no praise, no suggestions outside the catalogs.
If nothing survives verification, say so.
```

Swap `<TARGET>` for a diff, a path (`online-exam/team-d/src/modules/quiz`), or a whole
team folder.

---

## 6. Extending the catalogs

- Never renumber an existing rule. IDs are permanent; retire with `Status: retired`.
- New rules take the next free number in their band.
- Every rule needs all seven fields: `Severity`, `Applies to`, `Detect`, `Bad`, `Good`,
  `Why`, `Not a violation when`. A rule without a `Detect` line and an exception clause
  produces noise and does not belong here.
