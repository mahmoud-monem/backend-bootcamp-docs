# Sprint 2 — Network, Search & Cache

**Project:** ENR — Egyptian National Railways Ticket Booking (Bootcamp Project 3)
**Requirements:** [`requirements.md`](./requirements.md)
**Microservices guide:** [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)
**Design:** [Figma — Egyptian National Railways](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
**Jira import file:** [`sprint-2-network-search-stories.csv`](./sprint-2-network-search-stories.csv)

---

## How to read these stories

Each story describes **a service flow**: the request a client makes — the mobile app, an operator tool or a test script — the services involved and what each one does, or, for the platform stories no client calls directly, the job or the developer's flow. Then it lists the rules that flow must obey, the events it publishes, the edge cases it must survive, and the acceptance criteria you will be graded on.

**No client UI is built in this project.** The Figma design shows what the API must make possible — the data a client needs at each step and the refusals it must be able to explain — not screens to deliver.

What you will **not** find here: endpoint paths, request bodies, table or column names, queue names, routing keys or `.proto` definitions. Those are your design work — derive them from the flow, the rules and what the Figma screens need from the API. If a story says *"the refusal tells the client the profile must be completed first"*, deciding **how** you express that is part of the exercise.

Each story lists **domain events to publish**, named in plain language (*"a trip was published"*) rather than as routing keys, because the event catalogue — names, payloads, versions and consumers — is your deliverable (requirements §8). Next to each event is who consumes it and why. An event with no consumer is a question to answer, not a default.

**Every story starts its Service flow with a `Services:` line** naming the services involved and the role each one plays. The owner of the **first-listed** service is accountable for the story — its design, its tests and its demo — even when teammates write parts of it.

**What is different about this sprint.** Sprint 1 built the platform and the front door. Sprint 2 builds what travellers actually come for — the network and the search over it — and it is graded on three things that look easy and are not:

- **Time.** Every time is Cairo time, trains run overnight and arrive the next day, and Egypt changes its clocks twice a year. A search answers "which trains leave *Assiut* on *this Cairo date*", not "which trains leave Cairo".
- **The cache.** Redis makes search fast. It must never make search wrong, and it must never be the reason search is down. *The cache displays, the database decides.*
- **The first cross-service data flow.** Catalog publishes the trips that inventory will turn into seats — and inventory does not exist until Sprint 3. Nothing published this week may be lost before its consumer is born.

If you find yourself caching a search result that already says "sales open: yes", stop and re-read CAT-3.

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

**8 stories · 37 points**

| # | Story | Points |
|---|---|---|
| AUTH-5 | Complete your profile — the National ID, the phone, and the line between signed in and bookable | 3 |
| AUTH-6 | Saved passengers — the people a traveller books for | 3 |
| CAT-1 | Stations — a bilingual network you can find however you type it | 5 |
| CAT-2 | Trains, timetables, dated trips and fares — the model every seat will belong to | 5 |
| CAT-3 | One-way search — any two stops, in travel order, on a Cairo date | 8 |
| CAT-4 | Round trip, filters, sorting, a trip's stops and recent searches | 5 |
| CAT-5 | The cache layer — fast answers from Redis, correct answers from Postgres | 5 |
| OPS-1 | Operations API — publishing timetables and changing fares without rewriting history | 3 |

### Suggested order

```
CAT-1  Stations + station search       ── a train cannot stop at a station that does not exist
   │
   └─▶ CAT-2  Trains, timetables, trips, fares   ◀── the model Sprints 3–5 stand on
          │
          ├─▶ OPS-1  Publish timetable, change fare   (the seed goes through this same path)
          │
          └─▶ CAT-3  One-way search          ◀── the heart of the sprint
                 ├─▶ CAT-4  Round trip, filters, sort, stops, recent searches
                 └─▶ CAT-5  Cache layer      (design the keys WITH CAT-3, not after it)

AUTH-5  Complete profile ──▶ AUTH-6  Saved passengers        (identity — an independent track)
```

**Start CAT-1 and CAT-2 on day one.** CAT-2 is the story most likely to send you back to your ERD (daylight saving, fares per pair, coach layouts), and Sprint 3's inventory is built on the event it publishes. Design CAT-5's keys while CAT-3 is being written — a cache bolted on afterwards is a cache with the wrong keys.

### What carries over

| Earlier story | Used here by |
|---|---|
| PLAT-1 design gate — service map, event catalogue, cache plan | CAT-5 implements the cache plan key by key · every event in this sprint is already in the catalogue, or is added with a mentor review |
| PLAT-2 platform skeleton — envelope, localized error codes, config at boot, health | CAT-3 and CAT-4 field-level errors in Arabic and English · CAT-5 the health check that reports *degraded* |
| PLAT-3 messaging backbone — outbox, inbox, retry and dead-letter | AUTH-5 the profile-completed event · CAT-2 and OPS-1 trips published · CAT-5 the invalidation consumers |
| AUTH-1 gateway — local token verification, deny-by-default, rate limits | every endpoint of the sprint · OPS-1 operator-only access · AUTH-5 the "bookable traveller" access rule |
| AUTH-2 sign up and email verification | AUTH-5, AUTH-6 — only a verified traveller completes a profile or saves passengers |
| AUTH-3 sessions and refresh rotation | AUTH-5 — the traveller becomes bookable without signing in again · CAT-4 recent searches belong to the signed-in traveller |

### Suggested split

| Student (owns) | Takes this sprint | ≈ points |
|---|---|---|
| S1 — gateway + identity | AUTH-5 and AUTH-6 · the gateway routes for every story of the sprint · the "bookable traveller" and operator access rules · the seeded operator account (identity part of OPS-1) | 7 |
| S2 — catalog | CAT-2 and OPS-1 — and the review of almost every PR this sprint | 8 |
| S3 — inventory | CAT-5, plus CAT-4's recent searches (the per-user cache rule). Next sprint S3 feeds the availability keys from inventory's own events (SEAT-4) — so S3 designs those keys now | 7 |
| S4 — booking | CAT-3. In Sprint 3 booking asks catalog for a fare quote; whoever built search understands fares and segments best | 8 |
| S5 — payment + notification | CAT-1 · CAT-4's round trip, filters, sort and trip stops · the notification consumer of the profile-completed event | 8 |

This sprint is catalog-heavy. **The owner of a service reviews every pull request that touches it** — S2 will review most of the sprint, so S2's own load is kept to the model and the operations path.

### Definition of Done (every story)

- Every rule is enforced **in the service that owns it** — not in the gateway, not in the client. The gateway validates the shape of a request; the owner decides.
- Every acceptance criterion has a test, including the negative and boundary ones and, where stated, a **concurrent** one.
- Every edge case listed has a deliberate, defined behaviour — even if that behaviour is "refuse with this code".
- Every event is written to the **outbox in the same transaction** as the change it announces; every consumer is **idempotent** (inbox); a failing message is retried with backoff and then **dead-lettered**, never dropped.
- Every synchronous call has a **deadline**; a service that is down produces a defined error code, not a hang.
- No secret, token, full phone number or full National ID appears in any log, event or error response.
- Errors use the shared envelope and stable codes, with Arabic and English messages that name the field and what to do next (BR-56, BR-57).
- `.proto` contracts and event schemas are updated in `libs/contracts` with contract tests; the endpoint table, event catalogue, error catalogue and cache plan are updated when a story changes them.
- Money is integer piastres everywhere — in the database, on the wire and in tests. Times are Africa/Cairo and tested across a daylight saving change.
- It works in **`docker compose up` with the seed**, not only in unit tests.
- **Sprint-specific:** search p95 latency is measured with a **warm and a cold cache** (k6 or an equivalent script) and recorded in the README next to the cache hit rate.
- **Sprint-specific:** the seed is big enough that search, filters and the cache mean something — at least three lines, dozens of trains across several train types and every class, overnight trains to Aswan, and dated trips for the whole booking window.

---

## EPIC-2 — Epic: NETWORK & SEARCH — Profile, Timetables, Search and Cache

Everything a traveller needs before a seat can exist: the profile and passengers that make
them bookable, the railway network in two languages, trains and their coaches, timetables
and the dated trips generated from them, fares per train, class and pair of stations, the
search over all of it, the cache that makes search fast without making it wrong, and the
first operations path that lets ENR publish timetables and change fares.

Source of truth: sections 2.3, 4 (Modules 2, 3, 4 and 9), 5.2, 5.3, 5.8 (BR-50 to BR-52),
6.2, 6.3, 7.2 (Q3, Q4) and rulings D1, D2 and D4 of the requirements. The Figma Complete your
profile, Passengers tab, Home (One way, Round trip, with Train Number), station picker,
Calendar, Results with the date strip, Stops timeline and Results (No result) screens, the
add-passenger entry of Choose Passengers and the "Recent Search" wireframe are context for
what the API must make possible, not deliverables.

Depends on Sprint 1: the gateway, verified accounts and sessions, the envelope and error
codes, the outbox and inbox, and the cache plan reviewed at the design gate.

Sprint 3 depends on this one: inventory creates seats from the trips published here, the
hold is refused for a traveller this sprint marks as not bookable, and booking prices a
hold with the fares modelled here.

**Epic acceptance criteria**

- A traveller can complete their profile or skip it, and whether they are bookable is decided in exactly one place.
- A traveller can save up to 10 passengers, and nobody else can see, change or delete them.
- Any station can be found by a query in Arabic or English, however it is spelled.
- Any two stops of a trip, in travel order, on a Cairo date, return correct results — times, dates, duration, stops, seats and fare — including overnight trains and the daylight saving nights.
- Round trip, filters, sorting, a trip's stops and recent searches behave as BR-18, BR-20 and BR-21 define.
- Searches are served from Redis when warm and from Postgres when Redis is down; 200 simultaneous searches on a cold key cause one database rebuild.
- An operator can publish a timetable and change a fare; every action is audited, and every dated trip is published through the outbox ready for Sprint 3's inventory.
- Search p95 latency is measured warm and cold on a seed big enough to mean something.

---

## AUTH-5 — Complete your profile — the National ID, the phone, and the line between signed in and bookable

**3 points · Priority High · `identity,gateway,profile,sprint-2`**

### User story
As a traveller who signed up with only an email and a password
I want to give my name, phone number and National ID once, when I am ready
So that I can book later without filling the same form again

### Service flow
Services: identity (owner — profile, National ID and phone rules, the self passenger, the bookable state), gateway (REST edge; the "bookable traveller" access rule), notification (consumes the profile-completed event).

After the email is verified (Sprint 1), the traveller completes the profile — or skips it:

  client -> gateway: complete the profile of the signed-in account — full name, mobile
            number, National ID, profile type, nationality
    -> identity (gRPC): validate every field, normalize the digits and the phone, then in one
       transaction: complete the profile, create (or adopt) the self passenger, and write
       "a traveller's profile was completed" to the outbox
    <- the completed profile, and the traveller is bookable from the next request;
       or field-level errors naming each field, in the requested language
  outbox -> notification: the account and the name ("Ahmed Hussein")

  Skipping stores nothing: the traveller stays signed in but not bookable, and can complete
  the profile later.

  client -> gateway -> identity: read the profile — the same data back, the phone and the
            National ID in full to their owner only
  client -> gateway -> identity: the profile types and nationalities accepted today
            (Regular, Egyptian)

Skip exists because of the research: "Not all users want to provide their full 14-digit
National ID just to create an account." The redesign moved the National ID out of sign-up
and into a later profile step (D4). The price of that freedom is that the system now
has two kinds of signed-in traveller, and has to tell them apart in exactly one place.

### Rules and validations
- Three required facts: Full name, an Egyptian mobile number and a National ID (BR-10). Profile type and nationality have one value each today — Regular and Egyptian (BR-13). Only those values are accepted, each other value is refused with its own code, and both are kept as data a client reads from identity, so "Student" or a second nationality later is a new row, not a migration.
- Full name: Trimmed, inner spaces collapsed, Arabic or Latin letters, at least a first and a family name. It will be printed on tickets — decide the maximum length and document it.
- National ID structure (BR-11): Exactly 14 digits; the first is the century (`2` = 1900s, `3` = 2000s); the next six are a real birth date (YYMMDD) that is not in the future; the two after that are a known governorate code. Each failure has its own code so the response says exactly what is wrong. Decision: no check-digit test — its algorithm is not officially published, and a wrong guess would reject real people.
- Arabic digits: A National ID or phone submitted in Arabic digits (٠–٩) is accepted and normalized to the same value as its Western-digit form.
- The design's own sample is invalid: `26582264865` has 11 digits and is refused with the length code (§6.2). Seed and test data use synthetic, structurally valid IDs — never a real person's.
- Identity's governorate codes are its own: The codes inside a National ID are part of identity's validation. Do not call catalog — which groups stations by governorate — to validate an ID: two services, two purposes, no runtime dependency.
- Phone normalization (BR-10): An Egyptian mobile number is `010`, `011`, `012` or `015` followed by 8 digits. `01001234567`, `+201001234567`, `00201001234567` and `1001234567` (the number without its leading 0, as a client with a fixed +20 prefix sends it) are the same number, stored in one canonical form. Decision: the number is checked for format and uniqueness, not ownership — there is no SMS channel in this project.
- Unique where the rules say (BR-10, BR-12): The mobile number is unique across accounts. The National ID is unique across **account holders** (self passengers) — not across saved passengers. Both guarantees live in the database: two accounts claiming the same ID at the same instant cannot both succeed.
- Completing creates the self passenger: One transaction completes the profile and creates the traveller's own passenger with the same name, National ID and type (BR-10). Decision: if that National ID is already one of this traveller's saved passengers (they added themselves earlier), that entry becomes the self passenger — the list never holds the same ID twice (BR-12).
- Skip is allowed and harmless: Skipping stores nothing. A traveller who skipped can still search, save passengers and, later, save cards. Only holding a seat needs the profile (BR-10, D4).
- Bookable is one fact with one home (D4): Identity decides whether an account is bookable, and the system expresses it so that the "bookable traveller" access rule in your endpoint table is enforced at one point, not re-checked in every handler. Sprint 3's hold is the first endpoint behind it; this sprint builds and tests the rule. A refused request carries a *profile incomplete* code that tells the client the profile must be completed first.
- Takes effect now: Once the profile is completed, the traveller is bookable on their very next request, without signing out and in. If "bookable" travels inside the access token, decide how a token issued before completion stops saying "not bookable".
- Completed once: A second completion request on a complete profile is refused with a *profile already complete* code. Later changes go through the self passenger (AUTH-6) under the same uniqueness rules — a separate profile edit is not designed (Appendix A).
- Sensitive data (§7.2 Q9): The phone number and National ID are returned in full only to their owner, never appear in a log (mask them), and never travel in an event. Decide whether they are encrypted at rest — and if so, how the uniqueness rules above still work.
- Verified travellers only (BR-01, BR-05): An unverified account cannot complete a profile.

### Domain events to publish
- A traveller's profile was completed (consumed by notification, which keeps the traveller's name to address emails and push by name). It carries the account and the name — never the phone number or the National ID: no consumer needs them, so no consumer gets them. If your design enforces "bookable" from a copy kept outside identity instead of from the token, that service is a consumer too — name it in the event catalogue.

