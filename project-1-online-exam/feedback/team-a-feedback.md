# Team A — Code Review Feedback

**Reviewed:** `online-exam/team-a` · 85 TypeScript files, 5 spec files
**Standards applied:** [`clean-code.md`](../../standards/clean-code.md) · [`typescript.md`](../../standards/typescript.md) · [`testing-jest.md`](../../standards/testing-jest.md) · [Project Requirements](../requirements.md)

---

## 1. Summary

The strongest architecture of the five projects. Repository layer, dependency-injected
services, typed test doubles, Zod validation on every route, and a real attempt snapshot so
grading cannot be influenced by later edits to a quiz. The critical findings are not
structural — they are three specific rules from the requirements that were implemented
slightly differently than specified, plus unvalidated environment configuration.

| Area | Verdict |
| --- | --- |
| Architecture & layering | Strong — clean controller → service → repository, DI throughout |
| Type safety | Strong — `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` |
| Requirements coverage | Good, with 3 business-rule deviations |
| Testing | Good structure, gaps in required scenarios |
| Security | Good — no secret leaks in responses; env validation missing |

---

## 2. Requirements coverage

| Requirement | Status | Evidence |
| --- | --- | --- |
| M1 Register / Login / Forgot / Reset | ✅ | `modules/auth/auth.service.ts:50-160` |
| M2 Dashboard: passed, fastest, correct | ⚠️ | `modules/profile/profile.service.ts:9-30` — served from denormalised user counters; fastest-time counter is wrong (§3, CRIT-2) |
| M2 Quiz catalog | ✅ | `modules/quiz/quiz.service.ts` |
| M3 Start attempt, answers hidden | ✅ | `modules/attempt/attempt.service.ts:149-159` strips `correctOptionIds` and `isCorrect` |
| M3 Server-side timer + 10s grace | ⚠️ | `attempt.service.ts:111` — no grace period (§3, MAJ-1) |
| M3 Single/multi scoring | ⚠️ | `attempt.service.ts:309` exact set match ✅, but `question.points` never awarded (§3, MAJ-2) |
| M4 Result summary + review | ✅ | `attempt.service.ts:196-238`, answers revealed only after submit (`:210`) |
| §8.A.2 Env validated at startup | ❌ | `common/configs/env.config.ts:8-27` (§3, CRIT-1) |
| §8.A.3 AppError + global handler | ✅ | `common/utils/exception.util.ts`, `common/middlewares/globalError.middleware.ts` |
| §8.A.4 Validation middleware | ✅ | `common/middlewares/validation.middleware.ts` + Zod schemas per route |
| §8.A.5 Async errors reach `next` | ✅ | controllers use `try/catch → next(err)` |
| §8.B.4 Response envelope `{success,data}` / `{success,error:{code,message}}` | ❌ | `common/utils/response.util.ts:9-14` (§3, MAJ-3) |
| §9.1 `ScoreCalculator` util + spec | ❌ | no such module — grading is inline in `attempt.service.ts` |
| §9.2 AuthService tests | ✅ | `modules/auth/auth.spec.ts` — 25 tests |
| §9.3 QuizService tests (timer, scoring, idempotency) | ⚠️ | `modules/attempt/attempt.spec.ts` — 4 tests only |
| §9.4 Controller tests | ⚠️ | `quiz.spec.ts`, `diploma.spec.ts` cover controllers; no AuthController test |
| §3 Vertical slices (models inside module) | ⚠️ | `src/models/` and `src/common/repositories/` are global, not per-module |

---

## 3. Findings

### Critical

**CRIT-1 — `common/configs/env.config.ts:8-27` — TS-504, CC-701 (critical): every environment
variable is cast with `as string` instead of validated, so the app boots with `undefined`
secrets.**

```ts
export const USER_ACCESS_SECRET = process.env.USER_ACCESS_SECRET as string;
```

If `USER_ACCESS_SECRET` is absent, `jwt.sign` receives `undefined` and the failure surfaces
at the first login attempt in production, not at boot. Requirement §8.A.2 asks for
fail-fast validation.

Fix:

```ts
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().default(3000),
  MONGO_URI: z.string().url(),
  USER_ACCESS_SECRET: z.string().min(32),
  ADMIN_ACCESS_SECRET: z.string().min(32),
  REDIS_URL: z.string().url(),
});
export const env = envSchema.parse(process.env);   // throws at import, before listen()
```

You already depend on `zod` — this is a ten-line change.

**CRIT-2 — `modules/attempt/attempt.service.ts:357-366` — CC-805 (critical): `fastestTime` is
updated for every submitted attempt, including failed ones.**

