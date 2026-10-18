# Sprint 4 — Payment, Tickets & Cancellation

**Project:** ENR — Egyptian National Railways Ticket Booking (Bootcamp Project 3)
**Requirements:** [`requirements.md`](./requirements.md)
**Microservices guide:** [`microservices-nestjs-guide.md`](../microservices-nestjs-guide.md)
**Design:** [Figma — Egyptian National Railways](https://www.figma.com/design/73XvSxtAQaFdsdhW2RCSqR/Egyptian-national-railways?node-id=286-3279)
**Jira import file:** [`sprint-4-payment-tickets-stories.csv`](./sprint-4-payment-tickets-stories.csv)

---

## How to read these stories

Each story describes **a service flow**: the request a client makes (the mobile app, an operator tool, a conductor's scanner, a test script — or Stripe, calling your webhook), the services involved and what each one does, then the rules that flow must obey, the events it publishes, the edge cases that will bite, and the acceptance criteria you are graded on.

No client UI is built in this project. The Figma design shows what the API must make possible: which data a client needs, and which actions it can take. It is context, not a deliverable.

What you will **not** find here: endpoint paths, request bodies, table or column names, queue names, routing keys or `.proto` definitions. Those are your design work. Derive them from the flow, the rules and what the Figma screens need from the API. Events are named in plain language ("a payment succeeded") because the **event catalogue** (names, payloads, versions, producers and consumers) is one of your deliverables (requirements §8). Stripe's own names, such as its test card numbers and its webhook event types, are given where they are facts about Stripe. They were checked against Stripe's documentation; check them again against the API version you integrate.

Every story's service flow starts with a **`Services:`** line naming the services involved and the role each one plays. The owner of the **first-listed** service is accountable for the story: they make sure the pieces in the other services land, and they demo it. Being accountable does not mean coding alone. The owner of every other service on that line reviews the pull requests that touch their service.

**What is different about this sprint.** Sprints 1 to 3 never moved money. This one does, through a provider you do not control, and it is graded accordingly:

- Three parties now hold a version of the truth (Stripe, payment and booking), and they must agree after any crash, retry or duplicate message. *"Never charged without tickets, never tickets without a charge"* (BR-35) is the sentence this sprint is graded against.
- Stripe talks to you on its own schedule, through webhooks that can arrive twice, late or out of order. A webhook can also come from someone who is not Stripe.
- Several acceptance criteria can only be satisfied by a **concurrent test** or by **killing a service** in the middle of a flow and restarting it. Reasoning that the code "looks safe" does not satisfy them.
- The saga you drew in the Sprint 1 design gate now handles real money. Every compensation on that diagram has to exist.

If you find yourself writing *the client says the payment succeeded, so confirm the booking*, stop and re-read BR-33.

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
| PAY-1 | Saved cards — adding a card through Stripe without ever seeing its number | 5 |
| PAY-2 | Charging the booking — one charge, verified with Stripe, safe to retry | 8 |
| BOOK-3 | Confirmation — the saga completes, or puts everything back | 5 |
| TKT-1 | Tickets — one per passenger per leg, a QR a conductor can trust, and the confirmation email | 5 |
| BOOK-4 | Cancelling a ticket — the refund tiers, the preview, and the outcome-changed guard | 8 |
| PAY-3 | Refund lifecycle — from "In Progress" to "Refunded", and what happens when Stripe says no | 3 |

> **If capacity is tight, cut from the edges, never from the middle.** PAY-2's webhook verification, BOOK-3's late-payment branch and BOOK-4's cancel-once guarantee are what this sprint is graded on. What may slip to the first days of Sprint 5 is TKT-1's PDF and share link. Cut nothing that moves money.

### Suggested order

```
PAY-1  Saved cards  ───────────────┐
                                   ▼
PAY-2  Charging + verified webhooks     ◀── the money path. Webhook verification on day one.
   │
   └─▶ BOOK-3  Confirmation             ◀── BR-35 and BR-36 live here
          │
          └─▶ TKT-1  Tickets: QR, PDF, share link, email
                 │
                 └─▶ BOOK-4  Cancellation (the decision)
                        │
                        └─▶ PAY-3  Refund lifecycle (the money)

BOOK-4's refund policy is a pure function: start it on day one, in parallel with everything.
```

**Start PAY-2's webhook route on day one.** Until a signed Stripe event reaches payment through `docker compose up`, nothing after it can be tested end to end. PAY-2 does not have to wait for PAY-1's saved cards: build it against Stripe's test payment methods attached to a test customer, then plug in the saved cards when PAY-1 lands.

### What carries over

| Earlier story | Used here by |
|---|---|
| PLAT-2 envelope, localized error codes, config validated at boot | PAY-2: *card declined*, *hold expired*, *payment temporarily unavailable*. BOOK-4: *outcome changed*, *cancellation window closed*. Payment refuses to boot without its Stripe keys |
| PLAT-3 outbox, inbox, retry + dead-letter, the notification email path | Every event in this sprint. TKT-1, BOOK-4 and PAY-3 emails reach Mailpit through it |
| AUTH-1 gateway: routing, local token verification, deny-by-default, Redis rate limits | PAY-2: the public, raw-body webhook route (an exception to BR-01). TKT-1: the public share link and the conductor role. PAY-2 and BOOK-4 are the first to need the gateway's cached Idempotency-Key responses (§2.3), for confirming a payment and confirming a cancellation |
| AUTH-5 complete profile, AUTH-6 saved passengers | TKT-1: tickets copy the passenger's name and National ID when they are issued (BR-12, BR-37) |
| OPS-1 operator role | PAY-3: operators see and retry failed refunds |
| SEAT-2 group allocation (BR-25) | BOOK-3: re-allocating seats after a late payment |
| SEAT-3 holds: 15 minutes, all-or-nothing, expiry release, one 3-D Secure extension | PAY-2 asks for the extension. BOOK-3 turns the hold into sold seats, or re-acquires them (BR-36) |
| SEAT-4 availability counts through events | BOOK-3 and BOOK-4: selling and freeing seats changes "Available Seats" on search without anyone touching catalog's cache |
| BOOK-1 booking draft, price lock, the hold's expiry reported by the server | PAY-2: the amount charged is the locked total, never recomputed by payment |
| BOOK-2 saga orchestrator: persisted state, timeouts, compensation on hold expiry | BOOK-3 completes it. PAY-2 makes its *awaiting payment* step real |

### Suggested split

| Student | Takes this sprint | ≈ Points |
|---|---|---|
| S1 — gateway + identity | PAY-1 saved cards end to end (code in payment, reviewed by S5). The public raw-body webhook route for PAY-2. TKT-1's signed share link with its public route, plus the conductor role, the seeded conductor account and online verification | 7 |
| S2 — catalog | BOOK-4's refund policy (a pure function tested at every boundary in §11) and the preview. PAY-3's refund lifecycle with the operators' failed-refund list and retry. TKT-1's PDF in both languages (the bilingual station and class names are S2's data) | 7 |
| S3 — inventory | BOOK-3's inventory side: hold → sold, re-acquire or re-allocate after a late payment (BR-36). PAY-2's one 3-D Secure extension. BOOK-4's seat-segment release (BR-44) with its concurrency tests | 6 |
| S4 — booking | BOOK-3 saga completion, compensation and resume after a crash. TKT-1 ticket issue, the signed QR, the paged Paid and Canceled ticket lists. BOOK-4 confirm with the outcome-changed guard and cancel-once | 7 |
| S5 — payment + notification | PAY-2 charging, idempotency keys, webhook verification and deduplication, outbox, circuit breaker. The confirmation, cancellation and refund emails | 7 |

The owner of a service **reviews every pull request that touches it**, whoever wrote it. S5 reviews S1's PAY-1 and S2's PAY-3 work in payment. S4 reviews every change in booking. S3 reviews every change in inventory, including the ones driven from BOOK-3 and BOOK-4.

### Definition of Done (every story)

- Every rule is enforced **in the service that owns it**. The client, and every other service, is never trusted with a price, an amount, a status, a refund or a ticket's validity.
- Every acceptance criterion has a test, including the negative ones and, where stated, a **concurrent** test or a **kill-and-restart** test.
- Every edge case listed has a deliberate, documented behaviour.
- Every event goes through the producer's **outbox**. Every consumer is idempotent (inbox) and retries with backoff; a message that keeps failing ends in a dead-letter queue and is never dropped. A failing consumer never undoes the business change.
- Every synchronous call has a **deadline**: gRPC between services and every call to Stripe. Stripe calls sit behind a circuit breaker.
- No password, token, card data, Stripe key, Stripe client secret, full phone number or full National ID appears in any log.
- Errors use the envelope and stable codes with Arabic and English messages. The client can tell *card declined* from *hold expired* from *outcome changed* from *cancellation window closed* by the code alone. No Stripe message, gRPC status or SQL error ever reaches the client.
- Contracts are updated in `libs/contracts` (`.proto` files and event schemas) with contract tests. The event catalogue and the cache plan match the code.
- Money is integer piastres end to end: fares, fees, Stripe amounts, refunds. No ORM hands you a string.
- It works in `docker compose up`, including the Stripe CLI forwarder, not only in unit tests.
- **Sprint 4:** no card number, CVV or Stripe secret appears in any table, log, error response or anywhere in git history. Scan the history, not just the working tree.
- **Sprint 4:** every money path is proven idempotent by **replaying its webhook and its event** (payment succeeded, payment failed, ticket cancelled, refund completed, refund failed), with a test showing nothing changed the second time.
- **Sprint 4:** an end-to-end test runs search → hold → pay with a Stripe test card → tickets issued → confirmation email in Mailpit, against the running Compose stack.

---

## EPIC-4 — Epic: PAYMENT & TICKETS — Stripe, Confirmation, Tickets and Refunds

This epic covers everything from the traveller confirming the payment to the money coming back:
- saving a card through Stripe
- charging a booking exactly once
- the saga that turns a paid hold into tickets, or puts everything back
- the tickets themselves: signed QR, PDF, share link and confirmation email
- cancelling one ticket under the refund tiers
- following each refund until Stripe confirms it

Source of truth: requirements §4 Modules 6–8, §5.5–§5.7 (BR-30 to BR-47), §6.5 and §6.6, rulings D7, D8, D9, D11 and D13, and the UX research finding *"Saving card would make this easier"*. The Figma Payment, Add card, Payment Success, Settings → Payment Methods and Tickets screens are context for what the API must make possible, not deliverables: a list of saved cards with brand and last 4, a booking state that reports when the booking is confirmed, a Paid list and a Canceled list of tickets, the actions allowed on each ticket (share, QR, download, cancel), and each refund's status ("In Progress" or "Refunded").

Depends on Sprint 1 (gateway, deny-by-default, outbox and inbox, notification's email path), Sprint 2 (catalog and the availability cache) and Sprint 3 (holds with their one 3-D Secure extension, group allocation, the price lock and the booking saga). The saga sequence diagram from the Sprint 1 design gate (happy path, decline, hold expiry, late payment, crash in the middle) is implemented end to end here.

In this sprint, a bug can charge a real person twice or leave them paid with no ticket. Stripe is the first dependency the team does not control: it is slow, it is sometimes down, and it reports results on its own schedule.

**Epic acceptance criteria**

- A traveller saves a card through Stripe and pays with it. No card number, CVV or Stripe secret exists anywhere in the system or its git history.
- A booking is confirmed only after payment has verified the charge with Stripe. Repeated or concurrent confirm requests, retries, duplicate webhooks and crashed services never produce two charges, two bookings or a charge without tickets.
- A payment that succeeds after the hold expired ends with seats or with a full refund including the booking fee. It never ends with neither.
- Every confirmed booking has one ticket per passenger per leg, with a signed QR a conductor can verify offline, a PDF, an expiring share link and a confirmation email that arrives in Mailpit.
- Cancelling a ticket applies BR-42's tiers at the moment of confirmation. A changed outcome is refused, not silently applied. The cancellation frees exactly that seat on that segment and refunds exactly once.
- Every refund is tracked from "In Progress" to "Refunded". A refund that fails is visible to operators and can be retried. Refunds on a booking never exceed what was charged.
- Every money path is proven idempotent by replaying its webhook and its event.
- The end-to-end test search → hold → pay with a Stripe test card → tickets → email in Mailpit passes against `docker compose up`.

---

## PAY-1 — Saved cards — adding a card through Stripe without ever seeing its number

**5 points · Priority High · `payment,gateway,stripe,cards,sprint-4`**

### User story
As a traveller who takes the train every weekend
I want my card saved once and offered on every payment
So that paying takes one tap instead of typing sixteen digits on a moving train

### Service flow
Services: payment (owner: the traveller's Stripe customer, the setup flow and the saved card references), gateway (authenticates the traveller and routes the client's requests; never sees card data), Stripe (external: collects the card directly from the client through its SDK and runs the bank's check)

  List saved cards: client → gateway (traveller token) → payment
    → payment reads its own records; no call to Stripe
    → one list, in one order, for every place a client needs it (the saved-cards settings and
      the payment step of PAY-2). Per card: brand (Mastercard / Visa), last 4, expiry, holder
      name (e.g. "Ahmed ****1234"), whether it is the default, whether it can pay
    → an empty list is a normal answer: the traveller has no saved card yet

  Save a card:
    client asks to add a card → gateway → payment
      → payment finds or creates the traveller's Stripe customer (one per account)
      → payment starts a Stripe setup for that customer; nothing is charged
      → the response carries only what the client needs to finish the setup with Stripe's SDK
    client → Stripe directly: Stripe's SDK collects number, expiry and CVV; the bank may ask
      for 3-D Secure here. None of it passes through ENR.
    Stripe → payment's signed webhook (setup succeeded)          ┐ either or both; payment
    client reports the setup finished → gateway → payment        ┘ verifies with Stripe
      → payment records the card once, from what Stripe returns: brand, last 4 and expiry,
        never the number
      → the card is in the list

  Delete a card: client → gateway → payment
    → payment detaches the card from the traveller's Stripe customer and removes it from the list

Card number, expiry and CVV are never fields of any ENR request. Stripe's client SDK collects
them and sends them straight to Stripe. In tests, a script finishes the setup with Stripe's test
payment methods, as the SDK would.

The research is the reason this story exists. Frequent travellers had to retype their card on
every trip: *"Saving card would make this easier."* In the re-test, the traveller who had
complained found it solved.

### Rules and validations
- Card data never reaches ENR: The card number, expiry and CVV go from Stripe's SDK on the client directly to Stripe (BR-32). No ENR request, table, log or error ever contains them. The gateway's whitelist rejects a request that tries to send them, and the rejection does not echo the values back.
- One Stripe customer per account: Payment creates the traveller's Stripe customer when it is first needed and keeps the link to the account. If two devices add the traveller's first card at the same instant, the result is still one customer. That guarantee lives in payment's database. The call to Stripe also carries an idempotency key derived from the account, never from the email; Stripe advises against personal data in keys.
- Setup, not charge: Saving a card charges nothing. Payment starts a Stripe setup for the traveller's customer and gives the client only what it needs to finish that setup with Stripe's SDK. That client secret is sensitive (Stripe says never log it and never put it in a URL), so it appears in none of your logs and none of your URLs.
- Stripe says it is saved, not the client: A card appears in the list only when payment has verified with Stripe that the setup succeeded. Payment learns this in one of two ways: Stripe's signed setup-succeeded webhook (`setup_intent.succeeded`), or a server-side check with Stripe when the client reports back. Both may happen for the same card; together they record it once.
- What payment keeps: Stripe's references (customer and payment method), brand, last 4, expiry month and year, and the card holder name, so a card can be listed as "Ahmed ****1234". Nothing else (§7.1). These values come from what Stripe returns, never from what the client claims.
- One list: Every client step that needs the saved cards (the saved-cards settings, the payment step) reads the same list, in the same order. The list is read from payment's own records; returning it does not call Stripe.
- A default: The list marks exactly one usable card as the default. Decide the rule (for example: the card used last, otherwise the newest) and document it.
- Expired cards: A card past its expiry date is refused if used to pay, and is never the default. Decide whether the list leaves it out or returns it marked as expired, and document the choice; there is one list, so the choice holds everywhere.
- The same card twice: Stripe gives every card a fingerprint. Saving a card that is already saved does not create a second row. Decide whether saving it again is a quiet success or a clear "already saved" answer.
- Delete is final: Deleting a card detaches it from the traveller's Stripe customer, which Stripe makes permanent, and removes it from the list. Payments already made with the card keep their own record of its brand and last 4.
- Owned: Another traveller's card is a 404 everywhere: in the list, on delete, when completing a setup, and when used to pay in PAY-2 (BR-09).
- Stripe down: Listing still works from payment's own data. Adding or deleting a card fails within its deadline, with a code that tells the client to try again shortly. These calls sit behind the same circuit breaker as PAY-2.
- Keys: The Stripe secret key lives only in payment's environment and is validated at boot. The client only ever holds Stripe's publishable key.

### Domain events to publish
- None that another service needs this sprint. Saved cards are payment's private business: booking only ever says "charge this booking with this card reference", and payment answers. Do not publish an event nobody consumes; it is a contract you would maintain for no one.

### Edge cases to handle
- The traveller deletes a card while a payment started with it is still in progress.
- Stripe confirms the setup, but the client never reports back (it was killed, or lost its connection).
- The bank asks for 3-D Secure while the card is being saved (Stripe test card 4000 0025 0000 3155), and the challenge is abandoned.
- A card expires between being saved and being used.
- The traveller has no saved cards at all.
- Two devices add the traveller's first card at the same second.
- A setup-succeeded webhook arrives for a Stripe customer that payment has no record of.

### Acceptance criteria

- Given a traveller with no Stripe customer, when they save Stripe's test card 4242 4242 4242 4242, then one Stripe customer exists for the account, the saved-cards list returns it as a Visa ending 4242 with its expiry, and payment's database holds only Stripe references, brand, last 4, expiry and the holder name.
- Given the end-to-end run has finished, when every service's database, every log and every error response is searched for the test card number, its CVV, any Stripe client secret and the Stripe secret key, then none of them is found.
- Given an account with no Stripe customer, when two requests to add a card arrive at the same instant, then exactly one Stripe customer exists for that account, proven by a concurrent test.
- Given the client reports a card as saved but Stripe has no succeeded setup for it, when the list is read, then no new card appears.
- Given Stripe confirmed a setup and the client never reported back, when Stripe's webhook arrives, then the card appears in the list.
- Given the same setup-succeeded webhook is delivered twice, and the client also reports back, when all three are processed, then the card is recorded once.
- Given a card that asks for 3-D Secure on setup (4000 0025 0000 3155), when the challenge is completed, then the card is saved; when it is abandoned, then nothing is saved.
- Given traveller A's saved card, when traveller B tries to list it, delete it or pay with it, then the answer is 404.
- Given a saved card used for a past booking, when the traveller deletes it, then it is no longer in the list and is detached from the Stripe customer, and that booking's payment still records the brand and last 4.
- Given a card that is already saved, when the traveller saves it again, then the list still holds one entry and the documented answer is returned.
- Given a traveller with several cards, one of them expired, when the list is requested, then exactly one valid card is marked as the default by the documented rule, and a payment with the expired card is refused.
- Given Stripe is unreachable, when the traveller's saved cards are requested, then the list is returned from payment's data; when they try to add a card, then a retryable error code is returned within the deadline.
- Given a request to the gateway that carries card number or CVV fields, when it arrives, then it is rejected by validation and no log line contains the values.
- Given payment's Stripe secret key variable is missing or malformed, when payment starts, then it refuses to boot and says which variable is wrong.

---

## PAY-2 — Charging the booking — one charge, verified with Stripe, safe to retry

**8 points · Priority Highest · `payment,booking,inventory,gateway,stripe,money,sprint-4`**

### User story
As a traveller with seats held and a card chosen
I want to pay once, with my bank's check if my bank asks for one
So that my seats become tickets and I am never charged twice

### Service flow
Services: payment (owner: the booking's single Stripe payment, every call to Stripe, webhook verification and deduplication, payment events), booking (the saga: asks for the charge and reacts to the verified result), inventory (extends the hold once when the bank asks for 3-D Secure), gateway (routes the confirm-payment request with its Idempotency-Key; exposes the public, raw-body webhook route)

  Before paying, the client already holds, from BOOK-1, a live hold with its expiry
  (e.g. "Reservation Time 14:48") and the locked amounts:
    Price 65 EGP · Booking Fee (7.0%) 4.55 EGP · Total 69.55 EGP
  and the traveller's saved cards with the default marked (PAY-1).

  Confirm payment: client → gateway (Idempotency-Key) → booking, naming the booking and the
  chosen saved card. In order:
    1. Booking checks that the hold is live and belongs to this traveller.
    2. Booking asks payment to start the booking's charge for the locked total
       (6955 piastres in EGP) with the chosen card.
    3. Payment creates the booking's one Stripe payment, or finds it if it already exists.
    4. The response carries what the client needs to confirm the payment with Stripe's SDK.
       The client confirms it with Stripe. If the bank asks for 3-D Secure, the challenge
       runs between the client, Stripe and the bank, never through ENR.
    5. Stripe calls payment's webhook.
    6. Payment verifies the event, records it and publishes "a payment succeeded".
    7. Booking takes over (BOOK-3); the client reads the booking's state until it is final.

  Declined → the client is told with a code that names the problem and what the traveller
             can do (choose another saved card, or add one); the hold's expiry is unchanged
  Abandoned → if the traveller never confirms, nothing is charged and the hold keeps its expiry

When Stripe's SDK tells the client "it worked", that is a signal to start asking booking for
the outcome. It is not a confirmation (BR-33).

Developer flow: `docker compose up` starts the Stripe CLI, which runs `stripe listen --forward-to`
pointed at your webhook route and forwards your team's test-mode events to it. Each team uses its
own free Stripe test account. The API key and the signing secret that `stripe listen` prints go
into an untracked environment file, never into the repository.

### Rules and validations
- One booking, one charge: A booking has at most one Stripe payment, however many times the confirm request is sent, the client retries, or the traveller retries after a decline. When a card is declined, Stripe puts the payment back into a state where it can be retried with another payment method. Retry the same payment; never create a second one (BR-35).
- Idempotency key derived from the booking: Every call that creates something at Stripe carries an idempotency key derived from the booking. The key is never random per request and never contains personal data. If a key is reused with different parameters, Stripe refuses the request, which is one more reason the amount must always be the locked one. Stripe may forget keys after 24 hours, so the real guarantee is payment's own database: one payment record per booking.
- Retrying the confirm request is free: It carries an Idempotency-Key header at the gateway (§10.E), and the same key returns the same response. That cached response is a convenience held in Redis. It is not what prevents a second charge: payment's database and Stripe's idempotency key are. If Redis is down, a retried confirm request is slower, never doubled.
- The locked amount, in piastres: The charge is the locked total from BOOK-1 (BR-30, BR-31), in EGP and in integer piastres. EGP is a two-decimal currency at Stripe, so 69.55 EGP is 6955. Payment never computes a price and never accepts an amount from the client. When Stripe reports success, payment checks that the amount and currency Stripe reports match the amount booking asked for.
- Stripe's minimum charge: Stripe refuses charges below a minimum. That minimum depends on your Stripe account's settlement currency, not on EGP, so find it in Stripe's documentation for your test account. A total below it is refused with its own code before any call to Stripe.
- A live hold owned by this traveller: The confirm request works only against a live hold that belongs to this traveller (SEAT-3, BOOK-2). An expired hold is refused with *hold expired*, and Stripe is never called.
- 3-D Secure buys five minutes, once: When the bank asks for authentication, booking asks inventory to extend the hold once by 5 minutes (BR-26). Payment learns about the bank's request from Stripe: either the `payment_intent.requires_action` event or its own check of the payment's status with Stripe. The client's word is not enough. A second authentication on the same booking (after a decline and a retry) gets no second extension; inventory enforces "once".
- A decline keeps the hold: A declined payment creates no tickets. The hold stays until it expires, so the traveller can retry with another card on the same booking (BR-34). The error code tells the client which case it is: *card declined*, *authentication failed* or *payment temporarily unavailable*. Stripe's own message never reaches the client (§2.4).
- The client's word proves nothing: Booking acts only on payment's "payment succeeded". Payment publishes it only after a correctly signed Stripe webhook, or after its own server-side check with Stripe (BR-33).
- Public, but signed: The webhook route needs no traveller token. BR-01's list of public endpoints does not mention it, and this story rules that it is public. Nothing is accepted without a valid Stripe signature. Stripe computes that signature over the **raw request body**, so the route must receive the bytes exactly as Stripe sent them, before any JSON parsing; a body changed by one byte fails verification. Signatures older than the tolerance are refused, which protects against replay (Stripe's libraries default to 5 minutes). Only payment holds the Stripe API key and the webhook signing secret.
- Record, acknowledge, then process: A verified event is recorded durably in payment's inbox, keyed by Stripe's event ID, and Stripe gets a 2xx straight away. The business processing happens afterwards, asynchronously. Stripe retries deliveries that do not get a 2xx (only a few times over a few hours in test mode), so an event is never acknowledged before it is recorded.
- Duplicates and disorder: The same event ID is processed once. Stripe can also send two separate events for the same object and the same type, so deduplicate on the object and the type as well. Stripe does not guarantee delivery order, so a payment's status only moves forward: a late "payment failed" from an earlier attempt never overrides a recorded success. The events named here are Stripe's snapshot events, the kind that `stripe listen --forward-to` forwards.
- Outbox: Payment records the change and writes its event ("a payment succeeded", "a payment failed", "a payment needs the bank's authentication") in the same database transaction. Every consumer is idempotent.
- Deadline and circuit breaker: Every call to Stripe has a deadline. Repeated failures open a circuit breaker, and while it is open the confirm request fails fast with *payment temporarily unavailable*; the hold is not touched. A call that timed out has an unknown outcome, so it is retried only with the same idempotency key.
- Test cards, all of them in the test suite: success 4242 4242 4242 4242 · generic decline 4000 0000 0000 0002 · insufficient funds 4000 0000 0000 9995 · always asks for 3-D Secure 4000 0027 6000 3184 · asks for 3-D Secure, then declines 4000 0084 0000 1629. A card saved with 4000 0025 0000 3155 is authenticated during its setup (PAY-1) and not asked again, so use 3184 to test the hold extension.

### Domain events to publish
- A payment succeeded (booking completes the saga in BOOK-3).
- A payment failed, with a reason category and never Stripe's raw message (booking records the attempt and keeps the hold; no tickets are created).
- A payment needs the bank's authentication (booking asks inventory for the one hold extension).

### Edge cases to handle
- Stripe's webhook arrives before the response to the confirm request has reached the client.
- Stripe sends the same event three times, plus a "failed" event from an earlier attempt after the success.
- The same confirm request arrives twice within milliseconds (a double tap), or the client retries it after a network timeout (§6.5).
- The hold expires while the bank's 3-D Secure challenge is still open on the client (BOOK-3 handles a payment that succeeds late).
- Payment is down when the confirm request arrives. Decide what the response tells the client; the hold keeps its expiry (§6.5).
- The Stripe API key is wrong in one environment. Payment's boot check or readiness check finds out, not the first traveller (§6.5).
- A captured, correctly signed webhook is posted again an hour later.
- Stripe reports success for an amount different from the locked total. Treat it as an alarm, not a confirmation.

### Acceptance criteria

- Given a live hold with a total of 69.55 EGP and a saved Visa 4242 4242 4242 4242, when the traveller confirms and the payment is completed with Stripe as the client's SDK would complete it, then Stripe holds exactly one payment of 6955 in EGP for that booking, and booking receives "a payment succeeded" exactly once.
- Given two confirm requests for the same booking arrive at the same instant, when both complete, then exactly one Stripe payment exists for the booking, proven by a concurrent test.
- Given the client retries the confirm request with the same Idempotency-Key after a timeout, when it completes, then the original response is returned and nothing new is created at Stripe.
- Given the card 4000 0000 0000 0002, when the traveller confirms, then the answer is *card declined*, no ticket exists and the hold is live with its expiry unchanged; when the traveller then confirms with 4242 4242 4242 4242, then the payment succeeds and the booking still has one Stripe payment.
- Given the card 4000 0027 6000 3184, when the bank asks for authentication, then the hold is extended once by 5 minutes; given a later retry on the same booking also asks for authentication, then no second extension is granted.
- Given a request to the webhook route with no signature, a wrong signature or a body changed by one byte, when it arrives, then it is refused with 400, nothing is recorded and no event is published.
- Given a correctly signed event is delivered twice (resent by Stripe, or replayed within the tolerance), when both are processed, then the outcome is identical to processing it once: one record, one "payment succeeded" event, no change in booking.
- Given a "payment failed" event for an earlier attempt arrives after the payment succeeded, when it is processed, then the payment stays succeeded.
- Given the client reports success but neither a signed webhook nor payment's server-side check with Stripe confirms it, when booking is asked for the booking's state, then the booking is still awaiting payment and no ticket exists.
- Given Stripe is unreachable (simulated), when the traveller confirms, then *payment temporarily unavailable* is returned within the deadline and the hold is untouched; after repeated failures, further calls fail fast without waiting on Stripe; when Stripe returns, payments succeed again.
- Given payment is killed after recording a Stripe event but before its event is published, when payment restarts, then "a payment succeeded" is published exactly once.
- Given a total below Stripe's minimum charge for the team's account, when the traveller confirms, then the request is refused with a specific code and no call reaches Stripe.
- Given an expired hold, when the traveller confirms, then *hold expired* is returned and no Stripe payment exists for the booking.
- Given traveller A's booking, when traveller B tries to pay for it, or tries to pay for their own booking with A's card, then the answer is 404.
- Given the logs of a full payment run, when they are searched, then no client secret, signing secret, API key or card number appears.

---

## BOOK-3 — Confirmation — the saga completes, or puts everything back

**5 points · Priority Highest · `booking,inventory,payment,notification,saga,money,sprint-4`**

### User story
As a traveller who has just paid
I want my booking confirmed with real seats, or my money back if that is no longer possible
So that I am never charged without a ticket

### Service flow
Services: booking (owner: the saga orchestrator; persists every step, issues the tickets, decides compensations), inventory (turns the hold into sold seats; re-acquires or re-allocates seats after a late payment), payment (the source of "a payment succeeded"; cancels the charge or refunds it when booking asks), notification (tells the traveller when seats changed or the booking was refunded)

  Once Stripe's SDK finishes, the client asks booking for the booking's state until it is final:
    client → gateway → booking → awaiting payment, then confirmed (with its tickets, TKT-1,
    and any seats that changed) or refunded. The confirmed state is what the design's
    Payment Success step consumes.

  Behind it, for a hold that is still live:
    payment publishes "a payment succeeded" → booking checks the saga's persisted state
      → inventory turns the hold into sold seats
      → booking issues the tickets, marks the booking confirmed and records
        "a booking was confirmed", all in one local transaction
      → the next read of the booking's state reports it confirmed

And for the case requirements §1.3 calls *the project*: the payment succeeds **after** the hold expired (BR-36).

  same seats still free  → re-acquire them, confirm as normal
  other seats available  → allocate them under BR-25 (side by side, then the same coach, then
                           split), confirm, and tell the traveller which seats changed
  no seats at all        → refund in full, including the booking fee, and tell the traveller

### Rules and validations
- Only on a verified payment: The saga moves forward only on payment's "a payment succeeded" (BR-33), never on anything the client sends.
- Every step persisted: The result of each step is saved before the next step starts (BOOK-2's state machine). A restarted booking service resumes every unfinished saga from its last saved state, and no step runs twice in a way that changes its outcome.
- Every step idempotent: Inventory's hold → sold conversion is keyed by the booking, so asking twice sells once. Issuing tickets twice still produces one set, guaranteed in the database (one ticket per passenger per leg per booking), not by an `if`.
- Confirmation is atomic inside booking: The tickets, the booking's confirmed state and the "a booking was confirmed" outbox entry are committed together. They exist all together or not at all.
- Success means confirmed: Booking reports the booking as confirmed only once the saga has completed. Until then the booking's state says it is still awaiting payment. Decide how the client learns the outcome (polling with backoff, or a push), and what the API answers if confirmation takes longer than expected. The answer always comes from booking, never from Stripe's SDK.
- Late payment, same seats first (BR-36): If the hold expired before the payment succeeded, booking first asks inventory for the same seats on the same segments. That request goes through the same database guarantees as any hold (SEAT-3) and the same National ID rule (BR-27).
- Late payment, then new seats: If the same seats are gone, inventory allocates new seats in the same class under BR-25. The traveller pays the locked fare, no more and no less (BR-30). The traveller is told which seats changed, both in the booking's confirmed state and by a notification.
- Late payment, no seats: If no seats exist, booking asks payment to refund the whole charge, including the booking fee (BR-36). The booking ends refunded, no ticket exists, and the traveller is told.
- A round trip is one decision: BR-36 applies to the booking as a whole. Both legs are re-acquired or re-allocated, or the whole booking is refunded. One leg is never kept without the other (D7, BR-26).
- Too late to travel: If the late payment arrives when the trip is cancelled, already departed or closed for sales, booking treats it as "no seats" and refunds in full, including the fee (§6.7).
- Expiry with no payment: When a hold expires and no payment has succeeded, booking asks payment to cancel the booking's Stripe payment so it can never succeed later. Stripe allows cancelling a payment that has not succeeded. If Stripe answers that the payment already succeeded, the late-payment path above applies. The two outcomes race, and exactly one of them must win.
- A crash after the charge (BR-35): If booking is down when the payment succeeds, the event waits in RabbitMQ; the producer's outbox guarantees it was published. Booking processes it when it comes back. If booking crashes in the middle of the saga, it resumes on start. The full Stripe-versus-bookings reconciliation job is PLAT-4 in Sprint 5. This story must make "charged but not confirmed" **detectable**: a saga with a succeeded payment that has not reached a final state is visible, through a log, a metric or a query, so that PLAT-4 has something to find.
- Compensations never strand money: A refund or cancellation that booking asks for is retried until payment accepts it, and it ends in a dead-letter queue an operator can see, never silently dropped.
- Availability follows events: Selling, re-acquiring and re-allocating seats change "Available Seats" on search only through inventory's events (SEAT-4). Booking never touches catalog's cache.

### Domain events to publish
- A booking was confirmed (notification emails the tickets, TKT-1. In Sprint 5, push, the inbox and the reminder scheduler consume the same event).
- A booking's seats were changed after a late payment (notification tells the traveller which seats they now have).
- A booking was refunded in full after a late payment (notification tells the traveller; payment's refund lifecycle, PAY-3, tracks the money).

### Edge cases to handle
- The payment succeeds 30 seconds after the hold expired, and someone else has taken the seats (§6.5).
- The payment succeeds 30 seconds after the hold expired, and the seats are still free.
- Booking is killed between inventory selling the seats and the tickets being issued.
- Inventory is down when the payment succeeds, and the hold expires while booking is retrying.
- The same "a payment succeeded" is delivered twice.
- The trip is cancelled while the traveller is paying for it (§6.7).
- A round trip: the outbound seats can be re-acquired after a late payment, but the return leg is sold out.
- The hold expires and the payment succeeds at the same instant.

### Acceptance criteria

- Given a live hold and "a payment succeeded", when the saga completes, then exactly the held seats on exactly the held segments are sold, one ticket exists per passenger per leg, and reading the booking's state returns it confirmed with its tickets.
- Given the client reports success but no "a payment succeeded" has been published, when the client asks for the booking's state, then the booking is not confirmed and its state is still awaiting payment.
- Given "a payment succeeded" is delivered twice, when both are processed, then one set of tickets exists, inventory sold the seats once, and "a booking was confirmed" was published once.
- Given booking is killed after inventory sold the seats but before the tickets were issued, when booking restarts, then the tickets are issued exactly once and nothing is sold twice. The test kills and restarts the service; it does not simulate the crash.
- Given booking is stopped when the payment succeeds, when it starts again, then the booking is confirmed without anybody touching the database.
- Given the hold expired and the same seats are still free, when the payment succeeds, then the booking is confirmed with the same seats.
- Given the hold expired and its seats were taken but others are free in the same class, when the payment succeeds, then new seats are allocated in BR-25's order, the fare is unchanged, and the traveller is told which seats changed.
- Given the hold expired and no seat is free, when the payment succeeds, then the whole charge is refunded including the booking fee (6955 piastres for a 69.55 EGP total), no ticket exists, the booking ends refunded, and the traveller is told.
- Given a round trip whose return leg is sold out after a late payment, when the payment succeeds, then the whole booking is refunded and no outbound ticket exists.
- Given a hold expires with its payment not yet succeeded, when the compensation runs, then payment cancels the Stripe payment and it can no longer succeed.
- Given the hold's expiry and the payment's success happen at the same instant, when both have been processed, then exactly one outcome exists, either confirmed or the BR-36 path: never seats sold and a full refund together, and never neither. This is proven by a concurrent test.
- Given the trip is cancelled while the traveller is paying, when the payment succeeds, then the charge is refunded in full including the fee, and no ticket exists.
- Given a saga with a succeeded payment that has not reached a final state, when the team asks "who was charged without a ticket?", then the documented log, metric or query answers it.
- Given notification is down when a booking is confirmed, when the saga completes, then the booking is still confirmed and reading its state returns confirmed (BR-54).

---

## TKT-1 — Tickets — one per passenger per leg, a QR a conductor can trust, and the confirmation email

**5 points · Priority High · `booking,notification,gateway,identity,tickets,sprint-4`**

### User story
As a traveller with a confirmed booking for my friends and me
I want each person's ticket in the app, as a QR, a PDF and a link I can send them
So that each of us can board with our own ticket, even where there is no signal

### Service flow
Services: booking (owner: tickets and their printed copies, QR signing, PDF, share links, the Paid and Canceled lists, online verification), notification (the confirmation email with the tickets), gateway (the public share-link route; the conductor route under the conductor role), identity (the conductor role and a seeded conductor account)

  List tickets: client → gateway (traveller token) → booking, for the Paid list or the
  Canceled list (the Canceled list's refund statuses belong to PAY-3)
    → booking reads its own tickets; one paged response
    → each ticket carries everything a client shows for it, plus the actions allowed on it,
      for example:
        El Raml Station   12:30 PM   27 Oct, 2025      Duration 3hr 30mins
        Ramsis Station    04:00 PM   27 Oct, 2025
        Train number 916 · class in plain words · coach · seat · "Omar Khaled Hassan"
        fare 155 EGP
        allowed actions: share · QR · download · cancel (BOOK-4 says whether cancel is open)

  QR:        client → gateway → booking → the ticket's signed QR payload
  Share:     client → gateway → booking → a signed, expiring link to this one ticket
             anyone holding the link → gateway (public route) → booking → the ticket, read-only
  Download:  client → gateway → booking → the ticket's PDF, in the request's language

  On confirmation: "a booking was confirmed" (BOOK-3) → notification → one email with the
  tickets (in Mailpit, in development)

  On the train (a conductor's scanner; no ENR UI):
    offline  the scanner verifies the QR's signature with booking's public key, from the QR alone
    online   scanner → gateway (conductor role) → booking → valid, or rejected with its reason;
             this check knows about cancellations

### Rules and validations
- One ticket per passenger per leg (BR-37): A confirmed booking produces one ticket for each passenger on each leg, all under one booking reference. A round trip for 4 passengers is 8 tickets. The booking reference is unique, short and easy to read aloud at a ticket window.
- The ticket keeps its own copy (BR-37, BR-12): At issue time, the ticket copies the passenger's name and National ID, train number, class, coach, seat, the passenger's own boarding and alighting stations (not the train's first and last stations), the times, the dates and the fare. Editing the passenger or republishing the timetable later never changes an issued ticket.
- Plain-word classes (BR-22): The ticket carries the class as plain words in both languages, for example "Second Class AC". The Figma card's "AC 3 (4)" is the naming the research asked to retire.
- Self-contained, paged lists: The list returns everything a client shows for each ticket, plus the actions allowed on it, in one response that is paged. The number of database round trips does not grow with the number of tickets.
- Paid ordering (BR-40): Paid returns active tickets first, with the next departure first, then past tickets with the most recent first. Canceled returns cancelled and voided tickets with the refund status that PAY-3 maintains.
- The ticket's state decides its actions: An upcoming ticket allows share, QR, download and cancel. Whether cancelling is currently possible is BOOK-4's answer, and it is returned with the ticket. A cancelled or voided ticket allows no share and no download (BR-39), and booking refuses them if they are requested anyway. The client never decides this on its own.
- A signed QR (BR-38, D11): The QR payload is signed with an **asymmetric** key. The private key lives only in booking. Conductors hold the public key, which can verify a QR but cannot create one, so a conductor's device that is lost or copied cannot forge tickets. The payload carries a key identifier so the key can be rotated without invalidating tickets already issued.
- What the QR says: The QR carries exactly what a conductor needs to check the passenger in front of them: train number, date, boarding and alighting stations, coach, seat, passenger name, and only the last 4 digits of the National ID. The full National ID never goes into a QR or a share link, because anyone holding the paper or the link can read them.
- Online verification: An endpoint under the conductor role (BR-01) accepts a scanned QR. It answers *valid* only if the signature verifies **and** the ticket is active in booking's database. A cancelled or voided ticket is rejected with its reason, and so is a QR whose signature fails. The answer always comes from the database, never from a cache that could miss a cancellation. A seeded conductor account exists in every environment.
- Share (BR-39): Share produces a signed link that opens this one ticket, read-only, without signing in (one of BR-01's exceptions). The link expires at the ticket's arrival time at the passenger's alighting station, and stops working the moment the ticket is cancelled. A link that has been changed in any way is not found.
- Download (BR-39): Download produces a PDF containing the QR and everything printed on the ticket, generated by booking, which owns the PDF (§2.2). When the client requests it, it is in the request's language. When it is attached to an email, it is in the account's language (BR-56). Arabic is rendered right to left, with Arabic station and class names. A cancelled ticket cannot be downloaded.
- The confirmation email (BR-53): Notification consumes "a booking was confirmed" and sends one email with the tickets in the account's language. Decide whether the email carries the PDFs, the QR codes, or links to them, and which service renders them. Notification never reads booking's database: it uses what the event carries, or asks booking through a call.
- A notification failure is not a booking failure (BR-54): If notification is down, the booking stays confirmed and the email goes out once notification recovers. The same event never sends two emails.
- Owned: Another traveller's ticket is a 404 for the list, the QR, Share and Download (BR-09).

### Domain events to publish
- No new events. The tickets travel on "a booking was confirmed" (BOOK-3): notification emails them now, and in Sprint 5 the inbox, push and the reminder scheduler consume the same event. If you find you need "a ticket was issued", justify it in your event catalogue first.

### Edge cases to handle
- A saved passenger is renamed (AUTH-6) after their ticket was issued (§6.2).
- Somebody edits the seat number inside the QR payload and regenerates the QR image (§6.6).
- A shared link is opened after the ticket was cancelled (§6.6).
- A conductor with no signal scans a ticket that was cancelled an hour ago. The offline check cannot know; decide how the online check covers it, and document the limit.
- An overnight trip: a passenger boarding at Assiut at 04:10 the next day has the boarding date of their own station on the ticket.
- The QR signing key is rotated while tickets signed with the old key are still valid.
- The client's requests are in Arabic while the account's language is English.
- Notification is down at the moment of confirmation and recovers ten minutes later.

### Acceptance criteria

- Given a confirmed round trip for 3 passengers, when the Paid list is read, then 6 tickets exist under one booking reference, each with its own passenger, leg, coach and seat.
- Given a passenger is renamed after their ticket was issued, when the ticket is read, then it still carries the name it was issued with.
- Given a traveller with past and upcoming tickets, when the Paid list is read page by page, then the next departure comes first, past tickets follow most recent first, and no ticket repeats or goes missing.
- Given a page of Paid tickets, when it is returned, then each ticket carries the stations, times, dates, duration, train number, class in plain words, coach, seat, passenger name, fare and its allowed actions, and the number of database round trips does not grow with the number of tickets.
- Given a valid ticket's QR, when it is verified with only the public key and no network, then it passes; given the seat number in the payload is edited and the QR regenerated, then it fails.
- Given an attacker who holds only the public key, when they produce a QR, then it fails verification.
- Given an active ticket, when a conductor sends its QR to online verification, then it is valid; given the ticket is then cancelled, then the same QR is rejected as cancelled; given a traveller account calls online verification, then it is refused.
- Given a shared link, when it is opened without signing in, then the ticket is returned read-only; after the ticket's arrival time it is expired; after the ticket is cancelled it no longer opens; a changed link is not found.
- Given an upcoming ticket, when it is downloaded with Arabic and then with English as the request language, then each PDF contains the QR and the printed data in that language; given a cancelled ticket, then the download is refused.
- Given a booking is confirmed, when Mailpit is checked within the time your end-to-end test allows, then exactly one confirmation email with the tickets is there, in the account's language.
- Given "a booking was confirmed" is delivered twice, when both are processed, then one email is sent.
- Given notification is stopped during a confirmation, when it starts again, then the booking was confirmed all along and exactly one email arrives.
- Given traveller A's ticket, when traveller B requests it, its QR, a share link for it or its PDF, then the answer is 404.
- Given any QR payload or share link, when it is decoded, then it contains no full National ID.

---

## BOOK-4 — Cancelling a ticket — the refund tiers, the preview, and the outcome-changed guard

**8 points · Priority Highest · `booking,inventory,payment,notification,cancellation,refund,sprint-4`**

### User story
As a traveller whose plans changed for one of my passengers
I want to cancel that passenger's ticket and see what I will get back before I confirm
So that I am never surprised by the refund, and my other passengers keep their seats

### Service flow
Services: booking (owner: the refund policy, the preview, the decision and the ticket's state), inventory (frees exactly that seat on that segment), payment (moves the money: starts the refund and follows it in PAY-3), notification (the cancellation email), gateway (the cancellation confirmation's Idempotency-Key; routes the preview and the confirmation)

  Preview: client → gateway → booking, for one ticket
    → booking applies the refund policy at this instant
    → the response carries the outcome and a way to identify it, or why cancelling is refused

  Confirm the cancellation: client → gateway (Idempotency-Key) → booking, stating which
  outcome was accepted
    → booking recomputes the outcome; if it changed → refused with *outcome changed* and the
      new preview
    → otherwise booking cancels the ticket and records "a ticket was cancelled" in its outbox,
      in one transaction
    → inventory frees the seat; payment starts the refund (PAY-3)
    → the ticket is now in the Canceled list with its refund Pending
      ("In Progress", "Refund date: Pending")

The design has a cancel action on each ticket but no confirmation step, so you define what the
preview returns. It must give the client enough to say, for example: *"You will get back
77.50 EGP (50% of 155 EGP). The booking fee is not refundable."*, or to say why cancelling
is no longer possible and what the traveller can do instead.

The preview's possible answers, measured to the **passenger's boarding time**:

  24 hours or more              → 100% of the fare
  from 2 hours up to 24 hours   → 50% of the fare
  less than 2 hours, or departed → refused: "cancellation window closed"

Then the moment that makes this story 8 points. The traveller requests the preview at 24h05m,
talks to their friend, and confirms at 23h58m. The preview promised 100%; the rule now says 50%.

### Rules and validations
- The tiers (BR-42, D9): 24 hours or more before boarding → 100% of the fare. From 2 hours up to 24 hours → 50% of the fare. Less than 2 hours, or after departure → cancellation refused.
- Boundaries favour the passenger: Exactly 24h00m00s gets 100%; 23h59m59s gets 50%. Exactly 2h00m00s gets 50%; 1h59m59s is refused. Compute the remaining time from absolute instants, never from Cairo wall-clock arithmetic, so a daylight-saving night cannot move a boundary.
- The passenger's boarding time: The clock runs to the departure from the passenger's **boarding station**, not the train's first station. A passenger boarding at Banha on a Cairo → Alexandria train is measured to Banha. In this sprint it is the scheduled boarding time printed on the ticket; from Sprint 5 a declared delay moves it (BR-49a), so the boarding time is an input to the policy, never read from the ticket inside it. A delay of 60 minutes or more also unlocks BR-49 (next rule).
- A pure policy with Sprint 5 inputs already in it: The policy is a pure function of the boarding time, the current time, the fare, the delay at the boarding station and whether ENR cancelled the trip. It is unit-tested at every case in §11: 24h00m00s, 23h59m59s, 2h00m00s, 1h59m59s, after departure, delayed by 60 minutes or more (BR-49: a full fare refund even inside the last 2 hours, until the delayed departure), and trip cancelled. In this sprint the delay is always zero; OPS-2 feeds real values in Sprint 5 without changing the policy.
- The fare, never the fee (BR-45): A traveller's cancellation refunds a percentage of **that ticket's fare**. The booking fee is never part of it. Half of an odd number of piastres rounds half-up, in the passenger's favour, the same way BR-31 rounds. For example, 155 EGP gives 15500 piastres at 100% and 7750 at 50%.
- Preview, then a guarded confirm (BR-43): The preview returns the outcome and a way to identify it, and the confirmation states which outcome the traveller accepted. At the moment of confirmation, the server recomputes the outcome. If it differs (a boundary was crossed), the cancellation is **refused** with an *outcome changed* code (409, §10.E) and the new preview. It is never silently applied. DOCURA applied the new outcome and told the user afterwards; ENR's product team ruled that unfair.
- One ticket at a time (BR-41): Cancelling one passenger's ticket touches nothing else. Their other leg and the other passengers on the booking keep their tickets and their seats.
- Cancel once (BR-47): Two confirmations for the same ticket, from two devices at the same instant or from a retry, produce one cancellation and one refund. The second request is answered *already cancelled*. The guarantee is a conditional state change in booking's database, not an `if` that both requests can pass. A retry with the same Idempotency-Key gets the original response.
- Free exactly that seat (BR-44): Inventory releases exactly that ticket's seat on exactly that ticket's segment, and it becomes bookable by someone else at once. Other seats, and the same seat on other segments, are untouched. "Available Seats" on search updates through inventory's events (SEAT-4).
- Booking decides, payment moves the money: Booking computes the refund amount. Payment executes it and never recalculates it. Booking asks payment to refund, by a call or by an event; that is your choice, justified in your design as orchestration or choreography (§2.2). Either way, the request survives a crash and carries an idempotency key derived from the cancellation, so a retry never refunds twice.
- Decide first, then carry out reliably: The cancellation and its outbox entry are committed together. The seat release and the refund follow reliably even if inventory or payment is down at that moment: they can be late, never lost and never doubled. Ruling: an outage in another service must not push a traveller across a tier boundary, so it does not block the cancellation.
- Only what can be cancelled: Another traveller's ticket is a 404 (BR-09). An already cancelled or voided ticket, a departed one, and one inside the last 2 hours are each refused with their own code, and each refusal tells the traveller what they can still do (BR-57).
- The booking fee stays: When every ticket of a booking has been cancelled, the booking's state is cancelled, and the booking fee is still not refunded (BR-45). Only ENR cancelling the trip refunds the fee (BR-48, Sprint 5).
- Reminders are not this story's business: The cancel code contains no reference to reminders. "A ticket was cancelled" is the event that Sprint 5's reminder scheduler consumes (BR-55).

### Domain events to publish
- A ticket was cancelled, carrying the refund amount booking decided. Inventory frees the seat on that segment. Payment starts the refund if you chose the event as the request. Notification sends the cancellation email. In Sprint 5, the reminder scheduler drops that ticket's reminders.

### Edge cases to handle
- The traveller cancels at exactly 24h00m00s before boarding, and at 23h59m59s (§6.6).
- The preview says 100%, and the traveller confirms 2 minutes later at 23h58m (§6.6).
- The same ticket is cancelled from two devices at the same time (§6.6).
- The traveller cancels 2 of 3 tickets in a booking, one at a time; later ENR cancels the trip (Sprint 5). Keep enough per ticket (its fare and its share of the charged fee) for BR-48 to refund the third ticket correctly.
- A passenger boarding at Banha, on a train that left Cairo 40 minutes before.
- Inventory is down at the moment of cancellation; then payment is down.
- The ticket was confirmed after a late payment with re-allocated seats (BOOK-3); cancelling it must free the new seat, not the original one.
- The cancellation happens during the night daylight saving time starts or ends.

### Acceptance criteria

- Given a 155 EGP ticket 30 hours before boarding, when the preview is requested, then it offers 100% of the fare (15500 piastres) and states that the booking fee is not refunded.
- Given the same ticket 6 hours before boarding, when the preview is requested, then it offers 50% (7750 piastres).
- Given a controllable clock, when cancellation is attempted at exactly 24h00m00s, 23h59m59s, 2h00m00s and 1h59m59s before boarding, and after departure, then the outcomes are 100%, 50%, 50%, refused and refused.
- Given a passenger boarding at Banha on a Cairo → Alexandria train, when the preview is requested, then the time is measured to the departure from Banha.
- Given a preview of 100% at 24h05m, when the traveller confirms at 23h58m, then the cancellation is refused with *outcome changed* and the new 50% preview, and nothing is cancelled; when the traveller confirms that new preview, then the ticket is cancelled at 50%.
- Given a confirmed 100% cancellation, when it completes, then the ticket is in the Canceled list with its refund Pending ("In Progress", "Refund date: Pending"), and payment was asked to refund exactly 15500 piastres, with no part of the booking fee.
- Given a booking with three passengers, when one ticket is cancelled, then the other two tickets stay in the Paid list and their seats stay sold.
- Given a cancelled Cairo → Tanta ticket on seat 44, when another traveller holds seat 44 for Cairo → Tanta, then the hold succeeds; and a Tanta → Alexandria ticket on seat 44 is untouched.
- Given two confirmations for the same ticket arrive at the same instant, when both complete, then there is one cancellation and one refund request, and the other request is answered *already cancelled*. This is proven by a concurrent test.
- Given the client retries a confirmation with the same Idempotency-Key, when it completes, then the original response is returned and no second refund is requested.
- Given inventory is stopped, when a ticket is cancelled, then the cancellation is accepted; when inventory starts again, then the seat is freed exactly once.
- Given payment is stopped, when a ticket is cancelled, then the cancellation is accepted; when payment starts again, then exactly one refund is created.
- Given another traveller's ticket, an already cancelled ticket, and a departed ticket, when cancellation is attempted on each, then the answers are 404, *already cancelled* and *cancellation window closed*, each with an Arabic and English message.
- Given the policy's unit tests, when they run, then they cover a delay of 60 minutes or more inside the last 2 hours (full fare refund) and the trip-cancelled input, even though nothing feeds those inputs until Sprint 5.
- Given "a ticket was cancelled" is delivered twice, when both are processed, then the seat is freed once, one refund exists and one email is sent.

---

## PAY-3 — Refund lifecycle — from "In Progress" to "Refunded", and what happens when Stripe says no

**3 points · Priority High · `payment,booking,notification,gateway,refund,stripe,sprint-4`**

### User story
As a traveller who cancelled a ticket
I want to see my refund move from "In Progress" to "Refunded", with the date
So that I know my money is coming back, and I am not left wondering

### Service flow
Services: payment (owner: Stripe refunds, refund webhooks and the refund lifecycle; the operators' failed-refund list and retry), booking (keeps its own copy of each cancelled ticket's refund status, fed by payment's events), notification (the refund-completed email), gateway (operator routes under the operator role)

  Read the Canceled list: client → gateway → booking (TKT-1); no call to payment
    → each cancelled ticket carries its refund status from booking's own copy, for example:
        Pending    "In Progress"   fare 155 EGP   Refund date: Pending
        Refunded   "Refunded"      fare 155 EGP   Refund date: 29 Oct, 2025

  The money: a refund request from booking (BOOK-4, or BOOK-3's full refund) → payment
    → payment creates the Stripe refund
    → Stripe → payment's signed webhook (refund updates)
    → payment publishes "a refund was completed" or "a refund failed"
    → booking updates its copy; notification sends the "refund completed" email

  Operators (an operator tool or a script; no ENR UI):
    client → gateway (operator role) → payment → the failed refunds; retry one

The lifecycle (BR-46):

  Pending ──▶ Refunded ──▶ (Failed, if Stripe later reports failure)
     │
     └──────▶ Failed ──(operator retry)──▶ Pending

### Rules and validations
- Three states, and only these arrows: Pending (reported to the traveller as "In Progress", "Refund date: Pending") → Refunded (reported as "Refunded", with the date) when Stripe confirms the refund, or → Failed. Refunded → Failed when Stripe later reports a failure (next rule). An operator's retry takes Failed back to Pending (BR-46). No other transition exists.
- Stripe can change its answer: A refund that Stripe reported as succeeded can fail later. Stripe's test card 4000 0000 0000 5126 does exactly that and sends `refund.failed`. That is why Refunded → Failed is an arrow: the traveller stops being told "Refunded", and operators see the failure.
- The refund date: The refund date is the Cairo date on which Stripe confirmed the refund. Stripe notes that the credit can take about 5–10 business days to appear on the customer's statement after that, so the date means "Stripe confirmed it", not "it reached your account".
- Verified, deduplicated refund webhooks: Refund events (`refund.created`, `refund.updated`, `refund.failed`, and `charge.refunded`, which also covers partial refunds) arrive through the same verified, deduplicated webhook as PAY-2. Because order is not guaranteed, a late "pending" update never moves a Refunded refund backwards.
- Many partial refunds, one charge: Each cancelled ticket becomes a partial refund against the booking's single charge. Stripe allows several refunds on one charge.
- Never more than was charged (BR-46, §7.2 Q8): The total of all refunds on a booking never exceeds the amount charged. Payment enforces this in its own database, and the check holds when two tickets of the same booking are cancelled at the same instant. Stripe would also refuse a refund over the original amount, but Stripe is the last line of defence, not your guarantee. Payment refuses first.
- One refund per request, one attempt at a time: Each attempt to create a Stripe refund carries an idempotency key derived from the refund and the attempt number, so replaying the request (the call or the event from BOOK-4, or BOOK-3's full refund) never creates a second Stripe refund. An operator's retry after a failure is a **new attempt** with a new key: reusing the original key would only return the original, failed refund.
- Operators see and retry failures (BR-46): Under the operator role, payment lists failed refunds with the booking reference, the ticket, the amount and a failure-reason category, and no card data. An operator can retry one. Every retry is audited: who, when and which refund (in the spirit of BR-52).
- What the traveller is told about a failed refund: The design has only two statuses for a refund. Ruling: while a refund has failed and is waiting for an operator, the ticket still reports "In Progress" with "Refund date: Pending", plus a message code explaining that the refund is delayed and being handled. The traveller is never told "Refunded" for money that did not arrive.
- Booking keeps its own copy: The Canceled list reads the refund status from booking's own copy, which payment's events keep up to date. Returning the list makes no call to payment for any ticket.
- The email (BR-53, BR-54): "A refund was completed" makes notification send one email in the account's language. Delivering the event twice sends one email.
- Test cards for the lifecycle: refunds in test mode normally succeed immediately, so Pending lasts only an instant. A payment made with 4000 0000 0000 7726 gets a refund that stays pending and then succeeds (`refund.updated`); use it to demo Pending ("In Progress") → Refunded. A payment made with 4000 0000 0000 5126 gets a refund that fails later; use it to exercise Failed.

### Domain events to publish
- A refund was completed (booking marks its copy "Refunded" with the date; notification sends the refund email).
- A refund failed (booking keeps its copy reporting "In Progress" with the delay message code; operators find the refund in payment's failed list).

### Edge cases to handle
- Stripe sends the same webhook three times, out of order with a refund webhook (§6.5).
- Stripe's refund fails because the card was closed. What does the Canceled list report for that ticket? (§6.6)
- A refund Stripe reported as succeeded fails two days later, after the "refund completed" email was sent. Decide what the traveller is told, and document it.
- An operator sends the same retry twice in a row.
- Two tickets of the same booking are cancelled at the same instant, and their refunds are created at the same instant.

### Acceptance criteria

- Given a ticket was just cancelled, when the Canceled list is read, then the ticket's refund is Pending, reported as "In Progress" with "Refund date: Pending".
- Given a payment made with 4000 0000 0000 7726, when its refund is still pending at Stripe, then the Canceled list reports it "In Progress"; when Stripe's update reports it succeeded, then the Canceled list reports it "Refunded" with that day's Cairo date.
- Given a payment made with 4000 0000 0000 5126, when Stripe reports its refund failed, then the refund is Failed, it appears in the operators' failed list, and the Canceled list does not report it "Refunded".
- Given a failed refund, when an operator retries it, then a new Stripe refund attempt exists, the status is back to Pending, and the audit records who retried and when; given a traveller account tries the same operation, then it is refused.
- Given the same refund webhook is delivered three times, out of order with the payment's own webhooks, when all are processed, then the refund changed state once and one email was sent.
- Given two tickets of one booking are cancelled at the same instant, when both refunds are created, then both exist and their total does not exceed the amount charged. This is proven by a concurrent test.
- Given a refund request that would make the booking's total refunds exceed the charge, when payment receives it, then payment refuses it without calling Stripe.
- Given the same refund request is replayed, when it is processed again, then Stripe holds one refund for it.
- Given a refund is completed, when Mailpit is checked, then exactly one "refund completed" email is there, in the account's language.
- Given a Canceled list with 20 tickets, when it is returned, then no call is made to payment for any ticket.

---

## Explicitly out of scope for Sprint 4

- **ENR cancelling or delaying a trip** (BR-48, BR-49). BOOK-4's policy already accepts a delay and a trip-cancelled input. OPS-2 in Sprint 5 feeds them, voids the tickets and refunds the fee share.
- **Push notifications and the in-app inbox.** This sprint's events already carry what NOTIF-1 needs; only email is delivered now.
- **Reminders** 24 hours and 2 hours before boarding, and dropping them when a ticket is cancelled (NOTIF-2). "A ticket was cancelled" is published now so that NOTIF-2 has nothing to change in booking.
- **The full reconciliation job** comparing Stripe with bookings, and the stuck-saga sweeper (PLAT-4). BOOK-3 makes "charged but not confirmed" detectable; Sprint 5 makes it self-healing.
- **Tracing across services** with OpenTelemetry and Jaeger, and the kill-a-service drill (PLAT-5). The correlation ID from Sprint 1 must already reach Stripe calls and webhook processing in your logs.
- **Paying with a card that is not saved.** The design's only path to a new card is saving it first (PAY-1). Do not build a one-time card, but do not make PAY-2 depend on the card being saved in a way that would make one impossible.
- **Changing a booked ticket to another train**, discounted passenger types, and the visual seat map. These are named in Appendix A and not designed.
