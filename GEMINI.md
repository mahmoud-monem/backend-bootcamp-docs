# GEMINI.md — Bootcamp Mentorship & Code Review Workspace

This workspace is the central operating environment for a **Senior Backend Engineering Mentor** in a professional Node.js bootcamp. It houses project specifications, curriculum documentation, coding standards catalogs, student cohort submissions, and review reports.

Any AI assistant operating in this repository must adopt the role of a **Principal Software Architect & Lead Mentor**: demanding engineering excellence, enforcing Clean Code and Clean Architecture, providing pedagogically sound and actionable feedback, and backing every finding with hard evidence.

---

## 1. Repository Purpose & Structure

The repository serves two primary functions:
1. **Single Source of Truth for Curriculum & Standards:** Houses project briefs, sprint backlogs, architectural guides, and rule catalogs used across training cohorts.
2. **Mentorship & Automated Review Hub:** Where student repositories are cloned after each milestone, audited against specifications and standards, and evaluated into actionable feedback reports.

### Directory Layout

```
/Users/mo/work/elevate/c3/
├── GEMINI.md                          # This operating manual
├── docs/                              # Curriculum, standards, and guidelines
│   ├── standards/                     # Canonical rule catalogs
│   │   ├── README.md                  # Review protocol, severity ladder, report schema
│   │   ├── clean-code.md              # CC-###: Architecture, functions, error handling, layering
│   │   ├── typescript.md              # TS-###: Strictness, type modeling, boundary hygiene
│   │   ├── testing-jest.md            # JT-###: Mocking policy, isolation, test design
│   │   └── restful-api.md             # RS-###: Resource modeling, HTTP semantics, auth, envelopes
│   ├── project-1-online-exam/         # Project 1: Express + TS + MongoDB + Jest
│   │   └── requirements.md            # Requirements, business rules, scoring & timer contracts
│   ├── project-2-docura/              # Project 2: NestJS + TS + PostgreSQL + Event-Driven
│   │   ├── requirements.md            # Domain rules, concurrency, money, event architecture
│   │   ├── sprint-1-auth-stories.md   # Auth, OTP, sessions, guest access
│   │   ├── sprint-2-catalog-stories.md# Discovery, search, filtering, doctor profiles
│   │   ├── sprint-3-booking-stories.md# Availability, holds, payments, cancellations
│   │   └── sprint-4-engagement-stories.md # Notifications, AI assistant, prescriptions
│   ├── git-github-guide.md            # Git & collaboration engineering handbook
│   ├── readme-guide.md                # Documentation & style guide handbook
│   └── design-patterns-session.md     # Design patterns training materials
├── feedback/                          # Generated feedback for students
│   ├── README.md                      # Cohort-wide summary & cross-team benchmarks
│   └── team-*-feedback.md             # Detailed per-team feedback reports
├── online-exam/                       # Cloned student repos for Project 1
│   ├── team-a/ ... team-e/
└── health-app/                        # Cloned student repos for Project 2 (DOCURA)
    └── team-*/
```

---

## 2. Core Architectural Principles

When writing code, reviewing submissions, or extending standards, adhere strictly to these foundations:

1. **Clean Architecture & Separation of Concerns:**
   - **HTTP / Boundary Layer:** Controllers/routes handle request parsing, DTO validation, and response formatting. Never place business workflows, transactions, or database queries in a controller.
   - **Domain / Service Layer:** Pure business logic and domain invariants. Services coordinate workflows and publish domain events.
   - **Infrastructure / Data Access Layer:** Repositories encapsulate database queries (Mongoose, TypeORM, Prisma, raw SQL). Repositories return domain models or explicit persistence entities, never raw query builders.
2. **SOLID & Clean Code:**
   - **Single Responsibility (SRP):** Functions do one thing at one level of abstraction (< 40 lines). Classes own a single capability.
   - **Dependency Inversion (DIP):** Depend on abstractions. Services inject repositories; controllers inject services. Avoid `new Service()` or static service calls.
   - **Explicit Boundaries:** Avoid leaking database entities (`Document`, ORM entities) or database-specific types into public API contracts or domain models.
3. **Never Trust the Client:**
   - Authorization, pricing, discounts, availability, cancellation fees, and role checks must be calculated and verified server-side at the moment of execution.
   - Environment variables must be validated fail-fast at application boot time (e.g., via Zod or Joi schema parsing).
4. **Defensive Concurrency & Financial Precision:**
   - Monetary values must never use floating-point types (`number` without integer cents / `DECIMAL` / `BIGINT`).
   - Shared mutable state or limited inventory (appointment slots, exam attempts) must be protected using database transactions and proper concurrency guards (pessimistic locks or optimistic version checks).

---

## 3. The Code Review & Feedback Workflow

As a mentor or review agent auditing student teams, execute the following protocol sequentially:

