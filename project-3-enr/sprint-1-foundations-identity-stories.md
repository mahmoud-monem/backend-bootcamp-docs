# Sprint 1 — Foundations & Identity

**Project:** ENR — Egyptian National Railways Ticket Booking (Bootcamp Project 3)
**Requirements:** [`requirements.md`](./requirements.md)
**Microservices guide:** [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)
**Design:** [Figma — Egyptian National Railways](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
**Jira import file:** [`sprint-1-foundations-identity-stories.csv`](./sprint-1-foundations-identity-stories.csv)

---

## How to read these stories

Each story describes **a flow**: across the real Figma screens for the access stories, and through the hands of a developer, a reviewer or an operator for the platform stories. Then it lists the rules that flow must obey, the events it publishes, the edge cases, and the acceptance criteria you are graded on.

What you will **not** find here: endpoint paths, request bodies, table or column names, queue names, routing keys or `.proto` definitions. Those are your design work, and PLAT-1 is where you produce them. If a story says *"the response tells the app to open Verify your e-mail"*, deciding **how** you express that is part of the exercise.

Events are named in **plain language** (*"a verification code was issued"*) because the event catalogue is your deliverable. In PLAT-1, give each one its real past-tense name, its payload and its version, and use that name everywhere afterwards.

Every story starts its User flow with a **`Services:` line** naming the services involved and the role each plays. The owner of the **first-listed** service is accountable for the story: they answer for it at the demo and review every pull request that touches their service. That does not mean they build it alone (see the suggested split). For the platform stories, the first-listed service is the one whose owner leads the shared work this sprint.

**What is different about this sprint.** You are no longer building one application. You are building seven that must agree with each other, and Sprint 1 is graded on whether they do:

- The first two days produce **no code**. PLAT-1 is a gate: no service code is merged until your mentor signs the design.
- Four of the seven stories have no screen. Every screen in the next four sprints depends on them without saying so.
- PLAT-3 is the first time a business action in one service causes work in another. Get the outbox and the idempotent consumer right here, on a verification email, and Sprint 4 can trust them with refunds.
- Several acceptance criteria require **stopping something** in the middle of a flow: notification, RabbitMQ, Redis, Postgres. You prove those criteria by demoing them on the running Compose stack, not by explaining why they would work.

---

## Importing into Jira

**Jira → Settings → System → External System Import → CSV**, then map:

| CSV column | Jira field |
|---|---|
| Issue Type | Issue Type |
| Issue Key | *do not map* (local reference only) |
| Summary | Summary |
| Description | Description |
| Acceptance Criteria | Acceptance Criteria custom field, or append to Description |
| Priority | Priority |
| Labels | Labels (split on comma) |
| Story Points | Story point estimate |
| Epic Name | Epic Name (company-managed only) |
| Epic Link | Parent (team-managed) / Epic Link (company-managed) |
| Component | Components |
| Sprint | Sprint |

Keep the file order — the Epic row must import first. Dry-run on a sandbox project; a CSV import cannot be undone in bulk.

---

## Sprint at a glance

**7 stories · 33 points**

| # | Story | Points |
|---|---|---|
| PLAT-1 | Design gate — the service map, contracts and cache plan your mentor signs before any service code | 5 |
| PLAT-2 | Platform skeleton — seven services that boot, report their health and speak one language of errors | 5 |
| PLAT-3 | Messaging backbone — events that survive a crash, arrive once, and are never silently dropped | 5 |
| AUTH-1 | The gateway — one public door, tokens checked locally, and an honest answer when a service is down | 5 |
| AUTH-2 | Sign up and email verification — a code that arrives within 30 seconds, every time | 5 |
| AUTH-3 | Sign in, staying signed in, signing out and recovering a password | 5 |
| AUTH-4 | PIN and Touch ID — a quick unlock that is worthless on any other phone | 3 |

> **Three build days.** Days 1–2 are the design gate, which leaves three days for 28 points. If capacity is tight, AUTH-4 is the story to move to Sprint 2, because nothing else in this sprint depends on it. PLAT-2, PLAT-3 and AUTH-1 cannot move: every story in the next four sprints depends on them.

### Suggested order

```
PLAT-1  Design gate (days 1–2, whole team)  ── no service code merges before sign-off
   │
   └─▶ PLAT-2  Platform skeleton             ◀── first thing on day 3: everything stands on it
          │
          ├─▶ AUTH-1  Gateway                ── the only door; verifies what AUTH-3 issues
          │
          └─▶ PLAT-3  Messaging backbone     ── outbox, inbox, retry, dead-letter
                 │
                 └─▶ AUTH-2  Sign up + verification   ◀── the first real event: the code email
                        │
                        └─▶ AUTH-3  Sign in, sessions, forgot password  (reuses AUTH-2's codes)
                               │
                               └─▶ AUTH-4  PIN + Touch ID
```

**Close PLAT-1 by the end of day 2, and merge a thin PLAT-2 on the morning of day 3**: seven services that boot and answer their health checks, so the rest of the sprint has something to build on by lunchtime. Build **PLAT-3 and AUTH-2 side by side**, because the verification email is both PLAT-3's first real event and AUTH-2's whole promise. Agree on what the access token contains in PLAT-1, not on day 4, because AUTH-1 verifies what AUTH-3 issues.

### Suggested split

Five students, each owning one or two services (requirements §2.2). Ownership decides who **reviews**; the split below decides who **builds** this sprint. Roughly 6–8 points each:

| Student | Owns | Takes this sprint | ≈ Points |
|---|---|---|---|
| S1 | gateway + identity | PLAT-1 (gateway and identity parts, the endpoint table, the error catalogue) · AUTH-1 · from PLAT-2: the envelope, language selection and the correlation ID's origin at the gateway | 7–8 |
| S2 | catalog | PLAT-1 (catalog part) · PLAT-2 (monorepo, Compose, health and readiness, configuration validation, logging, graceful shutdown) · AUTH-4 | 7–8 |
| S3 | inventory | PLAT-1 (inventory part) · AUTH-3 | 6 |
| S4 | booking | PLAT-1 (booking part, plus assembling the gate: the saga diagram and the five dangerous edge cases) · AUTH-2 | 6–7 |
| S5 | payment + notification | PLAT-1 (payment and notification parts) · PLAT-3 (the shared library, and notification as its first consumer) | 6 |

S1 owns identity but builds none of AUTH-2, AUTH-3 or AUTH-4, and **reviews every one of their pull requests**. That is the ownership model working as intended: the owner of a service reviews every PR that touches it and is accountable for that service's design, README, contracts and tests, whoever typed the code.

### Definition of Done (every story)

- Every rule is enforced **in the service that owns it**: never by the app, and never by the gateway on another service's behalf.
- Every acceptance criterion has a test, including the negative, boundary and, where stated, **concurrent** ones. Integration tests run against real Postgres, Redis and RabbitMQ (Testcontainers), one database per service, never a shared test database.
- Every edge case listed has a deliberate, defined behaviour.
- Events leave through the **outbox**, carry the event envelope, and are consumed **idempotently**. A failing consumer retries with backoff and then dead-letters: it never undoes the producer's change and never drops the message.
- Every synchronous call has a **deadline**, and only idempotent calls are retried.
- Every Redis key the story writes is in the cache plan with its owner, TTL, what removes it, and what happens when Redis is down.
- No password, verification code, token, PIN, card data, full phone number or full National ID appears in any log of any of the seven services.
- Errors use the gateway's envelope and catalogue codes, with Arabic and English messages selected by `Accept-Language`.
- `.proto` files and event schemas live in `libs/contracts`, change in the same pull request as the code, and are covered by contract tests. The design-gate artifact changes with them.
- It works in `docker compose up` and is demoed against the running stack, not only in unit tests.
- The owner of every service the pull request touches has reviewed it.
- **This sprint:** `docker compose up` from a clean clone brings up all seven services healthy, and the design-gate artifacts are in the repository and signed off by the mentor.

---

## EPIC-1 — Epic: FOUNDATIONS — Platform, Gateway & Identity

This epic is the base every later sprint builds on. It covers the design your mentor signs
before any service code, a monorepo of seven services that starts with one command, the
messaging backbone every event will travel on, and the gateway that is the system's only
door. It also covers how a traveller gets in: sign up, email verification, sign in,
sessions, password recovery, and the PIN that unlocks one phone.

Source of truth: the Figma Start and Access screens (Splash, Choose Language, Onboarding 1–3,
Sign up, Password doesn't match, Verify your e-mail counting down and at 00:00, Sign in,
Create PIN, Confirm PIN, Enter your PIN and the Touch ID prompt), and the Presentation page's
research on code delivery and biometric login. Also sections 2, 3, 4 (Module 1), 5.1, 5.10,
6.1, 7, 8 and 9 and Appendix C of the requirements document.

Dependencies: none, because this is the first sprint. Every later sprint depends on it. Sprint 2's
Complete your profile unlocks the "bookable traveller" level defined here. The outbox built
here carries every seat, payment and refund event. Sprint 4's payments go through the
gateway built here.

The sprint opens with two days of design and no code. That time is not wasted: these are the
cheapest two days of the project in which to be wrong.

**Epic acceptance criteria**

- The design-gate artifacts are in the repository and signed off by the mentor, and no service code was merged before the sign-off.
- `docker compose up` from a clean clone brings up all seven services and their infrastructure, healthy.
- A visitor can choose a language, sign up with an email and a password, receive the code in that language within 30 seconds, verify, and land signed in.
- A traveller can sign in, stay signed in across restarts, sign out and recover a forgotten password, and is signed out everywhere after the recovery.
- A traveller can unlock the app with a PIN or Touch ID on the phone they set it up on, and nowhere else.
- Every request enters through the gateway, tokens are verified there without calling identity, and a route nobody opened is closed.
- The first event crosses services through the outbox, survives a stopped consumer and a stopped broker, and is delivered once.
- One correlation ID follows a request through HTTP, gRPC and RabbitMQ.
- Every error is a stable code with an Arabic and an English message, and no password, code, token or PIN appears in any log.

---

## PLAT-1 — Design gate — the service map, contracts and cache plan your mentor signs before any service code

**5 points · Priority Highest · `design-gate,architecture,gateway,identity,catalog,inventory,booking,payment,notification,sprint-1`**

### User story
As a team of five service owners about to build seven services in parallel
I want the whole system designed, written down and reviewed before anyone writes service code
So that our services fit together the first time, and we make our mistakes on paper while they are cheap to fix

### User flow (see Figma)
Services: booking (accountable: its owner assembles the gate package and draws the booking saga), gateway (the endpoint table and the error catalogue), identity, catalog, inventory, payment, notification (every owner contributes the ERD, the `.proto` methods, the events produced and consumed, and the cache keys of their own service).

There is no screen to build. The Figma is the input: every screen on the Final UI page, plus
the research on the Presentation page that explains why those screens look the way they do.

  Day 1, morning     the whole team walks every Appendix C screen, in order, and asks of each:
                     what does the app need here, which service owns that data, and who else
                     needs to know when it changes?
  Day 1, afternoon   the service map and the event catalogue, drafted together
  Day 2, morning     each owner writes their service's part: ERD and constraints, .proto
                     methods with deadlines and errors, cache keys. The gateway owner writes
                     the endpoint table and the error catalogue; the booking owner draws the saga
  Day 2, afternoon   cross-check (below) -> mentor review -> fixes -> sign-off recorded
  Day 3 onwards      code, against contracts everybody has already agreed

The cross-check is deliberately mechanical: every screen reaches an endpoint, every endpoint
reaches a service, every event has a producer and consumers, every cached key has an owner, and
every product ruling has a home.

### Rules and validations
- The gate: No service code is merged before the mentor signs off. Spikes on a branch are fine; merging them is not. The sign-off is recorded in the repository: who signed, when, and against which commit.
- In the repository, as source: Every artifact lives in the repository under one documented location, and diagrams are kept as text so that a change shows up in a diff. A photo of a whiteboard is a starting point, not an artifact.
- Service map (§2.2, §8): All seven services, the data each owns, every synchronous call (caller → callee, method) and every event (producer → consumers). The split in §2.2 is given and the map does not contradict it. How each service works inside is your design.
- ERD per service (§7.3): One diagram per service with the cardinalities inside it. References to another service's data are drawn differently: as an ID, never as a foreign key. Every constraint you rely on for correctness (unique, check, exclusion, foreign key) is listed with what it protects against, and each of the nine questions in §7.2 has a short written answer.
- Event catalogue (§8): For every event: a past-tense name, the producer, when it is published, its payload, its version, every consumer and what each consumer does with it. Every event has exactly one producer. An event with no consumer is either deleted or carries a written reason for existing.
- Contracts (§8): A `.proto` contract in `libs/contracts` for every synchronous call on the service map, with each method's deadline and the errors it can return. A call that is on the map but not in a contract, or in a contract but not on the map, fails the gate.
- Gateway endpoint table (§8): Method, path, purpose, access level (public, traveller, bookable traveller, operator, conductor — BR-01) and the services each endpoint calls, for every screen in Appendix C. That includes the three rows with no design: the notifications inbox, the operations API and conductor ticket verification. The public endpoints are exactly BR-01's list and no others.
- Error catalogue (§8): A stable code for every failure the app must react to differently, with its HTTP status and its Arabic and English messages (BR-56, BR-57). It starts with the examples in §8 and grows every sprint.
- Booking saga (§8, §7.2): A sequence diagram covering the happy path, a declined payment, a hold that expires, a payment that succeeds after the hold expired (BR-36) and a crash in the middle. Every compensation is named as the call or event that performs it. The diagram also states where the saga's state is persisted and what resumes the saga after a crash.
- Cache plan (§2.3): For every Redis key: its owner, contents, TTL, what invalidates it and what happens when Redis is unavailable. It covers every row of §2.3's required uses, and every key name is versioned. It also records the two decisions this sprint cannot start without: what "fail safe" means for sign-in limits, and what the gateway does when it cannot read the revoked-token list.
- The five most dangerous edge cases (§6): The team chooses them and explains each one across services, naming the call or event that handles it. "The other service handles it" is not an answer.
- Every ruling has a home (§9): For each of D1–D14, the service and the module, constraint or scheduled job that enforces it.
- The five decisions of §2.2: For each multi-service flow, whether it is orchestrated or choreographed. What crosses the wire. What each service copies from others and how stale that copy may be. For each of the seven services, what happens when it is down: what still works, what degrades, what fails and what the user sees. And where each cross-service invariant lives, starting with BR-27.
- Decided once, now: Money is stored as integer piastres in every ERD that holds money, and all business times are Africa/Cairo (BR-14). Write down where the conversions happen. Deciding either of these in Sprint 4 means a migration.
- Owned, signed and kept current: Every section names the student who owns it. Each owner approves the parts that describe their service, the whole team approves the cross-service parts, and the mentor signs the whole. After sign-off, a contract change starts as a pull request against the artifact, reviewed by every owner whose service it touches, and the code follows. An artifact that no longer matches the code is worse than none.

### Domain events to publish
- None. This story writes the catalogue that every later story publishes from. That includes the events this sprint names in plain language (PLAT-3, AUTH-2, AUTH-3, AUTH-4), which get their real names, payloads and versions here.

### Edge cases to handle
- Two owners both believe they own a piece of data, for example the National ID's uniqueness among account holders (identity) and one National ID per seat on overlapping segments (BR-27). Exactly one service enforces each invariant.
- A screen needs data from several services. The Tickets tab card shows the train, the passenger, the coach and seat, and the refund status. Decide whether the gateway aggregates or one service keeps a copy, and what that copy costs when the original changes.
- An event has a consumer but no producer, or a producer but no consumer. The cross-check should find this now, not Sprint 4.
- The mentor rejects part of the design on day 2. Decide what can start on day 3 and what has to wait, and record that decision with the sign-off.
- In Sprint 3 a contract turns out to be wrong. The change goes through the artifact first, so every consumer of that contract learns about it in review rather than from a failing demo.

### Acceptance criteria

- Given the repository at the sign-off commit, when a reviewer opens the documented location, then the service map, an ERD per service, the event catalogue, the `.proto` contracts, the endpoint table, the error catalogue, the saga diagram, the cache plan, the five edge cases and the rulings table are all there, and every diagram is kept as source.
- Given Appendix C, when each row is checked against the endpoint table, then every screen maps to at least one endpoint with its method, path, access level and the services it calls. This includes the notifications inbox, the operations API and conductor ticket verification.
- Given the endpoint table, when the public endpoints are listed, then they are exactly the exceptions in BR-01, and every other endpoint names its access level.
- Given the event catalogue, when each event is checked, then it has a past-tense name, exactly one producer, a version, a payload, and at least one consumer with what that consumer does, or a written reason for having none.
- Given the service map and `libs/contracts`, when they are compared, then every synchronous call on the map has a `.proto` method with a deadline and its errors, and every method in the contracts appears on the map.
- Given the cache plan, when each key is checked, then it has an owner, contents, a TTL, an invalidation trigger and a Redis-down behaviour, its name is versioned, and every row of §2.3's required uses is covered.
- Given the cache plan, when the entries for sign-in limits and the revoked-token list are read, then each states what happens when Redis is unavailable, and neither answer is "unlimited".
- Given D1 to D14, when each ruling is looked up, then it names the service and the module, constraint or job where it is enforced.
- Given each service's ERD, when it is reviewed, then references to other services are drawn as IDs and not foreign keys, every correctness constraint states what it protects, money is in integer piastres, and the nine questions of §7.2 are answered.
- Given the saga diagram, when it is reviewed, then it shows the happy path, a decline, a hold expiry, a late payment and a crash in the middle, with every compensation named as a call or an event.
- Given the five dangerous edge cases, when each is read, then it names the calls and events that handle it in every service involved.
- Given the §2.2 question "what happens when each service is down", when the answer is read, then all seven services are covered, each with what works, what degrades, what fails and what the user sees.
- Given the error catalogue, when it is read, then every code has an HTTP status and both an Arabic and an English message, and the examples from §8 are present.
- Given the repository history, when it is inspected, then no code under `apps/` was merged before the recorded sign-off, and every artifact section names its owner.

---

## PLAT-2 — Platform skeleton — seven services that boot, report their health and speak one language of errors

**5 points · Priority Highest · `platform,docker-compose,observability,gateway,identity,catalog,inventory,booking,payment,notification,sprint-1`**

### User story
As a developer on a five-person team
I want one repository in which `docker compose up` brings up all seven services, healthy, with the same envelope, errors, logs and correlation ID everywhere
So that every story from here on is built and demoed on the real system, and a problem in any service can be traced across all of them

### User flow (see Figma)
Services: catalog (accountable: its owner leads the shared skeleton, because catalog's own code starts in Sprint 2), gateway (the response envelope, language selection, and the point where the correlation ID is created), identity, inventory, booking, payment, notification (each one boots, validates its configuration, reports its health, logs and shuts down the same way).

No screen. This is the developer's flow, and the reviewer's:

  clone the repository -> follow the README, and nothing else
  -> docker compose up
       Postgres, Redis, RabbitMQ and Mailpit start and report healthy
       then the seven services start, each only after what it depends on is healthy
  -> the gateway's health check answers, and every service reports ready
  -> send any request through the gateway with Accept-Language: ar
       -> the answer is the envelope, with an Arabic message and a correlation ID
  -> search the logs of every service for that correlation ID
       -> every line the request produced, in every service, with no personal data in any of them

And the unhappy version of the same flow, which is the one you will be asked to demo:

  remove one variable from identity's configuration -> identity refuses to start and says which one
  stop Postgres  -> the services that need it report not ready, and recover on their own
  stop Redis     -> the services keep working, more slowly, and say so
  stop a service -> it finishes what it was doing and exits cleanly

### Rules and validations
- One monorepo, slices inside: A Nest monorepo with `apps/<service>` for the seven services and `libs/` for shared code (§2.5). Inside each service, use vertical slices: transport stays thin, rules live in domain services, and persistence lives in repositories (§2.4).
- Shared libraries are cross-cutting only: `libs/` holds contracts, messaging, observability and common code (envelope, error codes, configuration helpers), never domain logic. No app imports another app, and the build fails if one does. If two services import the same domain class, they are one service pretending to be two.
- A database and credentials per service: One Postgres container is fine, but each service has its own database and its own credentials, and those credentials cannot open any other service's database (§2.1). Each service owns its migrations and runs only its own.
- Compose is the platform: `docker compose up` starts Postgres, Redis, RabbitMQ, Mailpit and the seven services. Only the gateway is reachable from outside the Compose network. Developer consoles such as Mailpit's inbox may be published for local use; the services never are. The Stripe CLI forwarder joins in Sprint 4 and Jaeger in Sprint 5.
- Health, readiness, start order: Every service exposes liveness (the process is alive) and readiness (it can do its job: it can reach its database, the broker, and Redis if it uses Redis). Compose starts each service only once the infrastructure it needs is healthy. The gateway does not refuse to start because a downstream service is down; it reports that service as down and answers for it (AUTH-1).
- Redis down is degraded, not dead: A service that uses Redis reports Redis as unavailable in its health check and its logs, and keeps doing whatever its cache plan says it can still do without Redis (§2.3 rule 7).
- Configuration fails fast: Every service validates its whole environment against a schema at boot and refuses to start on a missing or malformed variable. The error names the variable and never prints a secret's value (§2.1). Every DOCURA team failed this one: a service started with a missing key and then failed on the first real request, in front of a reviewer. There are no defaults for secrets. An example environment file is committed; real secrets are not.
- One envelope: Every response through the gateway, success or error, has the same envelope. An error carries a stable, machine-readable code and a message. For a validation failure it also says which field and which value were wrong and what to do next (BR-57). Never "Something went wrong".
- Messages selected by code and language: Each code has an Arabic and an English message in one catalogue, and the gateway picks one using `Accept-Language` (`ar` or `en`, so `en-GB` counts as English; BR-56). A missing or unsupported language falls back to Arabic, the one documented default, applied everywhere. Adding a translation never touches business code (D14).
- Nothing internal leaks: No stack trace, SQL error, gRPC detail string or third-party message ever reaches the client (§2.4). An unexpected failure becomes a generic internal code in the envelope, and the full detail goes to that service's log under the correlation ID.
- Validation at every boundary: At the gateway, DTOs are checked by a global `ValidationPipe` with `whitelist` and `forbidNonWhitelisted`. gRPC and event payloads are validated again by each service that receives them (§2.1).
- One correlation ID, everywhere: The gateway generates a correlation ID for every request. A value sent by the client never replaces it. The gateway returns the ID in the response and passes it on in gRPC metadata and in AMQP message headers. A consumer that publishes a follow-up event keeps the same ID. Every log line carries it.
- Structured logs without personal data: One JSON line per log entry, with the service, level, correlation ID and message. No password, verification code, token, PIN, card data, full phone number or full National ID (§2.4), and email addresses are masked. Logging a whole request body is how a password ends up in a log, so do not do it.
- Graceful shutdown: On a stop signal a service first reports itself not ready and stops taking new work. It finishes in-flight HTTP and gRPC requests and the message it is handling, acknowledging only messages it finished. Then it closes its connections and exits within a documented time.

### Domain events to publish
- None of its own. This story makes the correlation ID travel in AMQP headers, and PLAT-3 sends the first event with it.

### Edge cases to handle
- A variable is present but malformed: a port of "abc", a URL with no scheme, a boolean written as "yes".
- Postgres takes forty seconds to initialise seven databases on the first start. The services wait instead of crash-looping.
- Redis is down when a service boots, versus Redis disappearing while the service runs.
- A request arrives while the service that should handle it is shutting down.
- A teammate's machine is not on Cairo time. Nothing in the platform depends on the host's timezone.
- A developer logs an incoming request body "just to debug", and it contains a password.

### Acceptance criteria

- Given a clean clone on a machine with only Docker, when the README is followed and `docker compose up` runs, then Postgres, Redis, RabbitMQ, Mailpit and all seven services report healthy without any manual step.
- Given one required variable is removed from a service's environment, when the service starts, then it refuses to start and the error names the variable. The same applies to a malformed value, and no secret's value appears in the output.
- Given one service's database credentials, when they are used to connect to another service's database, then the connection is refused.
- Given the running stack, when a service's port is called from outside the Compose network, then it cannot be reached. Only the gateway answers.
- Given Postgres is stopped, when readiness is checked, then the services that use it report not ready while liveness stays up, and when Postgres returns they become ready again without a restart.
- Given Redis is stopped while the stack runs, when the services keep receiving requests, then those that use Redis report the outage in their health checks and logs and keep serving what their cache plan allows.
- Given the stack is restarted five times in a row, when the logs are read, then no service crashed because a dependency was not ready yet.
- Given a request body with a field the endpoint does not declare, when it is sent, then it is rejected in the envelope with that field named.
- Given the same failing request sent with `Accept-Language: ar`, with `en` and with an unsupported language, when the responses are compared, then the code is identical in all three and the message is in Arabic, in English and in the documented default.
- Given a service throws an unexpected error, when the client reads the response, then it gets the envelope with a generic internal code and no stack trace, SQL or gRPC detail, and that service's log has the full error under the request's correlation ID.
- Given a request that crosses the gateway, a gRPC call and a RabbitMQ message, when the logs of all services are searched for its correlation ID, then every line the request produced is found in every service, and the client received the same ID.
- Given a request that carries a correlation ID made up by the client, when it is processed, then the gateway's own ID is used everywhere.
- Given a full sign-up and sign-in have been run, when every service's logs are searched, then no password, code, token, PIN or unmasked email address appears.
- Given a service is stopped while a request is in flight, when it shuts down, then that request completes, no new request is accepted, and the process exits within the documented time.
- Given code in one app imports from another app, when the project is built, then the build fails.

---

## PLAT-3 — Messaging backbone — events that survive a crash, arrive once, and are never silently dropped

**5 points · Priority Highest · `messaging,outbox,rabbitmq,identity,notification,sprint-1`**

### User story
As a service owner whose service must tell other services what happened
I want one shared, tested way to publish and consume events
So that a crash, a duplicate or a stopped service never loses a message, starting with the message users complained about most: the verification code

### User flow (see Figma)
Services: notification (accountable: the first consumer, which deduplicates, retries, dead-letters and sends the email to Mailpit), identity (the first producer, which records the event in the same transaction as the code), and the shared messaging library in `libs/` that every service uses from now on.

The Figma screen is AUTH-2's "Verify your e-mail", which says "We Have Sent Code To Your Email".
That sentence is a promise the backend has to keep after the response has already gone back
to the app. The research measured how badly the old app kept it: 3 of 4 users could not find
the code. The redesign's target is "OTP delivery 100% within 30s". This story is how you keep
that promise, and how you prove you kept it.

  the traveller taps Sign up (AUTH-2)
  -> identity saves the account and the code's hash, and in the SAME transaction records
     the event "a verification code was issued"
  -> identity answers the app: it does not wait for the email, and it never calls notification
  -> the relay publishes the recorded event to RabbitMQ
  -> notification receives it, checks whether it has already handled this event ID,
     renders the email in the account's language, sends it to Mailpit, records the event as
     handled, and only then acknowledges the message
  -> the email is in Mailpit within 30 seconds of the code being issued, and that time is measured

The demo for this story is the unhappy path. Stop notification, sign up, start notification,
and watch the email arrive. Publish the same event twice, and watch exactly one email arrive.

### Rules and validations
- The event is part of the change: The event is written in the same database transaction as the business change it describes. If the change rolls back, the event never existed. If it commits, the event will be published even if the service crashes the next instant. Publishing to RabbitMQ inside or before the transaction is the "dual write" bug this story exists to remove.
- The relay: The relay publishes recorded events to RabbitMQ and marks each one published only after the broker has confirmed it. Delivery is at-least-once. More than one instance of a service may run, so decide how two relays avoid competing for the same events. Duplicates are tolerated downstream; lost events are not.
- The event envelope: Every event carries an ID, a type, a version, the time it occurred, the correlation ID and the producer, plus its payload (§10.C). The schema lives in `libs/contracts`, and a contract test fails the build when a producer changes the schema in a way a consumer cannot read.
- Producers do not know their consumers: identity publishes "a verification code was issued" and does nothing else. It has no notification client, no email template and no SMTP setting. Adding a consumer later means adding a queue, not changing identity.
- Every consumer has its own queue: When two services are interested in the same event, each receives its own copy instead of competing for one. Exchanges, queues and dead-letter queues are declared in code by their owners. Nothing is created by hand in the RabbitMQ console.
- Consumers are idempotent: A consumer records the IDs of the events it has handled in its own database and skips any event it has already handled. The same event delivered twice produces one effect, which here means one email (BR-54).
- Acknowledge after, not before: A message is acknowledged only once its handling is finished. If a consumer crashes mid-handling, it receives the message again.
- Retry, then dead-letter, never drop: A temporary failure, such as SMTP not answering, is retried with growing delays a bounded number of times, and then the message goes to a dead-letter queue with the reason. A message that can never succeed (an invalid payload or an unknown version) goes to the dead-letter queue at once instead of being retried forever. A poison message never blocks the messages behind it. Replaying the dead-letter queue arrives in Sprint 5, but dead-lettered messages must be replayable from now on.
- The code crosses the wire once: The verification code travels in the event because notification has to put it in the email. identity stores only the code's hash. notification never stores the code (codes are email-only and never go in the inbox, BR-53), and neither service logs it. Any code left in a dead-lettered message stops working within 10 minutes.
- A dead code is not sent: If a code has already expired by the time notification handles its event, notification does not email it and records the event as skipped. An email containing a dead code is worse than no email.
- Measured within 30 seconds: For every verification email, record the time from the code being issued to Mailpit accepting the email. Report the 30-second target (BR-03) as the worst case and the 95th percentile over a run. Choose how quickly the relay picks up new events, and the retry delays, with that budget in mind.
- The email speaks the account's language: The email is rendered in the language stored on the account (BR-56). There is no request to read the language from, so it must come from the event itself or from notification's own copy of the account.
- Down is not broken: If notification is down, sign-up still succeeds at normal speed, and the email is delivered when notification returns. If RabbitMQ is down, sign-up still succeeds, and the event is relayed when RabbitMQ returns.

### Domain events to publish
- A verification code was issued, by identity, either for email verification (AUTH-2) or for a password reset (AUTH-3). notification consumes it and emails the code in the account's language. Every later event in the catalogue travels on this same backbone.

### Edge cases to handle
- RabbitMQ is down at the moment the sign-up commits.
- identity crashes after the commit and before the relay has published the event.
- notification crashes after Mailpit accepted the email but before it recorded the event as handled. Decide which way you fail, a rare duplicate email or a rare missing one, and say why. For a verification code, which is worse?
- Mailpit refuses connections for five minutes, then comes back.
- An event arrives with a version notification does not know, or with a required field missing.
- notification is down for twenty minutes, and the codes issued in that window have expired by the time it returns.
- Two instances of notification receive the same duplicated event at the same instant.

### Acceptance criteria

- Given a sign-up, when it commits, then exactly one "verification code was issued" event is recorded in identity in the same transaction. Given the transaction is forced to fail after the event was written, then no event exists.
- Given RabbitMQ is stopped, when a traveller signs up, then sign-up succeeds, and when RabbitMQ is started again the email arrives without anybody doing anything.
- Given identity is killed right after a sign-up commits and before its event is published, when identity restarts, then the event is published and the email arrives.
- Given notification is stopped, when a traveller signs up, then sign-up succeeds as fast as usual, and when notification is started again within the code's validity the email arrives.
- Given the same event is delivered twice, when both deliveries are processed, then Mailpit holds exactly one email.
- Given two notification instances receive the same duplicated event at the same instant, when both finish, then Mailpit holds exactly one email, proven by a concurrent test.
- Given Mailpit is unavailable, when an event is handled, then it is retried with growing delays and delivered once Mailpit returns. Given Mailpit never returns, then after the documented number of attempts the message is in the dead-letter queue with its reason.
- Given a malformed event or one with an unknown version, when it arrives, then it goes straight to the dead-letter queue without retries, and the messages behind it are still processed.
- Given notification returns after a code has expired, when it handles that code's event, then no email is sent and the skip is recorded.
- Given 50 sign-ups against the running Compose stack, when their emails are collected from Mailpit, then every one arrived within 30 seconds of its code being issued, and the worst case and the 95th percentile are recorded for the demo.
- Given an account whose language is Arabic, when its verification email arrives, then it is in Arabic. Given an English account, then it is in English.
- Given any event on the broker, when it is inspected, then it carries the full envelope (ID, type, version, time, correlation ID, producer), and the correlation ID matches the request that caused it.
- Given identity's code, when it is reviewed, then it contains no reference to notification, emails or SMTP. Given the event's schema is changed incompatibly, then the contract test fails the build.
- Given the logs of identity, notification and the message handling, and notification's database, when they are searched, then no verification code is found.

---

## AUTH-1 — The gateway — one public door, tokens checked locally, and an honest answer when a service is down

**5 points · Priority Highest · `gateway,auth,security,redis,sprint-1`**

### User story
As a traveller using the ENR app
I want every request to go through one door that knows who I am, lets me in only where I am allowed, and tells me plainly when something behind it is unavailable
So that my account is protected and the app never hangs

### User flow (see Figma)
Services: gateway (accountable: the only public entry point, which verifies tokens, enforces access levels, applies rate limits, routes over gRPC and maps errors into the envelope), identity (signs access tokens with its private key and makes the public key available to the gateway).

This story has no screen of its own; every screen in the app reaches the backend through it. From the app's side:

  the app calls /api/v1/... with or without an access token
  -> the gateway checks the rate limit for this client and this route
  -> the gateway verifies the token itself, with identity's public key and no call to identity
  -> the gateway checks that the token is not on the revoked list (signed out)
  -> the gateway checks the route's access level against the token
  -> the gateway calls the owning service over gRPC, with a deadline, passing who the caller is
  -> the service answers, fails, or does not answer in time
  -> the gateway answers the app in the envelope, in the app's language

Access levels (BR-01):
  public               sign up, verify email, resend code, sign in, refresh, forgot password,
                       reset password, PIN sign-in, opening a shared ticket link, health checks
  traveller            signed in with a verified email: everything not listed elsewhere
  bookable traveller   a traveller whose profile is complete (BR-10, D4): holds, payment, tickets
  operator             the operations API
  conductor            ticket verification

### Rules and validations
- The only door: The gateway is the only service the app can reach, and it serves the public API under `/api/v1` (§2.1). The services can only be reached from inside the Compose network (PLAT-2).
- Tokens verified locally: identity signs access tokens with RS256, using a private key that only identity holds. The gateway verifies every token with the public key, checking the signature, that the expected algorithm was used, the expiry, the issuer and the audience, without calling identity per request. A token that claims any other algorithm, including "none" or a symmetric one, is refused.
- Keys can rotate: Each token identifies the key that signed it, so identity can introduce a new key without signing everybody out. Tokens signed with the previous key keep working until they expire.
- What a token carries: The account, its access level and roles, the session, and a unique token ID. Never personal data: anyone holding a token can read it, so an email address inside it is a leak.
- Deny by default: Every route declares its access level. A route that declares nothing is closed, and a test that walks every route proves it. The public routes are exactly BR-01's list. An unverified account never receives a token (BR-05), so "traveller" always means a verified email.
- Bookable traveller is enforced once: The level exists from this sprint, even though nothing makes a traveller bookable until AUTH-5 in Sprint 2. A traveller who is not bookable is refused with the code that sends the app to "Complete your profile" (BR-10). That check happens in one place and is never repeated in each handler (D4). Decide how the gateway learns that a profile is complete, and what it costs if that information is stale.
- Staff roles: Operator and conductor are roles on staff accounts, carried in the token. The seed creates one operator account and one conductor account. A traveller's token never reaches an operator or conductor route, and staff routes are deny-by-default like everything else.
- Who the caller is, inside: The gateway passes the verified caller to services in gRPC metadata, and strips anything in the incoming request that looks like that internal metadata. A service never trusts an identity claim that came from outside, and the network stops outsiders from calling services directly.
- 404, not 403 (BR-09): Ownership is decided by the service that owns the record, because only that service has the data. "Not yours" and "does not exist" reach the app as the same 404 with the same body. A 403 would reveal that the record exists.
- Rate limits shared by every instance: Request limits are counted in Redis, so two gateway instances enforce one limit, not two. Public routes are limited per client, especially sign in, sign up, resend, forgot password and PIN sign-in. Signed-in routes are limited per account. A request over the limit gets a 429 with a retry hint (§10.E). These limits protect the system. The business limits of BR-04 belong to identity (AUTH-2, AUTH-3), not to the gateway.
- Revoked tokens (BR-06): Signing out puts the access token's ID on a revoked list in Redis. The gateway owns that list and checks it on every authenticated request. Each entry lives only until its token would have expired anyway, so the list never grows without bound. When identity revokes a whole session or a whole account (refresh-token reuse, password reset), those access tokens are refused within seconds. identity cannot write into the gateway's keys, so design how the gateway finds out.
- Redis unavailable: Decide, and document in the cache plan, what the gateway does for rate limits and for the revoked list when Redis is down. Sign-in rate limiting fails safe, never open (§2.3 rule 7). "Unlimited" is not an acceptable choice for any of them. The gateway's health check reports the degradation.
- Every call has a deadline: Each gRPC call uses the deadline set for that method in the contracts (PLAT-1). Only idempotent calls are retried, and only within the deadline.
- Errors mapped deliberately: A documented mapping turns each gRPC status into an HTTP status, and the service's error code passes through unchanged into the envelope (§10.A). A service that is down produces a "temporarily unavailable" code (503) with a retry hint. A service that missed its deadline produces a "timed out" code (504). Neither ever shows gRPC internals, and both are rendered in the app's language.

### Domain events to publish
- None of its own this sprint. If you choose events as the way the gateway learns that a session or an account was revoked, the gateway consumes them (AUTH-3).

### Edge cases to handle
- identity is down. Signed-in travellers keep using every other service, because tokens are verified locally. Sign in and sign up answer "temporarily unavailable".
- identity rotates its signing key while thousands of tokens are in use.
- The clocks of the identity and gateway containers differ by a few seconds. Decide the tolerance.
- A token is re-signed with HS256 using the public key as the secret (the classic algorithm-confusion attack).
- A downstream service accepts the connection and then never answers.
- A client sends a header that imitates the gateway's internal caller metadata and names another account.
- Redis restarts while a signed-out token is still within its lifetime.

### Acceptance criteria

- Given a valid access token, when a signed-in request is sent, then it reaches the owning service and identity receives no call for it. Prove it by stopping identity and seeing the request still succeed.
- Given an expired, tampered, wrongly signed, "none"-algorithm or HS256-re-signed token, when it is sent, then it is refused with 401 and a code that tells the app to refresh or sign in.
- Given a token signed with the previous key after a key rotation, when it is sent before it expires, then it is accepted.
- Given every route the gateway exposes, when a test walks them, then each one declares an access level, and the public ones are exactly BR-01's list.
- Given a traveller's token, when it calls an operator or conductor route, then it is refused. Given an operator's token on an operator route, then it is allowed.
- Given a traveller whose profile is incomplete, when they call a route that requires a bookable traveller, then it is refused with the code that sends the app to "Complete your profile".
- Given a request carrying a header that imitates the internal caller metadata for another account, when it reaches a service, then the service sees only the account from the token.
- Given a record that belongs to another account, when it is requested, then the response is a 404 identical in status and body to the response for a record that does not exist. This sprint's first owned record is a phone's PIN credential (AUTH-4).
- Given two gateway instances and a rate limit of N on sign in, when one client sends N+1 attempts alternating between the instances, then the last attempt gets a 429 with a retry hint, rather than the limit only tripping after 2N.
- Given a traveller signs out, when the same access token is used on the next request, then both gateway instances refuse it. Once the token's own expiry has passed, its entry is gone from the revoked list.
- Given Redis is stopped, when sign-in requests keep arriving, then they behave as the cache plan documents and are never unlimited, and the gateway's health check reports the degradation.
- Given the service behind a route is stopped, when the route is called, then the app receives a 503 envelope with a retry hint in the requested language within the deadline, and routes to other services keep working.
- Given a service that answers more slowly than its deadline, when it is called, then the gateway answers with a 504 envelope at the deadline, and a non-idempotent call is not retried.
- Given a service returns a domain error, when the gateway maps it, then the HTTP status follows the documented mapping, the code is unchanged, and no gRPC detail appears in the body.

---

## AUTH-2 — Sign up and email verification — a code that arrives within 30 seconds, every time

**5 points · Priority Highest · `identity,notification,gateway,auth,sign-up,sprint-1`**

### User story
As a first-time traveller
I want to create an account with just my email and a password, and confirm my email with a code that actually arrives
So that I can start planning trips without handing over my National ID up front

### User flow (see Figma)
Services: identity (accountable: accounts, codes, attempt and resend counters), notification (sends the code by email, through PLAT-3), gateway (public routes, rate limits, the language of every message).

Splash ("Egyptian National Railways")
-> Choose Language: English / Arabic, "Please choose your preferred language to continue", Continue
   No server call. From here on, every request carries the chosen language.
-> Onboarding 1–3: "Book Easily, Travel Smart" · "Plan Your Journey Ahead" · "Easy to Use, For Everyone"
-> Sign up: "Create your Egyptian Railways account"
     Email · Password · Confirm password (each with an eye toggle)
     "Already have an account? sign in"
     the error state "Password doesn't match" under Confirm password
-> Verify your e-mail: "We Have Sent Code To Your Email"
     the address the code went to, masked, with "Not your email?"
     the code field
     a countdown that runs from 02:00 to 00:00 (the Figma shows it at 01:26)
     "Send Again", which only works once the countdown reaches 00:00
-> the right code -> the email is verified and the traveller is signed in
-> Complete your profile (AUTH-5, Sprint 2 — "Skip" is allowed) -> Create PIN (AUTH-4) -> Home

"Not your email?" returns to Sign up with the form ready for another address.

Note what is NOT on the Sign up screen: name, phone and National ID. The research found that
"not all users want to provide their full 14-digit National ID just to create an account" (D4).
Those details come later, before the first seat hold.

### Rules and validations
- Three fields (BR-02): Sign up takes an email, a password and its confirmation, and nothing else. All field errors are returned at once, each naming its field and what to change (BR-57).
- Email normalization: The email is trimmed and compared case-insensitively, so " Ali@Example.COM " and "ali@example.com" are one account. Nothing else is rewritten; there are no provider-specific rules such as removing dots. Uniqueness on the normalized form is guaranteed by the database, not by a lookup before the insert.
- Password (BR-02): At least 8 characters, including a letter, a digit and a symbol. Decide a maximum length and say why (some hashing algorithms silently ignore everything after 72 bytes). The password is stored only as a slow, salted hash and never appears in a log, a response or an error.
- Confirmation: The confirmation must match the password exactly. A mismatch is the "Password doesn't match" error on the confirm field. The server checks it, even though the app checks it too.
- The account's language (BR-56): The language the traveller chose on "Choose Language" arrives with the sign-up request and is stored on the account. Emails and push messages use it, because later there is no request to read it from. A missing or unsupported value falls back to the documented default (PLAT-2).
- An unverified account can only verify (BR-05): The account exists, but it receives no session and can do nothing except verify. Signing in to it gives AUTH-3's "verification required" outcome.
- Signing up again: A sign-up for an email that belongs to an unverified account does not create a second account and does not answer "email taken". Instead it replaces that account's password and language and sends a new code, within the resend limits below. A sign-up for a verified email is refused with a code that points to Sign in and "Forgot your password ?". That answer does reveal that the email has an account. This is accepted here because the gateway rate-limits sign-up, and BR-08 closes the same leak on forgot password.
- The code (BR-03): 6 digits from a cryptographically secure source, leading zeros allowed. Valid for 10 minutes, single-use, and valid only for email verification: a password-reset code never verifies an email. The code is stored as a hash, and issuing a new code kills the previous one.
- Never in a response: The code never appears in any API response, log or error. The response says where the code went (masked) and how many seconds remain until "Send Again" is allowed.
- Wrong attempts (BR-04): A code dies after 5 wrong attempts. The right digits typed on the sixth try still fail, and the traveller must ask for a new code.
- Send Again (BR-04): Allowed 120 seconds after the previous code (the design's countdown), with at most 5 codes per hour per email, counting verification and password-reset codes together. The countdown comes from the server: the response carries the remaining seconds, so a reinstalled app or a second phone shows the same time.
- Counters that hold (BR-04, §2.3): Attempt and resend counters live in Redis, are owned by identity, and hold across identity instances and restarts. A Redis restart must not give anyone a fresh set of attempts. Decide how to guarantee that, write it in the cache plan, and prove it by restarting Redis in the middle of five wrong attempts (§6.1). When Redis is unreachable, attempts fail safe: they are refused with a retry hint, never allowed without limit.
- Delivered asynchronously, measured (BR-03): The email is sent through PLAT-3, never inside the sign-up request. Sign-up succeeds even when notification is down, and the code reaches Mailpit within 30 seconds, measured.
- Verified means signed in: The right code verifies the email and opens a session in exactly the way AUTH-3 issues one, so the traveller goes straight on to "Complete your profile" and "Create PIN".

### Domain events to publish
- An account was registered (notification keeps its own copy of the account's email address and language for every later email and push message).
- A verification code was issued (notification emails it in the account's language — PLAT-3).
- An email address was verified (no consumer is required this sprint; it is in the catalogue for any service that later needs to know an account became usable).

### Edge cases to handle
- A traveller signs up, never verifies, and signs up again a week later with the same email (§6.1).
- The verification email arrives after the traveller has already asked for a new code. The old code must not work (§6.1).
- Sign-ups for "Ali@Example.com" and "ali@example.com " arrive at the same instant.
- A code that starts with zero: 004213.
- Redis restarts in the middle of someone's 5 wrong attempts (§6.1).
- The traveller mistyped their email and tapped "Not your email?", and a day later the real owner of the mistyped address signs up.
- A script hammers "Send Again" for someone else's email to flood their inbox.

### Acceptance criteria

- Given a valid email, password and confirmation sent with Arabic as the language, when the traveller signs up, then an unverified account with language Arabic exists. The response contains no code, only the masked address and 120 seconds until Send Again, and an Arabic email with a 6-digit code reaches Mailpit within 30 seconds.
- Given " Ali@Example.COM ", when the traveller signs up, then the account's email is stored as "ali@example.com", and a later sign-up with "ali@example.com" is treated as the same account.
- Given two sign-ups for the same email in different letter case at the same instant, when both complete, then exactly one account exists, proven by a concurrent test.
- Given a password of 7 characters with a letter, a digit and a symbol, or one of 8 characters without a symbol or without a digit, when the traveller signs up, then it is refused on the password field with a message saying what is missing. Given "abcdef1!", then it is accepted.
- Given a confirmation that differs from the password, when the traveller signs up, then it is refused on the confirm field with the "Password doesn't match" message in the requested language.
- Given the right code entered 9 minutes 59 seconds after it was issued, when it is submitted, then the email is verified and the traveller is signed in. Given the same code submitted again, then it is refused.
- Given the right code entered 10 minutes 1 second after it was issued, when it is submitted, then it is refused as expired with a code that tells the app to offer "Send Again".
- Given 5 wrong attempts on a code, when the right code is then submitted, then it is refused and a new code is required.
- Given "Send Again" is requested 119 seconds after the previous code, when the request arrives, then it is refused with the seconds remaining and no email is sent. Given 120 seconds, then a new code is sent and the previous one no longer works.
- Given 5 codes issued for one email within an hour, when a sixth is requested, then it is refused with a retry hint. The limit still holds after identity restarts, after Redis restarts, and with requests spread across two identity instances.
- Given a code issued for a password reset, when it is submitted on "Verify your e-mail", then it is refused.
- Given an email that belongs to an unverified account, when someone signs up with it again, then no second account exists, the new password is the one that works after verification, and only the newest code verifies.
- Given an email that belongs to a verified account, when someone signs up with it, then it is refused with a code that points to sign in and password recovery.
- Given the traveller taps "Not your email?" and signs up with another address, when they verify that second address, then that account is verified, and the first address can still be signed up for by its real owner later.
- Given a complete sign-up and verification, when the logs of the gateway, identity and notification are searched, then no password and no code appears.

---

## AUTH-3 — Sign in, staying signed in, signing out and recovering a password

**5 points · Priority Highest · `identity,gateway,auth,session,password-reset,sprint-1`**

### User story
As a returning traveller
I want to sign in with my email and password, stay signed in between trips, sign out when I choose, and get back in if I forget my password
So that the app is quick for me and useless to anyone who takes my phone or guesses my password

### User flow (see Figma)
Services: identity (accountable: credentials, lockout, sessions and refresh tokens, reset codes), gateway (public routes, rate limits, the revoked-token list), notification (emails the reset code, through PLAT-3).

Sign in: "Sign in to book your next trip!"
  Email · Password
  "Forgot your password ?"
  "Don't have an account? sign up"
-> success -> on the first sign-in on this phone: Create PIN (AUTH-4) -> Home
-> unverified account: the app opens "Verify your e-mail" (AUTH-2)
-> locked account: the error says when the traveller can try again, and offers "Forgot your password ?"

While the traveller uses the app, the session renews itself in the background. Closing and
reopening the app does not ask for the password again.

Settings -> Logout -> the session on this phone ends.

"Forgot your password ?"
  -> the traveller enters their email
  -> a reset code arrives by email; the code step looks and behaves like "Verify your e-mail"
     (countdown, "Send Again")
  -> a new password and its confirmation
  -> done: every session on every device is signed out, and the traveller signs in with the new password

The Figma draws the link but not the recovery steps themselves. Those steps reuse the code
screen and the password fields above. You define the API.

### Rules and validations
- Same answer for unknown email and wrong password: An unknown email and a wrong password produce the same status, body and code, and take comparable time, so check a password hash even when there is no account. Sign in never reveals which emails have accounts.
- Verification required (BR-05): An unverified account with the right password gets a "verification required" outcome instead of a session, so the app opens "Verify your e-mail". A new code is sent if AUTH-2's resend rules allow it; otherwise the response carries the remaining countdown. With a wrong password, the account gets the ordinary wrong-credentials answer, so the verification state is revealed only to someone who knows the password.
- Lockout (BR-04): 10 failed sign-ins lock the account for 15 minutes. Failures are counted per normalized email whether or not an account exists, so the lock itself reveals nothing. A successful sign-in resets the count, and the count expires 15 minutes after the last failure. Attempts during the lock are refused, even with the right password; they are not counted and do not extend the lock. A completed password reset lifts the lock.
- The counter belongs to identity: Lockout is a business rule, so its counter lives in identity's Redis keys and holds across gateway instances, identity instances and restarts (§6.1). The gateway's rate limits (AUTH-1) are a separate protection. When Redis is unreachable, sign-in fails safe as the cache plan documents, never without limit.
- The session (BR-06): A successful sign-in issues a short-lived access token (15 minutes at most, RS256, see AUTH-1) and a refresh token. Decide and document how long a refresh token lives, so that travellers stay signed in across restarts (§4, Module 1). Refresh tokens are stored hashed, so a database leak does not hand out working sessions.
- Rotation: Every refresh returns a new access token and a new refresh token, and the old refresh token stops working. Two refreshes with the same token at the same instant must not both produce live successors. That guarantee lives in the database, not in an `if`.
- Reuse means theft (BR-06): If a refresh token that was already rotated is presented again, the whole session is revoked, the thief's tokens and the real app's alike, and the event is recorded. The real traveller has to sign in again; that is the price of not knowing which of the two is the thief.
- Sign out (BR-06): Logout revokes this session's refresh token and its access token immediately, so the gateway refuses that access token from the next request on (AUTH-1). Other devices stay signed in. Logout also revokes this phone's PIN credential (AUTH-4): after signing out, the PIN no longer opens the app.
- Forgot password, no enumeration (BR-08): The answer to "Forgot your password ?" is identical whether or not the email has an account: same status, same body, comparable time. An email goes out only if the account exists. The hourly limit counts unknown emails too, so even "too many requests" reveals nothing.
- The reset code (BR-08, BR-03, BR-04): 6 digits, valid for 10 minutes, single-use, and valid only for password reset. 5 wrong attempts kill it. "Send Again" works after 120 seconds, with at most 5 codes per hour per email, shared with verification codes. The code never appears in a response or a log.
- The new password: It follows the same strength rules as sign up (BR-02), must match its confirmation, and must differ from the current password (BR-08). A rejected new password does not use up the reset, so the traveller can try again while the reset is still valid.
- Reset signs out everywhere (BR-06): A completed reset revokes every session and every device's PIN credential on the account. The gateway refuses all of that account's access tokens within seconds, not after up to 15 minutes. A lost phone that was signed in with a PIN is signed out too (§6.1).
- An unverified account can reset: Completing a reset proves the traveller controls the inbox, so it also verifies an unverified account.

### Domain events to publish
- A traveller signed in (security audit; no consumer is required this sprint).
- An account was locked after failed sign-ins (security monitoring).
- A session was revoked, by sign-out, by refresh-token reuse or by a password reset (consumed by whatever makes the gateway refuse that session's access tokens).
- Reuse of a rotated refresh token was detected (security audit).
- A verification code was issued for a password reset (notification emails it — PLAT-3).
- A password was reset (the gateway refuses the account's outstanding access tokens; identity has already revoked its sessions and devices).

### Edge cases to handle
- Failed sign-ins for one account arrive through two gateway instances and two identity instances. Is the lock still at 10 (§6.1)?
- A refresh token is stolen and used after the real app has already rotated it (§6.1).
- The app refreshes from two screens at the same instant with the same refresh token. Decide whether a short grace window returns the same new pair or the reuse rule applies, and document it.
- The traveller resets their password while their lost phone is still signed in with a PIN (§6.1).
- An attacker sends 10 wrong passwords every 15 minutes to keep a traveller locked out. What can the real traveller still do?
- The access token expires in the middle of a chain of requests that started 14 minutes ago.
- Redis is down when a traveller signs in.

### Acceptance criteria

- Given a verified account and the right password, when the traveller signs in, then they receive an access token valid for at most 15 minutes and a refresh token, and the gateway accepts the access token without a call to identity.
- Given an unknown email, and separately a known email with a wrong password, when each signs in, then the status, body and code are identical and the response times are comparable.
- Given an unverified account and the right password, when the traveller signs in, then no session is issued and the outcome is "verification required". Given the wrong password, then the answer is the ordinary wrong-credentials one.
- Given 9 failed sign-ins followed by a successful one, when a wrong password is then sent, then the account is not locked, because the count started again.
- Given 10 failed sign-ins spread across two gateway instances and two identity instances, when the right password is then sent, then it is refused as locked with the time remaining, and after 15 minutes it succeeds.
- Given a refresh, when it completes, then a new pair is returned and the previous refresh token no longer works.
- Given two refreshes with the same refresh token at the same instant, when both complete, then at most one new refresh token is ever valid, proven by a concurrent test.
- Given a refresh token that was already rotated, when it is presented again, then the whole session is revoked: neither the thief's nor the real app's refresh token works, and the session's access tokens are refused within seconds.
- Given a traveller signs out, when their access token is used on the next request, then it is refused even though it has not expired, their refresh token is refused, and their other devices are still signed in.
- Given a registered email and an unregistered one, when "Forgot your password ?" is requested for each, then the responses are identical and only the registered email receives a message, and the same holds once the hourly limit is reached.
- Given a reset code, when it is submitted on "Verify your e-mail", then it is refused. The same applies to a verification code submitted as a reset code.
- Given a new password that is identical to the current one, weak, or not matching its confirmation, when it is submitted, then it is refused on that field, and the reset can still be completed afterwards with an acceptable password.
- Given a completed reset, when the traveller signs in, then the old password fails and the new one works, and every other device, including a phone signed in with a PIN, is signed out within seconds.
- Given Redis is unavailable, when sign-in attempts arrive, then they behave as the cache plan documents and failed attempts are never unlimited.
- Given a day of sign-ins, refreshes and resets, when every service's logs are searched, then no password, access token, refresh token or code appears.

---

## AUTH-4 — PIN and Touch ID — a quick unlock that is worthless on any other phone

**3 points · Priority High · `identity,gateway,auth,pin,device,sprint-1`**

### User story
As a frequent traveller
I want to open the app with a 4-digit PIN or my fingerprint instead of my password
So that booking my weekend train takes seconds, without making my account easier to steal

### User flow (see Figma)
Services: identity (accountable: device credentials, PIN hashes, wrong-PIN counting, revocation), gateway (the public PIN sign-in route and its rate limit).

After the first sign-in on a phone (via AUTH-2 verification or AUTH-3 sign in):
  Create PIN: "Set a PIN to secure your account." — 4 boxes
  -> Confirm PIN: "Set a PIN to secure your account." — 4 boxes
  -> Home

On every later launch on that phone:
  Enter your PIN — 4 boxes and a keypad, "Forgot Your PIN Code?"
  or the iOS prompt: Touch ID for "ENR" — "Authenticate to sign in"
  -> Home, signed in

"Forgot Your PIN Code?" -> Sign in with email and password (AUTH-3) -> Create PIN -> Confirm PIN -> Home.

The research is why this feature exists: the frequent traveller "felt satisfied and comfortable
using biometrics to log in". The rule is what makes it safe: the PIN protects one phone, not
the account (D12).

### Rules and validations
- One device, one credential (BR-07, D12): Creating a PIN registers this phone. identity issues a credential bound to this device and stores the PIN only as a hash. The PIN means nothing without that device credential, so typed on any other phone it is useless.
- Hard to brute-force offline: A 4-digit PIN has 10,000 possible values, and a plain hash of it can be reversed in milliseconds. Store it so that a copy of identity's database alone is not enough to test the 10,000 values: bind it to the device's secret, or to a key the database does not hold.
- Exactly 4 digits: Leading zeros are allowed. The confirmation must match before anything is stored.
- Signed in to register: Only a signed-in traveller can register a device. There is one credential per device per account, and registering again replaces the previous one.
- PIN sign-in issues a normal session: The device credential plus the right PIN gives the same access and refresh tokens as AUTH-3, tied to that device. A PIN never replaces the password on a new phone.
- Five wrong PINs (BR-07): The fifth wrong PIN revokes that device's credential, and from then on that phone needs the password and a new PIN. A right PIN resets the count. The count is atomic, so five wrong PINs sent at once cannot turn into six chances.
- Biometrics stay on the phone (BR-07): Touch ID or Face ID unlocks the same device credential on the phone. No fingerprint, face or biometric result ever reaches the server, and the API has no field that could carry one. A revoked credential cannot be opened with a fingerprint either.
- Forgot PIN: "Forgot Your PIN Code?" means a password sign-in followed by a new PIN. The new PIN replaces the old credential, and the old PIN stops working.
- What revokes a device: Five wrong PINs, signing out on that phone (AUTH-3), or a password reset, which revokes every device on the account (§6.1).
- Lockout does not lock the phone: The password lockout of BR-04 does not block PIN sign-in on a registered phone, which has its own five-attempt limit. That way, someone hammering the password cannot lock the real traveller out of their own phone.
- Your devices only: A device credential belongs to one account. Acting on another account's device gets a 404 (BR-09).
- Never logged: No PIN, PIN hash or device secret appears in any log or response.

### Domain events to publish
- A device was registered for PIN sign-in (security audit).
- A device credential was revoked, after five wrong PINs, a sign-out or a password reset (security audit; later, a "new sign-in on your account" notice could be triggered by it).

### Edge cases to handle
- The traveller signs in on a second phone. The PIN from the first phone must not be valid there (§6.1).
- The traveller resets their password while their lost phone is still set up with a PIN (§6.1).
- The app is reinstalled on the same phone. The device credential on the phone is gone, so the traveller signs in with the password and sets a new PIN.
- Five wrong PINs arrive in parallel from a scripted client.
- Four wrong PINs, then the right one, then four more wrong ones.

### Acceptance criteria

- Given a signed-in traveller on phone A, when they create and confirm a PIN, then phone A has a device credential and the PIN is stored only as a hash.
- Given a PIN and a confirmation that differ, when they are submitted, then nothing is registered and the error names the confirmation.
- Given phone A with a PIN, when the app is reopened and the right PIN entered, then the traveller is signed in without the password, with a new session tied to phone A.
- Given phone A's PIN typed on phone B for the same account, when it is submitted, then it is refused, and phone B must sign in with the password.
- Given 4 wrong PINs followed by the right one, when the right one is submitted, then sign-in succeeds and the count starts again.
- Given 5 wrong PINs on phone A, when the right PIN is entered next, then it is refused and the app is told the password is required.
- Given 5 wrong PINs sent at the same instant, when all of them complete, then the credential is revoked and no sixth attempt is ever evaluated, proven by a concurrent test.
- Given a revoked device credential, when the app tries to sign in through the Touch ID unlock, then sign-in is refused.
- Given the traveller signs out on phone A, when the PIN is entered on the next launch, then it is refused and the password is required.
- Given a password reset made on another phone, when phone A enters its right PIN, then it is refused.
- Given "Forgot Your PIN Code?" followed by a password sign-in and a new PIN, when the old PIN is entered, then it is refused, and the new PIN works.
- Given an account locked by 10 failed passwords, when phone A enters its right PIN, then the traveller is signed in.
- Given identity's database alone, when it is inspected, then no PIN is readable and the stored values cannot be checked against the 10,000 possible PINs without the device or a key held elsewhere. No PIN appears in any log.

---

## Explicitly out of scope for Sprint 1

- **Complete your profile, the National ID and saved passengers.** These come in Sprint 2 (AUTH-5, AUTH-6). The bookable-traveller level exists from AUTH-1, but nothing makes a traveller bookable until AUTH-5.
- **Settings → Profile.** This arrives with AUTH-5. Settings → Logout is covered by AUTH-3 in this sprint.
- **Idempotency-Key responses at the gateway** (§2.3). These arrive with the first request that moves money (Sprints 3–4). The Redis rules of AUTH-1 are the pattern to follow.
- **Push notifications, the in-app inbox and the unread count.** These come in Sprint 5. Verification and reset codes are email-only and never stored in the inbox (BR-53), so this sprint needs none of them.
- **Tracing, circuit breakers and dead-letter replay.** These come in Sprint 5. They build on the correlation ID from PLAT-2 and the replayable dead-letter queue from PLAT-3.
- **The Stripe CLI forwarder in Compose.** This arrives in Sprint 4.
- **Social sign-in, a "sign out everywhere" button, changing your email or password while signed in, account deletion and data export.** None of these is in the design. A password reset is the only thing that signs a traveller out everywhere.
