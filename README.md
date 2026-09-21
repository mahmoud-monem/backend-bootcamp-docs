# Bootcamp Documentation

Node.js Backend Engineering Bootcamp — project briefs, shared standards and review feedback.

## Projects

| # | Project | Focus | Stack |
|---|---|---|---|
| 1 | [Online Exam Platform](./project-1-online-exam/) | API extraction from a design · unit testing | Express · TypeScript · MongoDB · Jest |
| 2 | [DOCURA — Doctor Booking](./project-2-docura/) | **Event-driven architecture** · domain modeling · money and concurrency | **NestJS · TypeScript · PostgreSQL** |

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

## Shared across projects

- [`standards/`](./standards/) — the rule catalogs code is reviewed against (clean code, TypeScript, Jest, RESTful API)
- [`design-patterns-session.md`](./design-patterns-session.md) — design patterns session material
- [`git-github-guide.md`](./git-github-guide.md) — git, GitHub and version control handbook: commits, branching, pull requests, code review, releases
- [`readme-guide.md`](./readme-guide.md) — README handbook: what belongs in a README, and how to design and document the repo's coding style guide