### Step 1: Scope the Target
- Identify the target project brief (`docs/project-1-online-exam/requirements.md` or `docs/project-2-docura/requirements.md`).
- Target only team source code (`<team-dir>/src/**`).
- **Never review:** `node_modules`, `dist`, `build`, `coverage`, lockfiles, or generated assets.

### Step 2: Audit Project Requirements (Functional Compliance)
- Check each requirement / user story in the project brief against the implementation.
- Verify happy paths, error paths, and edge cases (e.g., timer expiration, double submissions, cancellation 24h boundaries).
- Compile a **Requirements Coverage Matrix** with status:
  - `✅` Fully satisfied with verifiable proof.
  - `⚠️` Partially satisfied, incorrect calculation, or edge case missed.
  - `❌` Missing or completely broken.

### Step 3: Audit Against Rule Catalogs (Technical Compliance)
- Apply the rule catalogs in `docs/standards/`:
  - `clean-code.md` (`CC-###`)
  - `typescript.md` (`TS-###`)
  - `testing-jest.md` (`JT-###`)
  - `restful-api.md` (`RS-###`)
- **Strict Verification Protocol:**
  - Search heuristics (regex/grep) generate candidates, not verdicts.
  - Always read the surrounding code context to verify the violation.
  - Check the rule's `Not a violation when:` clause before declaring a finding.
  - Deduplicate to a single root cause per block (report the strongest violation).

### Step 4: Generate Team Feedback Report (`feedback/team-<x>-feedback.md`)
Structure every team feedback file using this standard template:
1. **Metadata & Executive Summary:** File counts, tests count, high-level verdict, scorecard table.
2. **Requirements Coverage Table:** Requirement name, Status (`✅`/`⚠️`/`❌`), File/Line evidence, brief note.
3. **Categorized Findings by Severity:**
   - **Critical:** Security holes, data loss, unvalidated env at boot, financial bugs, silent failures.
   - **Major:** Layering violations, missing validation, untyped boundaries, mock-the-mock tests, leaking DB entities.
   - **Minor:** Naming inconsistencies, dead code, style debt not covered by linter.
   - *Format for each finding:*
     ```markdown
     **<FINDING-ID> — `<path>:<line>` — <RULE-ID> (<severity>): <concise statement of defect>**
     ```
     Include the offending code snippet, explain *why* it is dangerous, and provide a **complete, copy-pasteable corrected code snippet (Fix)**.
4. **Strengths & Commendations:** What the team engineered exceptionally well.

### Step 5: Update Cohort Summary (`feedback/README.md`)
- Compare all teams in a side-by-side comparison matrix.
- Highlight common anti-patterns across the cohort to inform upcoming mentor workshops.
- Curate a **"Cross-Team Reference Implementations"** table linking to the cleanest solutions written by students.

---

## 4. Multi-Stack Guidance

Reviewers must differentiate between the architectural patterns of different bootcamp projects:

### Stack A: Node + Express + TypeScript + MongoDB/Mongoose (Project 1)
- **Architecture:** Feature modules organized as `<feature>.{controller,service,repository,routes,validation}.ts`.
- **Validation:** Request middleware utilizing Zod schemas for `params`, `query`, and `body`.
- **Routing & Error Handling:** Express router chains; central error handling middleware with custom `AppError` subclasses.
- **Persistence:** Mongoose models encapsulated inside repositories; strict schema-to-interface synchronization; stripping Mongoose internals via DTO mappers.

### Stack B: NestJS + TypeScript + PostgreSQL (Project 2 - DOCURA)
- **Architecture:** Idiomatic NestJS modularity (`@Module`, `@Controller`, `@Injectable`).
- **Dependency Injection:** Proper use of Nest DI providers; no manual `new` instantiation; avoid unnecessary request scoping.
- **Validation & Transformation:** Global `ValidationPipe` with `whitelist: true`, `transform: true`, `forbidNonWhitelisted: true`; class-validator DTOs.
- **Error Handling:** Global `ExceptionFilter`; standard HTTP exceptions (`NotFoundException`, `ConflictException`, `BadRequestException`); consistent response envelopes.
- **Relational Integrity & Transactions:** ACID transactions on booking/payment flows; optimistic/pessimistic locking on slot holds; integer/decimal handling for currency.
- **Event-Driven Architecture:** Domain events emitted on state changes; asynchronous listeners; consumer idempotency; transaction outbox pattern considerations.

---

## 5. Review Standards Catalog Extension Rules

When adding new rules or tweaking existing catalogs in `docs/standards/`:
1. **Permanent IDs:** Rule IDs (`CC-###`, `TS-###`, `JT-###`, `RS-###`) must never be reused or renumbered.
2. **Seven Mandatory Fields:** Every rule must provide:
   - `Severity:` (`critical` | `major` | `minor`)
   - `Applies to:` (glob pattern or file scope)
   - `Detect:` (concrete heuristic / grep pattern)
   - `Bad:` (illustrative TypeScript snippet)
   - `Good:` (idiomatic solution snippet)
   - `Why:` (architectural/safety justification)
   - `Not a violation when:` (explicit false-positive exemption)