### Edge cases to handle
- The design's sample `26582264865` (11 digits); a birth date of 31 February; governorate code `99`; century digit `3` with a birth date later this year (§6.2).
- Two accounts claim the same National ID as their "self" at the same second (§6.2).
- The phone arrives as `+20 100 123 4567` with spaces, or in Arabic digits.
- A traveller who saved themselves as a passenger before completing the profile.
- A traveller with 10 saved passengers completes the profile.
- Someone submits a phone number that already belongs to another account — what does the message reveal about that account?
- A traveller who skipped reads their profile — decide what the response carries in place of the name (the design greets "Hello Ahmed Hussein") when there is none.
- Identity commits the profile, then RabbitMQ is down — the event must still reach notification later.

### Acceptance criteria

- Given a verified traveller with no profile, when they submit a valid name, phone and National ID, then the profile is complete, a self passenger exists with the same name and National ID, and the traveller is bookable on their next request without signing in again.
- Given the National ID `26582264865`, when it is submitted, then it is refused with the length code and the National ID field named.
- Given National IDs with 31 February, a future birth date, century digit `4` or governorate `99`, when each is submitted, then each is refused with its own code.
- Given a valid National ID in Arabic digits, when it is submitted, then it is accepted and stored identically to its Western-digit form.
- Given `01001234567` on one account, when another account submits `+201001234567`, then it is refused as a phone already in use, without revealing anything about the other account.
- Given two accounts submit the same National ID as their self at the same instant, when both complete, then exactly one succeeds and the other receives the *National ID in use* code — proven by a concurrent test.
- Given the same traveller submits the profile twice at once, when both requests complete, then exactly one profile, one self passenger and one event exist.
- Given a traveller who skipped, when they search or save a passenger, then it works; when a request reaches an endpoint carrying the "bookable traveller" rule, then it is refused with the *profile incomplete* code — proven this sprint with a test.
- Given a profile type other than Regular or a nationality other than Egyptian, when submitted, then it is refused with a specific code.
- Given a traveller who saved their own National ID as a passenger earlier, when they complete the profile, then that passenger becomes the self passenger and the ID appears once in their list.
- Given a completed profile, when the event is inspected, then it contains no phone number and no National ID; and when the logs of every service are searched, then neither appears in full.
- Given a complete profile, when the completion request is sent again, then it is refused with *profile already complete*.
- Given RabbitMQ is down at the moment the profile is completed, when it comes back, then notification receives the event exactly once.

