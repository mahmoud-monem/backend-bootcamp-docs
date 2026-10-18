# Bootcamp Documentation

Node.js Backend Engineering Bootcamp — project briefs, shared standards and review feedback.

## Projects

| # | Project | Focus | Stack |
|---|---|---|---|
| 1 | [Online Exam Platform](./project-1-online-exam/) | API extraction from a design · unit testing | Express · TypeScript · MongoDB · Jest |
| 2 | [DOCURA — Doctor Booking](./project-2-docura/) | **Event-driven architecture** · domain modeling · money and concurrency | **NestJS · TypeScript · PostgreSQL** |
| 3 | [ENR — Egyptian National Railways](./project-3-enr/) | **Microservice architecture** · sagas · **caching** · segment seat inventory · Stripe payments | **NestJS monorepo · PostgreSQL · Redis · RabbitMQ · gRPC · Stripe · Docker Compose** |

### Project 1 — Online Exam Platform
- [`requirements.md`](./project-1-online-exam/requirements.md) — project brief

### Project 2 — DOCURA
- [`requirements.md`](./project-2-docura/requirements.md) — business requirements, rules, edge cases and design challenges

Four sprints of user stories, each with a `.csv` of the same stories ready to import into Jira:

| Sprint | Stories | Points | Covers |
|---|---|---|---|
| [1 — Authentication](./project-2-docura/sprint-1-auth-stories.md) | 6 | 31 | Guest access, registration, OTP, sign-in, recovery, sessions |
| [2 — Discovery](./project-2-docura/sprint-2-catalog-stories.md) | 7 | 36 | Catalogue, home, search, filters, sorting, map, favourites |
| [3 — Booking](./project-2-docura/sprint-3-booking-stories.md) | 8 | 47 | Availability, slot holds, cards, payment, history, cancellation, reschedule, scheduled work |
| [4 — Engagement](./project-2-docura/sprint-4-engagement-stories.md) | 7 | 34 | Notifications, ask a doctor, AI assistant, safety, prescriptions, go-live |

### Project 3 — ENR
- [`requirements.md`](./project-3-enr/requirements.md) — business requirements, rules, edge cases, the mandatory service split and cache rules, and design challenges
- [`microservices-nestjs-guide.md`](./microservices-nestjs-guide.md) — how to build the services, and the `MS-###` rules they are reviewed against

3 teams × 5 students; every team builds the whole system and every student owns one or two services. Five one-week sprints of user stories, each with a `.csv` of the same stories ready to import into Jira:

| Sprint | Stories | Points | Covers |
|---|---|---|---|
| [1 — Foundations & Identity](./project-3-enr/sprint-1-foundations-identity-stories.md) | 7 | 33 | Design gate, platform skeleton, messaging backbone, gateway, sign-up, email OTP, sessions, PIN |
| [2 — Network & Search](./project-3-enr/sprint-2-network-search-stories.md) | 8 | 37 | Profile, passengers, stations, timetables, one-way and round-trip search, filters, cache layer, operations API |
| [3 — Seats & Holds](./project-3-enr/sprint-3-seats-holds-stories.md) | 6 | 34 | Segment seat inventory, group allocation, holds, availability counts, booking draft, booking saga |
| [4 — Payment & Tickets](./project-3-enr/sprint-4-payment-tickets-stories.md) | 6 | 34 | Stripe cards and charges, confirmation, tickets and QR, cancellation, refunds |
| [5 — Disruptions & Go-live](./project-3-enr/sprint-5-disruptions-golive-stories.md) | 6 | 31 | Trip delays and cancellations, notification center, reminders, reconciliation, resilience, load test, go-live |

## Shared across projects

- [`standards/`](./standards/) — the rule catalogs code is reviewed against (clean code, TypeScript, Jest, RESTful API)
- [`microservices-nestjs-guide.md`](./microservices-nestjs-guide.md) — microservices with NestJS handbook: boundaries, gateway, gRPC, RabbitMQ, outbox, sagas, caching, observability, resilience, testing
- [`design-patterns-session.md`](./design-patterns-session.md) — design patterns session material
- [`git-github-guide.md`](./git-github-guide.md) — git, GitHub and version control handbook: commits, branching, pull requests, code review, releases
- [`readme-guide.md`](./readme-guide.md) — README handbook: what belongs in a README, and how to design and document the repo's coding style guide
