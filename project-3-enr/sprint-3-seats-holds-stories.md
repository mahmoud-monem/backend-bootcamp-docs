# Sprint 3 — Seats, Holds & Booking

**Project:** ENR — Egyptian National Railways Ticket Booking (Bootcamp Project 3)
**Requirements:** [`requirements.md`](./requirements.md)
**Microservices guide:** [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)
**Design:** [Figma — Egyptian National Railways](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
**Jira import file:** [`sprint-3-seats-holds-stories.csv`](./sprint-3-seats-holds-stories.csv)

---

## How to read these stories

Same contract as Sprints 1 and 2: each story is **a service flow** — the request a client makes (the mobile app, an operator tool, a test script), or the work the system does on its own, and the services that request passes through — followed by the rules that flow must obey, the events it publishes, the edge cases, and the acceptance criteria you are graded on.

No client UI is built in this project. The Figma design shows what the API must make possible — which data a step needs, which refusals it must be able to tell apart — not screens you build.

What you will **not** find here: endpoint paths, request bodies, table or column names, queue names, routing keys or `.proto` definitions. Those are your design work — derive them from the flow, the rules and what the Figma screens need from the API. Events are named in plain language (*"a hold expired"*) because the event catalogue — names, payloads, versions, producers and consumers — is one of your deliverables (requirements §8).

**Every story names its services.** Each Service flow starts with a `Services:` line listing the services involved and the role each one plays. The owner of the **first-listed** service is accountable for the story — its design, its tests, and getting it merged — even when teammates write much of the code. Every other service on that line reviews the parts that touch it.

**What is different about this sprint.** This is the **hardest sprint in the project** — the concurrency core, the way BOOK-2 was in DOCURA — and it is graded harder for a reason:

- A bug here sells seat 44 on the Banha → Tanta stretch to two people, or lets one National ID sit in two seats on the same train.
- Several acceptance criteria can only be satisfied by a **concurrent test against real Postgres** — two requests released at the same instant, repeated until a lucky pass is impossible — not by reading the code and reasoning that it looks fine.
- The guarantee lives **in inventory's database**. If you find yourself writing *count the free seats, then insert the hold* as two statements with nothing in the database to stop a second traveller between them, stop and re-read SEAT-1 and SEAT-3.
- BOOK-2 is your first saga: one process across four services that survives a crash in the middle and undoes what it did when time runs out.
- SEAT-4 is the other half of the cache lesson: search may return a number a few seconds old; the hold never reads it (BR-29).

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

**6 stories · 34 points**

| # | Story | Points |
|---|---|---|
| SEAT-1 | Segment-aware seat inventory — a seat is taken from one stop to another, and the database guarantees it | 8 |
| SEAT-2 | Group seat allocation — side by side, then one coach, then split, and never silently | 5 |
| SEAT-3 | Holds — fifteen minutes, all or nothing, exactly one winner | 8 |
| SEAT-4 | Availability counts reach search — fast numbers from the cache, kept honest by events | 3 |
| BOOK-1 | The booking draft — passengers by reference, a locked price and a server-owned hold expiry | 5 |
| BOOK-2 | The booking saga — a persisted orchestrator that survives crashes and gives the seats back | 5 |

> **This is the hardest sprint in the project.** Points are in line with the other sprints, but the risk is not: SEAT-1 and SEAT-3 are where your data model meets two travellers at the same instant, and they are the stories most likely to send you back to the ERD. If capacity is tight, cut in this order: **(1)** SEAT-4's storm coalescing and Redis-down fallback — ship the plain event-to-cache update and carry the rest into Sprint 4 (search counts may lag; the hold is unaffected because it never reads them); **(2)** the 3-D Secure extension inside SEAT-3 — nothing calls it until PAY-2, so it can land with PAY-2. **Never cut** the database guarantee, the concurrency suite, automatic expiry, BR-27, or BOOK-2's compensation: Sprint 4 charges cards against these holds, and a hold that cannot be trusted turns every Sprint 4 story into a refund story.

### Suggested order

```
SEAT-1  Segment-aware inventory   ── seats must exist per trip and per segment before anyone can take one
   │
   ├─▶ SEAT-2  Group allocation    ── a pure function over the coach layout; start day one with fixtures
   │      │
   └──────┴─▶ SEAT-3  Holds         ◀── the concurrency guarantee lives here. Do it properly.
                 │
                 ├─▶ SEAT-4  Counts reach search
                 │
                 └─▶ BOOK-2  Saga orchestrator  ◀──▶  BOOK-1  Booking draft
                                                      (build together: BOOK-1 is the contract the client calls, BOOK-2 the engine)
```

**Write the concurrency tests on day one**, red, before the hold exists: two travellers, last seat; touching segments; overlapping segments; one National ID in two accounts. They are the specification of SEAT-1 and SEAT-3, and writing them first is the fastest way to discover whether your representation of "taken from stop *i* to stop *j*" can actually be protected by the database.

### What carries over

| Earlier story | Used here by |
|---|---|
| AUTH-5 complete profile and the bookable mark | SEAT-3, BOOK-1 — the single place BR-10 is enforced before the first hold |
| AUTH-6 saved passengers | BOOK-1 passengers by reference · SEAT-3 the National ID behind BR-27 |
| AUTH-1 gateway access levels and idempotency-key store | BOOK-1 — the bookable-traveller entry and the repeated submit |
| CAT-2 coach layouts, trips with ordered stops, fares per origin–destination and class | SEAT-1 seats and stop indexes · SEAT-2 adjacency · BOOK-1 the fare quote |
| CAT-3 / CAT-5 cached availability counts, key naming, stampede protection | SEAT-4 — the keys inventory's events now keep fresh |
| OPS-1 trip publication and fare changes | SEAT-1 consumes the publication · BOOK-1 the price lock against a fare change (BR-51) |
| PLAT-1 sequence diagram, `.proto` contracts, event catalogue | BOOK-2 — the code must match the diagram |
| PLAT-2 envelope and localized error codes | every refusal this sprint: seat taken, profile incomplete, hold expired, passenger already travelling |
| PLAT-3 outbox, inbox, retry and dead-letter | every event published and consumed this sprint |

### Suggested split

| Student | Owns | Takes this sprint | ≈ Points |
|---|---|---|---|
| S1 | gateway + identity | SEAT-2 group allocation (the algorithm and its unit tests) · BOOK-1's entry: the bookable-traveller check and the passenger lookup from identity | 6 |
| S2 | catalog | SEAT-4 (catalog's cached counts) · SEAT-1's catalog side (what the trip publication carries, versioning) · BOOK-1's fare quote | 6 |
| S3 | inventory | SEAT-1 (representation, database guarantee, the publication consumer) · SEAT-3's constraint and the concurrency suite, with S5 | 8 |
| S4 | booking | BOOK-2 saga orchestrator · BOOK-1 booking draft, fee and total, and the draft's one response | 7 |
| S5 | payment + notification | SEAT-3 holds: expiry release at volume, the extension, one hold per traveller, BR-27 · payment's start/cancel contract and the test double behind it for BOOK-2 | 7 |

Four of six stories land in inventory, so the inventory owner (S3) **reviews every pull request that touches inventory** — including S1's allocator and S5's expiry release. The same holds for every service: its owner reviews every PR that touches it, whoever wrote it.

### Definition of Done (every story)

- Every rule is enforced **in the service that owns it**; the client and the other services are never trusted with a price, a seat, a passenger's identity or an allowed action.
- Every acceptance criterion has a test, including the negative ones and, where stated, a **concurrent** one.
- Every edge case listed has a deliberate, defined behaviour.
- Concurrency guarantees live **in the database**, not in an `if` statement (requirements §2.4).
- Events are published through the **outbox**; every consumer is **idempotent** (inbox), retries with backoff and dead-letters what it cannot process — never drops it. A failing consumer never undoes the business change.
- Every synchronous call has a **deadline**; only idempotent calls are retried.
- No secret, token, full National ID or full phone number in any log, in any of the services this sprint touches — passengers' National IDs now travel between services.
- Money is integer piastres everywhere, including inside events and gRPC messages.
- Errors use the gateway envelope and stable codes with Arabic and English messages; a client can tell *seat taken*, *not enough seats*, *passenger already travelling*, *profile incomplete*, *sales closed* and *hold expired* apart.
- `.proto` contracts and event schemas are updated in `libs/contracts`, and their contract tests pass.
- It works with `docker compose up` against the running stack, not only in unit tests.
- **Sprint 3 additions:**
  - A **concurrency test suite runs in CI against real Postgres** (Testcontainers): every concurrent criterion in SEAT-3, each run many times per build (at least 50 iterations), so a lucky pass is not a pass. A concurrency criterion proven with mocks is not done.
  - The hold guarantee is **demonstrated live with two terminals** in the sprint demo: a trip with one seat left in a class, two travellers, both requests fired at the same moment against the Compose stack — one gets the booking draft with its hold, the other gets the seat-taken code. Then again with touching segments, where both win.
  - The booking saga's sequence diagram from PLAT-1 matches the code at the end of the sprint.

---

## EPIC-3 — Epic: SEATS & HOLDS — Inventory, Allocation and the Booking Saga

Everything between "the trip is on the timetable" and "these seats are mine while I pay":
the seats of every dated trip, tracked per segment; the server's group allocation; the
fifteen-minute hold with its expiry and its one extension; availability counts that follow
inventory into the search cache; the booking draft — passengers by reference, a locked
price, one fee and total, and the hold's expiry; and the booking saga, up to the moment it
waits for a payment.

Source of truth: requirements §1.2, §2.2, §2.3, Modules 5 and 6, §5.4, BR-30 and BR-31,
§6.4, §7.2 (questions 1, 2, 6 and 7) and rulings D1, D2, D3, D5, D6, D7 and D8. The Figma
screens — the Results card ("Available Seats: 170"), the Stops sheet, Choose Passengers, the
Review Tickets wireframe and the Payment booking summary — and the UX research on the
Presentation page ("Seats weren't next to each other", the "Auto" button) are context for
what the API must make possible, not deliverables.

Depends on Sprint 1 (platform, messaging backbone, gateway, sessions) and Sprint 2 (complete
profile, saved passengers, coach layouts, trips, fares, the search cache and trip publication).
Sprint 4 converts these holds into sales and charges real Stripe test cards against them.

This is the hardest sprint in the project. It is where two travellers collide, and where a
process first spans four services that can each fail on their own.

**Epic acceptance criteria**

- A published trip has its seats in inventory without anyone creating them by hand — exactly once, even when the publication arrives twice.
- A seat is taken per segment: overlapping segments exclude each other, touching segments share the seat — proven by concurrent tests against real Postgres.
- A group is seated side by side, else in one coach, else split — and a split is reported in the booking draft's response before payment can start.
- A hold lasts 15 minutes, covers every passenger and every leg or nothing, is decided by inventory's database, and is released automatically, at volume.
- One National ID never occupies two seats on overlapping segments of the same trip, across all accounts.
- Search counts follow inventory within the stated staleness bound, and search still works when Redis is down.
- A traveller's submitted passengers become a booking draft with a locked fare, a booking fee computed in one place, and the hold's expiry returned with the server's time.
- The booking saga is persisted, resumes after a crash, compensates when the hold expires, and its code matches the sequence diagram.
- Two terminals, one seat, same instant: one winner — live, in the demo.

---

## SEAT-1 — Segment-aware seat inventory — a seat is taken from one stop to another, and the database guarantees it

**8 points · Priority Highest · `inventory,catalog,seats,segments,concurrency,sprint-3`**

### User story
As the railway
I want every seat of every published trip tracked for each part of the route
So that a seat freed at Tanta is sold again from Tanta, and no seat is ever sold twice for the same stretch

### Service flow
Services: inventory (owner — creates each trip's seats and decides who occupies which seat on which segment), catalog (producer — publishes the trip with its ordered stops and coach layout, CAT-2 / OPS-1)

No traveller request. The flow starts with an operator and ends on a number search already
returns:

  the operator publishes a timetable (OPS-1)
  -> catalog generates the dated trips: "train 916 on 27 October"
  -> catalog announces, through its outbox, that each trip was published
  -> inventory consumes it (inbox) and creates every seat of every coach for that trip
  -> search can now return "Available Seats: 170" for that trip — and that number is inventory's

Then the part this story exists for. Take the stops of a Cairo → Alexandria train, as
catalog returns them:

  Ramsis Station, Cairo 07:30 PM -> Banha -> Tanta -> Damnhour -> Sidi Gaber -> ElRaml Station, Alex 11:00 PM

Seat 44 in coach 5 is not one thing that is either sold or free. It is free or taken for
each stretch of that line. A passenger from Cairo to Tanta and another from Tanta to
Alexandria can both sit in it. A passenger from Banha to Sidi Gaber cannot, if either of
them already does.

### Rules and validations
- Seats come from the publication: Inventory creates a dated trip's seats from its coach layout when it learns the trip was published (OPS-1). Nobody creates seats by hand, and the seed goes through the same path as production.
- Inventory keeps its own copy: Inventory never reads catalog's database (§2.2). It keeps what it needs — the trip's ordered stops, its coaches, their class and their seats, and whatever the layout says about which seats sit together (SEAT-2). Decide whether the publication carries all of that or inventory asks catalog for it, and write down the trade-off.
- A segment is stop *i* to stop *j*: Every occupation is for a segment between two stops of the trip, in travel order (BR-15). A station the trip does not stop at, the same station twice, or a destination before the origin is refused with a code naming the problem.
- Taken per segment (BR-23, D1): A seat is free for a segment only if no active hold and no sold ticket on that seat overlaps it. Your representation covers both *held* and *sold*, so that Sprint 4 can turn a hold into a sale (BOOK-3) and free one passenger's segment alone (BR-44) without a migration.
- Touching is not overlapping: Cairo → Tanta and Tanta → Alexandria share a seat; Cairo → Alexandria and Banha → Tanta do not. Define a segment so that touching ends never collide, write the definition down once, and test every pair on a six-stop trip (§11, unit test 4).
- Segments are stops, not clock times: Overlap is decided by stop order. An overnight trip, a next-day arrival or a daylight-saving night must not change which segments overlap.
- The guarantee lives in the database (§2.4, §7.2 Q1–Q2): Two overlapping occupations of the same seat on the same trip cannot both be written — whatever the application code does, however many instances run. *Hint: PostgreSQL has range types and exclusion constraints. Find out what `EXCLUDE USING gist` can guarantee that an `if` statement cannot.* The table design is yours; your mentor reviews the constraint, not the `if`.
- Availability count (BR-24): The count for a trip, class and segment is the number of seats of that class free for the **whole** segment. It is computed from inventory's database, and a page of search results gets its counts without one query per trip or per segment.
- Exactly once: The same publication delivered twice creates the seats once (PLAT-3 inbox, plus a natural uniqueness for "this seat of this coach on this trip").
- All or nothing per trip: A trip's seats appear in one transaction. A half-built trip — some coaches created, others not — never exists, and a trip is not on sale until its seats are complete.
- Newer wins: Publications carry a version. An older publication arriving after a newer one never overwrites it, and a later publication can never re-open a trip that was closed.
- Occupied seats are never removed: A re-publication that would remove a coach or seat with an active hold or a sold ticket is not applied silently (BR-50). It is refused, logged and dead-lettered where an operator can see it.
- Closing must be possible: Sprint 5 (OPS-2) closes a cancelled trip's inventory — no new holds, existing records kept for the refunds. Nothing triggers it yet, but the model must allow it now.
- Inventory is the only writer: No other service writes seats or occupations. Catalog's counts are a copy (SEAT-4).

### Domain events to publish
- A trip's seats were created and the trip is on sale (consumed by catalog to fill its first availability counts, SEAT-4).
- A trip's publication was refused by inventory (dead-lettered and logged, so an operator sees a re-publication that would have removed an occupied seat).

### Edge cases to handle
- The publication is delivered twice — inventory crashed after committing the seats and before acknowledging the message.
- Version 2 of a trip's publication (a coach added) arrives before version 1.
- The publication arrives before catalog can answer for that trip — catalog is still generating the rest of the timetable, is restarting, or does not know the layout yet — when inventory asks for it.
- An operator publishes 30 days of timetable for the whole seeded network at once: thousands of trips, hundreds of thousands of seats.
- A coach layout with a gap in its numbering (no seat 13), or a class with no coaches on this train.
- A trip that crosses midnight and a daylight-saving change between two of its stops.

### Acceptance criteria

- Given OPS-1 publishes a trip whose train has coaches of two classes, when inventory processes it, then every seat of every coach exists for that trip with its class, and the count for every segment and class equals the layout's seat count.
- Given the same publication is delivered twice, when both deliveries are processed, then the seats exist once and every count is unchanged.
- Given seat 44 is held Cairo → Tanta, when counts are computed, then seat 44 is free for Tanta → Alexandria and taken for Banha → Tanta and for Cairo → Alexandria.
- Given the application's own checks are bypassed and two overlapping occupations of the same seat are written directly, when the second write runs, then the database refuses it — proven by a test against real Postgres.
- Given the same direct write for two touching segments, when both run, then the database accepts both.
- Given every pair of segments on a six-stop trip, when overlap is evaluated, then touching pairs never overlap and every other intersecting pair does — a table-driven unit test.
- Given an older publication arrives after a newer one, when it is processed, then the trip keeps the newer stops and layout.
- Given catalog cannot supply the layout yet, when the publication is processed, then no seat of that trip exists, the message is retried with backoff, and it is dead-lettered after the limit — never dropped.
- Given a hold is requested for a trip inventory has not created yet, when it arrives, then it is refused with a *not on sale yet* code rather than an error.
- Given a segment whose destination comes before its origin, or a station the trip does not stop at, when a count or a hold is requested, then it is refused with a code naming the station.
- Given a re-publication that would remove a coach with an active hold, when it is processed, then the occupied seats remain and the refusal is logged and dead-lettered.
- Given a trip marked closed, when a hold is requested, then it is refused and its existing occupations are untouched.
- Given a page of 20 trips, when inventory returns their counts, then the number of database round trips does not grow with the number of trips or segments.

---

## SEAT-2 — Group seat allocation — side by side, then one coach, then split, and never silently

**5 points · Priority High · `inventory,catalog,allocation,sprint-3`**

### User story
As a traveller booking for my friends or family
I want the railway to seat us next to each other without me having to ask
So that I never restart a booking just to sit beside my companion

### Service flow
Services: inventory (owner — allocates the seats inside the hold), catalog (source of the coach layouts and their seat arrangement, CAT-2), booking (returns the allocation in the booking draft, BOOK-1)

  the client submits three saved passengers for a trip, a class and a segment (BOOK-1)
  -> gateway -> booking -> inventory (gRPC): hold seats for three passengers
  -> inventory reads its copy of the coach layout and the seats of that class free for the
     whole segment
  -> the allocator picks: side by side in one coach, else one coach, else split
  -> inventory writes the hold; the database constraint decides (SEAT-1, SEAT-3)
  -> the hold carries, per passenger, the coach and seat, and the level reached:

       "Passenger · Coach 5 · Seat 44"
       "Passenger · Coach 5 · Seat 45"
       "Passenger · Coach 5 · Seat 46"      outcome: side by side

  -> booking returns them in the booking draft, with the split notice when there is one

The research is why this story exists. In the old app, a user booking for two said
"Seats weren't next to each other." and restarted the whole booking twice to sit beside
their companion. Every participant was confused by an "Auto" button that silently assigned
seats. The redesign takes the choice away from the client: the server allocates, keeps the
group together, and **reports the result before payment can start**. There is no seat map —
the research lists it as "What's Next", and it is not designed (Appendix A) — so no request
ever names a seat.

### Rules and validations
- The server allocates (BR-25, D2): The client never sends a seat or a coach. A request that tries is rejected by validation, not quietly ignored.
- The order is fixed: For a group, try in this order: **all side by side in one coach**, then **all in one coach**, then **split across coaches**. A same-coach allocation always beats a split, even when the split would keep pairs together.
- A split is as small as it can be: When the group must be split, use as few coaches as possible and keep as many companions side by side as the layout allows. Document the tie-break.
- Adjacency comes from the layout, not from arithmetic: "Side by side" is defined by the coach layout CAT-2 stores — rows, aisles, which seats share a row — never by "44 and 45 are consecutive numbers". Define it for groups of 2, 3 and 4 on every layout you seed, with a small diagram in inventory's README.
- Only seats free for the whole segment: Candidates are seats of the requested class that are free for the entire requested segment (BR-23, BR-24). A seat free from Cairo to Tanta but taken from Tanta onward is not a candidate for Cairo → Alexandria.
- One class, one segment per leg: Every passenger on a leg travels in the class searched, between the same two stations.
- Deterministic: The same free seats and the same group always produce the same allocation. No randomness; ties are broken by a documented rule.
- Pure and testable: The algorithm is a function of the layout, the free seats and the group size — no database, no clock, no network — so it is unit-tested for groups of 1 to 4 at every level of the fallback (§11, unit test 5).
- Allocation reads, the database decides: The allocator chooses from a snapshot; the constraint from SEAT-1 is what makes the choice true. When the write fails because another traveller took a chosen seat in between, allocate again from a fresh snapshot — a bounded number of times — then give up with the seat-taken code. Never loop forever.
- Never silent: The result says which level was reached — side by side, same coach, or split — so the booking draft's response reports it before payment (BR-25). The design has no copy for the split notice; give it a stable code with Arabic and English messages, carried in that response.
- All or nothing: A group of four is never offered three seats. When the class does not have enough seats on the segment, the request is refused whole, with a code and the number of seats that are free, so the client can suggest fewer passengers or another class (BR-57).
- Each leg on its own: On a round trip, each leg is allocated independently; the outbound can be side by side while the return is split, and each leg reports its own outcome.
- One traveller alone: Decide whether a single traveller is steered away from breaking up a free side-by-side pair, so groups later still find one. Document the choice either way.

### Domain events to publish
- None of its own. The allocation outcome (side by side, same coach, split) travels in the *seats were held* event from SEAT-3, so the team can measure how often groups are split — the research problem this story fixes.

### Edge cases to handle
- A group of 4 asks for seats in a coach where only 3 adjacent seats are free (§6.4).
- Coach 3 has four free seats scattered; coaches 5 and 6 each have a free side-by-side pair. The rule says coach 3.
- Seats 44 and 45 are numbered consecutively but sit across the aisle from each other in this layout.
- A seat is free for most of the requested segment but taken for one stop of it.
- The seat the allocator chose is taken by another traveller between the snapshot and the write.
- The layout of a train is republished with a different arrangement while holds exist on it.

### Acceptance criteria

- Given a coach with two side-by-side seats free, when a group of 2 is allocated, then they get those seats and the outcome is *side by side*.
- Given no coach has three side-by-side seats free but coach 5 has three free seats, when a group of 3 is allocated, then all three are in coach 5 and the outcome is *same coach*.
- Given no coach has four free seats, when a group of 4 is allocated, then it is split across the fewest coaches possible and the outcome is *split*.
- Given coach 3 has four scattered free seats and coaches 5 and 6 each have a side-by-side pair, when a group of 4 is allocated, then all four are in coach 3.
- Given only 3 seats of the class are free on the segment, when a group of 4 asks, then nothing is held and the refusal says 3 seats are free.
- Given two seats numbered consecutively but across the aisle in the layout, when side-by-side seats are sought, then they are not treated as side by side.
- Given a seat free for part of the requested segment only, when the group is allocated, then that seat is never chosen.
- Given the same free seats and the same group twice, when the allocation runs, then the result is identical.
- Given a request that names a seat or a coach, when it reaches the gateway, then it is rejected by validation.
- Given the allocator's chosen seat is taken concurrently, when the write fails, then the allocator retries from a fresh snapshot and either holds other seats or gives up with the seat-taken code after the documented number of attempts.
- Given a round trip, when both legs are allocated, then each leg reports its own outcome and the booking draft's response can carry a split on one leg only.
- Given the unit-test suite, when it runs, then groups of 1, 2, 3 and 4 are covered at each of the three levels without a database.

---

## SEAT-3 — Holds — fifteen minutes, all or nothing, exactly one winner

**8 points · Priority Highest · `inventory,booking,gateway,holds,concurrency,sprint-3`**

### User story
As a traveller who has just submitted my passengers
I want the seats I was given to be mine while I review and pay
So that nobody else can take them halfway through, and I know exactly how long I have

### Service flow
Services: inventory (owner — holds, expiry, the database guarantee), booking (the only caller, through the saga in BOOK-2), gateway (the bookable-traveller entry check, BR-10), identity (source of the passengers' National IDs, through booking)

  the client submits passengers (BOOK-1)
  -> gateway: bookable-traveller check (BR-10)
  -> booking -> inventory (gRPC, with booking's key): hold the allocated seats, every
     passenger, every leg
  -> inventory's database decides; the hold gets its expiry on inventory's clock
  -> the response carries the seats, the hold's expiry and the server's current time —
     what a client shows as "Reservation Time 14:48"
  -> (Sprint 4) payment starts the bank's 3-D Secure step -> inventory extends the hold once

Now the same moment from two clients. Two travellers on the same train, the same class, the
last free seat on overlapping segments, both submit in the same second. One of them gets the
hold. The other gets the seat-taken code, with a message saying the seats were just taken
and what to do next — choose another train or another class. Neither of them gets a server
error, and no response ever tells a traveller they have a seat they do not have.

Nothing about the hold is decided by a cached number. "Available Seats: 170" in search
results is a cached copy (SEAT-4); the hold asks inventory's database, at that moment.

### Rules and validations
- What a hold is: A hold is the allocated seats (SEAT-2), on a segment, for each passenger on each leg, owned by one traveller's booking, with an expiry. Held seats are offered to nobody else.
- All or nothing (BR-26, D7): A hold covers every passenger and every leg of a round trip, or nothing. If the return leg has no seats, no outbound seat stays held and no *seats were held* event is published. Both legs live in inventory's database — prefer a design where all-or-nothing is one transaction rather than a compensation.
- Never from cache (BR-29): The hold is decided by inventory's database at the moment of the hold. No cached count is consulted, on any path.
- Exactly one winner (§7.2 Q2): When requests overlap on the same seat and segment at the same instant, exactly one succeeds. The other receives the **seat-taken** code — `409 Conflict` at the gateway — with a message that says what to do next (BR-57). Touching segments both succeed.
- Fifteen minutes on inventory's clock (BR-26, D3): The expiry is a timestamp set by inventory and returned with the hold. Booking passes it to the client with the server's current time, which is all a client needs to show "Reservation Time 14:48"; nobody else computes their own fifteen minutes. Use one clock for every instance.
- A hold never outlives the sale: Sales close 15 minutes before departure from the passenger's origin (BR-16). A hold is refused once sales are closed for any leg, and its expiry is the earlier of fifteen minutes and that closing time.
- One extension: The hold is extended once by 5 minutes when the bank's 3-D Secure step starts (BR-26). The operation is built and tested now; PAY-2 calls it in Sprint 4. A second extension, or an extension of an expired hold, is refused.
- Expired means free, immediately: A hold whose expiry has passed blocks nobody, even if the release job has not reached it yet. Correctness never depends on the job being on time.
- Released automatically, reliably, at volume (D3): Expired holds are released by inventory without anybody asking — not by in-memory timers that die with the process. With several instances running, each hold is released exactly once. 10,000 holds expiring in the same minute are all released, and their events published, within 60 seconds of their expiry.
- Safe to retry: A hold request carries a key from booking. Repeating it after a timeout returns the same hold, never a second one. Releasing is idempotent too, and a hold can be released by that key — so a hold booking never heard back about is never orphaned.
- One active hold per traveller (BR-28): Starting a new hold releases the previous one **in the same transaction**; if the new hold fails, the previous one is untouched. Two sessions of the same traveller (two phones) holding at the same instant end with exactly one active hold — guaranteed by the database, not by a check.
- One National ID, one seat (BR-27, D6, §7.2 Q7): The same National ID cannot occupy two seats on overlapping segments of the same trip, across all bookings and all accounts. An active hold counts as occupying. Exactly one service enforces this, in its own database, at the moment of the hold: inventory already has the seats and segments, booking already has the passengers and bookings. Choose, write the choice in the service map, and make it hold under concurrency. The refusal names the passenger (masked National ID), not the seat.
- Least personal data: The enforcing service needs to compare National IDs, not to read them. Store and send the least that allows the comparison (§7.2 Q9), and never log it.
- Bookable travellers only (BR-10, D4): A traveller whose profile is incomplete is refused at the single entry every hold passes through — the bookable-traveller access level built on AUTH-5 — with the profile-incomplete code, which tells the client the profile must be completed first. It is not re-implemented in booking or inventory.
- Only through booking: Inventory's hold operation is an internal call. The gateway exposes no way to hold seats except starting a booking (BOOK-1).

### Domain events to publish
- Seats were held, with the allocation outcome (consumed by the availability pipeline in SEAT-4, and by the team's split-rate measurement).
- A hold was released — by the traveller, by a newer hold, or by compensation (consumed by SEAT-4).
- A hold expired (consumed by booking to end the saga in BOOK-2, and by SEAT-4).
- A hold was extended (consumed by booking, so the booking draft and the payment step return the new expiry).

### Edge cases to handle
- Two travellers try to hold the last seat on the same segment at the same instant (§6.4).
- A holds Cairo → Tanta on seat 44 while B holds Tanta → Alexandria on seat 44 at the same instant — both must succeed; and A holds Cairo → Alexandria while B holds Banha → Tanta — exactly one succeeds (§6.4).
- A round trip whose return leg has no seats (§6.4).
- The same National ID is added to two bookings on overlapping segments of the same trip, by two different accounts, in the same second (§6.4).
- A traveller holds from two phones — two sessions of the same account — at the same time (§6.4).
- A hold expires while the traveller is in the bank's 3-D Secure step (§6.4) — before or after the extension request arrives.
- 10,000 holds expire in the same minute (§6.4); and the release job is down for an hour, then comes back.
- A traveller abandons a hold and immediately asks for the same trip again — the seats they were holding must be available to their new hold.

### Acceptance criteria

- Given free seats and a bookable traveller with two passengers, when the hold is created, then it carries a seat per passenger and an expiry 15 minutes ahead on inventory's clock.
- Given one free seat left in a class on a segment, when two travellers request it at the same instant, then exactly one holds it and the other receives the seat-taken code — proven by a concurrent test against real Postgres, repeated at least 50 times.
- Given a class with a single seat, when one traveller holds Cairo → Tanta and another holds Tanta → Alexandria at the same instant, then both succeed with the same seat.
- Given a class with a single seat, when one traveller holds Cairo → Alexandria and another holds Banha → Tanta at the same instant, then exactly one succeeds.
- Given a round trip whose return leg has no seats in the class, when the hold is requested, then it is refused, no outbound seat is held, and no *seats were held* event is published.
- Given two accounts add the same National ID on overlapping segments of the same trip at the same instant, when both holds complete, then exactly one succeeds and the other is refused with the passenger-already-travelling code; on touching segments, both succeed.
- Given a traveller with an active hold, when they start a new one, then the previous one is released in the same transaction, and when two sessions of that traveller hold at the same instant, then exactly one active hold remains.
- Given a hold that expired 30 seconds ago and has not been released yet, when another traveller asks for those seats, then they get them.
- Given 10,000 holds expiring in the same minute and two inventory instances running, when the release runs, then every hold is released exactly once and its event published within 60 seconds of its expiry.
- Given an active hold, when it is extended, then it gains 5 minutes once; a second extension is refused, and an extension of an expired hold is refused.
- Given a trip whose cached count says 0 while inventory has free seats, when a hold is requested, then it succeeds; given a cached count of 170 while inventory has none, then it is refused.
- Given booking repeats a hold request with the same key after a timeout, when it completes, then the original hold is returned and no second hold exists.
- Given a traveller whose profile is incomplete (AUTH-5), when they submit passengers, then nothing is held, inventory is never called, and the response carries the profile-incomplete code.
- Given a trip departing the passenger's origin in 14 minutes, when a hold is requested, then it is refused with the sales-closed code; given 20 minutes, then the hold expires when sales close, 5 minutes later.

---

## SEAT-4 — Availability counts reach search — fast numbers from the cache, kept honest by events

**3 points · Priority Medium · `inventory,catalog,cache,redis,events,sprint-3`**

### User story
As a traveller comparing trains in search results
I want the available-seat count of each trip to be close to the truth
So that I do not pick a train that has been full for an hour

### Service flow
Services: inventory (owner of the truth — publishes when a trip's availability changes), catalog (consumer — keeps the cached counts returned with search results, CAT-3 / CAT-5)

  a hold, a release, an expiry, or a trip going on sale, in inventory
  -> inventory publishes, through its outbox, that the trip's availability changed
  -> catalog consumes it (inbox) and refreshes its cached counts in Redis (CAT-5 keys)
  -> the next search returns each trip with its count: "Available Seats: 170"

  a group of three holds seats on that trip      -> the next search returns 167
  their hold expires fifteen minutes later       -> the next search returns 170 again
  the last seat in the class is held             -> search returns the trip marked sold out (BR-16), not omitted

The number search returns is catalog's cached copy of a number inventory owns (§2.3). This
story is the pipe between them, and the rules about how far behind the copy may fall.

### Rules and validations
- Inventory owns the number: Catalog serves a copy, kept fresh by inventory's events (§2.3). Catalog never computes availability on its own and never asks inventory on every search.
- Every change publishes: A hold, a release, an expiry and a trip going on sale (SEAT-1) each lead to an availability change being published. Sales and cancellations join them in Sprint 4.
- The count is per segment: A hold on Banha → Tanta changes the count of every segment that overlaps it — Cairo → Alexandria included — and leaves touching segments unchanged. Decide whether the event carries the new counts or only says "this trip changed" and catalog re-reads, and write down the trade-off.
- Staleness bound (decision): A count returned by search is at most **5 seconds** behind inventory in normal operation, and never older than its key's TTL — at most **60 seconds**, jittered — if events stop flowing. Measure the lag and show it in the demo.
- Newer wins: Changes carry a per-trip version or sequence. An older change arriving late never overwrites a newer count, and a duplicate changes nothing (inbox).
- The cache serves search, the database decides (BR-29): No hold, allocation or booking path reads these keys. A stale count can mislead a traveller for seconds; it can never sell a seat.
- Storms are coalesced: When a popular trip sells out, hundreds of changes per minute for one trip must not become hundreds of cache rewrites and an ever-growing queue. Coalesce per trip — on the publishing side, the consuming side, or both — so the number of rewrites per trip per second is bounded and the last state always lands.
- Sold out is returned, not hidden: A trip whose count is zero stays in search results, marked sold out (BR-16).
- Redis down = slower, not broken (§2.3 rule 7): Search still returns counts from a source you name — a batched call to inventory with a deadline, or catalog's own copy fed by the same events. If no source answers, the result is still returned with the count marked unknown, and an unknown count never marks a trip sold out. The health check says *degraded*.
- No update is lost to an outage: A change that cannot be written because Redis is down is retried or made harmless by the next read; after Redis returns, no count older than the TTL is ever served.
- Keys follow CAT-5: Versioned names, jittered TTLs, single-flight rebuilds, and hit/miss measured for the count keys separately from the search-result keys. If your search-result keys embed counts, a seat change must reach them too — decide whether counts live under their own keys and are merged at read time.

### Domain events to publish
- The availability of a trip changed (consumed by catalog to refresh its cached counts).

### Edge cases to handle
- A popular trip sells out: 500 holds and releases on it in one minute.
- Redis is down while counts change, then comes back still holding the old values.
- A release is processed before the hold that preceded it.
- Inventory is down while travellers search.
- A change arrives for a trip that has already departed or is no longer shown in search.

### Acceptance criteria

- Given search returns 170 seats for a trip in a class, when a group of 3 holds seats on that segment, then a search within 5 seconds returns 167.
- Given that hold expires, when the next search runs within 5 seconds of the release, then it returns 170 again.
- Given a hold on Banha → Tanta, when counts are refreshed, then Cairo → Alexandria and Banha → Tanta decrease and Tanta → Alexandria does not.
- Given the last seat in a class is held, when the trip is searched, then it is returned marked sold out, not omitted.
- Given the same change is delivered twice, or an older change arrives after a newer one, when both are processed, then the count search returns matches inventory.
- Given 500 holds and releases on one trip within a minute, when the storm ends, then the count search returns equals inventory's within 5 seconds, cache writes per trip stayed within the documented bound, and the queue is empty.
- Given Redis is stopped, when travellers search, then results still return with counts, the health check reports degraded, and no request fails.
- Given Redis and inventory are both unavailable, when travellers search, then results return with counts marked unknown and no trip is marked sold out for lack of data.
- Given a count key is inspected, when its TTL is read, then it is versioned, at most 60 seconds, and jittered.
- Given the demo, when asked, then the team reports the hit rate of the count keys and the measured event-to-cache lag.
- Given a change that fails processing repeatedly, when retries are exhausted, then it is dead-lettered and can be replayed.

---

## BOOK-1 — The booking draft — passengers by reference, a locked price and a server-owned hold expiry

**5 points · Priority High · `booking,catalog,identity,gateway,money,sprint-3`**

### User story
As a bookable traveller who has found my train
I want to pick who is travelling and see exactly where each of us sits and what it costs before I pay
So that nothing changes between what I agreed to and what I am charged

### Service flow
Services: booking (owner — the booking draft, the price lock, the fee and total, the one response the client reads), gateway (the bookable-traveller entry and the repeated-submit guard), identity (the traveller's saved passengers, AUTH-6), catalog (the fare quote), inventory (the hold, through the saga in BOOK-2)

  the client submits the trip (or both legs of a round trip), the class, the two stations,
  and which saved passengers are travelling — by reference, nothing else
  -> gateway: bookable-traveller check (BR-10); the same submit repeated is one submit
  -> booking: 1 to 4 distinct passengers, each belonging to this traveller
  -> identity, or booking's own copy of it: each passenger's name and National ID — snapshotted
  -> catalog: the fare of each leg's train, class and origin–destination — locked (BR-30)
  -> inventory: allocate and hold seats for every passenger on every leg (SEAT-2, SEAT-3),
     through the saga (BOOK-2)
  -> booking computes the fee and the total in one place (BR-31)
  -> one response carries everything the Review Tickets step needs:

       per leg and per passenger: the name, "Passenger · Coach 5 · Seat 44"
       the class, the allocation outcome, and the split notice when the group was split (SEAT-2)
       each fare, the fee and the total: Price 6500 · Booking Fee (7.0%) 455 · Total 6955 piastres
       the hold's expiry and the server's current time — "Reservation Time 14:48"

  then, on the same draft:
  the client records that the traveller accepted the terms -> booking stores it, with its time
  the client continues to payment -> refused without the terms; otherwise the saga starts the charge (BOOK-2)
  the client abandons the draft -> booking releases the hold now

The Payment step's booking summary — "Price 65 EGP · Booking Fee (7.0%) 4.55 EGP · Total
69.55 EGP" — arrives with PAY-1 / PAY-2 in Sprint 4; the numbers are decided here.

On a round trip, the same response carries both legs, each with its own coaches and seats,
under one hold expiry and one total.

### Rules and validations
- Bookable travellers only (BR-10, D4): Starting a booking is open only to a traveller whose profile is complete, checked once at the entry (AUTH-5). The refusal carries the profile-incomplete code, which tells the client the profile must be completed first.
- Passengers by reference (BR-09): The client sends which saved passengers were chosen, never their names or National IDs. Booking gets them from identity — by a gRPC call, or from its own copy fed by identity's events (§2.2) — decide and justify. Another traveller's passenger — or booking draft — is a 404.
- One to four, all different (BR-28, D5): Zero passengers, more than four, or the same passenger twice is refused with a code naming the problem. Groups above four are not supported (Appendix A).
- Adding a passenger is AUTH-6: A new passenger is created through identity's saved-passenger operations from Sprint 2, with their 10-passenger limit (BR-12), and is then submitted by reference like any other. Booking never creates passengers.
- A snapshot at the hold: Booking keeps its own copy of each passenger as reviewed — the name and the National ID the hold was checked against. Editing the passenger afterwards does not change this booking (BR-12, BR-37).
- The price comes from catalog and is locked (BR-30, BR-51): For each leg, booking asks catalog for the fare of that train, class and origin–destination pair, and stores it with the booking. A fare change after the hold does not affect it. The client never sends a price.
- Fee and total in one place (BR-31, D8): Booking fee = 7% of the **total** fare of every passenger on every leg, rounded half-up to the piastre; Total = fares + fee; integer piastres throughout. The calculation has exactly one home, owned by booking: payment receives the amount and never recomputes it, and the refund policy will use the same home. It does not live in a shared library — libraries never hold domain logic (§2.1).
- The rounding is on the total: 65 EGP is 6500 piastres, fee 455, total 6955 — the Figma's 69.55 EGP. Three tickets at 10.50 EGP: 3150 × 7% = 220.5, so the fee is 221 piastres. Per ticket it would have been 74 × 3 = 222. BR-31 says 221 (§6.5).
- The draft is complete in one response: One response carries everything the Review Tickets step needs — per leg and per passenger the name, coach and seat; the class; the allocation outcome and any split notice; each fare; the fee; the total; the hold's expiry and the server's current time. The client makes no call per passenger. It does not need, and does not get, full National IDs.
- The expiry is the server's: The response carries the expiry inventory set (SEAT-3) — or its extension — and the server's current time, so "Reservation Time 14:48" comes out the same from a second phone, after a reload, or when the draft is read again much later. A client never derives it from a timer it started at submit.
- Terms before payment: The traveller's acceptance of the Terms and Conditions and Privacy Policy must be recorded before payment can start. Booking records the acceptance and its time, and refuses to start payment without it — nothing the client checks on its side is the guard.
- Round trip is one booking (D7, BR-20): Two legs, one hold, one payment. The server re-checks that the return trip departs after the outbound arrives.
- Abandoning gives the seats back: When the client abandons the draft — the traveller backs out of Review Tickets or gives up on the booking — booking releases the hold now (BR-25), not fifteen minutes later.
- One submit, one booking: The same submit sent twice — a double tap, or a client retry after a timeout — creates one booking and one hold.
- Errors say what to change (BR-57): seats just taken — choose another train or class; not enough seats — fewer passengers or another class; passenger already travelling — which passenger; sales closed — another train; profile incomplete — complete it.

### Domain events to publish
- A booking was started and is awaiting payment (BOOK-2 publishes it when the saga reaches that state).
- A booking was abandoned by the traveller (consumed by reconciliation in Sprint 5, and by the team's drop-off measurement — the research KPI is drop-off ≤ 15%).

### Edge cases to handle
- The fare changes between the search and the submit — and again after the hold.
- A saved passenger is deleted from one phone a moment before another phone submits that passenger.
- The Figma's own sample National ID, `26582264865`, has 11 digits (§6.2): a passenger like that cannot exist after AUTH-6, so your seed must not contain one either.
- The client reads the booking draft again 14 minutes after submitting.
- Three tickets at 10.50 EGP — the rounding case above.
- The return leg's train has no seats in the chosen class.
- The traveller submits themselves and three others, then starts another booking from a second phone.

### Acceptance criteria

- Given a bookable traveller submits two saved passengers, when the booking draft is returned, then it lists each passenger with a coach and seat, the class, the locked fare, the fee, the total, the hold's expiry and the server's current time — in one response.
- Given one passenger on a 65 EGP fare, when the summary is built, then Price is 6500, Booking Fee 455 and Total 6955 piastres.
- Given three passengers at 10.50 EGP, when the summary is built, then the fee is 221 piastres and the total 3371 — and a unit test shows the per-ticket method would have produced 222.
- Given an operator changes the fare after the hold, when the booking draft is read again, then the original fare and total are unchanged.
- Given a passenger that belongs to another traveller, when it is submitted, then the response is 404 and nothing is held.
- Given zero passengers, five passengers, or the same passenger twice, when submitted, then the request is refused with a code naming the problem.
- Given a request that carries a name, a National ID, a price or a seat, when it reaches the gateway, then validation rejects it.
- Given a round trip, when the booking draft is returned, then both legs appear with their seats, under one hold, one fee computed on the sum of all fares, and one total.
- Given the allocation was split, when the booking draft is returned, then the split outcome and its notice are in the response before any payment is possible.
- Given terms were not accepted, when the client continues to payment, then it is refused with the terms-required code and the saga does not advance.
- Given the booking draft is read from a second phone three minutes after submitting, when the response is returned, then it carries the expiry inventory set and the server's current time, about 12 minutes apart.
- Given the client abandons the booking draft, when the request completes, then the hold is released immediately and the search count recovers (SEAT-4).
- Given the same submit is sent twice at the same instant, when both requests complete, then one booking and one hold exist.
- Given a passenger is renamed after the hold, when the booking draft is read, then it carries the name as it was at the hold.

---

## BOOK-2 — The booking saga — a persisted orchestrator that survives crashes and gives the seats back

**5 points · Priority Highest · `booking,inventory,catalog,payment,saga,sprint-3`**

### User story
As the platform
I need one persisted process that takes a booking from the traveller's submit to "awaiting payment", and undoes it when time runs out
So that no seat stays held for a booking that died, and no step is done twice because a service was slow

### Service flow
Services: booking (owner — orchestrates and persists the saga), catalog (fare quote, gRPC), inventory (hold, release and extend, gRPC; the *hold expired* event), payment (start and cancel a charge through its real gRPC contract — answered by a test double behind that contract this sprint)

No request of its own. This is what happens between the client submitting passengers
(BOOK-1) and the charge starting, drawn in the booking-saga sequence diagram you produced in
PLAT-1:

  the client submits passengers (BOOK-1)
  -> booking records the booking before calling anyone                    nothing held yet
  -> catalog: what does this cost?                       (deadline)       price locked, BR-30
  -> inventory: hold these passengers on these legs      (deadline)       seats held, expiry known
  the client reads the draft -> terms accepted -> the client continues to payment
  -> payment: start a charge for this total, in EGP      (deadline)       awaiting payment
  ...Sprint 4 (BOOK-3) takes it from here: payment succeeded -> confirm -> tickets

  Off the happy path:
  the hold expires          -> booking: expired · cancel the started charge · the seats are back
  the traveller abandons    -> booking: abandoned · release the hold now
  a step times out          -> uncertain, not failed: ask again with the same key
  booking crashes mid-way   -> on restart, every unfinished saga resumes from its record

Requirements §2.2: *the booking service orchestrates a booking — it asks catalog for a
price, asks inventory to hold seats, asks payment to start a charge, and reacts to the
payment result. If any step fails, it runs the compensations.*

### Rules and validations
- Orchestrated and persisted (§2.2, §7.2 Q6): The saga's state lives in booking's database. After a restart it can be rebuilt entirely from there; nothing that matters lives only in memory.
- States you can tell apart: At minimum: nothing done yet; price locked; seats held; payment started and awaiting the result; and the terminal outcomes expired, abandoned and failed — each knowing whether a compensation is still owed. Name and draw them; the names are yours.
- Only legal transitions, one at a time: A transition is guarded in the database, so two things happening at once — the hold expiring while the traveller continues to payment, two booking instances resuming the same saga — produce exactly one outcome.
- Record before you call: Booking persists what it is about to do before each remote step, so that after a crash it knows which steps may have happened. No database transaction is ever held open across a remote call, and there is no distributed transaction.
- Every step has a deadline: Every gRPC call has one, documented next to its contract. The traveller's request answers within the gateway's deadline even when a service is slow — with a retryable code, never by hanging.
- A timeout is uncertain, not failed: Every step carries a key derived from the booking. Repeating it returns the original result (SEAT-3, the payment contract), so retrying after a timeout never holds twice or starts two charges.
- Resume after a crash: Unfinished sagas whose step deadline has passed are picked up — on start and periodically — by exactly one instance. PLAT-4 (Sprint 5) adds the sweeper that reports sagas that cannot resume and reconciles them with Stripe.
- The hold sets the clock: The saga's deadline is the hold's expiry as inventory reported it, including any extension. Booking never keeps its own fifteen minutes.
- Compensation on expiry: Whichever comes first — the *hold expired* event or booking's own deadline — the booking becomes expired, the hold is released (releasing an already-expired hold is a success) and any started charge is cancelled.
- Compensation is never forgotten: A compensation that fails is retried with backoff until it succeeds, and dead-lettered where someone sees it if it cannot. The hold's own expiry is the last safety net, not the plan.
- What a failure costs depends on where: If the quote fails, nothing is held and nothing is compensated; the traveller is told to try again. If payment cannot start a charge, the hold stays until it expires and the traveller may retry with the same booking (§6.5, in the spirit of BR-34).
- The real contract, a double behind it: Booking calls payment's real gRPC contract from `libs/contracts` — booking reference, amount in piastres from the one fee-and-total home (D8), currency EGP, an idempotency key. This sprint, payment answers through a test double **inside payment, behind that contract**. Booking contains no fake, no flag and no "if test"; when Stripe arrives in Sprint 4, not one line of booking changes.
- A charge that cannot be cancelled: If payment answers that the charge already succeeded while the saga is compensating, the saga does not mark it failed. It records the case for the late-payment path BOOK-3 builds in Sprint 4 (BR-36) and logs an alert — it never drops it.
- One correlation ID: The same ID follows the saga through HTTP, gRPC and RabbitMQ, so one search in the logs shows the whole booking.
- The diagram is the code: The sequence diagram from PLAT-1 shows what this sprint built — happy path to awaiting payment, expiry, abandon, crash. A change to the flow updates the diagram in the same pull request.

### Domain events to publish
- A booking is awaiting payment (for observability now; consumed by reconciliation in Sprint 5).
- A booking expired before payment (consumed by reconciliation in Sprint 5, and by the drop-off measurement).
- Consumes: *a hold expired* and *a hold was extended* from inventory (SEAT-3).

### Edge cases to handle
- Booking crashes after inventory created the hold but before booking recorded it.
- Inventory answers after booking's deadline — the hold exists, booking thinks the call failed.
- The *hold expired* event arrives twice, or before booking recorded the hold.
- The payment service is down when the traveller continues to payment (§6.5).
- Two booking instances resume the same saga at the same moment.
- The hold expires at the exact moment the traveller continues to payment.
- A trip is cancelled while the traveller is mid-payment (§6.7) — Sprint 5 handles it, but nothing here may make it impossible.

### Acceptance criteria

- Given a successful submit and a continue to payment, when the saga is traced, then it passes through every state in order to awaiting payment, the payment test double returns a payment reference, and one correlation ID spans gateway, booking, catalog, inventory and payment.
- Given booking is killed after inventory created the hold and before the state was recorded, when booking restarts, then the saga resumes, finds the same hold through its key, and no second hold exists.
- Given inventory times out after creating the hold, when booking retries with the same key, then the original hold is returned.
- Given a hold expires while the booking is awaiting payment, when the *hold expired* event is processed, then the booking is expired, the started charge is cancelled and the seats are free — without any request from the traveller.
- Given the *hold expired* event is delivered twice, when both deliveries are processed, then compensation ran once.
- Given the hold expires at the same instant the traveller continues to payment, when both complete, then exactly one outcome is recorded.
- Given the payment service is down, when the traveller continues to payment, then they receive a retryable code, the hold remains until its expiry, and a retry within the hold succeeds once payment is back.
- Given the payment double reports that the charge already succeeded during compensation, when the saga handles it, then the booking is not marked failed, the case is recorded for the late-payment path, and an alert is logged.
- Given two booking instances running, when the same unfinished saga is due, then each step is performed once.
- Given catalog does not answer within its deadline, when the traveller submits, then they get a retryable code within the gateway's deadline and nothing is held.
- Given inventory is down when a compensation runs, when it comes back, then the release is retried until it succeeds and is never lost.
- Given booking's source code, when it is reviewed, then it contains no fake payment, and the same build works unchanged against Sprint 4's payment service.
- Given the PLAT-1 sequence diagram, when it is walked in the demo next to the code, then every state, call, deadline and compensation shown exists in the code, and nothing in the code is missing from it.

---

## Explicitly out of scope for Sprint 3

- **The payment step, saved cards and the real charge.** BOOK-2 reaches *awaiting payment* against a test double behind payment's contract; PAY-1 and PAY-2 make it real with Stripe in Sprint 4, including the 3-D Secure step that calls SEAT-3's extension.
- **Turning a hold into a sale and issuing tickets.** BOOK-3 and TKT-1 in Sprint 4 — including a payment that succeeds after the hold expired (BR-36).
- **Cancellation and refunds.** BOOK-4 in Sprint 4 frees one ticket's seat on one segment; SEAT-1's representation must already allow it.
- **Closing a cancelled trip's inventory.** OPS-2 in Sprint 5; SEAT-1 leaves room for it.
- **The stuck-saga sweeper and Stripe reconciliation.** PLAT-4 in Sprint 5; BOOK-2 resumes what it can now.
- **Load-testing seat contention with k6.** PLAT-6 in Sprint 5; this sprint's concurrency suite proves correctness, not throughput.
- **A seat map, train-type information, groups above four.** Not designed (Appendix A). Nothing in the hold should make letting a traveller choose a seat impossible later, but no API here offers it.