---

## AUTH-6 — Saved passengers — the people a traveller books for

**3 points · Priority High · `identity,gateway,passengers,sprint-2`**

### User story
As a traveller who books for family and friends
I want their names and National IDs saved once
So that booking for them is a tap, not a form (research: frequent users re-enter details every time)

### Service flow
Services: identity (owner — saved passengers and their rules), gateway (REST edge; someone else's passenger is a 404).

  client -> gateway -> identity (gRPC): the signed-in traveller's passengers
    <- the whole list, self first, then the others in a documented order; per passenger the
       full name ("Omar Khaled Hassan"), the National ID in full, the type named in the
       requested language ("Regular" / "كوامل"), and whether it is the self passenger

  client -> gateway -> identity (gRPC): add, edit or delete a passenger — full name,
            National ID, type (Regular)
    identity: AUTH-5's name and National ID rules, the limit, one National ID per list,
              ownership
    <- the saved passenger; or a field-level error naming the field; or a 404 for
       someone else's passenger

The design offers the same add in two places — the Passengers tab and, during a booking,
Choose Passengers. It is one request against one list: a passenger added through either is
in the list both read. Selecting up to four of them for a booking is Sprint 3 (BOOK-1).

Look at the National ID in the design: `26582264865` again — 11 digits. Your API refuses it.

### Rules and validations
- What a passenger is: A full name, a National ID and a type (BR-12), validated with exactly the same name and National ID rules as AUTH-5 (BR-11), Arabic digits included. One implementation, not two.
- Up to 10 (BR-12): A traveller saves up to 10 passengers. Decision: the self passenger does not count — it is created by the profile, not saved — so a full list is 10 plus self. The eleventh is refused with a code that tells the client the limit.
- The limit holds under concurrency: Two adds at the same instant on a list of 9 leave 10, not 11. The guarantee lives in the database, not in a count followed by an insert.
- Once per list, shareable across lists (BR-12): A National ID appears at most once in one traveller's list, but two travellers may both save the same person. The uniqueness is per account and enforced by the database — on add and on edit.
- The self passenger: Appears first, is marked as the traveller themself, and cannot be deleted (BR-12). Its name and National ID are the profile's — one fact, one place: editing the self passenger is how the profile changes, under AUTH-5's rules (the ID stays unique across account holders).
- The self ID lock (BR-12): The self passenger's National ID cannot change once a ticket was issued for it. Tickets live in booking (Sprint 4), so identity has to learn it from there — decide how (an event identity consumes, or a question it asks) and build the rule now. Until Sprint 4 the answer is always "no ticket".
- Tickets keep their own copy (BR-12, BR-37): Editing or deleting a passenger never changes a ticket already issued. No other service may hold a reference it expects identity to keep alive.
- Types are data (BR-13): Only Regular today, returned with its name in the requested language ("Regular" / "كوامل"). A discounted type later is a new row, not a migration of existing passengers.
- Owned (BR-09): Someone else's passenger is a 404 for read, edit and delete — even when its identifier is valid.
- Bounded and ordered: The list is small (at most 11) and returned whole: self first, then the others in a documented order.
- Sensitive (§7.2 Q9): The National ID is returned in full to its owner, as the design shows it, never logged in full and never put in an event.
- Verified travellers only (BR-01): A traveller does not need a complete profile to save passengers.

### Domain events to publish
- A saved passenger was added, changed or removed — optional, because no service needs it in this project: booking reads the chosen passengers from identity when the hold is made (Sprint 3) and copies them onto the booking. If you publish it anyway, it carries identifiers only, never the National ID. Decide, and write the reason in your event catalogue — an event nobody consumes is not free.

### Edge cases to handle
- One traveller saves the same National ID twice; a different traveller saves the same person (§6.2).
- A passenger is edited after a ticket was issued for them — what does the ticket show? (§6.2 — its own copy, from Sprint 4.)
- A passenger's National ID is edited to one already in the same list.
- Two clients of the same traveller add the 10th and the 11th passenger at the same second.
- A crafted request tries to delete the self passenger.
- A passenger is deleted by one client while another client, which read the list a moment earlier, still holds its identifier for a booking (from Sprint 3, a hold names it).
- A traveller who never completed the profile saves passengers, later adds their own National ID, then completes the profile (AUTH-5 merges it).

### Acceptance criteria

- Given a traveller with 3 saved passengers, when they add a valid one, then it appears in their passenger list — the one list both the Passengers tab and Choose Passengers read.
- Given a traveller with a self passenger and 10 saved passengers, when they add an 11th, then it is refused with the limit code.
- Given a list of 9, when two adds arrive at the same instant, then exactly one succeeds and the list holds 10 — proven by a concurrent test.
- Given a National ID already in the traveller's list, when it is added again or another passenger is edited to it, then it is refused with the duplicate code naming the National ID field.
- Given traveller A saved a person, when traveller B saves the same National ID, then it succeeds.
- Given the self passenger, when deleting it is attempted, then it is refused with a code saying it cannot be deleted.
- Given the self passenger's name is edited, when the profile is read, then it returns the same name.
- Given the self passenger's National ID is edited to one that another account holder uses as their self, then it is refused (BR-10).
- Given traveller A's passenger, when traveller B reads, edits or deletes it, then the answer is 404 and the passenger is unchanged.
- Given `26582264865` or any structurally invalid National ID, when a passenger is added, then it is refused with the same codes AUTH-5 uses.
- Given a passenger list, when it is read in Arabic, then self comes first, the order is the documented one, and the type reads "كوامل".
- Given any passenger operation, when the logs are searched, then no full National ID appears.

---

## CAT-1 — Stations — a bilingual network you can find however you type it

**5 points · Priority Highest · `catalog,gateway,stations,cache,sprint-2`**

### User story
As a traveller choosing where to travel from and to
I want to find my station by typing it the way I type it — Arabic or English, with or without the hamza
So that the first step of every trip never fails me

### Service flow
Services: catalog (owner — stations, governorates, search normalization, the station-list cache), gateway (REST edge; the requested language).

  client -> gateway -> catalog (gRPC): the station list in the requested language,
            optionally narrowed by a query ("alex", "اسكندريه")
    catalog: the list from Redis (cache-aside) or Postgres; a query is normalized (BR-22)
             and matched against station names, alternative spellings, cities and governorates
    <- governorates in a documented order, each with its active stations, every name in the
       requested language — the whole grouped list in one response; or a well-formed
       empty list

The same list serves both the origin and the destination.

The design's station picker lists Cairo, Giza, Alexandria, Qalyubia, Monufia, Beheira,
Kafr El Sheikh, Dakahlia, Sharqia, Gharbia, Ismailia, Damietta… — those are governorates,
not stations. Alexandria has two stations a traveller boards at (Sidi Gaber and El Raml);
Cairo has Ramsis. The requirements ask for stations "grouped the way the design's station
picker shows them" — so governorates are the groups and stations are the choices.

### Rules and validations
- Bilingual: Every station, city and governorate has an Arabic and an English name (BR-56). A response uses the requested language throughout — never a mix of the two in one response (research finding).
- Grouped by governorate: The list is governorates, each with its stations, in a documented order that is the same in both languages (the design's order is not alphabetical — decide and keep it). Decision: an origin or a destination is always a station, never a whole governorate.
- Either language, whatever the requested language: A query in Arabic with English as the requested language still finds the station, and the other way round. Results are named in the requested language.
- Arabic normalization (BR-22): `أ إ آ → ا`, `ة → ه`, `ى → ي`, and diacritics (tashkeel) and the tatweel removed — applied identically to the stored names and to the query. Decision: a word's leading definite article "ال" is ignored too; otherwise "اسكندريه" never finds "الإسكندرية" (§6.3).
- English (BR-22): Case-insensitive, punctuation and hyphens ignored ("Kafr El-Zayat" = "kafr el zayat"), matching from the start of any word — so "alex" finds Alexandria and "gaber" finds Sidi Gaber.
- Spellings vary: Egyptian names have several English transliterations (Ramsis / Ramses, Assiut / Asyut, Banha / Benha). Decision: a station may carry alternative spellings that search also matches; the seed includes the common ones.
- Wider than the station name: A query matches a station's own name, its city and its governorate, so "Alexandria" lists both Alexandria stations.
- The query is text, not a pattern: A `%` or `_` in the query matches nothing special. An empty query returns the whole grouped list; an over-long one is refused with a field error.
- Only active stations: A closed station is not offered. A station that any trip still stops at is deactivated, never deleted.
- Swap is free (BR-15): Swapping From and To is the client's job; the server treats the reversed pair as just another search. Every station offered is valid as both origin and destination. Decision: the station list for To is not filtered by From — an unreachable pair is answered by search with a reason (CAT-3).
- A realistic seed, in both languages: at minimum Cairo – Alexandria (Ramsis, Banha, Tanta, Kafr El Zayat, Damanhour, Sidi Gaber, El Raml) and Cairo – Aswan (Ramsis, Giza, Beni Suef, Minya, Assiut, Sohag, Qena, Luxor, Aswan), plus at least one more line reaching the governorates the design's station picker shows (for example Cairo – Zagazig – Ismailia – Port Said, or Tanta – Mansoura – Damietta). The design's names are kept for the stations it shows ("Ramsis Station", "El Raml Station"); its "Damnhour" is a typo — the catalog's name wins.
- Cached (§2.3): The station list is owned by catalog, cached with a long, jittered TTL, and invalidated by the event that changes a station — not by waiting for the TTL. An Arabic request never receives the English list from the cache, or the other way round.
- One response, not N: The whole grouped list comes in one response, and the number of database round trips does not grow with the number of stations.

### Domain events to publish
- A station was added or changed (consumed by catalog's own cache invalidation in CAT-5, and from Sprint 4 by booking, which keeps its own copy of the station names it prints on tickets).

### Edge cases to handle
- "اسكندريه", "إسكندرية", "الاسكندرية" and "alex" (§6.3).
- "سيدى جابر" typed with `ى`, "اسيوط" without the hamza, "الاقصر" against "الأقصر".
- A query typed with tashkeel: "طَنْطَا".
- The query is `%`.
- A station is renamed while the list is cached.
- Redis is down when the station list is requested.
- The request carries no language, or one the system does not support.

### Acceptance criteria

- Given the seed, when the station list is requested in English and then in Arabic, then both contain the same stations, grouped by governorate in the same order, each named in its language.
- Given the queries "اسكندريه", "إسكندرية", "الاسكندرية" and "alex", when each is searched, then each returns the Alexandria stations.
- Given the queries "Ramses" and "Benha", when searched, then Ramsis and Banha are found.
- Given the query "KAFR el-zayat", when searched, then Kafr El Zayat is found.
- Given a query in Arabic while the requested language is English, when searched, then the stations are found and returned with their English names.
- Given the query `%`, when searched, then a well-formed empty result is returned — not every station.
- Given a query that matches nothing, when searched, then a well-formed empty list is returned, not an error.
- Given the station list is cached, when a station is renamed through catalog, then the next request returns the new name without waiting for the TTL.
- Given a warm cache, when the list is requested, then no database query runs; given Redis is stopped, then the list is still returned, from the database.
- Given a deactivated station, when the station list is requested, then it is not in it.
- Given the normalization rules, when unit-tested with every variant in BR-22 and §6.3, then each variant normalizes to the same form as its plain spelling.
- Given the seed, when it is inspected, then every station of both required lines exists in Arabic and English, under its governorate.

---

## CAT-2 — Trains, timetables, dated trips and fares — the model every seat will belong to

**5 points · Priority Highest · `catalog,timetable,fares,events,sprint-2`**

### User story
As the platform
I need trains, classes, coaches, timetables, dated trips and fares modelled once and correctly
So that search can find a trip, inventory can create its seats, and booking can price it

### Service flow
Services: catalog (owner — trains, train types, classes, coach layouts, timetables, dated trips, fares, trip generation), inventory (from Sprint 3, consumer of "a trip was published" — nothing published this sprint may be lost before it exists).

No client request of its own: this is the model behind every search result, and the job that
turns timetables into dated trips.

  a timetable is published (OPS-1 — the seed goes through the same path)
    -> catalog, scheduled generation job: the dated trips of the booking window, each in one
       transaction with its stops, its composition and "a trip was published" in the outbox
    -> outbox -> RabbitMQ -> inventory (Sprint 3), which creates the trip's seats without
       calling catalog back — until inventory exists, nothing published may be lost

Read one search result backwards to see what the model must hold:

  "El Raml Station 12:30 PM 27 Oct 2025"  ...  "Ramsis Station 04:00 PM"
      -> a dated trip: one train on one date, its stops, an arrival and a departure at each
  "1st Class"               -> a class, in plain words, with its own coaches
  "Available Seats: 170"    -> the seats in that class's coaches (counted by inventory from Sprint 3)
  "4 Stops"                 -> the ordered stops between the two stations
  "Ticket Fees 155 EGP"     -> a fare for that train, that class and that pair of stations

and a trip's stops:

  Ramsis Station, Cairo 07:30 PM -> Banha -> Tanta -> Damnhour -> Sidi Gaber
  -> ElRaml Station, Alex 11:00 PM

The design's old ticket card says Train class "AC 3 (4)". The research found that meant
nothing to occasional travellers ("use 2nd Class AC") — that is the copy this story retires.

### Rules and validations
- Trains: A train has a unique number (916, 969…) and a type (for example VIP, Spanish, Russian, Talgo, Ordinary), with type names in both languages — the CAT-4 filter uses them.
- Classes in plain words (BR-22, research): "First Class AC", "Second Class AC", "Second Class", "Third Class" — never "AC 1" or "AC 3 (4)" — in Arabic and English. If you keep internal codes, they never reach a response.
- Coach layouts: A layout is a reusable seating plan — rows, seat numbers, where the aisle is — for example a 2+1 First Class AC coach and a 2+2 Second Class AC coach. A train's composition is an ordered list of numbered coaches, each with a class and a layout. Seat numbers are unique within a coach ("Coach 5, Seat 44").
- Adjacent is defined, not guessed (D2, BR-25): The layout states which seats are side by side — same row, no aisle between them — and whatever else SEAT-2 needs to seat a group of 2, 3 or 4 together next sprint. Write the definition down; SEAT-2 builds on it.
- Capacity is derived: A class's capacity on a trip is the number of seats in that class's coaches. It is never typed in separately, so it can never disagree with the seats.
- Timetable: The pattern a train runs on — the days it runs, the ordered stops with an arrival and a departure at each (no arrival at the first, no departure at the last), effective from a date.
- Dated trip (BR-14): "Train 916 on 27 October" — one run, with its own stops, times and composition copied from the timetable version it was generated from. Unique per train and date, enforced by the database.
- Stops have positions (D1): Every stop of a trip has a stable position along it. A segment is "from stop *i* to stop *j*" — the unit inventory protects in Sprint 3 — so a position never changes silently once a trip is published (OPS-1).
- Generated ahead (§7.2 Q3): Dated trips exist for the whole 30-day booking window plus a margin you choose, extended by a scheduled job that is safe to run twice and safe to run on two catalog instances at once — no duplicate trips, no duplicate events. A missed day must not leave the end of the window empty. The catalog README says what generates trips, how far ahead and why.
- Cairo time, daylight saving included (BR-14): All times are Africa/Cairo, and Egypt observes daylight saving time (from late April to late October). Times are turned into instants through the IANA time zone, never through a fixed `+02:00` or `+03:00`. Decide and document what a timetable time that does not exist on the spring-forward night becomes, and which occurrence an ambiguous time on the fall-back night means.
- Next-day arrivals (BR-14): A trip leaving Cairo at 23:00 and reaching Aswan at 09:30 has stops on two calendar dates; each stop knows its own date. Durations are differences between instants, not between clock readings.
- Fares (BR-19, §7.2 Q4): A fare depends on train, class and origin–destination pair, in integer piastres. Every in-order pair of stops, for every class a trip carries, resolves to exactly one positive whole number of piastres — a missing fare is a publishing error, not a search-time surprise. Do it without a row per pair per train per date, decide and enforce whether a sub-segment may cost more than a longer segment containing it, and explain the design in the catalog README.
- Fares have a history (BR-30, BR-51): A fare records the instant it took effect, so "what did this pair cost at 10:42 yesterday" is answerable. Sprint 3's quote reads the current fare; OPS-1 adds new ones.
- Seeded through the domain: The seed creates timetables through the same path the operator uses (OPS-1), so seeded trips are generated and published exactly like real ones — no raw inserts that skip the outbox.
- Published reliably: Every generated trip is announced through the outbox with what inventory needs in Sprint 3 to create its seats without calling back. Inventory does not exist yet — decide how those events wait for it (a queue declared now, or a replay). A message published to an exchange with no queue bound to it is gone.

### Domain events to publish
- A trip was published (consumed from Sprint 3 by inventory, which creates the trip's seats per class and segment). Versioned; carries the trip and train identity, the train type, the service date, the ordered stops with their positions and times, and the coach composition with seat numbers and adjacency.
- A trip was withdrawn (consumed from Sprint 3 by inventory: a dated trip that will no longer run, removed before anything was sold — see OPS-1).
- A timetable was published (consumed by catalog's own cache invalidation, CAT-5).

### Edge cases to handle
- The overnight train: Cairo 23:00 → Aswan 09:30, with a passenger boarding at Assiut at 04:10 (§6.3).
- The daylight saving nights — a trip timetabled at 00:30 on the night the clocks change (§6.3).
- A train that runs only on Fridays.
- A timetable with one pair of stops missing a fare.
- The generation job runs twice in the same minute, on two instances.
- The generation job was down for three days.
- A 2+1 layout, where one side of every row has a single seat.

### Acceptance criteria

- Given the seed, when a dated trip is read, then it has ordered stops with arrival and departure instants, a train type, a composition by coach and class, and class names in plain words in both languages.
- Given the overnight Cairo → Aswan trip, when its stops are read, then the stops after midnight carry the next day's date.
- Given a trip that runs across a daylight saving change, when its duration is computed, then it equals the real elapsed time — not the difference between the two clock readings.
- Given a timetable time that does not exist on the spring-forward night, when trips are generated for that night, then the documented rule applies — proven by a test.
- Given a train that runs only on certain days, when trips are generated, then trips exist on those dates and on no others.
- Given the generation job, when it is run twice in a row or on two instances at once, then no duplicate trip and no duplicate event exists.
- Given today, when trips are counted per day, then every day of the booking window has its trips.
- Given a train, a class and any two of its stops in travel order, when the fare is asked for, then exactly one fare in integer piastres is returned; given a timetable with a pair missing a fare, when it is published, then it is refused naming the pair.
- Given a fare that changed yesterday at 10:00, when the fare at 09:59 and at 10:01 is asked for, then the old and the new fare are returned.
- Given a class's coaches, when the class capacity on a trip is read, then it equals the number of seats in those coaches.
- Given a 2+1 and a 2+2 layout, when the adjacency definition is unit-tested, then it says for every seat which seats are side by side and never pairs two seats across the aisle.
- Given trips were published while no consumer existed, when a consumer standing in for inventory starts, then it receives every trip published so far.
- Given a published trip, when its event is inspected, then it validates against its schema in `libs/contracts` and contains everything needed to create the trip's seats without calling catalog.
- Given any response, when class names are read, then none is a code like "AC 2" or "AC 3 (4)".

---

## CAT-3 — One-way search — any two stops, in travel order, on a Cairo date

**8 points · Priority Highest · `catalog,gateway,search,sprint-2`**

### User story
As a traveller
I want every train between my two stations on my date, with times, seats and the price
So that I can choose one in seconds instead of guessing

### Service flow
Services: catalog (owner — search, fares, the result and its cache keys), gateway (REST edge; request shape; a deadline on the catalog call; the requested language).

  client -> gateway: one-way search — From, To, departure date, seat class and an optional
            train number, in the requested language
    gateway: checks the shape, then calls catalog with a deadline
    -> catalog (gRPC): validates the stations, date, class and train number; reads the day's
       trips for that pair and class (cache-aside, CAT-5); drops what is no longer on sale
       now; merges availability; sorts by departure from the origin
    <- the five days of the date strip, decided by the server, and one result per trip,
       for example:
         El Raml Station 12:30 PM 27 Oct 2025 -> Ramsis Station 04:00 PM 27 Oct 2025
         train number and type · class · 170 seats available · 210 minutes · 4 stops
         between · 15500 piastres (155 EGP), without the booking fee
       or an empty list with the reason it is empty, in the requested language:
         "Sorry, no trains available on the selected date"
         (Arabic: "عفوا, لا يوجد مواعيد متاحة في التاريخ المختار")
       or a refusal naming the field to change

Another day of the date strip is another search. A trip's stops are CAT-4. Choosing a trip
starts the booking (Sprint 3).

The design's sample data disagree with each other — the header says Wed 24 Oct, the card
says 27 Oct 2025. Your responses must not: a result's departure date is the searched date.

### Rules and validations
- Signed in (BR-01): Search requires a verified traveller — this design has no guest mode. The results themselves are the same for everyone, which is what makes them cacheable.
- Any two stops, in travel order (BR-15): A trip matches when it stops at From and later at To. Banha → Tanta is a valid search on a Cairo → Alexandria train; Tanta → Banha on the same train is not. From equal to To is refused, naming To.
- A Cairo date, at the passenger's origin (BR-16): A trip appears under the date it departs the passenger's origin station in Cairo time — not the date the train left its first station.
- Only what can be sold (BR-16): Not departed, not cancelled, sales open, and within the 30-day booking window. Sales close 15 minutes before departure from the passenger's origin station — exactly 15 minutes before is closed — and a closed trip is not listed. Decision: the window is today plus the next 29 days (30 Cairo calendar days), one definition used by search, the date strip, the bookable range a refused date states and trip generation.
- Sold out is returned (BR-16): A trip with zero seats in the chosen class on that segment is returned and marked sold out, not hidden. A train that has no coach of the chosen class is not a result at all.
- Seat Class is required: Every search is for exactly one class, and every result's seats and fare are for it. Decision: Seat Class is required and its absence is a field error. The class list a client offers comes from catalog, in plain words (CAT-2).
- Train number (BR-18): Optional; narrows the search to that train. Arabic digits are accepted. An unknown train number is a field error on Train Number; a known train that does not run between the two stations in that direction is an empty result with that reason.
- Every result carries (BR-17): Train number, train type, class, departure station, time and date, arrival station, time and date, duration, the number of stops between the two stations (excluding both), available seats for that class on that segment, and the fare without the booking fee (BR-19) in integer piastres with its currency.
- Times a client cannot get wrong: Times are returned as Cairo local times with their offset, so a client whose device is set to another time zone still reads 12:30 PM, never a converted time. The duration is a number of minutes computed from instants, so it is right across midnight and across a daylight saving change.
- Date strip: Five days around the selected date, never including a day before today or past the window — near either edge the selected day moves off-centre. The server says which days they are, so the window rule lives in one place.
- Availability, honestly provisional: Until inventory exists (Sprint 3, SEAT-4), the count is the class capacity of the trip. The response's shape, the availability source (behind one seam) and its cache keys are designed now for the real per-segment count (BR-24) — Sprint 3 changes where the number comes from, not the response and not the keys.
- Errors say what to change (BR-57, research: "I don't know what to change: station, date, or round-trip option"): Each refusal names the field and the fix — From or To unknown or inactive, the same station twice, a date in the past, a date past the window (with the last bookable date), an unknown class, an unknown train number. Empty results say why too: "No train runs between these stations in this direction" (change the stations) is different from "Sorry, no trains available on the selected date" (change the date) and from "this train does not serve this route" (change or clear the train number).
- Bounded, complete, ordered: One day between two stations is tens of trips; it is returned complete, sorted by departure time from the origin (the default, BR-18), and CAT-4's filters and other sorts apply to the whole day, not a page. The number of database round trips does not grow with the number of results.
- Cached, but never frozen in time (CAT-5): Results are served cache-aside, and the time-dependent part of BR-16 — not departed, sales open — is applied on every read. A list cached at 13:50 must not offer, at 14:00, a trip whose sales closed at 13:55.
- When a service is down: The gateway's call to catalog has a deadline; if catalog is slow or stopped, the client receives a *search unavailable, try again* code within it, never a hang. Identity being down does not stop search — tokens are verified at the gateway.

### Domain events to publish
- None. Search is a read. Resist publishing "a search was performed" through the outbox for analytics — it would turn the most frequent read in the system into a write. (Recording the search for "Recent Search" is CAT-4 and stays inside catalog.)

### Edge cases to handle
- The 23:00 Cairo → Aswan train and a passenger boarding at Assiut at 04:10 — which date does it appear under? (§6.3)
- Searching today at 14:00: a trip that left at 13:50, one leaving at 14:10 (sales closed), one leaving at 14:15 (§6.3).
- Tanta → Cairo with the number of a Cairo → Alexandria train (§6.3).
- The selected date is the last day of the window, and the strip wants two days after it.
- A trip departing at 00:30 on a daylight saving night.
- A client whose device is set to Dubai time.
- Catalog is stopped while a traveller searches.
- A route served only by a Friday train, searched on a Wednesday.

### Acceptance criteria

- Given a Cairo → Alexandria train stopping at Banha and Tanta, when Banha → Tanta is searched, then that trip is returned with Banha's and Tanta's times, the number of stops between them and the Banha–Tanta fare.
- Given the same train, when Tanta → Banha is searched with its train number, then an empty result is returned with the wrong-direction reason.
- Given From equal to To, when searched, then it is refused naming To.
- Given the 23:00 Cairo → Aswan train, when Assiut → Aswan is searched for the day after the train's departure date, then the trip appears; when searched for the train's own departure date, then it does not.
- Given now is 14:00 Cairo, when today is searched, then trips departing the origin at 13:50, 14:10 and exactly 14:15 are absent and one departing at 14:16 is present.
- Given a date before today or after the window, when searched, then it is refused naming the date and stating the bookable range.
- Given a trip whose availability source reports zero seats in the chosen class, when searched, then it is returned and marked sold out.
- Given a train without the chosen class, when searched, then it is not returned.
- Given a route with trains on other days but none on the selected date, when searched, then the empty result carries the "no trains on the selected date" reason; given two stations no train serves in that direction, then it carries the "no trains on this route" reason.
- Given any result, when it is inspected, then it carries every BR-17 field, the fare in integer piastres without the booking fee, and a duration computed from instants.
- Given the selected date is today, when the date strip is returned, then it starts today; given the last day of the window, then it contains no later day.
- Given an Arabic request, when results are returned, then station, class and train type names and every message are in Arabic.
- Given a search that returns 30 trips, when it is traced, then the number of database round trips is the same as for a search that returns 3.
- Given catalog is stopped, when a traveller searches, then the gateway answers within its deadline with the *search unavailable* code; given identity is stopped, then search still works.
- Given a result list cached before a trip's sales closed, when it is read after they closed, then that trip is absent.

---

## CAT-4 — Round trip, filters, sorting, a trip's stops and recent searches

**5 points · Priority High · `catalog,gateway,search,round-trip,sprint-2`**

### User story
As a traveller planning a return journey, or one who knows exactly when and how much
I want a round trip in one flow, filters for time, price and train type, the stops of each trip, and my recent searches ready to repeat
So that I find the right train without starting over (research: "No way to narrow results")

### Service flow
Services: catalog (owner — round-trip search, filters, sorting, a trip's stops, recent searches), gateway (REST edge; passes the signed-in traveller so recent searches are theirs).

Round trip — two lists, the second computed from the choice made on the first:

  client -> gateway -> catalog (gRPC): outbound search — CAT-3's search plus a return date
    <- the outbound list (CAT-3), with that day's filter options
  client -> gateway -> catalog (gRPC): the return list for the chosen outbound trip, on
            the return date
    catalog: reads that trip's arrival itself; searches the reverse route on the return date
    <- only trips that leave after the chosen outbound trip arrives, or an empty list with
       its reason
  the two chosen trips go to the hold (Sprint 3: one booking, two legs, one payment — D7)

Any list takes filters (departure time window, maximum fare, train type) and a sort
(departure time, duration, fare). The design has no filter screen — the research asked for
it ("Users wanted to filter by time and price") and BR-18 defines it.

A trip's stops — what the design's "4 Stops" opens:

  client -> gateway -> catalog (gRPC): the stops of a trip between the passenger's stations
    catalog: trip stops, cache-aside
    <- in travel order, each with its Cairo time, and its date where it changes:
         Ramsis Station, Cairo 07:30 PM -> Banha -> Tanta -> Damanhour -> Sidi Gaber
         -> El Raml Station, Alexandria 11:00 PM

Recent searches — the design's "Recent Search" wireframe:

  every valid search -> catalog records it for the signed-in traveller
  client -> gateway -> catalog (gRPC): the signed-in traveller's recent searches
    <- the last five, newest first, each with what a client needs to run it again

### Rules and validations
- Return date (BR-20): On or after the departure date and within the booking window; otherwise refused, naming Return Date.
- The exact reverse: The return search goes from the outbound To to the outbound From, in the same class (a round-trip search has one seat class for both legs). Decision: a train number narrows the outbound leg only — a return train has its own number.
- After arrival, decided by the server (BR-20): The return list is computed for a chosen outbound trip and contains only trips that depart the return origin after that outbound trip arrives there. The server reads the outbound arrival itself — it never trusts a time sent by the client. Decision: strictly after, with no minimum connection time.
- Empty lists explain themselves (BR-57): "No return trains on that date" differs from "nothing leaves after your outbound trip arrives" — the second tells the traveller to change the return date or the outbound trip (the research's "...or round-trip option"). A list emptied by filters carries its own reason too: change the filters, not the date.
- Filters (BR-18): A departure time window (Cairo local time at the passenger's origin, within the selected day, start before end), a maximum fare (integer piastres, compared with the fare **without** the booking fee), and one or more train types. Invalid values are field errors. On a round trip, filters apply to each list separately.
- Filter options come from the results: The response tells the client which train types appear in that day's results and the fare range, so a client never offers a filter that can only return nothing.
- Sorting (BR-18): Departure time (default), duration, fare — with a documented tie-break, so the same request always returns the same order.
- Filters do not multiply the cache (CAT-5): Filtering and sorting are applied to the day's cached results on read, so a new filter combination is never a new database query.
- A trip's stops: The passenger's segment — boarding stop, every stop between, alighting stop — each with its Cairo local time, and its date wherever it differs from the boarding date. The number of stops between always equals the search result's "N Stops" count. Names come from the catalog: "Damanhour", not the design's "Damnhour".
- Trip stops are cached (§2.3): Owned by catalog, long jittered TTL, invalidated when that trip's stops or times change.
- Recent searches (BR-21): The last 5 per traveller, newest first, without duplicates. Decision: two searches are the same when route, class, one way or round trip, and train number are the same; repeating one moves it to the top with its new dates.
- What gets recorded: A valid search is recorded whether or not it found trains (it is the search the traveller may want to repeat); an invalid request is not. Recording never slows a search down and never makes it fail.
- Stale entries: A recent search whose departure date has passed is returned without its dates, so the client asks for new ones. An entry whose station has been deactivated is dropped. Station names are returned in the requested language.
- Per traveller, never shared (§2.3 rule 6): Recent searches are never cached under a shared key, never mixed into the shared search cache, and never visible to another account.
- Five, even under concurrency: Two searches at the same instant from two clients of the same traveller leave at most 5 entries and no duplicate — the guarantee lives in the data store, not in a read-then-write.

### Domain events to publish
- None. Recent searches are catalog's own data in catalog's own database; publishing them would put a traveller's travel pattern on the broker with no consumer that needs it.

### Edge cases to handle
- The outbound trip arrives at 23:50 and the return date is the same day.
- An overnight outbound trip arrives the day after the chosen return date.
- A time window of 22:00–02:00 (crossing midnight).
- A maximum fare below every fare of the day.
- The same search repeated six times, then six different ones.
- Two travellers search the same route in the same second — each sees only their own recent searches.
- A trip's stops are read while its timetable is republished.
- The chosen outbound trip sells out or is withdrawn between the two lists.

### Acceptance criteria

- Given a return date before the departure date, when searched, then it is refused naming Return Date.
- Given a chosen Cairo → Alexandria trip arriving at 16:00, when the return list for the same day is requested, then only Alexandria → Cairo trips departing after 16:00 are returned.
- Given an outbound trip arriving after the last return train of the return date, when the return list is requested, then it is empty with the reason that tells the traveller to change the return date or the outbound trip.
- Given a train number on a round trip, when both lists are requested, then it narrows the outbound list only.
- Given a 06:00–12:00 window, when applied, then only trips departing the passenger's origin in that window remain; given a window whose start is after its end, then it is refused naming the field.
- Given a maximum fare of 155 EGP, when applied, then a 155 EGP trip is included and a 156 EGP trip is excluded — both compared without the booking fee.
- Given a train type filter, when applied, then only trips of that type remain; given an unknown type, then it is a field error.
- Given sorting by duration and then by fare, when each is requested twice, then the order follows the sort with the documented tie-break and is identical both times.
- Given filters that remove every trip, when applied, then the empty reason is "no trains match your filters", not "no trains on the selected date".
- Given a search result with 4 stops between, when that trip's stops are requested, then six entries are returned in order — boarding, four between, alighting — with times, and dates where they change.
- Given a traveller who searched seven different routes, when their recent searches are read, then the last five are returned newest first; given one of them is searched again, then it moves to the top and appears once.
- Given traveller A's recent searches, when traveller B reads theirs, then none of A's appear — and no recent-search value is stored under a key that does not identify its account.
- Given two searches by the same traveller at the same instant, when both complete, then at most five entries exist and none is duplicated — proven by a concurrent test.
- Given a recent search whose departure date has passed, when it is read, then it is returned without its dates.
- Given the recent-search store fails, when a traveller searches, then the results are still returned.

---

## CAT-5 — The cache layer — fast answers from Redis, correct answers from Postgres

**5 points · Priority Highest · `catalog,redis,cache,sprint-2`**

### User story
As the owner of the catalog service
I need the most-read data in the system served from Redis without ever trusting Redis with a decision
So that search stays fast under load, correct after a change, and alive when Redis is not

### Service flow
Services: catalog (owner — every key in this story: what it holds, its TTL, what removes it), gateway (aggregates health; reports catalog as degraded, not down, when Redis is gone).

No client request of its own: the cache sits inside catalog, behind CAT-1, CAT-3 and CAT-4.

  read:        catalog -> Redis; on a miss, one rebuild from Postgres -> written back to Redis
  invalidate:  a timetable published, a fare changed, a station changed -> catalog's inbox
               -> exactly the affected entries become unreachable

The developer's flow — and the demo you will be asked to run:

  1. Search Cairo → Alexandria for tomorrow, First Class AC: cold, built from Postgres, written to Redis.
  2. Search it again: warm, no database query at all.
  3. Change that fare through OPS-1: the next search returns the new fare, without waiting for a TTL.
  4. Fire 200 identical searches at a cold key at once: the database rebuilds it once.
  5. Stop Redis: searches still answer, slower; the health check says degraded; the logs say why.
  6. Start Redis again: the cache is used again, with no restart.
  7. Show the search cache hit rate. You will be asked for it.

### Rules and validations
- Cache-aside for catalog's required uses (§2.3): The station list (CAT-1), trip stops (CAT-4) and search results (CAT-3). A read tries Redis, falls back to Postgres on a miss, and writes the result back.
- Every key has an owner, a TTL and an invalidation trigger (§2.3 rule 2): Written in the cache plan (PLAT-1) and true in the code. A key whose removing event you cannot name is a bug.
- Named and versioned keys (§2.3 rule 5): Changing the shape of a cached value is a version bump; keys of the old version are simply never read again.
- TTL with jitter (§2.3 rule 4): Long for stations and trip stops, short for search results; every TTL is randomized within a documented band, so keys written together do not expire together.
- No stampede (§2.3 rule 3): When a key is missing, one request rebuilds it and the others wait briefly for that result instead of all querying Postgres. This holds across catalog instances, not only inside one process: 200 simultaneous searches on a cold key mean one database rebuild.
- Event-driven invalidation: A timetable published or a fare changed (OPS-1), or a station changed (CAT-1), removes — or makes unreachable — exactly the affected entries, through catalog's inbox (idempotent). Unrelated routes stay cached.
- At scale: Invalidation never depends on listing keys in Redis (no `KEYS` on a production Redis). Decide how 10,000 affected entries become unreachable at once (§6.3).
- Invalidation beats a slow rebuild: A rebuild that read the database before a change must not write its stale result back after the invalidation. Design for the interleaving, then test it.
- Availability under its own keys (§2.3): "Available Seats" per trip, class and segment lives in its own keys and is merged into the results at read time. In Sprint 3, inventory's events (SEAT-4) update just those keys: a seat sold never invalidates a results list, and a timetable change never touches counts it did not affect.
- The cache displays, the database decides (§2.3 rule 1, BR-29): No decision reads from the cache — not the trip generator, not OPS-1's validation, not next sprint's hold, not the fare quote booking will ask for. A cached fare or count is for display only.
- Empty answers are cached too: A popular search with no trains is cached (with a shorter TTL) and invalidated when a timetable for that route is published — otherwise it is the most expensive search in the system.
- Nothing personal in a shared value (§2.3 rule 6): CAT-4's recent searches are not in these keys, and nothing in a shared value depends on who asked. Decide whether values hold both languages or are keyed per language — an English request never receives Arabic names.
- Redis down means slower, not broken (§2.3 rule 7): Every Redis call has its own short timeout, so a hanging Redis adds at most that much to a search. When Redis is unreachable, searches are served from Postgres, catalog's health check reports degraded (not down), the outage is logged once rather than once per request, and the cache is used again automatically when Redis returns. Stampede protection degrades too — it is never the reason a search fails.
- Measured (§2.3 rule 8): Hits and misses are counted per key family and exposed, so the search hit rate can be shown at the demo from real counters.
- Bounded: Redis runs in Compose with a memory limit and an eviction policy; an evicted key is just a miss.

### Domain events to publish
- None. This story consumes: a timetable was published, a fare was changed, a station was added or changed (from OPS-1 and CAT-1) — and, from Sprint 3, inventory's availability events (SEAT-4).

### Edge cases to handle
- The timetable is republished while 10,000 search results are cached (§6.3).
- A fare changes in the 50 milliseconds between a rebuild's database read and its Redis write.
- Redis restarts empty at peak time.
- Redis is slow — 2 seconds per call — rather than down.
- The same invalidation event is delivered twice, or after a newer one.
- Two catalog instances, one cold key, 200 requests split between them.
- Redis is full and starts evicting.

### Acceptance criteria

- Given a cold search, when it runs, then its result is written to Redis; when it is repeated, then no database query runs and a hit is counted.
- Given 200 simultaneous searches on a cold key spread across two catalog instances, when they complete, then the database was queried once to rebuild it and all 200 received the same correct answer — proven by a concurrent test that counts queries.
- Given a cached search, when that route's fare changes through OPS-1, then the next search returns the new fare without waiting for the TTL, and searches on unrelated routes are still served from the cache.
- Given a rebuild that read the old fare before a change and writes after the invalidation, when the key is read next, then the old fare is not returned — proven by a test that forces that interleaving.
- Given the same invalidation event delivered twice, when both are processed, then the outcome equals processing it once.
- Given 1,000 keys written in the same second, when their TTLs are inspected, then they are spread across the documented band.
- Given Redis is stopped, when a traveller searches, then correct results come from Postgres, catalog's health reports degraded, and the outage is logged once; given Redis starts again, then the cache is used again without restarting catalog.
- Given Redis answers every call after 2 seconds, when a traveller searches, then the search takes no longer than the Redis timeout plus the normal database time.
- Given a cached results list and a change in availability only, when the results are read, then the new count appears and the list itself was not rebuilt — demonstrated through the availability seam until SEAT-4 feeds it.
- Given the same search in Arabic and in English, when both are served from the cache, then each receives names in its own language.
- Given a change to a cached value's shape, when the key version is bumped, then values of the old version are never read.
- Given any operation that decides something in catalog, when it is traced, then it read Postgres, never Redis.
- Given the metrics, when the search hit rate is asked for, then it is computed from real hit and miss counters.

---

## OPS-1 — Operations API — publishing timetables and changing fares without rewriting history

**3 points · Priority High · `catalog,identity,gateway,operations,audit,sprint-2`**

### User story
As an ENR operator
I want to publish a timetable and change a fare through an API
So that the network travellers search is the one ENR runs — and every change can be traced to a person

### Service flow
Services: catalog (owner — the operations API, timetable versions, fares, trip generation, the audit trail), identity (the seeded operator account and its role), gateway (REST edge; operator-only access), inventory (from Sprint 3, consumer of trips published and withdrawn).

No UI in this project (requirements, §3 and Appendix C). The operator works through the API
with a tool such as curl or Postman; every call below passes the gateway's operator-only rule
before it reaches catalog:

  sign in as the seeded operator (identity)
  -> publish a timetable for train 916: running days, ordered stops with times,
     composition, fares, effective from a date
     -> catalog validates it whole, then in one transaction: the version, its audit entry
        and its events in the outbox
     -> the dated trips for the booking window are generated, each one published for inventory
     -> the search cache for the affected routes is invalidated
     -> the response says how many trips were created, changed and withdrawn
  -> change the First Class AC fare Cairo → Alexandria on train 916, from now on
     -> new searches return the new fare; holds already made keep theirs (Sprint 3, BR-30)
  -> read the audit trail: who did what, when, and how many tickets it touched

Delaying and cancelling a trip are the other operations; they come in Sprint 5 (OPS-2).

### Rules and validations
- Operator only (BR-01): The operations API requires the operator role, and a traveller is refused (deny by default — AUTH-1). The role is never self-assigned: signing up never produces an operator. The operator account is seeded by identity, and its password comes from configuration, never from the repository.
- Validate whole, then write: A timetable is validated in full before anything is written — known stations, at least two stops, no station twice, times increasing along the route, a fare for every in-order pair and class (CAT-2), a composition of known layouts. A publish is all-or-nothing, and its refusal names every problem found.
- Versions with an effective date: A published version applies from a date; dated trips before that date keep the version they were generated from.
- Never silently rewrite a sold trip (BR-50): A trip with sold or held seats keeps its times and stops. A publish that would change its times is refused (that is the delay operation, Sprint 5); removing a stop that a ticket or hold boards or leaves at is refused. Any other change to such a trip — adding a stop, removing one nobody uses — is refused unless your Sprint 3 inventory design can remap its segments: decide and document. The refusal lists the affected trips and counts, so the operator can choose a later effective date.
- Catalog does not own tickets: Decide how catalog learns that a trip has sold or held seats touching a stop — a question to inventory at publish time, or a local copy fed by inventory's events. If the answer cannot be obtained (inventory is down), the publish is refused, never treated as "no tickets". In this sprint the answer is always "none"; the rule and its seam are built now.
- Trips that stop running are withdrawn: A republish that removes a dated trip with no sales withdraws it and announces it, so inventory never keeps seats for a train that will not run.
- Fare changes for future sales only (BR-51, BR-30): A new fare takes effect now or at a future instant, never in the past, and the previous fare stays in the history (CAT-2). It applies to new holds only — Sprint 3's held bookings keep the price they were shown.
- Fare validation: Positive integer piastres, the documented sub-segment rule (CAT-2), and only for a train, class and pair that exist.
- Safe to retry: The same publish or fare change sent twice — after a timeout, by a retrying client — is applied once: no second version, no duplicate trips, no duplicate events, one audit entry.
- Audited (BR-52): Every action records who (the operator account), when, what (the action and its before and after), how many tickets it affected (zero this sprint — the field exists now), and the correlation ID. The audit entry is written in the same transaction as the change: no change without its record, no record without its change. The trail is append-only and readable by operators.
- Events through the outbox: The change and its events commit together; caches invalidate (CAT-5) and, from Sprint 3, inventory creates or removes seats.

### Domain events to publish
- A timetable was published (consumed by catalog's cache invalidation, CAT-5).
- A trip was published, and a trip was withdrawn (consumed from Sprint 3 by inventory, which creates or removes the trip's seats — CAT-2).
- A fare was changed (consumed by catalog's cache invalidation, CAT-5; carries the train, class, pair, the new fare and the instant it takes effect).

### Edge cases to handle
- Republishing a timetable whose trips already have tickets (BR-50) — none exist this sprint, so the test supplies the answer through the seam.
- Inventory is down when a publish needs to know about sold seats.
- A publish fails validation on the 14th stop of 15.
- The operator sends the same publish twice after a timeout.
- A fare is set so low that the total with the 7% fee falls below Stripe's minimum charge for EGP (§6.5) — decide whether operations may set it.
- Two operators change the same fare in the same second.
- A fare changes while 10,000 searches are cached.

### Acceptance criteria

- Given the seeded operator, when a valid timetable is published, then dated trips exist for the booking window, each was published through the outbox, and the response reports how many were created, changed and withdrawn.
- Given a traveller's token, when any operations endpoint is called, then it is refused; given a fresh sign-up, then the new account has no operator role.
- Given a timetable missing a fare for one pair, or with times going backwards, when it is published, then the whole publish is refused naming each problem and nothing was written.
- Given a published trip with sold seats (supplied through the seam), when a republish changes its times or removes a stop a ticket uses, then it is refused listing the affected trips, and nothing was applied.
- Given the sold-seats answer cannot be obtained, when a republish that touches published trips is sent, then it is refused — never treated as zero tickets.
- Given the same publish sent twice, when both complete, then one version, no duplicate trips, no duplicate events and one audit entry exist.
- Given a fare change effective now, when a traveller searches next, then the new fare is returned, and the fare history shows the previous fare with the period it applied.
- Given a fare change dated in the past, when it is sent, then it is refused naming the effective date.
- Given any operations action, when the audit trail is read, then it shows who, when, what (before and after), the number of affected tickets and the correlation ID.
- Given a fare change whose transaction fails, when the system is inspected, then neither the new fare, nor its audit entry, nor its event exists.
- Given a republish that removes a dated trip with no sales, when it completes, then the trip is withdrawn and the withdrawal was published.

---

## Explicitly out of scope for Sprint 2

- **Real availability counts.** "Available Seats" is the class capacity until inventory creates seats from the trips published here (SEAT-1) and feeds the availability keys (SEAT-4) in Sprint 3.
- **The profile check at the hold.** AUTH-5 builds and tests the "bookable traveller" rule; the hold that uses it is Sprint 3 (SEAT-3, BOOK-1).
- **Delaying and cancelling a trip.** OPS-1 publishes timetables and changes fares; delays and cancellations, and the fan-out they cause, are Sprint 5 (OPS-2).
- **A visual seat map and train-type information** — the research's "What's next", never designed (Appendix A).
- **A separate profile edit, discounted passenger types, non-Egyptian travellers** — not designed (Appendix A); the profile changes through the self passenger (AUTH-6), and the model must allow the rest later without a migration (BR-13).
- **An operator app.** The operations API has no UI in this project.