```ts
await this.userRepo.updateOne({
  filter: { _id: userId },
  update: {
    $inc: { correctAnswers: correctCount, quizzesPassed: passed ? 1 : 0 },
    $min: { fastestTime: timeSpentSeconds },      // <- unconditional
  },
});
```

Requirement §4 Module 2 defines fastest completion time as *"Shortest time taken to
complete any **passed** quiz."* A student who fails a quiz in 20 seconds permanently sets
their dashboard record to 20s. `quizzesPassed` on the line above correctly guards on
`passed`; `$min` does not.

Fix — build the update conditionally:

```ts
const update = {
  $inc: { correctAnswers: correctCount, quizzesPassed: passed ? 1 : 0 },
  ...(passed && { $min: { fastestTime: timeSpentSeconds } }),
};
```

Then add the regression test: *"does not update fastestTime when the attempt failed"*.

### Major

**MAJ-1 — `modules/attempt/attempt.service.ts:111` — requirement §7.2 (major): the 10-second
grace period is missing from the expiry deadline.**

```ts
const expiresAt = new Date(now.getTime() + timeLimitSeconds * 1000);
```

Spec: `startTime + (durationMinutes * 60 + 10) * 1000`. Without the grace window a student
who submits on the last second loses the attempt to network latency. Note the deadline is
snapshotted at start, which is the right design — only the `+10s` term is missing.

**MAJ-2 — `modules/attempt/attempt.service.ts:313-318` — requirement §7.1 (major): scoring
counts questions rather than awarding `question.points`.**

```ts
const correctCount = gradedQuestions.filter(({ isCorrect }) => isCorrect).length;
const scorePercentage = Number(((correctCount / attempt.totalQuestionsSnapshot) * 100).toFixed(2));
```

Spec awards `question.points` per correct question, so questions can be weighted. The exact
set-match rule (`sameIdSet`, `:34-39`) is correct and handles both single and multi-choice —
only the weighting is missing. Add `points` to the question snapshot and sum earned/total.

**MAJ-3 — `common/utils/response.util.ts:9-14` and `common/middlewares/globalError.middleware.ts:43-47`
— requirement §8.B.4 (major): response envelope does not match the specified contract.**

Current success body: `{ statusCode, message, data }`. Current error body:
`{ statusCode, message }`. Spec requires:

```json
{ "success": true, "data": { } }
{ "success": false, "error": { "code": "EXPIRED_TIMER", "message": "..." } }
```

No machine-readable `code` is emitted anywhere, so a client cannot distinguish an expired
timer from a duplicate submission without string-matching the message. Add a `code` to
your exception classes and map it in the handler.

**MAJ-4 — `common/middlewares/globalError.middleware.ts:36` — CC-406 (major): unknown errors
default to HTTP 502.**

```ts
status = status || 502;
```

502 means "bad gateway" — an upstream failure. An unhandled application error is 500
(requirement §8.B.3). The 5xx message masking on `:38-41` is good practice; keep it.

**MAJ-5 — requirement §9.1 (major): there is no `ScoreCalculator` module or spec.**

Grading logic lives inside `attempt.service.ts` (`normalizeIds`, `sameIdSet`,
`buildQuestionSnapshot`), so it can only be tested through a service call with four mocked
repositories. The requirement calls for `modules/quiz/utils/score-calculator.ts` with its
own spec covering: single correct/incorrect, multi exact match, multi partial selection,
multi with an extra incorrect selection. Extracting it is ~20 lines and makes those four
cases trivial to test.

**MAJ-6 — `modules/attempt/attempt.spec.ts` (major): only 4 tests for the most complex service
in the project.**

Requirement §9.3 names three scenarios explicitly. Missing: timer-expiry path
(`:253-259`), double-submission rejection (`:250-252`), and the `updateResult.modifiedCount !== 1`
race guard (`:346-350`). All three are reachable with the mocks you already build.

### Minor

- **`modules/auth/auth.service.ts:67` — CC-701 (minor):** `process.env.USER_ACCESS_SECRET_EXPIRATION || '1h'`
  read inside a service. Move to the env module from CRIT-1.
- **`modules/auth/auth.service.ts:91` — TS-104 (minor):** `(error as any).code` — the line
  above already narrows with `'code' in error`, so the cast is unnecessary.
- **`common/services/securtiy.service.ts` — CC-002 (minor):** filename typo ("securtiy"),
  propagated into every importing module.
- **`src/models/`, `src/common/repositories/` — requirement §3 (minor):** the spec asks for
  vertical slices with `auth.model.ts` inside `modules/auth/`. Your global model and
  repository folders are a technical-layer split. Defensible, but state the reasoning to
  your mentor since it is an explicit design point in the brief.
- **`modules/profile/profile.service.ts:59-81` — CC-502 (minor):** `updateProfile` issues
  three sequential round trips (find, update, find). One `findOneAndUpdate` with
  `{ new: true }` does the same work.
