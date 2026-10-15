# Sprint 5 — Disruptions, Notifications & Go-live

**Project:** ENR — Egyptian National Railways Ticket Booking (Bootcamp Project 3)
**Requirements:** [`requirements.md`](./requirements.md)
**Microservices guide:** [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)
**Design:** [Figma — Egyptian National Railways](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
**Jira import file:** [`sprint-5-disruptions-golive-stories.csv`](./sprint-5-disruptions-golive-stories.csv)

---

## How to read these stories

Each story describes **a flow**, then the rules that flow must obey, the domain events, the edge cases and the acceptance criteria you are graded on. Where the Figma has screens, the flow walks through them. Where it has none, the flow follows the operator or developer instead.

What you will **not** find here: endpoint paths, request bodies, table or column names, queue names, routing keys or `.proto` definitions. Those are your design work. Derive them from the flow, the rules, the Figma screens and your Sprint 1 design gate. If a story says *"the operator can see when the fan-out has finished"*, deciding **how** you express that is part of the exercise.

Each story lists **domain events to publish** in plain language: *"a trip was cancelled"*, not a routing key. The event catalogue (names, payloads, versions, consumers) is your deliverable (requirements §8). The thing that happens and the things that react to it live in different services. The code that cancels a trip never calls the code that sends an email.

**Every story starts its User flow with a `Services:` line.** It names the services involved and the role each one plays. The owner of the **first-listed** service is accountable for the story: they make sure it is designed, split across the team, reviewed and demonstrated. Being accountable does not mean coding alone. Most stories in this sprint touch three or more services, and the owner of each one reviews every pull request that touches it (requirements §2.2, *Ownership*).

**What is different about this sprint.**

- **It is the unhappy path at scale.** OPS-2 is one operator request that reaches every service and 400 tickets. The requirements open with this case (§1.3: *"a train cancelled with 400 tickets sold — that is the project"*). It is why you built the outbox, the inbox and the saga in Sprints 1–4.
- **It is mostly consumption.** NOTIF-1 and NOTIF-2 react to events that already exist. If sending a notification forces you to edit booking, payment or catalog, those services were built wrong, and this sprint is where you find out.
- **Half the sprint has no screen.** PLAT-4, PLAT-5 and PLAT-6 are platform stories. Their flows belong to an operator or a developer. You prove their acceptance criteria by running the stack: killing services, replaying events, loading it with k6. Reading the code and reasoning that it looks fine does not count.
- **It ends with proof.** PLAT-6 is the capstone. It covers the whole journey from a clean clone, the seat guarantee under load, and whether the design you drew in Sprint 1 matches the system you built.

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

**6 stories · 31 points**

| # | Story | Points |
|---|---|---|
| OPS-2 | Trip delayed or cancelled by ENR — one operator action, 400 tickets, and knowing when it has finished | 8 |
| NOTIF-1 | Notification center — the bell, the inbox, and every message in the traveller's language | 5 |
| NOTIF-2 | Reminders — 24 hours and 2 hours before boarding, and never for a train that will not run | 3 |
| PLAT-4 | Reconciliation and stuck sagas — money and tickets agree again, before a traveller notices | 5 |
| PLAT-5 | Resilience and observability — one booking in one trace, and every service killed on purpose | 5 |
| PLAT-6 | Go-live — the whole journey from a clean clone, under load, and as designed | 5 |

### Suggested order

```
OPS-2   Trip delayed / cancelled         ◀── start day one: the hardest story, every service takes part
   │
   ├─▶ NOTIF-1  Notification center      ── the consumer four sprints of events have been waiting for
   │      └─▶ NOTIF-2  Reminders         ── uses NOTIF-1's channels; moves with OPS-2's delays
   │
   └─▶ PLAT-4   Reconciliation + sweeper ── also finds a fan-out that stopped halfway

PLAT-5  Traces, deadlines, breaker, drills  ◀── start day one in parallel: you will debug OPS-2 with it
   │
   └─▶ PLAT-6  Go-live: load, security, docs, demo   ── the last two days; needs everything above
```

**Start OPS-2 and PLAT-5 on day one.** OPS-2 is the story most likely to send you back to your Sprint 4 code: a refund path that only works one ticket at a time, or a ticket issuer that does not know a trip can be cancelled. PLAT-5 is what lets you see why the fan-out stopped at ticket 213. **Keep the last two days for PLAT-6.** A go-live started on the final afternoon is a demo, not a go-live.

### What carries over

| Earlier story | Used here by |
|---|---|
| BOOK-4 cancellation — refund tiers, preview → confirm, outcome-changed guard | OPS-2 — the free cancellation after a delay of 60 minutes or more is a new branch of the same policy, not a second policy |
| PAY-3 refund lifecycle — Pending → Refunded, Failed visible to operators | OPS-2 — 400 refunds through the same lifecycle · PLAT-4 — refunds recorded but missing at Stripe, or the reverse |
| PAY-2 idempotency keys, verified and deduplicated webhooks | OPS-2 — one refund per ticket when "trip cancelled" arrives twice · PLAT-4 — what reconciliation compares · PLAT-5 — which calls are safe to retry |
| TKT-1 tickets, signed QR, share link, Paid / Canceled tabs | OPS-2 — voided tickets move to Canceled and fail online verification · PLAT-6 — tampered QR codes and share links in the security pass |
| SEAT-1 / SEAT-3 segment inventory and holds | OPS-2 — the trip's seats are closed and its holds released · PLAT-6 — the k6 seat-contention test |
| CAT-3 search · CAT-5 cache layer — invalidation, Redis-down fallback, hit/miss metrics | OPS-2 — a cancelled trip leaves every cached result, a delayed one shows its new times · NOTIF-1 — the same rules for the unread count · PLAT-6 — the search hit rate under load |
| OPS-1 operations API and the operator role | OPS-2 — delay and cancel join publish and change fare |
| BOOK-2 / BOOK-3 saga state machine, late payment after hold expiry | PLAT-4 — the sweeper resumes or compensates what they persisted · OPS-2 — a trip cancelled while a traveller is mid-payment |
| PLAT-3 messaging backbone — outbox, inbox, retry, dead-letter | every story · PLAT-4 — inspecting and replaying the dead letters it has been collecting |
| PLAT-3's first flow — identity → notification sends the code email | NOTIF-1 — codes stay email-only while every other kind gains push and inbox |
| PLAT-2 correlation ID and health checks · AUTH-1 gateway rate limits | PLAT-5 — traces, logs, readiness, graceful shutdown · PLAT-6 — the security pass |
| PLAT-1 design-gate artifacts | PLAT-6 — updated to match the code |

### Suggested split

| Student (owns) | Takes this sprint | ≈ Points |
|---|---|---|
| S1 — gateway + identity | PLAT-5: traces through the gateway, the correlation ID in every log line, the deadline audit, graceful shutdown, collecting the drill results · NOTIF-1 gateway side (bell and inbox API, device-token registration) · PLAT-6 security pass | 6 |
| S2 — catalog | OPS-2 operations side: delay and cancel, shifting the stops, leaving search, cache invalidation, the audit · PLAT-4 dead-letter inspection and replay · PLAT-6 search under load, the seed, the scripted demo | 6 |
| S3 — inventory | OPS-2 closing the trip's seats and releasing its holds · NOTIF-2 reminders · PLAT-6 k6 seat-contention test | 5 |
| S4 — booking | OPS-2 void-and-refund fan-out, completion tracking, the mid-payment race, free cancellation after a delay · PLAT-4 stuck-saga sweeper and the charged-without-tickets repair | 7 |
| S5 — payment + notification | NOTIF-1 inbox, unread count, templates and channels · PLAT-4 Stripe-side reconciliation · PLAT-5 circuit breaker on Stripe · OPS-2 refunds within Stripe's rate limit | 7 |

Every owner runs the PLAT-5 kill drill **for their own service** and writes their own README (PLAT-6). The owner of a service reviews every pull request that touches it. In this sprint that means the booking owner reviews OPS-2's operator side wherever it changes what booking receives, and the notification owner reviews every template, whoever wrote it.

### Definition of Done (every story)

- Every rule is enforced **in the service that owns it**. The app, and every other service, is never trusted with a price, a refund amount, a status or an allowed action.
- Every acceptance criterion has a test, including the negative ones and, where stated, a **concurrent** one. Integration tests run against real Postgres, Redis and RabbitMQ (Testcontainers).
- Every edge case listed has a deliberate, defined behaviour.
- Every event goes through the producer's **transactional outbox**. Every consumer is **idempotent**. A failing message is retried with backoff and then dead-lettered, never dropped.
- Every synchronous call has a **deadline**, and only idempotent calls are retried.
- No password, code, token, PIN, card data, full phone number or full National ID appears in any log, span attribute or dead-letter payload.
- Errors go through the gateway envelope with stable error codes and Arabic and English messages.
- Contracts (`.proto` files and event schemas) are updated in `libs/contracts`, and their contract tests pass.
- Money is integer piastres everywhere, including every refund sent to Stripe.
- It works in `docker compose up`, not only in unit tests.
- **Sprint 5:** the full journey works from a clean clone.
- **Sprint 5:** switching off the notification service never blocks a booking, a cancellation or a trip cancellation.
- **Sprint 5:** replaying every event produces no duplicate charge, refund or notification.

---

## EPIC-5 — Epic: DISRUPTIONS & GO-LIVE — Fan-out, Notifications, Resilience and Launch

The last week, and the one that tests whether the architecture holds. When ENR delays or cancels a train, the whole system has to react: 400 tickets voided and refunded exactly once, the trip gone from search, and every traveller told. The epic then adds the notification center the previous sprints have been publishing to, and the reminders. It adds the reconciliation that catches what the saga missed and the observability that lets you prove what happened. It ends with a go-live that runs the whole journey from a clean clone, under load.

Source of truth: two Figma areas. The first is the Home header, with "Hello Ahmed Hussein" and the bell's unread badge. The second is the Tickets → Canceled states: "In Progress" with "Refund date: Pending", and "Refunded" with its date. Also requirements §2.2 (questions 1 and 4), §2.3, §4 Modules 9 and 10, §5.8, §5.9, §6.5–§6.7, §9 (D10, D11, D13) and §11. Three things here have no design at all: the operations API, the notification inbox and the platform stories. Their contracts are written, not drawn.

Depends on every earlier sprint. OPS-2 reuses BOOK-4's cancellation policy and PAY-3's refund lifecycle instead of writing a second refund path. PLAT-4 repairs what BOOK-2 and BOOK-3 persisted. PLAT-6 needs everything.

**Epic acceptance criteria**

- An operator can cancel a trip with 400 tickets across 150 bookings. Every ticket is voided and refunded exactly once, at 100% of its fare plus its fee share. The trip leaves search and can no longer be held, and the operator can see when the fan-out has finished.
- An operator can delay a trip. The later stops shift and travellers are told. A delay of 60 minutes or more at a passenger's boarding station opens a free full-fare cancellation.
- Every event in BR-53 reaches the traveller by email, push and inbox, in the account's language, exactly once. Codes never appear in the inbox.
- Reminders arrive 24 hours and 2 hours before boarding. They move with delays and never fire for cancelled tickets or trips.
- Money and tickets agree after any crash. Reconciliation finds and repairs or flags every mismatch, stuck sagas are resumed or compensated, and dead letters can be inspected and replayed.
- One booking can be followed through every service in Jaeger and in the logs. Each of the seven services has been killed mid-saga, with a documented outcome.
- The full journey runs from a clean clone. Switching off the notification service blocks nothing, replaying every event duplicates nothing, and the seat guarantee holds under load.

---

## OPS-2 — Trip delayed or cancelled by ENR — one operator action, 400 tickets, and knowing when it has finished

**8 points · Priority Highest · `catalog,booking,inventory,payment,notification,gateway,operations,disruptions,sprint-5`**

### User story
As an ENR operator
I want to delay or cancel a trip with one action
So that every traveller on it is refunded or told the new times, nobody is left holding a ticket for a train that will not run, and I can see when all of that is done

### User flow (see Figma)
Services: catalog (the operations API: the delay or cancellation, the trip's stops and state, leaving search, cache invalidation, the audit), booking (finds every affected ticket, voids it, decides each refund, opens the free cancellation after a long delay, reports when the fan-out has finished), inventory (closes the trip's seats and releases its holds), payment (issues every refund at Stripe and tracks it to Refunded), notification (tells every affected traveller), gateway (the operator role and routing)

No screen. The operator works through the operations API (requirements Appendix C: "No design — you define the API"):

  CANCEL    train 916 on 27 Oct, reason "track maintenance"
            -> accepted at once; the work continues in the background
  PROGRESS  of that cancellation, asked as often as the operator likes
            -> 400 tickets in 150 bookings · 400 voided · 391 Refunded · 9 Pending · 0 Failed
               travellers notified: 150 of 150 · finished: no
            -> a minute later: finished: yes — 400 Refunded, 0 Failed

  DELAY     train 916 on 27 Oct by 45 minutes from Banha
            -> Banha's departure and every later stop move by 45 minutes
  DELAY     the same trip by another 20 minutes from Banha
            -> Banha and every later stop are now 65 minutes late

What travellers see, on screens built in Sprints 2 and 4:

  Tickets -> Canceled: the voided ticket with the orange "In Progress" badge and "Refund date: Pending",
  then the green "Refunded" badge with its date ("Refund date: 29 Oct, 2025")
  the bell on Home gains an unread badge; an email arrives (Mailpit) and a push
  a search Cairo → Alexandria for 27 Oct no longer shows train 916 — if it was the only train,
  "Sorry, no trains available on the selected date" with "Select Another Date"
  after the 65-minute delay: Tickets -> Paid -> "⋮" -> "Cancel trip" previews a full fare refund,
  even though boarding is only 90 minutes away

Do not confuse the two cancellations. The app's "Cancel trip" action is a traveller cancelling their own ticket (BOOK-4). This story is ENR cancelling the whole train run for everyone on it.

Cancelling one ticket is not the hard part; BOOK-4 already does it. The hard part is doing it 400 times across five services, when any of them can be down and any message can arrive twice, and then knowing that it has finished.

### Rules and validations
- Operators only, every action audited: Delay and cancel require the operator role (BR-01), and a traveller's token is refused. Every action records who, when, what (trip, stop, minutes, reason) and how many tickets it affected (BR-52). The ticket count is known only once booking has found the tickets, so the fan-out completes the audit record. Nobody guesses the count at the start.
- Cancelling is final and has a window: A trip can be cancelled until it departs its first stop. After that the request is refused with a specific code, because a breakdown en route is out of scope. A cancelled trip cannot be delayed or un-cancelled. Cancelling an already-cancelled trip starts nothing new and returns the existing cancellation and its progress, so an operator's retry is safe.
- Gone from sale, at once and everywhere: A cancelled trip leaves search (BR-16), including every cached result, stops timeline and availability count that contains it (§2.3 rule 2): name the keys and the event that removes them. Its seats can no longer be held (BR-48). Inventory refuses the hold from its own database, whatever a stale cache still shows (BR-29).
- Every active ticket is voided: Each active ticket on the trip becomes *voided*, which is a different state from *cancelled by its traveller*. The ticket moves to the Canceled tab with its refund badge (BR-40), fails online verification (BR-38), and can no longer be shared or downloaded (BR-39). Its seat does not return to sale, because the trip is closed.
- The refund is the full price: Each voided ticket is refunded 100% of its fare plus its share of the booking fee (BR-48, D8, D10), in integer piastres. The refund goes through PAY-3's lifecycle: Pending ("In Progress", "Refund date: Pending"), then Refunded with the date, or Failed and visible to operators (BR-46). BOOK-4's refund path is reused, not copied. 400 refunds in a burst will hit Stripe's rate limit, so find it in Stripe's documentation. A rate-limited refund is retried with backoff and is never dropped.
- Fee shares add up exactly: The booking fixes each ticket's fee share. The share does not depend on which tickets are still active. When every ticket of a booking is voided, the shares add up to exactly the fee charged: never a piastre more, never a piastre less. Rounding 7% of each fare separately does not achieve that. Take three tickets at 65.50 EGP: the fee charged was 13.76 EGP (7% of 196.50 is 13.755, rounded half-up). But 7% of each ticket is 4.585, rounded to 4.59, and three of those make 13.77. Decide where the extra piastre goes, and prove it with a test.
- Only what is still on the trip: Some tickets were cancelled by their traveller before ENR's decision. Each one keeps the refund BR-42 gave it, and its fee share is not refunded (BR-45). In a round trip, only the tickets on the cancelled trip are voided. The other leg stays valid and can be cancelled under the normal tiers, and the notification says so.
- One refund per ticket, ever: Nothing produces a second refund for the same ticket (BR-47, BR-35). That covers a duplicate "trip cancelled", a retried call, a restarted consumer, and a traveller cancelling the same ticket at the same instant. The guarantee lives in the database and in the Stripe idempotency key, not in an `if`. The refunds of one booking never exceed what was charged (BR-46).
- Orchestrated or choreographed — your call, written down: Decide whether the fan-out is orchestrated or choreographed, and justify it in the service map (requirements §2.2, question 1). The justification must answer three questions. Who knows when it is finished? Who retries a ticket that got stuck? What happens while one of the five services is down?
- Know when it is finished: For every cancellation the operator can see these numbers: tickets found, voided, Refunded, Pending and Failed, and travellers notified. The operator can also see whether the fan-out is finished (D10). *Finished* means every ticket is voided and every refund has reached a final state: Refunded, or Failed and visible to operators. Notifications are reported next to these numbers but never hold the fan-out open (BR-54).
- Mid-payment is not an exception: Once booking knows a trip is cancelled, no ticket is ever issued for it. Equally, no ticket issued a moment before the cancellation escapes the fan-out. A hold on the trip is released, and the traveller's Confirm is refused with a code saying ENR cancelled the trip. If a payment still succeeds at Stripe, it is refunded in full, booking fee included, and the traveller is told (BR-36).
- A delay shifts the rest of the trip: A delay is a positive number of minutes declared from a stop. That stop's departure and every later arrival and departure move by that amount (BR-49). Delays accumulate per stop: 45 and then 20 minutes from the same stop make 65. A delay is the only way to move the times of a trip with sold tickets (BR-50). It never changes seats, and it is refused for a cancelled or arrived trip.
- After a delay, the delayed time is the time: Every rule that uses the passenger's boarding time uses the delayed one. That covers the refund tiers (BR-42), the sales closing (BR-16), the reminders (NOTIF-2) and the share link's expiry at arrival (BR-39). The ticket keeps its printed scheduled times (BR-37) and also carries the current delay, and its signed QR is not re-issued. In search, the trip stays under its scheduled date, even when the delay pushes it past midnight.
- Sixty minutes buys a free cancellation: When the delay at the passenger's boarding station is 60 minutes or more, BOOK-4's preview offers 100% of the fare for that ticket, regardless of BR-42. That holds even inside the last 2 hours, until the delayed departure (BR-49). The booking fee is still not refunded, because this is the traveller's cancellation (BR-45). If a second delay changes the outcome between preview and confirm, the outcome-changed guard applies (BR-43).
- Every affected traveller is told, once: Each affected booking produces one notification to its account holder, in the account's language (BR-56). It says which tickets are affected and what the refund is, or gives the new times and says whether free cancellation is open. A delay affects a traveller only if their boarding or alighting time moved. While notification is down, the message is delayed, but the cancellation is not (BR-54).

### Domain events to publish
- A trip was cancelled (inventory closes its seats and releases its holds; booking voids its tickets and starts their refunds; catalog drops it from every cached search result)
- A trip was delayed (booking records the delay against each affected ticket and opens the free cancellation at 60 minutes or more; notification moves the reminders in NOTIF-2; catalog refreshes cached results and stop timelines)
- A trip's seats were closed (catalog's availability cache; the fan-out's progress)
- A booking's tickets were voided by a trip cancellation (payment issues the refunds, if you chose events over calls; notification tells the account holder, once per booking)
- A booking was affected by a delay (notification tells the account holder the new times and whether free cancellation is open)
- A refund was completed — already published by PAY-3 (booking shows "Refunded"; the fan-out counts it; notification tells the traveller)
- A trip cancellation finished (the audit record gets its final count; operators see it)

### Edge cases to handle
- ENR cancels a trip with 400 tickets across 150 bookings while the notification service is down (§6.7).
- The same "trip cancelled" arrives twice, and the second copy arrives while the first is still halfway through.
- Booking crashes after voiding 200 of the 400 tickets.
- A traveller cancels 2 of 3 tickets, one at a time, and then ENR cancels the trip (§6.6). Separately, a traveller confirms their own cancellation in the second between ENR's decision and booking hearing about it. Decide who wins, and make sure that traveller never gets less than a voided ticket would have received.
- A trip is cancelled while a traveller is on the 3-D Secure page for it (§6.7).
- The trip is delayed by 45 minutes from Cairo and later by 20 minutes from Tanta. A passenger boarding at Banha is 45 minutes late, and one boarding at Tanta is 65 minutes late (§6.7).
- Stripe refuses 3 of the 400 refunds because the cards were closed.
- Redis is unreachable at the moment the cancelled trip's search keys should be removed.

### Acceptance criteria

- Given a trip with 400 active tickets in 150 bookings, when an operator cancels it, then every ticket is voided and refunded exactly once at 100% of its fare plus its fee share, and the progress reports finished with 400 Refunded.
- Given a booking of three tickets at 65.50 EGP with a fee of 13.76 EGP, when its trip is cancelled, then the three fee shares add up to exactly 1,376 piastres and the three refunds add up to exactly the 21,026 piastres charged.
- Given a cancelled trip, when anyone searches its route and date, then it is not returned, even if the result was cached a minute before the cancellation. When a traveller tries to hold a seat on it, then the hold is refused with a trip-cancelled code.
- Given the notification service is stopped, when an operator cancels a trip, then every ticket is still voided and refunded. When notification restarts, every affected account holder receives exactly one email, one push and one inbox entry per booking.
- Given "trip cancelled" is delivered twice, including while the first delivery is still being processed, when both are handled, then there is one refund per ticket and one notification per booking, proven by a replay test.
- Given booking is killed after voiding half of the tickets, when it restarts, then the fan-out resumes and finishes with exactly one refund per ticket.
- Given a traveller cancels a ticket at the same instant ENR cancels its trip, when both complete, then exactly one refund exists for that ticket, proven by a concurrent test.
- Given a booking of three tickets where the traveller cancelled two earlier, when ENR cancels the trip, then only the third is voided and refunded at 100% of its fare plus its fee share, and the two earlier refunds are unchanged.
- Given a traveller holds seats on a trip and has not paid, when the trip is cancelled, then the hold is released and their Confirm is refused with a code saying the trip was cancelled, and nothing is charged.
- Given a payment for a trip succeeds at Stripe after the trip was cancelled, when booking receives it, then no ticket is issued, the full amount including the booking fee is refunded, and the traveller is told.
- Given a round-trip booking whose outbound trip is cancelled, when the fan-out finishes, then only the outbound tickets are voided, and the return tickets are still in Paid.
- Given delays of 45 and then 20 minutes from Banha, when a passenger boarding at Banha previews a cancellation 90 minutes before the delayed departure, then the preview offers 100% of the fare without the booking fee. With a delay of exactly 60 minutes it is also offered. With 59 minutes, BR-42 applies and the cancellation is refused. After the delayed departure, it is refused.
- Given a delay declared from Tanta, when the trip is read, then Tanta's departure and every later stop have shifted and the earlier stops have not. Search results and the stops timeline show the new times. A passenger boarding at Banha is told of the later arrival and is offered no free cancellation.
- Given an operator cancels a trip that has already departed its first stop, or delays a cancelled trip, then the request is refused with a specific code. Given a traveller's token, then every operations action is refused.
- Given any delay or cancellation, when the audit is read, then it shows who, when, what, and the number of tickets affected once the fan-out has finished.

---

## NOTIF-1 — Notification center — the bell, the inbox, and every message in the traveller's language

**5 points · Priority High · `notification,gateway,identity,cache,events,sprint-5`**

### User story
As a traveller
I want a bell that tells me when something happened, and one place listing everything the app has told me
So that I never miss a confirmation, a delay or a refund, even when the email went to spam

### User flow (see Figma)
Services: notification (the inbox, the unread count, device tokens, templates, email and push delivery), gateway (the bell and inbox API, ownership checks), identity (the account's email and language, sent as events, of which notification keeps its own copy), booking, payment and catalog (they publish the events this story consumes and change nothing)

Home -> the header: "Hello Ahmed Hussein" and the bell, with an unread badge.
  The app reads the badge on every screen that shows it, so reading it must be cheap.

Tap the bell -> the notification inbox. THERE IS NO FIGMA SCREEN FOR IT (requirements Appendix A).
  You define its API, and the app builds the screen from it:
    newest first, one page at a time
    each entry: a title and a body in the traveller's language, when it happened,
    read or unread, and what it points at (a booking reference, a ticket)
  tap an entry -> it is marked read -> the badge goes down by one
  "mark all as read" -> the badge disappears

The app starts on a phone -> it registers that device's push token.
Sign out -> that device stops receiving this account's pushes.

Away from the screen, every notified event (BR-53) goes out on its channels:

  Booking confirmed          email with the tickets · push · inbox
  Ticket cancelled           email · push · inbox
  Refund completed           email · push · inbox
  Trip delayed               email · push · inbox
  Trip cancelled             email · push · inbox
  Reminder, 24 h and 2 h     email · push · inbox   (scheduled by NOTIF-2)
  Verification / reset code  email only — never in the inbox, never pushed

Nothing in this story creates a notification. Sprints 1–5 already publish the events. This story is the consumer that writes them down and sends them, plus the API that reads them back.

### Rules and validations
- Written by consumers, never by features: Every notification is created by notification reacting to an event that already happened. Booking, payment and catalog contain no reference to notifications, and this story changes none of their code. If it has to, an earlier story was built wrong.
- Channels follow BR-53: A confirmed booking produces an email with the tickets, a push and an inbox entry. Ticket cancelled, refund completed, trip delayed, trip cancelled and reminders produce an email, a push and an inbox entry. Adding a new kind later must not touch the service that produces its event.
- Codes are email-only: Verification and reset codes go by email only. They are never stored in the inbox, never pushed and never logged (BR-03, BR-53). The inbox must be unable to hold a code, even by mistake.
- The account's language: Emails and pushes are rendered in the language stored on the account at the moment of sending, from Arabic and English templates (BR-56). Notification keeps its own copy of each account's email address and language, fed by identity's events. Inbox entries are stored as a message code with parameters, with station and class names in both languages, and rendered in the request's language when read. A kind that lacks either template fails a test before it can fail a traveller.
- Newest first, paged: The list is newest first and paged. Paging never repeats or skips an entry when new notifications arrive between two pages. Each entry carries what the app needs to open it, so the app makes no extra request per entry.
- Read and unread: Mark one as read, and mark all as read, are both safe to repeat. Mark all affects only the entries that existed when it was asked. A notification that arrives in the same instant stays unread.
- Owned and private: Another account's notification is a 404, whether it is read, marked or counted (BR-09).
- The badge is cached, deliberately: The unread count lives in Redis under a per-account key, never a shared one (§2.3 rule 6). It has an owner, a jittered TTL and named invalidation triggers: a new entry, mark one, mark all. Decide between invalidating and incrementing, and defend the choice, because a count that drifts is a bug. Rebuilding the count causes no stampede (rule 3), and hits and misses are measured (rule 8).
- Redis down means slower, not wrong: Without Redis, the badge is counted from notification's database, and the logs and health check say the cache is degraded (§2.3 rule 7). The count is still correct.
- One device, one token: Each device registers one push token, and the token belongs to one account at a time. Re-registering replaces it and signing out removes it. A device that changes hands never receives the previous account's pushes. Tokens the provider rejects are removed, not retried forever.
- Simulated push, real failures: A simulated push provider is acceptable, writing to the log or to Mailpit. It must still be able to fail and to reject a token, so the failure paths are tested.
- Each channel on its own: A failing channel never stops the others. Each delivery is retried with backoff and then recorded as failed where an operator can see it (PLAT-4).
- Once and only once: The same event never notifies the same person twice on any channel. Deduplicate by event and recipient, in the database. A notification failure never undoes a booking, a cancellation or a refund (BR-54).
- Lock screens are public: A push or inbox entry never carries a National ID, a full phone number, a QR payload or a share link.

### Domain events to publish
- No new business events. This story consumes these: an account was verified or changed its language or email (identity, feeding notification's own copy), a booking was confirmed, a ticket was cancelled, a refund was completed, a booking was affected by a delay or a trip cancellation (OPS-2), and a reminder fell due (NOTIF-2)
- A delivery failed for good on one channel (recorded inside notification; surfaced to operators with the dead letters in PLAT-4)

### Edge cases to handle
- The notification service is down for an hour while bookings, cancellations and a trip cancellation continue.
- A traveller switches the account language from English to Arabic between an event and its email.
- A traveller signs out on a phone, and someone else signs in on the same phone.
- A new notification arrives while the badge is being rebuilt after a cache miss. The old count must not stay cached until the TTL.
- A traveller has 10,000 notifications.
- An entry points to a ticket that has since been voided.
- The push provider rejects every token for an hour.

### Acceptance criteria

- Given a booking is confirmed, when the event is consumed, then an email with the tickets arrives in Mailpit in the account's language, the simulated provider records a push, an inbox entry exists, and the badge goes up by one.
- Given each kind of notification in BR-53, when its event occurs, then it produces its email, push and inbox entry, and every kind renders correctly in Arabic and in English.
- Given a verification or reset code is sent, when the inbox, the push records and the logs of every service are inspected, then the code appears in the email only.
- Given a traveller with more notifications than one page, when every page is walked while new notifications arrive, then entries come newest first and none repeats or is missing.
- Given an unread entry, when it is marked read, then the badge goes down by one. When it is marked read again, nothing changes.
- Given "mark all as read" and a new notification arrive at the same instant, when both complete, then the new notification is unread and the badge shows 1.
- Given account A's notification, when account B reads, marks or counts it, then it is not found.
- Given the badge was read once, when it is read again, then Redis serves it (a hit is recorded). After any mark-as-read, the next read returns the correct count.
- Given Redis is stopped, when the bell and the inbox are requested, then both still answer correctly from the database, and the health check reports the cache as degraded.
- Given the same event is delivered twice, when both deliveries are consumed, then exactly one inbox entry, one email and one push exist.
- Given the push provider rejects a device token, when a notification is created, then the inbox entry and the email still exist, and the token is removed.
- Given a phone signs out of account A and into account B, when A receives a notification, then nothing is pushed to that phone.
- Given an account whose language is Arabic, when it receives a notification and the inbox is then read with `Accept-Language: en`, then the email and the push were in Arabic and the inbox entry renders in English.
- Given the notification service is stopped, when a traveller books and cancels a ticket, then both succeed. When notification restarts, each event notifies exactly once.

---

## NOTIF-2 — Reminders — 24 hours and 2 hours before boarding, and never for a train that will not run

**3 points · Priority High · `notification,booking,catalog,scheduler,sprint-5`**

### User story
As a traveller with a ticket
I want to be reminded the day before and shortly before my train leaves my station
So that I do not miss it, and I am never reminded of a train I cancelled or that ENR cancelled

### User flow (see Figma)
Services: notification (schedules, sends, moves and drops reminders), booking (publishes the ticket events reminders are built from, unchanged), catalog (publishes the delays that move them)

No screen. The reminders arrive through NOTIF-1's channels: email, push and the inbox behind the bell.

  A traveller books train 916 for two passengers, boarding at Banha on 27 Oct at 08:20.
  26 Oct, 08:20   the 24-hour reminder: train 916, Banha → Alexandria, tomorrow at 08:20,
                  Coach 5, Seats 44 and 45
  ENR delays the trip by 65 minutes at Banha -> boarding is now 09:25
  27 Oct, 07:25   the 2-hour reminder, moved with the delay

  The traveller cancels one of the two tickets -> the reminder names only the remaining passenger.
  Both tickets cancelled, or the trip cancelled by ENR -> nothing is sent.

A reminder is work that happens because time passed, not because anyone asked. Something has to remember it across restarts and send it exactly once, even with two instances running.

### Rules and validations
- Two reminders per leg: Each booking leg gets one reminder 24 hours and one 2 hours before the passenger's boarding time (BR-53). They go to the account holder by email, push and inbox (NOTIF-1), and list the passengers who still hold a ticket on that leg.
- Real hours, Cairo display: "24 hours before" means 24 elapsed hours before the boarding instant, so a daylight saving change does not move it. Times in the message are Africa/Cairo (BR-14).
- Scheduled and persisted, not a timer: Reminders are scheduled when tickets are issued and stored in notification's database. An in-memory timer or queue loses them on the first restart.
- Never for a train that will not run: A reminder is dropped when every ticket on its leg is cancelled or voided, or when the trip is cancelled (BR-55). When only some tickets are cancelled, the reminder goes out and names the passengers still travelling. This happens through events. Booking never calls notification.
- The last second: A ticket may be cancelled a moment before its reminder fires, while the "ticket cancelled" event is still in the queue. Decide how the reminder learns about it: check at send time, or accept the window. Document what window remains.
- Delays move it: A delay at the passenger's boarding station moves both reminders by the delay (BR-49). A reminder that was already sent is not sent again, because the delay notification already tells the traveller the new time.
- Too late is not sent: A reminder whose time had already passed when the tickets were issued is not sent. A booking made 10 hours before boarding gets only the 2-hour reminder. A booking made 1 hour before gets none.
- After a restart: A reminder missed while the service was down is sent late only while the passenger's train has not yet left their boarding station. If both reminders are overdue, only the 2-hour one is sent. A reminder whose boarding time has passed is dropped and recorded.
- Once, whatever runs it: If the scheduler runs twice, or two instances run it at the same instant, each reminder is still sent once (BR-54). The guarantee is a claim in the database, not a lock in memory.
- Duplicate events, one schedule: The same "tickets issued", "ticket cancelled" or "trip delayed" event delivered twice leaves exactly one schedule per leg per reminder.
- The account's language at send time: A reminder is rendered in the language stored on the account when it is sent, not when it was scheduled (BR-56).

### Domain events to publish
- A reminder fell due (consumed inside notification by NOTIF-1's delivery: email, push and inbox)
- Consumed, not published: tickets were issued, a ticket was cancelled or voided, a trip was delayed, a trip was cancelled

### Edge cases to handle
- A reminder was scheduled for a ticket that was cancelled one minute later (§6.7).
- A booking for two passengers where one ticket is cancelled.
- The notification service is down from 05:00 to 07:00, and the 2-hour reminder for an 08:20 boarding was due at 06:20.
- A delay of 30 minutes arrives two minutes after the 2-hour reminder was sent.
- A boarding at 00:30 on the night Egypt's daylight saving time ends (§6.3).
- 5,000 reminders fall due in the same minute for a holiday train.
- A round trip: the outbound and the return legs each have their own reminders.

### Acceptance criteria

- Given tickets issued three days before boarding, when the clock reaches 24 hours and then 2 hours before boarding, then one reminder is sent at each point, by email, push and inbox, in the account's language.
- Given tickets issued 10 hours before boarding, when the reminders are inspected, then only the 2-hour reminder exists.
- Given a ticket cancelled one minute after its reminders were scheduled, when the reminder times pass, then nothing is sent.
- Given one of two tickets on a leg is cancelled, when the reminder is sent, then it names only the remaining passenger.
- Given the trip is cancelled by ENR, when the reminder times pass, then no reminder is sent for any of its tickets.
- Given a delay of 65 minutes at the boarding station before the 2-hour reminder is due, when the reminder fires, then it fires 2 hours before the delayed boarding time. Given a delay after a reminder was sent, then that reminder is not sent again.
- Given the scheduler is run twice in a row, or by two instances at the same instant, when the results are compared, then each reminder was sent exactly once.
- Given notification is stopped across a reminder's time and restarted before boarding, then the reminder is sent once. If it restarts after boarding, then nothing is sent and the drop is recorded.
- Given both reminders are overdue when notification restarts, when it catches up, then only the 2-hour reminder is sent.
- Given the "tickets issued" event is delivered twice, when both deliveries are consumed, then one reminder per tier exists for that leg.
- Given a boarding at 00:30 on the night daylight saving time ends, when the 24-hour reminder fires, then it fires 24 elapsed hours before boarding.

---

## PLAT-4 — Reconciliation and stuck sagas — money and tickets agree again, before a traveller notices

**5 points · Priority Highest · `payment,booking,inventory,reconciliation,saga,dead-letter,sprint-5`**

### User story
As the team running ENR
I need jobs that compare what Stripe did with what we recorded, and that finish or undo work that stopped halfway
So that nobody is charged without a ticket or holds a ticket without a charge, and we find out before they phone us

### User flow (see Figma)
Services: payment (compares Stripe's charges and refunds with its own records and repairs refunds), booking (compares charges with bookings and tickets and owns the stuck-saga sweeper), inventory (answers whether a repaired booking's seats can be re-acquired), notification (tells a traveller when a repair moved their money or issued their tickets), every service (its own dead-letter queues, inspected and replayed)

No screen. There are three jobs that run on their own, and two tools an operator uses:

  RECONCILE      often over a recent window, and once a day over a longer one
                   Stripe says              ENR says                   -> then
                   charge succeeded         no tickets                 -> finish the booking, or refund in full
                   no successful charge     tickets exist              -> flag for a person; never auto-void
                   no such refund           refund Pending             -> issue it, with the same idempotency key
                   refund succeeded         no refund, or Pending      -> record it as Refunded, or flag it
  SWEEP          sagas sitting in a non-final state longer than they should -> resume or compensate
  REPORT         every finding and every repair: what, why, before and after, done by the job or by an operator
  DEAD LETTERS   list what failed, why, and its correlation ID -> fix the cause -> replay one, or all

Requirements §6.5 asks the question this story answers: *"Stripe succeeds, then the booking service crashes before confirming. The customer has been charged and has no ticket. How do you find out, and how do you fix it?"*

### Rules and validations
- Compare, do not trust: A scheduled job compares what Stripe recorded with what payment and booking recorded: Stripe's charges and refunds against ENR's bookings, tickets and refunds. It reads Stripe through Stripe's API, paged, and assumes nothing (D13). It runs often over a recent window, and daily over a longer one.
- Charged without tickets: This is the §6.5 case: Stripe took the money, but booking never confirmed. The booking is finished under BOOK-3's late-payment rules (BR-36). It gets the same seats if they are still free, or new seats and a message to the traveller. If no seats exist, the payment is refunded in full, booking fee included, and the traveller is told.
- Tickets without a charge: Never repaired automatically. The tickets are flagged for a person and an alert is raised. A job does not get to void the ticket of someone who may already be on the train. Finding one means BR-35 was broken somewhere, so find out how.
- Refund recorded, missing at Stripe: Issue the refund with the same idempotency key the first attempt used. If that attempt did reach Stripe, it is not repeated.
- Refund at Stripe, not recorded: If it matches a Pending refund, the webhook was lost: record it as Refunded with Stripe's date (BR-46). If it matches nothing, for example a refund made by hand in the Stripe dashboard, flag it. A charge whose amount differs from the booking's total is also flagged, never adjusted.
- Leave the normal path alone: A payment a few seconds old is still in flight. Choose a grace period, so reconciliation never races a webhook that is already on its way.
- Safe to repeat, and audited: Running a job twice, or on two instances at once, produces the same findings and one repair each. Every finding and every repair is recorded: what was found, why, the state before and after, and whether the job or an operator acted.
- Half data repairs nothing: If Stripe or a service becomes unreachable mid-run, the run stops and records why. It repairs nothing it could not verify, and the next run starts again.
- Stuck means overdue: For every non-final saga state, define how long it may last and what *overdue* means. One example is a saga still awaiting payment after its hold expired plus a margin. The same sweeper finds an OPS-2 cancellation whose fan-out has not finished in its expected time.
- Resume or compensate, never both: For each state, the sweeper either resumes or compensates. A saga whose payment succeeded but whose tickets were not issued is resumed and finished. A saga with no payment whose hold expired is compensated: the seats are released and its open PaymentIntent is cancelled, so it cannot succeed later. Every step is the same idempotent call the saga itself uses. When the sweeper and a late webhook act on the same saga at once, there is one outcome, and the database guards the state transition.
- Give up loudly: A saga that still fails after a bounded number of attempts is parked in a state operators can see, with the reason. It is not retried forever.
- Dead letters you can read: For every service, an operator can list the dead-lettered messages, each with its event type, why it failed, the number of attempts, the correlation ID, and the first and last failure. Personal data is masked.
- Dead letters you can replay: An operator can replay one message, or every message in one queue. A replayed message goes through the normal consumer and its inbox deduplication, so replaying something already processed changes nothing. A message that fails again goes back to the dead-letter queue instead of looping. These tools need the operator role or a documented command, and a traveller can never reach them.

### Domain events to publish
- A payment discrepancy was found (operators; the audit)
- A discrepancy was repaired (the audit; notification tells the traveller when money moved or tickets were issued late)
- A stuck saga was resumed or compensated (the audit; the saga's own events then continue as normal: a booking confirmed, seats released)
- A dead-lettered message was replayed (the audit)

### Edge cases to handle
- Stripe succeeds and booking crashes before confirming, and meanwhile the seats were sold to someone else (§6.5).
- A charge is three seconds old when the job reads it.
- A refund webhook is lost, and the refund would otherwise stay Pending forever.
- Someone refunds a charge by hand in the Stripe dashboard.
- Stripe's API goes down in the middle of a run.
- The sweeper resumes a saga at the same instant its delayed webhook arrives.
- A dead-lettered "booking confirmed" is replayed after the trip it refers to was cancelled.

### Acceptance criteria

- Given Stripe charged a booking and booking was killed before confirming, when reconciliation runs, then the booking is completed under BR-36 with the same seats, new seats, or a full refund including the fee. The traveller is told and the repair is audited.
- Given a successful charge younger than the grace period, when reconciliation runs, then it is not touched.
- Given tickets exist with no successful charge at Stripe, when reconciliation runs, then they are flagged and an alert is raised, and they are not voided.
- Given a refund recorded as Pending that Stripe has no record of, when reconciliation runs, then exactly one refund is created at Stripe. When it runs again, no second refund is created.
- Given a refund at Stripe that matches a Pending refund whose webhook was lost, when reconciliation runs, then the ticket shows "Refunded" with Stripe's date. Given a refund at Stripe that matches nothing, then it is flagged.
- Given the job is run twice in a row, or by two instances at once, when the results are compared, then the findings and repairs are identical to a single run.
- Given Stripe becomes unreachable mid-run, when the run ends, then it has stopped with a recorded reason and repaired nothing it could not verify.
- Given a saga awaiting payment after its hold expired with no successful charge, when the sweeper runs, then its seats are released and its PaymentIntent is cancelled.
- Given a saga whose payment succeeded but whose tickets were never issued, when the sweeper runs, then the saga is finished and its tickets exist exactly once.
- Given the sweeper and a late webhook act on the same saga at the same instant, when both complete, then there is exactly one outcome, proven by a concurrent test.
- Given a saga that fails to resume more than the allowed number of times, when the sweeper gives up, then the saga is parked, visible to operators, with the reason.
- Given a consumer fails on a message until its retries are exhausted, when the dead letters are listed, then the message appears with its reason, attempts and correlation ID, and its personal data masked.
- Given the cause is fixed and the message is replayed, when it is processed, then its effect happens once. When it is replayed again, nothing changes.
- Given a traveller's token, when the reconciliation report or the dead-letter tools are requested, then the request is refused.

---

## PLAT-5 — Resilience and observability — one booking in one trace, and every service killed on purpose

**5 points · Priority High · `gateway,identity,catalog,inventory,booking,payment,notification,observability,resilience,sprint-5`**

### User story
As a developer on call for ENR
I want to follow one booking through seven services, and to know what each service does when another one fails
So that I can answer "why did this user receive two refund emails?" from evidence, not from guesses

### User flow (see Figma)
Services: gateway (starts every trace and correlation ID), every service (carries both through gRPC and RabbitMQ, logs with them, sets deadlines, shuts down gracefully), payment (the circuit breaker on Stripe)

No screen. A developer's walk-through:

  docker compose up                (Jaeger joins the stack this sprint)
  book a trip in the app
  open Jaeger -> search by the booking reference
    -> gateway HTTP -> booking -> catalog (quote) -> inventory (hold) -> payment -> Stripe
    -> Stripe's webhook -> payment -> "payment succeeded" over RabbitMQ -> booking
    -> inventory (hold becomes sale) -> tickets issued -> notification -> Mailpit
  search the logs of all seven services for that booking's correlation ID
    -> every line of that booking, and nothing else

  docker kill payment while the traveller is on the 3-D Secure page
    -> what does the traveller see? what happens to the hold? -> restart -> what happens now?
  ... and the same for each of the seven services

Requirements §2.2 asks two questions this story answers with evidence. *"Why did this user receive two refund emails?"* is answered by the traces and the logs. *"What happens when each service is down?"* (question 4) is answered by the drill.

### Rules and validations
- One booking, one story in Jaeger: Every service is instrumented with OpenTelemetry and exports to Jaeger, which is added to Compose. Context crosses HTTP, gRPC metadata and RabbitMQ message headers. Every span that touches a booking carries its booking reference, so a single search shows the whole booking.
- Where traces break: An event published by the outbox relay must continue the trace of the transaction that wrote it, not start the relay's own. Stripe's webhook arrives as a new request and must be linked back to the trace that created the payment. Traces usually break in these two places. Prove that yours do not.
- The correlation ID in every log line: Every log line of every service is structured and carries the correlation ID and the trace ID. That includes requests, consumers, the outbox relay and scheduled jobs, where each run gets its own ID. A search for one ID across the seven services tells one booking's story and nothing else.
- No personal data in telemetry: The logging rule also covers span attributes and baggage. They must contain no token, code, PIN, card data, full phone number, full National ID or email address.
- Every gRPC call has a deadline, audited: The repository holds a table of every synchronous call: caller, callee, method, deadline, and why that number. An automated check fails the build when a call has no deadline. Deadlines shrink down the chain, so a call never waits longer than its caller has left.
- Circuit breaker on Stripe: Payment wraps its Stripe calls in a circuit breaker. The breaker opens after a threshold of failures or timeouts. While it is open, calls fail fast with a code the app can act on, and the hold stays, so the traveller can retry before it expires (BR-34). The breaker then probes half-open and closes on success. A refund attempted while it is open is queued and retried, never lost. Inbound webhooks are not affected.
- Retry only what is safe: Retries use exponential backoff with jitter, bounded by the deadline, and apply only to idempotent calls: reads, and writes that carry an idempotency key. A call that is not idempotent is attempted once and its failure is reported. Write the list down.
- Health tells the truth: Liveness (the process is alive) and readiness (it can serve requests) are separate checks. A service whose Redis is down is ready but degraded (§2.3 rule 7). Compose uses readiness to decide start order.
- Graceful shutdown: On SIGTERM, a service first turns not-ready. It stops accepting requests and taking new messages, and finishes in-flight work within a bounded time. It acknowledges only the messages it finished, so the rest return to the queue. Then it flushes the outbox relay and its spans and closes its connections. Compose's stop grace period is longer than that bound.
- Kill-a-service drill, seven times: For each of the seven services, kill it at a named moment mid-saga, with no graceful shutdown. Record the moment, what still works, what degrades, what fails, and the message the user sees. Record what happens when the service comes back, and whether money and tickets agree afterwards, meaning reconciliation (PLAT-4) finds nothing. This is the written answer to requirements §2.2 question 4, produced by running the drill.
- Expected outcomes are specific: For example, with identity down, signed-in travellers keep searching and booking because the gateway verifies tokens itself, while signing in fails with a clear code. With notification down, nothing blocks, and messages arrive later, exactly once. Every service gets an answer this concrete.
- Telemetry never blocks: Tracing is never on a request's critical path. If Jaeger is down, spans are dropped and the services carry on.

### Domain events to publish
- No new business events. Every existing event now carries the trace context next to the correlation ID in its envelope (requirements §10.C).

### Edge cases to handle
- Stripe is slow rather than down: the charge call times out. Did the charge happen? It must not be retried blindly. Resolve it through the idempotency key or the webhook.
- A consumer is killed halfway through handling a message.
- SIGTERM arrives in the middle of an outbox relay batch.
- Every instance retries a failed call at the same instant (this is what the jitter is for).
- The breaker opens while 30 travellers are on the payment screen.
- The gateway's own budget is shorter than the sum of the deadlines below it.
- Jaeger is down.

### Acceptance criteria

- Given a booking made through the app, when its reference is searched in Jaeger, then one trace, or traces linked to each other, shows the HTTP requests, every gRPC call, every RabbitMQ publish and consume, the Stripe call and the webhook.
- Given an event published through the outbox, when its consumer's span is inspected, then it belongs to the trace of the request that caused the event.
- Given a booking's correlation ID, when the logs of all seven services are searched, then every line of that booking is found, each one carries the ID, and no other booking's lines appear.
- Given the logs and spans of a full demo run, when they are scanned, then no token, code, PIN, card data, full phone number, full National ID or email address appears.
- Given a gRPC call without a deadline is added to any service, when the automated check runs, then the build fails.
- Given a downstream service is made slower than its deadline, when the app calls through the gateway, then it receives a mapped, recoverable error within the gateway's budget and never hangs.
- Given Stripe fails beyond the threshold, when travellers confirm payments, then the breaker opens, Confirm fails fast with a documented code, and the hold remains. When Stripe recovers, a half-open probe closes the breaker.
- Given a refund requested while the breaker is open, when Stripe recovers, then exactly one refund exists.
- Given a non-idempotent call fails, when the attempts are inspected, then it was attempted once. Given an idempotent call fails transiently, then it was retried at growing, jittered intervals, all within its deadline.
- Given a service receives SIGTERM while handling requests and messages, when it stops, then in-flight work has completed, unfinished messages are back in the queue, and nothing was lost or processed twice.
- Given the drill is run for each of the seven services, when the observed outcome is compared with the documented one, then they match, and reconciliation afterwards finds no mismatch.
- Given identity is stopped, when a signed-in traveller searches and books, then both work. When someone tries to sign in, they receive a clear error code.
- Given Jaeger is stopped, when a booking is made, then it succeeds.

---

## PLAT-6 — Go-live — the whole journey from a clean clone, under load, and as designed

**5 points · Priority Highest · `gateway,identity,catalog,inventory,booking,payment,notification,capstone,load-test,security,sprint-5`**

### User story
As a team about to present ENR
I want to demonstrate the whole traveller journey, show that the seat guarantee holds under load, and show that the architecture we designed is the one we built
So that the system is judged on what it does, not on what we say it does

### User flow (see Figma)
Services: gateway (the security pass: rate limits, the edge), inventory (seat contention under load), catalog (search under load and the cache hit rate; the seed's network), booking (signed QR codes and share links; the journey's tickets), every service (its README, its contracts, its runbook, its place in the design)

The journey, end to end, on the seed, run by a script against a fresh `docker compose up`:

  choose Arabic -> sign up -> the code reaches Mailpit within 30 seconds -> verify -> sign in -> set a PIN ->
  skip "Complete your profile" -> search Cairo → Alexandria -> try to hold -> sent back to complete the profile ->
  complete it -> filter by departure time and price -> "Available Seats: 170", "4 Stops" ->
  choose three passengers -> Review Tickets: "Coach 5 · Seat 44", 45 and 46, "Reservation Time 14:48" ->
  pay with a saved card and a 3-D Secure test card -> "Payment Success!" "Thank you! Your booking has been confirmed!" ->
  three tickets in Paid with QR, share link and PDF -> the email in Mailpit -> the bell's badge ->
  cancel one ticket 30 hours ahead -> "In Progress", then "Refunded" ->
  the operator delays the trip by 45 and then 20 minutes -> push and inbox -> a free cancellation is offered ->
  the operator cancels another seeded trip -> all its travellers refunded, progress finished ->
  kill payment mid-saga -> restart -> reconciliation finds nothing ->
  open Jaeger -> the booking, end to end

Every sprint appears in that paragraph. If any step needs a manual database edit to work, it is not finished. This story adapts DOCURA's go-live: the same proof, now across seven services instead of one.

### Rules and validations
- It starts from nothing: A reviewer clones the repository and copies the example environment (Stripe test keys and the Stripe CLI login). They run `docker compose up` and reach a running, seeded system. No step needs a manual database edit, and no step works only on the author's machine.
- The seed tells a story: The seed contains a realistic ENR network and an operator account. It contains the traveller "Ahmed Hussein" with saved passengers, a saved test card, Paid tickets, and one "In Progress" and one "Refunded" ticket in Canceled, plus unread notifications. It includes a popular weekend trip nearly sold out, a delayed trip and a cancelled trip. Trips are generated relative to today, so they always fall inside the 30-day booking window (BR-16). The seed contains no real person's data: National IDs are structurally valid and invented (BR-11).
- The demo is a script: The journey above runs as a script against the gateway and the operations API, and it asserts every step. It runs twice in a row without a reset.
- Seat contention under k6: Pick one popular trip with its last few seats. Hundreds of virtual users try to hold them at the same instant: singles and groups, on overlapping and on touching segments (BR-23). After the run, an invariant check against inventory's database (not the API) must find zero seats sold or held twice on overlapping segments. The losers must get the documented seat-taken answer, not a server error or a timeout. The report gives error rates by kind and p50, p95 and p99 latency.
- Search under load: Run a realistic mix of routes and dates through search, and report the cache hit rate (§2.3 rule 8), the latency percentiles and the database load. Run it again with Redis stopped and report how much slower it is: slower, not broken (§2.3 rule 7).
- Targets before results: Set the targets for latency, error rate and hit rate before running. Commit the k6 scripts and the results, and say plainly which targets were met.
- Nothing dangerous in the repository: Scan the whole history for secrets, not only the current files. Stripe keys come only from the environment, and no real person's data exists anywhere.
- Personal data masked in logs: Capture the logs of every service during a full demo run and check them. They must contain no National ID, full phone number, email address, code, token, PIN or card data (requirements §2.4).
- Signatures hold: A QR whose seat was edited and regenerated fails both offline and online verification (BR-38, D11). An altered share link and an expired share link are refused (BR-39). A webhook without a valid Stripe signature is refused (BR-33).
- Rate limits are real: Sign-in, code resend, search, seat holds and the operations API are rate-limited at the gateway. The counters are shared across gateway instances, and a limited request gets a 429 with a retry hint (§2.3, §10.E).
- A README per service: Each service's README names its owner and says what it owns. It lists the contracts it serves and calls, the events it publishes and consumes, the cache keys it owns, its environment variables, and how to run its tests. It ends with a runbook: what to do when the service is down, how to replay its dead letters, and its drill outcome from PLAT-5.
- The design matches the code: The design-gate artifacts are updated to match the code: the service map, the event catalogue, the `.proto` contracts, the gateway endpoint table, the error catalogue, the booking saga diagram and the cache plan. The event catalogue is checked automatically against what is actually published and consumed.
- Every ruling has a home: A table leads from each ruling, D1 to D14, to the service and the module, constraint or job that enforces it, and to the test that proves it (requirements §9). The repository also contains the written answers to the five §2.2 questions and the five edge cases you consider most dangerous (requirements §6).
- Rules are tested where they live: When a business rule is deliberately broken in the code, at least one test fails.

### Domain events to publish
- None new. The event catalogue is checked against every event the running system publishes and consumes.

### Edge cases to handle
- A reviewer runs the demo on a machine whose timezone is not Cairo.
- A reviewer runs the demo twice in a row without resetting.
- The seed runs on a different day, and the seeded trips must still be in the future and inside the 30-day window.
- The k6 run leaves hundreds of holds behind, and the demo right after it must still work.
- The Stripe CLI is not logged in. The stack must say so, instead of silently never confirming a payment.
- A secret was committed in Sprint 1 and deleted in Sprint 2.

### Acceptance criteria

- Given a clean machine, when a reviewer follows the root README, then the system runs with the seed and without manual database edits.
- Given the scripted demo, when it runs against a fresh stack, and then a second time without a reset, then every step passes both times.
- Given the k6 seat-contention test, when it ends, then the invariant check against inventory's database finds zero seats sold or held twice on overlapping segments. Every loser received the documented seat-taken code and not a server error, and the error rates and p50, p95 and p99 latencies are in the report.
- Given the search load test, when it runs, then the cache hit rate and latencies are reported. When it runs again with Redis stopped, then search still answers and the slowdown is reported.
- Given the notification service is switched off, when a traveller books, a traveller cancels a ticket and an operator cancels a trip, then all three succeed. When notification is switched back on, each event notifies exactly once.
- Given every domain event is replayed, when the results are compared, then there is no duplicate charge, refund, ticket or notification.
- Given the repository and its full history, when they are scanned for secrets, then none is found.
- Given the logs of every service after a full demo run, when they are searched, then no National ID, full phone number, email address, code, token, PIN or card data is found.
- Given a QR whose seat was edited and regenerated, an altered share link and an expired share link, when each is verified, then each is refused (§6.6).
- Given sign-in, resend, search and hold requests beyond their limits, sent across two gateway instances, when they arrive, then the limited requests receive a 429 with a retry hint.
- Given each of the seven services, when its README is read, then it names its owner, its contracts, its events and its cache keys, and contains a runbook.
- Given the event catalogue in the repository, when the automated check compares it with the running system, then every published event and every consumer match.
- Given rulings D1 to D14, when the table is followed, then each one leads to code, a constraint or a scheduled job, and to a test.
- Given the test suite passes, when a business rule is deliberately broken in the code, then at least one test fails.

---

## What is left undone — deliberately

With Sprint 5 the designed product is complete. The items below stay unbuilt because **no design exists for them** (requirements, Appendix A), and inventing them is not the exercise. Nothing you built should make them impossible, though:

- **A visual seat map** — the research's top "what's next". The server allocates seats today (D2). A map would let the traveller choose, and your allocation and hold model should not need a rewrite to allow it.
- **Train-type information** — what a train type means and why it costs what it does.
- **Contact Us** and **FAQs** — buttons in Settings with no screens behind them.
- **Discounted passenger types and children's fares** — student, senior, military, child. BR-13 already required a model that allows them without migrating existing data. This is where you find out whether yours does.
- **Non-Egyptian travellers** — passports instead of National IDs, and every rule that assumes 14 digits.
- **Changing a ticket to another train** — today that is a cancellation under the refund tiers, followed by a new booking.
- **Group bookings above 4**, **loyalty or subscriptions**, and **the profile edit screen**.
- **An operator app and a conductor app** — the operations API and the ticket verification exist; their screens do not.

Two more items are not about design at all. They make a good final conversation with your mentor about what you would build next:

- **Account deletion and data export** — required before any real launch. In a system of seven services, "delete my account" is a fan-out in its own right. You need to know which service holds what about this person, and which copies must stay for accounting (tickets keep their own copy). And, as in OPS-2, you need to know when it has finished.
- **Kubernetes deployment** — Compose proves the system runs. It does not prove the system scales, rolls out without downtime or survives a node dying. This sprint's health checks, graceful shutdown and stateless services are the prerequisites. The next questions are secrets management, autoscaling consumers, and running two instances of every scheduled job safely.
