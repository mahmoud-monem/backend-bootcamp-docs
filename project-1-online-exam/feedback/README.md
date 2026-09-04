# Online Exam Platform — Cohort Review Summary

Five projects reviewed against the [rule catalogs](../../standards/) and the
[project requirements](../requirements.md).

| Team | [A](./team-a-feedback.md) | [B](./team-b-feedback.md) | [C](./team-c-feedback.md) | [D](./team-d-feedback.md) | [E](./team-e-feedback.md) |
| --- | :-: | :-: | :-: | :-: | :-: |
| TS files / spec files | 85 / 5 | 57 / 4 | 57 / 4 | 68 / 14 | 24 / **0** |
| Layering (controller→service→repo) | ✅ | ✅ | ⚠️ | ✅ | ✅ |
| §7.1 scoring rules correct | ⚠️ no points | ❌ partial credit + wrong denominator | ❌ no multi-choice | ⚠️ text match, duplicates pass | ✅ |
| §7.2 timer + 10s grace | ⚠️ no grace | ✅ | ❌ absent | ❌ absent | ✅ |
| §4 M2 dashboard stats | ⚠️ fastest-time bug | ⚠️ computed in Node | ❌ never written | ✅ | ❌ no module |
| §8.A.2 env validated at boot | ❌ | ❌ | ⚠️ partial | ❌ | ❌ |
| §8.B.4 response envelope | ❌ | ⚠️ | ❌ 3 shapes | ⚠️ | ⚠️ |
| Secrets / internals leaked | — | password hash | stack trace, fallback secret | password hash | stack + whole error, open admin routes |
| §9 required tests | ⚠️ partial | ⚠️ partial | ⚠️ partial | ✅ | ❌ none |
| REST: resource naming | ✅ | ⚠️ `/dashboard` = diplomas | ⚠️ verb-first paths | ⚠️ singular + camelCase | ⚠️ singular |
| REST: safe methods | ✅ | ❌ `DELETE` returns stats | ❌ `GET` starts an attempt | ✅ | ✅ |
| REST: `/api/v1` prefix | ❌ no prefix | ❌ no prefix | ⚠️ `/api`, no version | ✅ | ❌ auth at root |
| REST: helmet / CORS / rate limit | ⚠️ wildcard CORS, limiter unused | ⚠️ limiter commented out | ❌ none | ❌ helmet unused | ❌ none |

## Patterns across the cohort

**Every team failed §8.A.2 (validate env at startup).** Five different ways to write the same
bug: `as string` (A), `String(process.env.X)` (B), partial `required()` (C), `!` (D), `|| ""`
(E). Worth one shared session — a 10-line Zod/Joi env module solves it everywhere.

**No team fully implemented §8.B.4 (response envelope).** The spec's
`{success:false, error:{code, message}}` gives clients a machine-readable error code; every
project currently forces clients to string-match a message.

**Three of five have no server-side timer, or an incomplete one** — the rule the brief states
twice (§1.4, §7.2). Teams B and E implemented it exactly; use their code as the reference.

**Scoring is the most-failed business rule.** Only team E awards `question.points` and enforces
exact multi-choice matching. The failure modes are instructive: partial credit (B, C),
denominator = answers submitted rather than questions asked (B), text instead of ids (D).

**Testing maturity varies widest.** Team D wrote 159 tests including controllers; team E wrote
none. Teams A–C wrote tests that pass but skip the scenarios §9 names explicitly — the timer,
the double-submit, and the three multi-choice cases.

**No team secured the HTTP surface.** Not one has rate limiting active on login or OTP —
two teams installed the library and left it unused or commented out. Three have `helmet` in
`package.json` and never import it. Four use `cors()` with no origin allowlist. Only team D
versioned its API.

**Two teams broke HTTP method semantics** in ways that break clients before they break code:
team C starts a timed attempt on `GET` (prefetchable), team B serves dashboard statistics from
a `DELETE` that is also shadowed by an earlier `/:id` route and therefore never runs.

## Suggested cross-team reference implementations

| Concern | Look at |
| --- | --- |
| Scoring util (§7.1) | `team-e/src/modules/quiz/utils/score-calculator.ts` |
| Timer validation (§7.2) | `team-b/src/modules/quiz/quiz.service.ts:287-292` |
| Dashboard aggregation (§4 M2) | `team-d/src/modules/dashboard/dashboard.service.ts:9-57` |
| Attempt snapshot + concurrency guard | `team-a/src/modules/attempt/attempt.service.ts:134-140, 328-350` |
| Error class design | `team-c/src/utils/errors/ApiError.ts` |
| Test structure & typed doubles | `team-a/src/modules/attempt/attempt.spec.ts` |
| Controller unit tests (§9.4) | `team-d/src/modules/quizAttempt/quizAttempt.controller.spec.ts` |