- **`modules/attempt/attempt.service.ts:95-102` — design (minor):** the active-attempt guard
  filters on `userId` + status only, so an in-progress attempt on quiz A blocks starting
  quiz B. Intentional? If not, add `quizId` to the filter.

---

## 3b. REST API findings (`RS-###`)

Reviewed against [`restful-api.md`](../../standards/restful-api.md).

**RS-505 (major) — `src/app.ts:26`: CORS is wildcard *and* credentialed; the installed rate
limiter is never used.**

```ts
APP.use(cors({ origin: '*', credentials: true }));
```

`origin: '*'` with `credentials: true` is rejected outright by browsers — the combination
cannot work as written, and as a policy it allows any site to call the API. `express-rate-limit`
is in `dependencies` but appears in no source file, so `/auth/login` and the forgot-password
OTP endpoint accept unlimited attempts.

```ts
APP.use(cors({ origin: env.CLIENT_URL, credentials: true }));
APP.use(ROUTES.AUTH.BASE, rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }), authRouter);
```

**RS-001 (minor) — `src/routes.ts:24-27`: verbs in the profile paths.**

```ts
PROFILE: { GETPROFILE: '/get-profile', UPDATEPROFILE: '/update-profile/', GETPROFILEBYID: '/get-profile/:id' }
```

Produces `GET /profile/get-profile` and `PATCH /profile/update-profile/` (trailing slash).
The method already says get and update:

```ts
PROFILE: { BASE: '/profile', BY_ID: '/:id' }
// GET /profile, PATCH /profile, GET /profile/:id
```

**RS-006 + RS-601 (minor) — `src/app.ts:42-46`: no `/api` prefix and no version segment.**
Routers mount at `/auth`, `/quizzes`, `/attempts`, `/diplomas`, `/profile`. Prefix them with
`/api/v1` so a gateway can target the API as a unit and the first breaking change does not
break every client.

**RS-402 (major) — `common/middlewares/globalError.middleware.ts:43-47`: no machine-readable
`error.code`.** Same defect as MAJ-3; the code is the half that lets a client distinguish an
expired timer from a duplicate submission without matching on message text.

**RS-306 (minor) — `src/app.ts:27`: `express.json()` with no `limit`.** Add `{ limit: '100kb' }`.

**Clean:** plural collections throughout (`/quizzes`, `/attempts`, `/diplomas`); state
transitions modelled as sub-resources (`/quizzes/:id/start`, `/attempts/:id/submit`, RS-004);
`201` used for registration and creation (RS-201); ownership enforced in the query filter
rather than after the fetch (`attempt.service.ts:198-200`, RS-503); `helmet()` enabled;
validation bound per route for body, params **and** query (RS-301, RS-302); duration fields
carry their unit in the name — `timeLimitSeconds`, `timeSpentSeconds` (RS-604).

---

## 4. Done well — keep doing this

- **Attempt snapshotting** (`attempt.service.ts:134-140`). Grading runs against the question
  set captured at start time, so editing a quiz mid-attempt cannot change a student's score.
  Only one team did this.
- **Conditional-write concurrency guard** (`:328-350`). The update filters on
  `status: IN_PROGRESS` and `expiresAt: { $gte: now }`, then verifies `modifiedCount === 1`.
  That is a correct optimistic-locking pattern and it closes the double-submit race.
- **Compensating rollback** (`:161-164`). If snapshot creation fails, the attempt row is
  deleted. This is the only place in all five projects that handles a partial multi-step write.
- **Answer reveal gated on status** (`:210`, `:232-235`) — correct answers are spread into the
  response only once the attempt is submitted.
- **Test doubles are typed** (`attempt.spec.ts:12-22`) and fixtures are builders with
  overrides (`buildAttempt`). This is exactly the pattern `JT-104` and `JT-206` ask for.
- **`tsconfig.json`** enables `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `noImplicitReturns`, and `noUnusedLocals` — the strictest configuration of the five teams.
- Zod validation bound per route with `validate({ params, body, query })`
  (`attempt.router.ts:18-34`) — nothing reaches a controller unvalidated.

---

## 5. Priority order

1. **CRIT-1** env validation (10 lines, prevents a production outage class).
2. **CRIT-2** `fastestTime` guard + regression test.
3. **MAJ-1** add the `+10s` grace period.
4. **MAJ-3** switch to the `{success, data}` / `{success, error:{code, message}}` envelope.
5. **MAJ-5 + MAJ-2** extract `ScoreCalculator`, add `points` weighting, add its four required tests.
6. **MAJ-6** three missing `AttemptService` tests, **MAJ-4** 500 default.
7. Minors as a cleanup pass.
