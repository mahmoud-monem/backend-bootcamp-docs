# Project 3: ENR — Egyptian National Railways Ticket Booking

> **Bootcamp:** Node.js Backend Engineering Bootcamp
> **Project #:** 3 (Third Project)
> **Primary Goals:** **Microservice architecture** · distributed data and sagas · **caching** · segment-based seat inventory under concurrency · real payments (Stripe test mode)
> **Tech Stack:** Node.js · **NestJS** (monorepo) · TypeScript · **PostgreSQL** (database per service) · **Redis** · **RabbitMQ** · **gRPC** · **Stripe** · Docker Compose
> **Figma Design:** [Egyptian National Railways (Figma)](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
> **Team setup:** 3 teams × 5 students · **5 sprints of one week** · every team builds the whole system · every student **owns** one or two services

---

## 0. How to Use This Document

You have **four sources of truth**, and you are expected to cross-read all of them:

| Source | What it gives you |
|---|---|
| **This document** | The business: who the users are, what the product must do, the rules it must never break, the edge cases that will bite you, and the architecture you are required to use |
| **The user stories** (`sprint-1-…` to `sprint-5-…`) | The backend work, story by story: the service flow (what a client sends, which services act, what the response or event carries), with rules and acceptance criteria per story |
| **The Figma design** | Context, not a deliverable: no client UI is built in this project. The screens, states and copy show what your API must make possible. The **Presentation** page is the UX research behind it — read it, it explains *why* the product behaves the way it does |
| **[`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)** | *How* to build microservices with NestJS, and the `MS-###` rules your code is reviewed against |

**What this document deliberately does NOT give you:** database tables, endpoints, `.proto` files, queue names or an event catalogue. **Those are your deliverables.** You are now designing a *system of services*, which means one more question on top of "what fields does a ticket have?" — **"which service owns a ticket, and who else needs to know about it?"**

> 💡 **The skill being trained:** splitting a product into services that can be built, deployed and broken independently — and keeping the data correct when the network between them is not.

---

## 1. Project Overview & Philosophy

**ENR** is the mobile app of the Egyptian National Railways: search train schedules between any two stations, book seats for up to four people, pay by card, carry the ticket as a QR code, and cancel with a clear refund.

The design you are building is a **redesign**. A UX team studied the existing app, ran usability tests with real users, and rebuilt the flows. Their findings are in the Figma *Presentation* page, and they are your requirements just as much as the screens are.

### 1.1 What the research found

| Finding | What users said or did |
|---|---|
| The verification code was lost | *"Where is the OTP?!"* — 3 of 4 users could not find the code. The redesign's target: **the code arrives within 30 seconds, every time** |
| Group seats were scattered | *"Seats weren't next to each other."* — a user restarted the whole booking twice to sit next to their companion |
| The "Auto" button | Every participant was confused by an action that silently assigned seats |
| Cards had to be re-entered | *"Saving card would make this easier"* — frequent travellers type their card every trip |
| Errors did not help | *"The error is too generic — I don't know what to change: station, date, or round-trip option."* |
| No way to narrow results | Users wanted to filter by time and price |
| Mixed languages | Arabic and English appeared on the same screen; terms like "AC 2" meant nothing to occasional travellers |
| National ID up front | *"Not all users want to provide their full 14-digit National ID just to create an account."* |

### 1.2 What that means for the backend

1. **The seat the user sees is the seat they get.** If a user is shown "Coach 5, Seat 44" next to their friend in "Seat 45", that is what they pay for — not a reshuffle after payment.
2. **Money and tickets always agree.** Nobody is ever charged without getting a ticket, and nobody ever holds a ticket that was not paid for. With five services and a payment provider in between, this is now genuinely hard.
3. **Fast answers come from the cache; correct answers come from the database.** Search results can be a few seconds stale. A seat hold cannot.
4. **Every error tells the user what to do next.** "Sorry, no trains available on the selected date — Select Another Date" is the standard the design sets. Your error codes must make that possible.

### 1.3 Design principles for trainees

1. **Services own data, not tables.** A service is the only thing allowed to read or write its database. If another service needs that data, it asks (gRPC) or listens (events).
2. **Design for the network failing.** Every call between services can time out, be retried, or arrive twice. Your design must stay correct when it does.
3. **Never trust the client — or another service's input blindly.** Prices, seat assignments, refund amounts and ticket validity are decided by the service that owns them.
4. **Money is integers.** Piastres, not floating-point pounds.
5. **The unhappy path is the grade.** Two people taking the last seat, a payment that succeeds after the hold expired, a train cancelled with 400 tickets sold — that is the project.

---

## 2. Mandatory Technical Requirements

These are **not optional** and are the main grading criteria.

### 2.1 Stack

| Requirement | Detail |
|---|---|
| **NestJS monorepo** | One repository per team, Nest monorepo mode: `apps/<service>` and `libs/<shared>`. Shared libraries hold cross-cutting code only (logging, config, envelope, outbox, contracts) — **never domain logic** |
| **PostgreSQL, database per service** | One Postgres container is fine, but every service has **its own database and its own credentials**. No service ever connects to another service's database. ORM is your choice — justify it |
| **Redis** | Mandatory. See §2.3 |
| **RabbitMQ** | All asynchronous communication between services (domain events) |
| **gRPC** | All synchronous service-to-service calls, with `.proto` contracts versioned in `libs/contracts` |
| **API gateway** | The **only** service the mobile app talks to. REST under `/api/v1`, one response envelope |
| **Stripe (test mode)** | Card payments, saved cards, refunds, webhooks. Each team uses its own free Stripe test account. Real card numbers never touch your servers |
| **Docker Compose** | `docker compose up` brings up every service plus Postgres, Redis, RabbitMQ, Mailpit (email) and the Stripe CLI webhook forwarder. Health checks decide start order |
| **TypeScript strict** | No `any` in domain code |
| **Validation** | DTOs + global `ValidationPipe` with `whitelist` and `forbidNonWhitelisted` at the gateway; gRPC and event payloads validated at every service boundary |
| **Configuration** | Every service validates its environment **at boot** and refuses to start on a missing or malformed variable |
| **Testing** | Unit tests for every business rule, integration tests per service with real Postgres/Redis/RabbitMQ (Testcontainers), contract tests for `.proto` files and event schemas, end-to-end tests for the money flows |

### 2.2 Microservice architecture — **the core learning goal**

The system is split into the services below. **The split is given; the internals are yours.**

| Service | Owns | Depends on |
|---|---|---|
| **gateway** | The public REST API, token verification, rate limiting, request routing and aggregation | calls every service (gRPC) |
| **identity** | Accounts, email verification, sessions, PIN devices, the traveller's profile and saved passengers | listens to booking's issued tickets (to lock the self passenger's National ID, BR-12) |
| **catalog** | Stations, trains, classes, coach layouts, timetables and dated trips, fares, search, recent searches, the **operations API** (timetables, fares, delays, cancellations) | listens to inventory's availability changes; needs to know whether a trip has sold seats before a timetable change (BR-50) |
| **inventory** | Which seat is taken on which part of which trip, seat holds, seat allocation, availability counts | listens to catalog's trip publications, delays and cancellations |
| **booking** | Bookings, the **booking saga**, tickets (QR, PDF, share), cancellation and refund policy | calls catalog, inventory, payment; needs identity's passengers (a call or a copy — your decision); listens to catalog's delays and cancellations |
| **payment** | Stripe customers, saved cards, charges, refunds, Stripe webhooks | calls Stripe; receives Stripe's webhooks |
| **notification** | Email, push and the in-app inbox behind the bell | listens to everyone's events |

**What that means in practice:**

- The booking service **orchestrates** a booking (a saga): it asks catalog for a price, asks inventory to hold seats, asks payment to start a charge, and reacts to the payment result. If any step fails, it runs the **compensations** (release seats, cancel the charge, refund).
- Services never share tables, never import each other's domain code, and never call each other's databases. A service that needs a station name for a ticket **keeps its own copy**, fed by events.
- When something business-meaningful happens, the owning service **publishes an event** through a **transactional outbox**. Every consumer is **idempotent**. Failed messages are retried with backoff and finally **dead-lettered**, never dropped.
- Every synchronous call has a **deadline**. A slow service must not freeze the gateway.
- One **correlation ID** follows a request through HTTP, gRPC and RabbitMQ, so you can answer: *"why did this user receive two refund emails?"*

**You must decide and justify:**

1. **Orchestration or choreography** for each multi-service flow. The booking flow is orchestrated (that is given). Is the trip-cancellation fan-out orchestrated or choreographed? Why?
2. **What crosses the wire.** For every sync call and every event: the payload, who validates it, and what happens when a field is added next sprint.
3. **What each service copies from others** — and how stale that copy may be.
4. **What happens when each service is down.** Walk through it for all seven: what still works, what degrades, what fails, and what message the user sees.
5. **Where each cross-service invariant lives.** Example: *"the same National ID cannot hold two seats at the same time on the same train"* involves passengers (identity), seats (inventory) and bookings (booking). Exactly one service must enforce it. Which one, and with which data?

**Ownership.** Each student owns one or two services (suggested split: *gateway + identity* · *catalog* · *inventory* · *booking* · *payment + notification*). The owner is accountable for that service's design, its README, its contracts and its tests, and **reviews every pull request that touches it**. Sprint stories are split across the whole team — owning a service does not mean coding alone.

> 🎯 **Design deliverable (Sprint 1, day 2):** a **service map** (services, owned data, sync calls, events), the **event catalogue**, the **`.proto` contracts**, the **gateway endpoint table**, the **error catalogue**, an **ERD per service** (§7.3), a **sequence diagram of the booking saga** including its compensations, the **cache plan** (§2.3), the **five most dangerous edge cases** (§6) and **where each ruling lives** (§9). Mentor review **before** writing service code.

### 2.3 Caching — **mandatory**

Every team uses Redis, and uses it deliberately. A cache that was added "because it was required" and that nothing measures does not count.

**Required uses:**

| Where | What | Why |
|---|---|---|
| catalog | Station list, trip stops, search results | The most-read data in the system, changed rarely |
| catalog | Availability counts on search results ("Available Seats: 170") | Read on every search; owned by inventory, so kept fresh by events |
| gateway | Rate-limit counters, revoked-token list, idempotency-key responses | Must be shared by every gateway instance |
| identity | Verification-code attempt and resend counters, failed sign-in counters | Must survive a restart and work across instances |
| notification | Unread count behind the bell | Read on every screen |

**Rules the cache must follow:**

1. **The cache displays, the database decides.** A cached "170 seats" is fine on a results screen. A seat hold never consults the cache.
2. **Every cached value has an owner, a TTL and an invalidation trigger.** If you cannot say what event removes a key, the key is a bug.
3. **No stampedes.** When a popular key expires, one request rebuilds it; the others do not all hit the database.
4. **TTLs are jittered**, so a thousand keys written together do not expire together.
5. **Keys are named and versioned** (`catalog:v1:search:…`), so a format change is a version bump, not a production incident.
6. **Per-user data is never cached under a shared key.** One traveller's recent searches must never appear for another.
7. **Redis down = slower, not broken.** If Redis disappears, searches still work (from the database) and the system says so in its logs and health checks. Sign-in rate limiting must fail **safe**, not open — decide what that means.
8. **Hit and miss rates are measured.** You will be asked for the search cache hit rate during the demo.

> 🎯 **Design deliverable:** the **cache plan** — for every key: owner, contents, TTL, what invalidates it, and what happens when Redis is unavailable.

### 2.4 Carried over from Projects 1 and 2

Every mandatory requirement of the previous projects still applies, in every service:

- Vertical slices inside each service; controllers handle transport only, services hold rules, repositories hold persistence.
- Centralized error handling — no stack trace, SQL error, gRPC internals or Stripe message ever reaches the client.
- One response envelope at the gateway; **stable, machine-readable error codes**; messages selected by code.
- Guards that **deny by default**; a new endpoint is closed until someone opens it.
- **Someone else's record is a `404`, not a `403`.**
- No password, code, token, PIN, card data, full phone number or full National ID in any log.
- Every operation that costs money or sends a message is **safe to retry**.
- Money as integer piastres. (Every DOCURA team was bitten by TypeORM returning `numeric` columns as strings. Do not be bitten twice.)
- Concurrency guarantees live in the **database**, not in an `if` statement.

### 2.5 Suggested repository structure

```
enr/
├── apps/
│   ├── gateway/
│   ├── identity/
│   ├── catalog/
│   ├── inventory/
│   ├── booking/
│   ├── payment/
│   └── notification/
├── libs/
│   ├── contracts/        # .proto files and event schemas — the only thing services share about each other
│   ├── messaging/        # outbox, inbox, RabbitMQ topology, retry and dead-letter handling
│   ├── observability/    # logger, correlation ID, tracing, health
│   └── common/           # envelope, error codes, config validation helpers
├── docker-compose.yml
└── README.md
```

Each `apps/<service>` keeps its own vertical slices, migrations, seed and tests. If two services import the same *domain* class from `libs/`, they are one service pretending to be two.

---

## 3. The Actors

| Actor | Who they are | What they can do |
|---|---|---|
| **Visitor** | Has the app, no session | Choose a language, sign up, verify email, sign in, recover a password. **Nothing else** — there is no guest mode in this design |
| **Traveller** | Signed-in account with a verified email | Search, filter, view stops, manage saved passengers and cards, read notifications |
| **Bookable traveller** | A traveller whose profile is complete (§5.2) | Everything above, plus hold seats, pay, hold tickets, cancel |
| **Passenger** | A person on a ticket. May or may not be the account holder | Nothing — passengers are data. The account holder books for them |
| **Operator** | ENR staff, seeded account with an `operator` role | The operations API: publish timetables, change fares, delay or cancel a trip. **No UI** in this project |
| **Conductor** | ENR staff on the train, seeded account with a `conductor` role | Scans a ticket QR. Needs a way to verify it — offline from the QR alone, and online when there is signal. **No UI** in this project |
| **Stripe** | External payment provider | Charges cards, issues refunds, and calls your webhook |

**Ownership rule:** bookings, tickets, saved passengers, saved cards and notifications belong to exactly one account. Asking for someone else's returns **404**.

---

## 4. Functional Requirements

Read them alongside the Figma screens. Every screen in Appendix C is in scope.

### Module 1 — Access (`gateway`, `identity`)

- **Language** — chosen on first launch; every server message and email follows it (BR-56).
- **Sign up** — email, password, confirm password. That is all; the rest comes later (BR-02).
- **Verify email** — a code sent by email, a countdown, "Send Again", "Not your email?" (BR-03, BR-04).
- **Sign in** — email and password; "Forgot your password?" recovers by email code (BR-06, BR-08).
- **PIN and biometrics** — after the first sign-in on a device the traveller sets a 4-digit PIN; next time, the PIN (or Touch ID, which unlocks the same device credential) signs them in (BR-07).
- **Sessions** — stay signed in across restarts, sign out, sign out everywhere after a password reset.

### Module 2 — Profile and passengers (`identity`)

- **Complete your profile** — full name, Egyptian mobile number, National ID; profile type is Regular, nationality is Egyptian. The traveller may skip it, but cannot hold a seat until it is done (BR-10).
- **Passengers tab** — saved passengers with name, type and National ID; add, edit, delete (BR-12).

### Module 3 — Network and timetable (`catalog`)

- **Stations** in Arabic and English, searchable in both, grouped by governorate as in the design's station picker.
- **Trains** with a number (916, 969…), a type, and coaches of different **classes**.
- **Timetables** — which days a train runs, the ordered list of stops, and arrival/departure time at each.
- **Dated trips** — "train 916 on 27 October" is the thing a seat belongs to.
- **Fares** — per train, class and origin–destination pair.

### Module 4 — Search (`catalog`)

- **Home** — One way / Round trip, From, To (with swap), Departure Date, Return Date, Seat Class, and an optional Train Number.
- **Results** — header with route, date and class; a **5-day date strip**; one card per trip with stations, times, dates, duration, available seats, number of stops and the fare; an edit action that returns to the search form.
- **Stops** — tapping "4 Stops" shows the timeline of stations and times for that trip.
- **Empty state** — *"Sorry, no trains available on the selected date"* with **Select Another Date**.
- **Filters and sorting** — what the research asked for: departure time window, maximum price, train type; sort by departure time, duration or price.
- **Recent searches** — on Home, per traveller.

### Module 5 — Seats and holds (`inventory`)

- **Availability** per trip, class and *segment* (the part of the route between the passenger's two stations).
- **Allocation** — the server picks the seats, keeping groups together (BR-25).
- **Holds** — seats reserved for 15 minutes while the traveller reviews and pays (BR-26).

### Module 6 — Booking and tickets (`booking`)

- **Choose passengers** — from saved passengers, up to four, with "+ Add new passenger".
- **Review tickets** — per passenger: coach and seat; the terms checkbox; the **reservation countdown**.
- **Payment summary** — Price, Booking Fee (7.0%), Total.
- **Confirmation** — *"Payment Success! Thank you! Your booking has been confirmed!"*
- **Tickets tab** — Paid and Canceled. Each card: stations, times, dates, duration, train number, class, coach/seat, passenger name, fare. Actions: **Share, QR, Download, Cancel trip**. Cancelled cards show a refund badge: *In Progress* with "Refund date: Pending", or *Refunded* with the date.

### Module 7 — Payments (`payment`)

- **Saved cards** — list, add, delete; Settings → Payment Methods and the payment screen show the same list. A card is added through Stripe on the device; your backend never sees the number.
- **Charging** — one charge per booking, in EGP, with 3-D Secure when the bank asks for it.
- **Refunds** — full or partial, per ticket, tracked until Stripe confirms them.

### Module 8 — Cancellation and refunds (`booking`, `payment`)

- **Cancel trip** on a ticket shows what the traveller will get back **before** they confirm, then cancels and starts the refund (§5.7).

### Module 9 — Operations and disruptions (`catalog` ops API → everyone)

- **Publish** a timetable and generate dated trips ahead of time.
- **Change fares** for future sales.
- **Delay** a trip by N minutes, or **cancel** it — and let the rest of the system react (§5.8).

### Module 10 — Notifications (`notification`)

- **Email** — verification codes, booking confirmation with tickets, cancellation, refund completed, delay, trip cancelled, reminders.
- **Push** — the same events, to registered devices (a simulated push provider is acceptable).
- **In-app inbox** — behind the bell; unread badge; mark as read. *The inbox screen is not designed — you define its API.*

### Module 11 — Settings

- **Profile**, **Payment Methods**, **Logout**. Theme and language are device preferences, but the account's language is stored on the server so emails and push use it (BR-56). *Contact Us* and *FAQs* are not designed — out of scope.

---

## 5. Business Rules

Breaking one of these is a functional bug.

### 5.1 Accounts and access

| # | Rule |
|---|---|
| BR-01 | Every endpoint requires a signed-in traveller with a verified email, **except**: sign up, verify email, resend code, sign in, refresh, forgot password, reset password, PIN sign-in, opening a shared ticket link, the Stripe webhook (which must verify Stripe's signature instead), and health checks. Operator and conductor endpoints require their role. |
| BR-02 | Sign up takes an **email, a password and its confirmation**. The email is unique, case-insensitive and trimmed. The password has **at least 8 characters including a letter, a digit and a symbol**. The confirmation must match. |
| BR-03 | Email verification uses a **6-digit code, valid for 10 minutes**, single-use, valid only for the purpose it was issued for (verification vs password reset). Requesting a new code kills the previous one. The code is **never** in an API response. It is sent asynchronously and must reach the inbox **within 30 seconds** — measure it. |
| BR-04 | A code dies after **5 wrong attempts**. "Send Again" is allowed after **120 seconds** (the design's countdown), at most **5 codes per hour per email**. **10 failed sign-ins lock the account for 15 minutes** (failures are counted per normalized email, a success resets the count, and the count expires 15 minutes after the last failure). Counters live in Redis and work across instances and restarts. |
| BR-05 | An unverified account can do nothing except verify. Signing in to it returns a *verification required* outcome so the app can show the Verify screen. |
| BR-06 | A session is a **short-lived access token (≤ 15 minutes)** and a **rotating refresh token** whose lifetime you choose and justify (at most 30 days). Reusing an already-rotated refresh token revokes the whole session. Sign-out revokes the refresh token and the access token immediately (the gateway rejects it until it expires). A password reset signs the account out everywhere. |
| BR-07 | A **4-digit PIN** unlocks **one registered device**. The PIN is stored hashed and is useless on any other device. It never replaces the password on a new device. **5 wrong PINs** revoke that device's credential and require the password. Touch ID / Face ID never reach the server — they unlock the same device credential the PIN does. |
| BR-08 | "Forgot password" sends a reset code by email (same rules as BR-03/BR-04). The response is **identical whether or not the email exists**. The new password must differ from the current one. |
| BR-09 | Someone else's booking, ticket, passenger, card or notification is a **404**. |

### 5.2 Profile and passengers

| # | Rule |
|---|---|
| BR-10 | Before the **first seat hold**, the traveller must complete their profile: **full name, Egyptian mobile number, National ID**. Skipping is allowed; the hold is refused with a code that sends the app to *Complete your profile*. The mobile number is unique across accounts (normalized: `01001234567` and `+201001234567` are the same). The National ID is unique across **account holders**. Completing the profile creates the traveller's own **"self" passenger**. Ownership of the phone number and National ID is **not verified** in this project (there is no SMS channel); a value already claimed by another account is refused with a specific error, and resolving such disputes is out of scope (Appendix A). |
| BR-11 | A National ID is **14 digits** whose structure is valid: century digit (`2` = 1900s, `3` = 2000s), a real birth date that is not in the future, and a known governorate code. Anything else is rejected with a specific error. |
| BR-12 | A traveller saves **up to 10 passengers** besides their self passenger: full name, National ID, type. A National ID appears **at most once in one traveller's list** — but two travellers may both save the same person. Editing or deleting a passenger never changes a ticket already issued: tickets keep their own copy. The self passenger cannot be deleted, and its National ID cannot change once a ticket was issued for it. |
| BR-13 | The only passenger type today is **Regular**, and the only nationality is **Egyptian**. Your model must allow discounted types (student, senior, military, child) later without a migration of existing data. |

### 5.3 Network, search and fares

| # | Rule |
|---|---|
| BR-14 | A **trip** is one dated run of a train, with an ordered list of stops and an arrival and departure time at each. All times are **Africa/Cairo**. A trip may arrive the **next day**. |
| BR-15 | A traveller may search between **any two stations a trip stops at, in travel order**. Banha → Tanta is a valid search on a Cairo → Alexandria train. The same station as From and To is rejected. |
| BR-16 | Search returns trips that **depart the origin station on the selected Cairo date**, have not departed yet, are within the **booking window — today plus the next 29 days, in Cairo calendar days** — are not cancelled, and whose sales are open. **Sales close 15 minutes before departure from the passenger's origin station.** A trip with **zero seats** in the chosen class is **shown as sold out**, not hidden. |
| BR-17 | Each result carries: train number, train type, class, departure station, time and date, arrival station, time and date, duration, the number of stops **between** the passenger's stations, available seats for that class **on that segment**, and the fare. |
| BR-18 | Optional Train Number narrows the search to that train. Filters: departure time window, maximum fare, train type. Sorting: **departure time (default)**, duration, fare. Filters apply to the fare **without** the booking fee. |
| BR-19 | A fare depends on **train, class, and origin–destination pair**. The fare shown in results excludes the 7% booking fee; the payment summary shows both. |
| BR-20 | **Round trip:** the return date is on or after the departure date; the return search is the reverse route; a return trip must **depart after** the chosen outbound trip **arrives**. |
| BR-21 | **Recent searches:** the last **5** per traveller, newest first, without duplicates. |
| BR-22 | Station search works in **Arabic and English**. Arabic matching ignores the differences users don't type consistently: `أ إ آ → ا`, `ة → ه`, `ى → ي`, diacritics, tatweel, and a leading `ال` — so "اسكندريه" finds "الإسكندرية". Class names are plain words ("First Class AC"), never codes like "AC 1" (research finding). |

### 5.4 Seats and holds

| # | Rule |
|---|---|
| BR-23 | Seats are **segment-aware.** A seat sold from Cairo to Tanta is **free again from Tanta onward**. A seat is free for a segment only if **no active hold and no sold ticket overlaps** that segment. Touching segments do not overlap: Cairo→Tanta and Tanta→Alexandria can share a seat. |
| BR-24 | The availability count for a result is the number of seats in that class that are free for the **whole** requested segment. |
| BR-25 | **The server allocates seats.** For a group it tries, in order: **all side by side in one coach**, then **all in one coach**, then **split across coaches**. A split is **shown on Review Tickets before payment** — never silently. The traveller can accept the allocation or abandon the hold. |
| BR-26 | A hold reserves the allocated seats for **15 minutes** (the design's *Reservation Time* countdown). It is extended **once by 5 minutes** when the bank's 3-D Secure step starts. An expired hold is **released automatically** and its seats return to the pool. A hold covers **all passengers and all legs, or nothing**. A hold — and its extension — never outlives the close of sales at the passenger's origin (BR-16). |
| BR-27 | The same **National ID cannot occupy two seats on overlapping segments of the same trip** — across all bookings and all accounts. An active hold counts as occupying, so the duplicate is refused when the second hold is made, not at payment. |
| BR-28 | A booking has **1 to 4 passengers**. A traveller has **at most one active hold**; starting a new one releases the previous one. |
| BR-29 | A hold **never** uses a cached availability count. It is decided by inventory's database at the moment of the hold. |

### 5.5 Booking and payment

| # | Rule |
|---|---|
| BR-30 | The fare shown when the hold was created is the fare charged. A fare change after that does not affect the held booking. |
| BR-31 | **Booking fee = 7% of the total fare**, rounded **half-up to the nearest piastre**. **Total = fares + booking fee.** All amounts are integer piastres everywhere, including Stripe. |
| BR-32 | Payment is by card through Stripe. Card number and CVV **never** reach your servers, logs or database. A card is saved only when the traveller saves it; a saved card is a Stripe payment method attached to the traveller's Stripe customer. |
| BR-33 | A booking is **confirmed only when the payment service has verified the payment with Stripe** (signed webhook or a server-side check) — never because the app says the payment succeeded. |
| BR-34 | A declined payment creates **no tickets**. The hold stays until it expires, so the traveller can retry with another card. |
| BR-35 | **Never charged without tickets, never tickets without a charge.** A double tap, an app retry, a duplicate webhook or a crashed service never produces two charges, two bookings or missing tickets. |
| BR-36 | If a payment succeeds **after the hold expired**: re-acquire the same seats if they are still free; otherwise allocate new seats under BR-25 and tell the traveller; if no seats exist, **refund in full, including the booking fee**, and tell the traveller. |

### 5.6 Tickets

| # | Rule |
|---|---|
| BR-37 | A confirmed booking produces **one ticket per passenger per leg**, under one **booking reference**. A ticket keeps its own copy of everything printed on it: passenger name and National ID, train, class, coach, seat, stations, times, fare. |
| BR-38 | The ticket **QR is a signed payload** that a conductor can verify **offline**, from the QR alone. The online verification endpoint additionally rejects a **cancelled or voided** ticket. A forged or altered QR fails both. |
| BR-39 | **Share** produces a signed link that opens the ticket without signing in and **expires when the trip arrives**. **Download** produces a PDF in the request's language; the PDF attached to the confirmation email uses the account's language (BR-56). A cancelled ticket can no longer be shared or downloaded. |
| BR-40 | The Tickets tab has **Paid** (upcoming tickets first, soonest departure first; then past tickets, most recent first) and **Canceled** (cancelled or voided tickets with their refund status). |

### 5.7 Cancellation and refunds

| # | Rule |
|---|---|
| BR-41 | Each ticket — one passenger, one leg — is cancelled **on its own**. Cancelling one passenger's ticket does not touch the others. |
| BR-42 | The refund depends on the time left until the **passenger's boarding time**: **24 hours or more → 100% of the fare** · **2 to 24 hours → 50% of the fare** · **less than 2 hours, or after departure → cancellation refused**. The boundaries favour the passenger: exactly 24h00m00s is 100%, exactly 2h00m00s is 50%. Half of an odd number of piastres rounds half-up. |
| BR-43 | The traveller sees the refund amount **before** confirming. The server decides at the moment of confirmation; if the outcome changed since the preview (a boundary was crossed), the cancellation is **refused with an *outcome changed* code and the new preview** — never silently applied. |
| BR-44 | Cancelling frees **exactly that ticket's seat on that ticket's segment**, immediately bookable by someone else. |
| BR-45 | On a traveller's cancellation, the **booking fee is never refunded**. |
| BR-46 | A refund has its own lifecycle: **Pending** (badge *In Progress*, "Refund date: Pending") → **Refunded** (badge *Refunded*, the date) when Stripe confirms it — or **Failed**, which operators must be able to see and retry. The sum of all refunds on a booking never exceeds what was charged. |
| BR-47 | A ticket can be cancelled only once. Two cancellations of the same ticket at the same time produce **one** refund. |

### 5.8 Operations and disruptions

| # | Rule |
|---|---|
| BR-48 | When ENR **cancels a trip**, every ticket on it is **voided** and refunded **100% of the fare plus that ticket's share of the booking fee**. Each ticket's fee share is fixed once, when the booking is paid, and the shares of a booking **add up exactly to the fee charged** — so voiding every ticket returns exactly the fee, never a piastre more or less. Every affected traveller is notified. The trip disappears from search, and its seats can no longer be held. A trip can be cancelled only before it leaves its first stop. |
| BR-49 | When ENR **delays a trip**, the delay is declared **from a stop**: that stop's departure and every later stop shift by N minutes, delays on the same trip **add up**, and every affected traveller is notified. If the total delay at the passenger's boarding station is **60 minutes or more**, that ticket may be cancelled for a **full fare refund** regardless of BR-42 — including inside the last 2 hours, until the delayed departure. |
| BR-49a | After a delay, every rule that uses a boarding, departure or arrival time — sales close (BR-16), the refund tiers (BR-42), the share-link expiry (BR-39) and reminders (BR-53) — uses the **delayed** time. Search still lists the trip under its scheduled date. |
| BR-50 | A published trip with sold tickets can't be silently rewritten. Time changes go through the *delay* operation. Removing a stop that tickets board or leave at is refused. |
| BR-51 | A fare change applies to **new holds only** (BR-30). |
| BR-52 | Every operations action is **audited**: who, when, what, and how many tickets it affected. |

### 5.9 Notifications

| # | Rule |
|---|---|
| BR-53 | Notified events — each by **email, push and inbox**: **booking confirmed** (the email carries the tickets), **ticket cancelled**, **refund completed**, **trip delayed**, **trip cancelled**, **reminders** 24 hours and 2 hours before boarding. Verification and reset codes are email-only and never stored in the inbox. |
| BR-54 | A notification failure **never** undoes a business action, and the same event never notifies twice. |
| BR-55 | Reminders for a cancelled ticket or a cancelled trip are **never sent**. |

### 5.10 Language and errors

| # | Rule |
|---|---|
| BR-56 | Every message shown to a user is identified by a **stable code** and rendered in the requested language (`Accept-Language`: `ar` or `en`; anything else falls back to Arabic). Emails and push use the **language stored on the account**, because there is no request to read it from. Station and class names exist in both languages. |
| BR-57 | Every error tells the user **what to change**: which field, which value, and what to do next — never "Something went wrong". (Research finding: *"I don't know what to change: station, date, or round-trip option."*) |

---

## 6. Edge Cases

The happy path is about 20% of this project. These separate a working system from a demo.

### 6.1 Access

- A user signs up, never verifies, and signs up again a week later with the same email.
- The verification email arrives after the user already requested a new code. The old code must not work.
- Two identity instances count wrong sign-in attempts for the same account. Is the lock still at 10?
- Redis restarts in the middle of someone's 5 wrong attempts.
- A user signs in on a second phone. Is the PIN from the first phone valid there? (It must not be.)
- A refresh token is stolen and used after the real app already rotated it.
- A user resets their password. Their lost phone is still signed in with a PIN.

### 6.2 Profile and passengers

- The design's own sample National ID, `26582264865`, has **11 digits**. What does your API say about it?
- A National ID with birth date 31 February, or with governorate code `99`.
- A traveller tries to save the same National ID twice — and a different traveller saves the same person.
- A passenger is edited after a ticket was issued for them. What does the ticket show?
- Two accounts claim the same National ID as their "self".

### 6.3 Network and search

- An overnight train leaves Cairo at 23:00 and reaches Aswan at 09:30 the next day. Which date does it appear under for a passenger boarding at Assiut at 04:10?
- The night **daylight saving time** starts or ends in Egypt. A trip departs at 00:30 that night.
- A search for Tanta → Cairo on a train that runs Cairo → Alexandria. (Wrong direction — no result.)
- A search for today, at 14:00, for a trip that left at 13:50 — and for one leaving at 14:10 (sales closed).
- A trip is fully sold on Cairo → Tanta but has free seats Tanta → Alexandria.
- A user types "اسكندريه", "إسكندرية" and "alex". All three should find Alexandria.
- The timetable is republished while 10,000 search results are cached.

### 6.4 Seats and holds — the hard part

- **Two travellers try to hold the last seat on the same segment at the same instant.** Exactly one wins; the other gets a clear, recoverable error.
- Traveller A holds Cairo → Tanta on seat 44 while traveller B holds Tanta → Alexandria on seat 44 at the same instant. **Both must succeed.**
- Traveller A holds Cairo → Alexandria; B holds Banha → Tanta on the same seat at the same instant. **Exactly one** succeeds.
- A group of 4 asks for seats in a coach where only 3 adjacent seats are free.
- A round trip whose return leg has no seats. The outbound seats must not stay held.
- The same National ID is added to two different bookings on overlapping segments of the same trip, by two different accounts, at the same second.
- A traveller opens the app on two phones and holds on both.
- A hold expires while the traveller is on the 3-D Secure page.
- 10,000 holds expire in the same minute.

### 6.5 Payment and booking — money

- **Stripe succeeds, then the booking service crashes before confirming.** The customer has been charged and has no ticket. How do you find out, and how do you fix it?
- Stripe sends the same webhook three times, out of order with a refund webhook.
- The app retries "Confirm" because the network was slow. Same question: one charge?
- The payment succeeds 30 seconds after the hold expired, and the seats were taken by someone else.
- A fare is so small that the total is below Stripe's minimum charge amount. (Stripe sets the minimum by your account's settlement currency, not by EGP — find out what it is for your test account and decide what the traveller sees.)
- Rounding: 3 tickets at 65 EGP each — is the fee computed on the total or per ticket, and do the two ways ever differ by a piastre? (BR-31 says on the total. Prove you follow it.)
- The payment service is down when the traveller taps Confirm. What does the traveller see, and what happens to the hold?
- The Stripe API key is wrong in production. Who finds out, and when?

### 6.6 Tickets, cancellation and refunds

- A traveller cancels at **exactly 24h00m00s** before boarding — and at 23h59m59s.
- The preview says 100%; the traveller confirms 2 minutes later at 23h58m.
- A traveller cancels the same ticket from two phones at the same time.
- A traveller cancels 2 of 3 tickets in a booking, one at a time. Then ENR cancels the trip. What is refunded for the third, and for the booking fee?
- Stripe's refund fails (the card was closed). What does the Canceled tab show?
- A shared ticket link is opened after the ticket was cancelled.
- Somebody edits the seat number inside the QR payload and regenerates the QR image.

### 6.7 Disruptions and notifications

- ENR cancels a trip with **400 tickets** across 150 bookings. The notification service is down during the cancellation.
- ENR delays a trip by 45 minutes, then by another 20. Is the 60-minute free-cancel threshold reached?
- A reminder was scheduled for a ticket that was cancelled one minute later.
- A trip is cancelled while a traveller is mid-payment for it.
- The same `trip.cancelled` event is delivered twice. Nobody gets two refunds; nobody gets two emails.

> 🎯 **Deliverable:** pick the **five edge cases you consider most dangerous**, explain how your design handles each one across services, and review it with your mentor. "The other service handles it" is not an answer unless you can name the event or call that makes it so.

---

## 7. Data Modeling Challenge

> 💡 **You design the schemas — one per service. This section gives you the domain, not the tables.**

### 7.1 Entities you will need to discover

Group them by **owning service**:

**identity** — accounts, credentials, verification codes (or their hashes), sessions and refresh tokens, devices and their PIN credentials, profiles, saved passengers.

**catalog** — stations (bilingual), governorates, trains and train types, classes, coach layouts and seat maps, timetables (the pattern a train runs on), dated trips with their stops and times, fares, delays and cancellations, recent searches.

**inventory** — the seats of each dated trip and which segments of the route each seat is taken for, holds and their expiry.

**booking** — bookings, legs, tickets and their own copies of passenger and trip data, the saga's state, cancellations, refund decisions, local copies of the reference data tickets need.

**payment** — Stripe customers, saved card references (brand, last 4, expiry, holder name — nothing more), charges, refunds, received webhook events.

**notification** — device push tokens, the inbox, delivery attempts, scheduled reminders, language per account.

**Every service** — its outbox and its inbox (processed event IDs).

### 7.2 Design decisions you must make and defend

1. **How do you represent "seat 44 is taken from Banha to Tanta"?** One row per seat per segment? One row per seat with a range of stop indexes? A bitmask? *Hint: PostgreSQL has range types and exclusion constraints. Find out what `EXCLUDE USING gist` can guarantee that an `if` statement cannot.*
2. **How is "two travellers, one seat, same instant" made impossible?** Where exactly does the guarantee live, and what error does the loser get?
3. **Timetable vs dated trip.** Do you store trips ahead of time or compute them? How far ahead? What generates them, and what happens on a DST night?
4. **Where does the fare live** so that it depends on train, class and origin–destination — without a table with a million rows?
5. **What does booking store about a trip, a station or a passenger** it does not own — and what happens to that copy when the original changes?
6. **What is the saga's state**, where is it persisted, and what resumes it after a crash?
7. **Who enforces BR-27** (one National ID, one seat per overlapping segment)? It needs passenger data and seat data at once.
8. **How is money stored**, and how do you prove that refunds never exceed the charge?
9. **Which data is sensitive** (National ID, phone, email) and what does that change about how you store, return and log it?

### 7.3 What you must submit before writing entity code

- An **ERD per service**, with the relationships and cardinalities inside each service, and the **references across services** drawn differently (an ID, not a foreign key).
- The **constraints** you rely on for correctness — unique, check, exclusion, foreign keys — and what each one protects against.
- A short answer to each question in §7.2.

> 📝 **Mentor review required** before you generate migrations.

---

## 8. Service Design & API Extraction Challenge

You are not given endpoints, `.proto` files or events. Derive them from the Figma screens, the stories and this document.

**Deliverables before writing controllers:**

1. **Service map** — every service, what it owns, every sync call (caller → callee, method) and every event (producer → consumers).
2. **Gateway endpoint table** — method, path, purpose, access (public / traveller / bookable traveller / operator / conductor), and which services it calls, for every screen in Appendix C.
3. **`.proto` contracts** for every synchronous call, with deadlines and the errors each method can return.
4. **Event catalogue** — name (past tense), producer, when it is published, payload, version, every consumer and what each consumer does with it.
5. **Error catalogue** — stable codes for every failure the app must react to differently (e.g. *email taken*, *code expired*, *profile incomplete*, *seat taken*, *hold expired*, *payment declined*, *outcome changed*, *cancellation window closed*), with the HTTP status and the Arabic and English message.
6. **Booking saga sequence diagram** — happy path, decline, hold expiry, late payment, crash in the middle.
7. **Cache plan** (§2.3).

> 📝 **Mentor review required** (Sprint 1, day 2) before implementing services.

---

## 9. Product Decisions (already made)

The design does not answer everything. The product team ruled on the questions below. **These are requirements.** The right-hand column is still your work.

| # | The question | The ruling | What you must design |
|---|---|---|---|
| D1 | Does a seat belong to the whole train run, or to a part of it? | **To a segment.** A seat freed at Tanta is sold again from Tanta (BR-23). | A representation of "taken from stop *i* to stop *j*" that the database can protect against overlap |
| D2 | Who picks the seats? | **The server**, keeping the group side by side, then in one coach, then split — and a split is shown before payment (BR-25). | An allocation algorithm over a coach layout, and a definition of "adjacent" for that layout |
| D3 | How long can seats be held? | **15 minutes**, plus **5** once for 3-D Secure (BR-26). | What releases abandoned holds — reliably, at volume — and what the traveller sees after expiry |
| D4 | Must users give a National ID at sign-up? | **No.** Sign-up is email + password. The profile, including the National ID, is required before the **first hold** (BR-10). | How the system expresses "signed in, but not bookable" — enforced once, not in every handler |
| D5 | How many passengers per booking? | **1 to 4**, from saved passengers (BR-28). | — |
| D6 | Can one person be on two seats? | **No** — not on overlapping segments of the same trip, across all accounts (BR-27). | Which service owns this invariant and with which data |
| D7 | What is a round trip? | **One booking, two legs, one payment**, held together or not at all (BR-20, BR-26, BR-37). | How a two-leg hold is all-or-nothing when each leg is a different trip |
| D8 | How is the booking fee calculated? | **7% of the total fare, half-up to the piastre** (BR-31). Never refunded on a traveller's cancellation (BR-45); refunded on ENR's (BR-48). | Where the calculation lives so that payment, booking and refunds can never disagree |
| D9 | What are the refund tiers? | **≥24h → 100%** · **2–24h → 50%** · **<2h → refused**; boundaries favour the passenger (BR-42). | A pure, tested policy and a guard for the outcome changing between preview and confirm |
| D10 | What happens when ENR cancels or delays? | **Cancel → full refund including fee share, tickets voided.** **Delay ≥ 60 min → free cancellation** (BR-48, BR-49). | The fan-out across services, and how you know it finished for all 400 tickets |
| D11 | How is a ticket checked on the train? | A **signed QR** verifiable **offline**; an online check also rejects cancelled tickets (BR-38). | Which signature scheme lets a conductor verify without holding a secret that could forge tickets |
| D12 | What does a PIN protect? | **One device.** Never a password substitute on another device (BR-07). | The device credential, and how the PIN unlocks it without the server ever storing the PIN in clear |
| D13 | Payments in which provider and currency? | **Stripe test mode, EGP, integer piastres** (BR-31, BR-32). | Idempotency keys, webhook verification and deduplication, reconciliation |
| D14 | Which language does the server speak? | **Both.** Codes in the API, `ar`/`en` rendering, account language for emails and push (BR-56). | How messages stay translatable without touching business code |

> 🎯 **Deliverable:** for each ruling, show your mentor **where** it is enforced — the service, the module, the constraint or the scheduled job. A ruling with no visible home is a ruling you have not implemented.

---

## 10. Best Practices

### A. Engineering

1. **Separation of concerns, per service** — transport (HTTP, gRPC, AMQP handlers) is thin; rules live in domain services or policies; persistence lives in repositories.
2. **Fail fast on configuration** — every service, at boot.
3. **Centralized error handling** — and a deliberate mapping between domain errors, gRPC status codes and HTTP status codes.
4. **Never log a secret or personal data** — and remember logs are now spread across seven services.
5. **Idempotency everywhere money or messages are involved.**

### B. Microservices

Follow the `MS-###` rules in [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md). The ones reviewers will check first: database per service, no shared domain code, deadlines on every call, outbox for every event, idempotent consumers, dead-letter queues, correlation IDs, health checks, graceful shutdown.

### C. Events

1. **Past tense, business language** — `trip.cancelled`, `seats.held`, `payment.succeeded`.
2. **An envelope on every event** — ID, type, version, time, correlation ID, producer.
3. **Consumers never break producers**, and are **safe to run twice**.
4. **Retry with backoff, then dead-letter** — and have a way to replay the dead-letter queue.

### D. Cache

See §2.3. The test: for any key, you can say **who writes it, what removes it, and what happens without it**.

### E. REST (gateway)

Resource-oriented plural paths · correct verbs and status codes · one envelope · pagination on every list · `409 Conflict` when a seat was taken or a cancellation outcome changed · `422` for a request that is valid JSON but breaks a rule · `429` with a retry hint for rate limits · an **`Idempotency-Key`** header on every request that creates money movement.

---

## 11. Testing Requirements

**Unit tests** for every rule, not for getters:

1. **Refund policy** — 24h00m00s, 23h59m59s, 2h00m00s, 1h59m59s, after departure, delayed ≥ 60 min, trip cancelled.
2. **Money** — 7% fee rounding, totals, partial refunds never exceeding the charge, fee share on trip cancellation.
3. **National ID validation** — length, century digit, impossible dates, future dates, unknown governorate.
4. **Segments** — overlap vs touching segments; availability counts per segment.
5. **Allocation** — adjacent, same coach, split; groups of 1 to 4.
6. **Search** — any two stops in order, wrong direction, next-day arrival, sales closed, booking window.
7. **Arabic normalization** — the variants in §6.3.

**Concurrency tests** — two holds for the last seat; overlapping vs touching segments at the same instant; the same National ID in two bookings; the same ticket cancelled twice.

**Integration tests per service** — with real Postgres, Redis and RabbitMQ (Testcontainers). No shared test database between services.

**Contract tests** — every `.proto` and every event schema: a consumer's expectations break the build when a producer changes them.

**End-to-end tests** through Docker Compose for the flows that move money:

- search → hold → pay (Stripe test card) → tickets issued → confirmation email in Mailpit;
- cancel → refund decision → seat released → refund webhook → *Refunded*;
- payment succeeds after the hold expired;
- trip cancelled → every ticket voided and refunded exactly once.

---

## 12. Execution Roadmap

| Sprint | Theme | Stories |
|---|---|---|
| **1** | Foundations & Identity — **design gate on days 1–2** | [`sprint-1-foundations-identity-stories.md`](./sprint-1-foundations-identity-stories.md) |
| **2** | Network, Search & Cache | [`sprint-2-network-search-stories.md`](./sprint-2-network-search-stories.md) |
| **3** | Seats, Holds & Booking | [`sprint-3-seats-holds-stories.md`](./sprint-3-seats-holds-stories.md) |
| **4** | Payment, Tickets & Cancellation | [`sprint-4-payment-tickets-stories.md`](./sprint-4-payment-tickets-stories.md) |
| **5** | Disruptions, Notifications & Go-live | [`sprint-5-disruptions-golive-stories.md`](./sprint-5-disruptions-golive-stories.md) |

Each sprint is one week. Every sprint ends with a **demo against the running Compose stack**, not against a single service with mocks.

---

## Appendix A — Not Designed

These appear in the research or as buttons, but the Figma has **no screens** for them. Do not invent them — but do not make them impossible either.

Visual seat map (the research's top "what's next") · verifying ownership of a phone number or National ID · train-type information · the notifications inbox screen · profile edit screen · Contact Us · FAQs · discounted passenger types · non-Egyptian travellers · children's fares · changing a booked ticket to another train · group bookings above 4 · loyalty or subscriptions · an operator or conductor app.

## Appendix B — Glossary

**Station** — a place a train stops. **Train** — a numbered service (916) with a type and coaches. **Trip** — one dated run of a train. **Stop** — a station on a trip, with an index, an arrival and a departure time. **Segment** — the part of a trip between a passenger's boarding and alighting stops. **Class** — a seating category with its own coaches and fares. **Hold** — seats reserved temporarily while paying. **Booking** — one purchase: up to 4 passengers, one or two legs, one payment. **Leg** — one direction of a booking. **Ticket** — one passenger on one leg. **Booking reference** — the code that identifies a booking. **Saga** — a multi-step, multi-service process with compensations. **Outbox / inbox** — the tables that make publishing and consuming events reliable. **Piastre** — 1/100 of an Egyptian pound.

## Appendix C — Screens to Cover

Every screen below is in the Figma file (in English and Arabic) and is in scope.

| Area | Screens |
|---|---|
| **Start** | Splash · Choose Language · Onboarding 1–3 |
| **Access** | Sign up · Password doesn't match · Verify your e-mail (counting down, and at 00:00) · Sign in · "Forgot your password ?" (a link only — the recovery steps reuse the Verify pattern) · Create PIN · Confirm PIN · Enter your PIN · Touch ID prompt |
| **Profile** | Complete your profile |
| **Search** | Home (One way, Round trip, with Train Number) · Station picker · Calendar · Results with date strip · Stops timeline · No results |
| **Booking** | Choose Passengers · Add passenger · Review Tickets (wireframe) · Payment · Add card · Payment Success |
| **Tickets** | Paid · Canceled (*In Progress* and *Refunded*) · Share · QR · Download · Cancel trip |
| **Passengers** | List · Add · Edit · Delete |
| **Settings** | Profile · Payment Methods · Logout |
| **No design — you define the API** | Notifications inbox · Operations API · Conductor ticket verification |

## Appendix D — Known Inconsistencies in the Design

The Figma is a UX redesign, not a specification. Where it disagrees with this document, **this document wins**:

| In the design | The rule |
|---|---|
| Class labels "1st Class" (results header) and "AC 3 (4)" (ticket card) | Plain class names in both languages (BR-22) — the research itself asked to drop codes like "AC 3" |
| The sample National ID `26582264865` has 11 digits | 14 digits with a valid structure (BR-11) |
| Sample dates disagree (header "Wed, 24 Oct", card "27 Oct, 2025") | Results show the dates of the selected day's trips (BR-16, BR-17) |
| "Damnhour" in the stops timeline | Station names come from your seeded data, spelled correctly in both languages |
| The Verify screen shows the email in full ("olivia@untitledui.com") | The response carries a **masked** email |
| The ticket card shows no coach or seat | A ticket shows its coach and seat (BR-37) — the app needs them to find the seat |
| Review Tickets exists only as a wireframe, without the fee and total | The fee and total are computed when the hold is made and shown on the Payment screen (BR-31) |
