# Team C — Code Review Feedback

**Reviewed:** `online-exam/team-c` · 57 TypeScript files, 4 test files
**Standards applied:** [`clean-code.md`](../../standards/clean-code.md) · [`typescript.md`](../../standards/typescript.md) · [`testing-jest.md`](../../standards/testing-jest.md) · [Project Requirements](../requirements.md)

---

## 1. Summary

Several individually good pieces — a clean `ApiError` class with named factories, an
`asyncHandler`, an env module with a `required()` guard, Zod validation schemas — but they
are not wired together consistently. The result is that **two different authentication
middlewares verify tokens with two different secrets, neither of which matches the secret
used to sign them**, and the quiz endpoints therefore cannot authenticate a real user. On
top of that, the timer rule from §7.2 is not implemented at all, and the three dashboard
statistics are read but never written.

The recurring theme is duplication: two auth middlewares, two `authenticate` functions, two
`user.routes.ts`, two diploma service files. Each copy drifted, and the drift is where the
bugs are.

| Area | Verdict |
| --- | --- |
| Architecture & layering | Good intent (repos, services, validation); bypassed in places |
| Type safety | Mixed — `strict` on, but `(req as any)` and `any` on auth boundaries |
| Requirements coverage | Timer missing, dashboard stats never computed, multi-choice unsupported |
| Testing | 4 files, heavy over-mocking; no timer/controller tests |
| Security | Stack traces returned to clients; hard-coded fallback JWT secret |

---

## 2. Requirements coverage

| Requirement | Status | Evidence |
| --- | --- | --- |
| M1 Register / Login / Forgot / Reset | ✅ | `modules/auth/auth.service.ts:31-140` |
| M2 Dashboard: passed, fastest, correct | ❌ | `passedQuizzesCount` / `numOfCorrectAnswers` are read in 4 places and written in **none** (§3, CRIT-3) |
| M2 Quiz catalog | ✅ | `modules/quiz/quiz.services.ts` |
| M3 Start attempt, answers hidden | ✅ | `quiz.services.ts:37-41` — maps to `{_id, question, options}` only |
| M3 Server-side timer + 10s grace | ❌ | no expiry check anywhere in `submitQuiz` (§3, CRIT-2) |
| M3 Single/multi scoring | ❌ | `quiz.services.ts:145-153` — one answer per question, `includes()` (§3, CRIT-4) |
| M4 Result summary + review | ✅ | `submitQuiz` return + `getQuizAnswers` |
| §8.A.2 Env validated at startup | ⚠️ | `config/env.ts` validates `MONGO_URI`/`JWT_SECRET`; `ACCESS_SIGNITURE`, `REFRESH_SIGNITURE`, `BEARER` bypass it entirely |
| §8.A.3 AppError + global handler | ⚠️ | `ApiError` is good; the handler leaks stack traces (§3, CRIT-5) |
| §8.A.4 Validation middleware | ✅ | `modules/*/*.validation.ts` with Zod |
| §8.A.5 Async errors reach `next` | ⚠️ | `asyncHandler` exists but is not applied to `auth` or `quizHistory` (§3, CRIT-1) |
| §8.B.4 Response envelope | ❌ | `{status:'SUCCESS', message, quiz}` / `{status:"success", data}` / `{msg, stack, status}` — three shapes, none matching the spec |
| §9.1 `ScoreCalculator` util + spec | ❌ | no such module; grading is inline in `submitQuiz` |
| §9.2 AuthService tests | ⚠️ | `modules/auth/auth.test.ts` exists but mocks bcrypt away (§3, MAJ-4) |
| §9.3 QuizService tests | ⚠️ | `modules/quiz/quiz.test.ts` — no timer test (nothing to test), no double-submit test |
| §9.4 Controller tests | ❌ | none |

---

## 3. Findings

### Critical

**CRIT-1 — `common/middleware/auth.middleware.ts:53-62` — CC-501, §8.A.5 (critical): the `auth`
middleware is `async` and is not wrapped in `asyncHandler`, on Express 4.**

```ts
export const auth = async (req: AuthRequest, res: Response, next: NextFunction) => {
  const data = await decodeToken({ ... })     // throws ApiError.unauthorized()
  req.user = data
  next()
}
```

`express@4.22.2` does not forward rejected promises from middleware to the error handler
(that landed in Express 5). Every failure path inside `decodeToken` — missing header, bad
signature, unknown user, unverified account — becomes an unhandled promise rejection. The
client gets no 401 and no response at all; the request hangs until timeout.

The fix is already in your codebase — use it:

```ts
export const auth = asyncHandler(async (req, res, next) => { ... })
```

Same defect at `modules/quiz/quiz.controller.ts:26-41` (`quizHistory` is a bare `async`
handler).

**CRIT-2 — `modules/quiz/quiz.services.ts:110-189` — requirement §7.2 (critical): `submitQuiz`
performs no timer validation.**

The method checks that the quiz exists, that an attempt exists, that it was not already
submitted, and that the answer count matches — then grades. There is no comparison of
`Date.now()` against `attempt.startQuiz + duration`. A student can start a quiz, leave, and
submit a day later at full credit. Requirement §7.2 specifies:

```ts
const maxAllowedTimeMs = startTime.getTime() + (durationMinutes * 60 + 10) * 1000;
if (now.getTime() > maxAllowedTimeMs) {
  // score 0, passed false
}
```

`startQuiz` already records the timestamp (`:48`), so only the check is missing.

**CRIT-3 — `modules/quiz/quiz.services.ts:174-180` — requirement §4 Module 2 (critical): two of
the three dashboard statistics are never written.**

`passedQuizzesCount` and `numOfCorrectAnswers` are defined on the schema
(`DB/models/user.models.ts:55,65`), typed (`common/types/user.types.ts:20,22`), and returned
by the profile endpoint (`modules/profile/profile.service.ts:40-42`) — but no code path
assigns them. Every user's dashboard shows 0 passed quizzes and 0 correct answers forever.
`submitQuiz` updates only `fastestQuizTime`.

Fix — update all three in one write when an attempt is submitted:

```ts
await userRepo.findOneAndUpdate({
  filter: { _id: userId },
  update: {
    $inc: { numOfCorrectAnswers: correctAnswersCount, passedQuizzesCount: isPassed ? 1 : 0 },
    ...(isPassed && { $min: { fastestQuizTime: takenTimeSeconds } }),
  },
});
```

Note the fastest-time guard: §4 defines it over **passed** quizzes only, and in **seconds** —
`:159-161` currently stores minutes (`/ 60000`, rounded), so any quiz finished in under 30
seconds records as `0`.

**CRIT-4 — `modules/quiz/quiz.services.ts:145-153` — requirement §7.1 (critical): multi-choice
questions are not supported, and single-choice grading gives credit for any one correct
option.**

```ts
answers: string[]                                   // one answer per question, by index
questions.forEach((question, index) => {
  const userAnswer = answers[index]
  const isCorrect = question.correctAnswers.includes(userAnswer)
```

Three problems in four lines:

1. The payload is a positional `string[]`, so a client that reorders or omits one answer
   silently shifts every subsequent grade. Requirement M3 expects answers keyed by question.
2. `correctAnswers.includes(userAnswer)` — on a question with correct options {A, B},
   submitting only A scores full credit. §7.1 requires an exact set match.
3. There is no way to submit multiple selections at all.

Fix: accept `{ questionId, selectedOptionIds[] }[]`, look up by id, and compare sets.

**CRIT-5 — `bootstrap.ts:29-35` — CC-405 (critical): the error handler returns the stack trace
to the client.**

```ts
app.use((err: IError, req, res, next) => {
  res.status(err.statusCode || 500).json({
    msg: err.message,
    stack: err.stack,          // file paths, dependency versions, internal structure
    status: err.statusCode || 500
  })
})
```

Requirement §8.A.3: *"Never let standard Express route handlers ... leak raw stack traces to
users."* Log the stack server-side; return `{success: false, error: {code, message}}`. Gate
any debug detail behind `NODE_ENV !== 'production'`.

**CRIT-6 — `common/middleware/auth.middleware.ts:81` — CC-702 (critical): hard-coded fallback
JWT secret on a live route.**

```ts
signature: process.env.JWT_SECRET || "your_default_secret",
```

This middleware is mounted at `modules/profile/user.routes.ts:8`. If `JWT_SECRET` is unset,
tokens verify against a secret that is published in your repository — anyone can mint an
admin token. Never provide a fallback for a secret; fail at boot instead.

**CRIT-7 — `common/middleware/auth.middleware.ts:37-43` vs `utils/jwt.ts:12` — CC-005 (critical):
tokens are signed and verified with different secrets and different payload keys.**

| Step | Code | Secret | Claim read |
| --- | --- | --- | --- |
| Login issues token | `utils/jwt.ts:12` | `env.jwtSecret` (`JWT_SECRET`) | writes `userId` |
| Quiz routes verify | `auth.middleware.ts:40` | `process.env.ACCESS_SIGNITURE` | reads `payload._id` |
| Profile route verifies | `auth.middleware.ts:81` | `process.env.JWT_SECRET` | reads whole payload |

Unless `ACCESS_SIGNITURE` happens to hold the same string as `JWT_SECRET`, every request to
`/api/quizzes/start/:quizId`, `/submit/:quizId`, and `/:quizId/answers` fails verification.
Even when the secrets match, `payload._id` is `undefined` (the token carries `userId`), so
`userModel.findById({ id: undefined })` returns null and the request 401s — or, per CRIT-1,
hangs. `modules/profile/profile.service.ts:10-11` is already working around this
(`tokenData.userId || tokenData.id || tokenData._id`), which is the symptom, not the fix.

Pick one signing utility, one secret, one payload shape.

### Major

**MAJ-1 — duplicated modules — CC-301, CC-604 (major).**

| Duplicate | Status |
| --- | --- |
| `common/middleware/auth.middleware.ts` exports both `auth` and `authenticate` | different secrets (CRIT-7) |
| `modules/user/user.controller.ts:7` exports a third `authenticate` | mounted at `modules/user/user.routes.ts:6` |
| `modules/profile/user.routes.ts` and `modules/user/user.routes.ts` | both serve `/profile`; only the first is mounted in `bootstrap.ts:24` |
| `modules/diploma/diploma.service.ts` and `diploma.services.ts` | identical 16-line files; only `.service.ts` is imported |

Delete the unreferenced copies. Duplication on an auth path is how CRIT-6 and CRIT-7 came to
exist in the first place.

**MAJ-2 — `modules/quiz/quiz.services.ts:70-108` — CC-202, CC-502 (major): direct model access
and an N+1 query loop.**

```ts
const attempts = await QuizAttemptModel.find({ userId });
for (const attempt of attempts) {
  const quiz = await Quiz.findById(attempt.quizId);
  const questionsCount = await Question.countDocuments({ quizId: quiz._id });
```

You built `DB/repos/*` — this method bypasses it and calls three Mongoose models directly.
It also runs two queries **per attempt**: 100 attempts is 201 round trips. Use a single
aggregation, or `$in` + a lookup map.

Separately, `:90-92` reconstructs the correct-answer count from the stored percentage
(`Math.round((attempt.score / 100) * questionsCount)`) instead of storing it at submit time.
That is invented data — it will disagree with the real count as soon as rounding bites.

**MAJ-3 — `modules/quiz/quiz.services.ts:156` — CC-703 (major): the pass mark is hard-coded to 50.**

```ts
const isPassed = scorePercentage >= 50
```

Requirement §5 stores `passScorePercentage` on the quiz. Read it from `quiz`, and keep the
default in one named constant.

**MAJ-4 — `modules/auth/auth.test.ts:27-34` — JT-201, requirement §9.2 (major): `bcrypt` is
mocked, so the "password hashing" test verifies nothing.**

```ts
jest.mock('bcrypt', () => ({ hash: jest.fn().mockResolvedValue('hashed_password'), ... }))
```

Requirement §9.2 asks for *"Password hashing verification"*. With `bcrypt.hash` stubbed, the
test asserts that a mock returned the string the test itself configured. Either let bcrypt
run (it is fast enough at 10 rounds for a handful of tests) or assert the property that
matters: *the value passed to `userRepo.create` is not the plaintext password*.

**MAJ-5 — `jest.config.js:14-19` — JT-003 (major): TypeScript diagnostics are muted in tests.**

```js
diagnostics: { ignoreCodes: [151002, 2307] }
```

`2307` is "cannot find module". Suppressing it means a test can import a path that does not
resolve and still run, with the import silently `undefined`. Fix the `moduleNameMapper`
paths instead — you already have four mappings; the non-relative imports (`DB/repos/...`,
`utils/errors/...`) are what forced this workaround.

**MAJ-6 — `modules/quiz/quiz.services.ts:24-29` — requirement M3 (major): a user can never
retake a quiz, and "already started" is conflated with "already submitted".**

```ts
const alreadyStarted = await quizAttemptRepo.findOne({ filter: { quizId, userId } })
if (alreadyStarted) throw ApiError.conflict('Quiz already started')
```

Any historical attempt blocks a new one forever. Filter on `status: 'IN_PROGRESS'` (add the
field) so a finished attempt does not block a retake, and so an abandoned attempt can be
resumed or expired.

**MAJ-7 — response envelopes are inconsistent within a single file — CC-005, §8.B.4 (major).**

`quiz.controller.ts:18-22` returns `{status:'SUCCESS', message, quiz}`; `:36-40` returns
`{status:"success", message, data}`; the error handler returns `{msg, stack, status}`. Three
shapes, different casing, payload sometimes under `data` and sometimes under a feature name.
Pick the spec's envelope and apply it everywhere.

### Minor

- **`utils/secuirty/` — CC-002 (minor):** directory name typo, imported by 4 modules.
- **`modules/quiz/quiz.controller.ts:14` — TS-501 (minor):** `(req as any).user?.id`. Declare
  the Express `Request` augmentation once in `types/express.d.ts`; you already have an
  `AuthRequest` interface at `auth.middleware.ts:16` that nothing uses.
- **`modules/profile/profile.service.ts:5` — TS-101 (minor):** `getProfile(tokenData: any)`.
- **`modules/profile/profile.service.ts:17` — CC-208 (minor):** `new userRepository()` inside
  the method body; construct once at module scope like the other services.
- **`modules/quiz/quiz.controller.ts:26-41` — CC-604 (minor):** `quizHistory` is exported but
  mounted on no route (`quiz.routes.ts` has 4 routes, none for history). Note that it reads
  the user id from `req.body.user?.id` — the request **body** is client-controlled, so if you
  do mount it, that is an authorization hole. Take the id from `req.user`.
- **`bootstrap.ts:16` — CC-701 (minor):** `process.env.PORT || 5000` read in the bootstrap
  while `config/env.ts` already exposes `env.port`.
- **`bootstrap.ts:29-41` — CC-006 (minor):** the error handler is registered before the 404
  handler; conventional order is routes → 404 → error handler.
- **`quiz.services.ts:15-16` — TS-605 (minor):** `class QuizService` with arrow-function
  properties. Use methods; arrow properties are only needed when the function is detached
  from its receiver.

---

## 3b. REST API findings (`RS-###`)

Reviewed against [`restful-api.md`](../../standards/restful-api.md).

**RS-101 (critical) — `modules/quiz/quiz.routes.ts:9`: a `GET` request starts a quiz attempt.**

```ts
quizRouter.get('/start/:quizId', auth, startQuizController)
```

`startQuiz` writes an attempt row (`quiz.services.ts:43-52`) and, because of MAJ-6, that row
permanently blocks the student from ever starting the quiz again. `GET` must be safe:
browsers prefetch links, proxies retry them, and crawlers follow them. A link preview can
consume a student's only attempt.

```ts
quizRouter.post('/:quizId/attempts', auth, validate({ params: quizIdSchema }), startQuizController)
```

**RS-001 + RS-003 (major) — `modules/quiz/quiz.routes.ts:9-11`: verb-first paths that hide the
hierarchy.**

| Now | Should be |
| --- | --- |
| `GET /api/quizzes/start/:quizId` | `POST /api/quizzes/:quizId/attempts` |
| `POST /api/quizzes/submit/:quizId` | `POST /api/quizzes/:quizId/attempts/:attemptId/submission` |
| `GET /api/quizzes/:quizId/answers` | `GET /api/quizzes/:quizId/attempts/:attemptId/review` |

`/start/:quizId` also shadows nothing today but sits in the same namespace as
`GET /:quizId` — `/api/quizzes/start` would match the metadata route with `quizId="start"`.

**RS-303 (critical) — `modules/quiz/quiz.controller.ts:31`: user identity read from the request
body.**

```ts
const userId = req.body.user?.id;   // client-controlled
const history = await quizService.quizHistory(userId, type);
```

This handler is currently mounted on no route (see the minor finding in §3), which is the only
reason it is not a live vulnerability. Two things to fix before it ships: take the id from
`req.user`, and note that `QuizAttemptModel.find({ userId: undefined })` matches **every**
document — Mongoose strips undefined keys — so the failure mode is "returns all users' history",
not "returns nothing".

**RS-505 (major) — `bootstrap.ts:17-19`: no `helmet`, unrestricted CORS, no rate limiting.**

```ts
app.use(cors())        // reflects any origin
app.use(express.json())// no size limit (RS-306)
```

`helmet` is in `dependencies` but imported nowhere. `/api/auth/login`, `/forgot-password`, and
`/verify-otp` have no rate limit — the OTP is six digits, so unlimited attempts make it
guessable.

**RS-401 (major) — three response shapes.**

```ts
res.status(200).json({ status: 'SUCCESS', message: '...', quiz })      // quiz.controller.ts:18
res.status(200).json({ status: 'success', message: '...', data })      // quiz.controller.ts:36
res.status(404).send('Not found')                                      // bootstrap.ts:37 — plain text
res.status(...).json({ msg, stack, status })                           // bootstrap.ts:30
```

Different casing, payload under a feature name in one place and `data` in another, and a
non-JSON 404 from a JSON API. Adopt `{success, data}` / `{success, error:{code, message}}`
everywhere including the 404 handler.

**RS-403 (critical) — `bootstrap.ts:32`: `stack` in the response body.** Same as CRIT-5.

**RS-601 (minor) — `bootstrap.ts:23-26`: no version segment.** `/api/auth` → `/api/v1/auth`.

**Clean:** `/api` prefix applied to every router; plural collections (`/api/quizzes`,
`/api/diplomas`, `/api/users`); auth paths are correct kebab-case nouns and non-CRUD actions
(`/forgot-password`, `/verify-otp`, `/reset-password` — RS-004 handled well); `ApiError`
already carries the right status per condition (400/401/403/404/409), so RS-203 and RS-205 are
satisfied at the source — they are only lost by the handler.

---

## 4. Done well — keep doing this

- **`utils/errors/ApiError.ts`** — a clean, complete error hierarchy: status code,
  `isOperational` flag, prototype restoration for `instanceof` across the transpile boundary,
  `captureStackTrace`, and named factories. The best error class of the five teams. It just
  needs a global handler that respects it.
- **`config/env.ts:5-11`** — the `required(key)` helper that throws at import time is exactly
  the fail-fast pattern §8.A.2 asks for. Extend it to cover *all* secrets and the problem in
  CRIT-6 disappears.
- **`utils/asyncHandler.ts`** — correct implementation. Apply it consistently (CRIT-1).
- **Correct answers are stripped at start** (`quiz.services.ts:36-41`) with a comment saying
  why.
- **`forgotPassword` does not reveal whether an email exists** (`auth.service.ts:87-90`) —
  deliberate anti-enumeration handling with a comment. Good security instinct.
- **Zod schemas per module** and typed inputs (`RegisterInput`, `LoginInput`) derived from
  them — this is the `TS-202` "derive, don't duplicate" pattern.

---

## 5. Priority order

1. **CRIT-7 + CRIT-6 + MAJ-1** — collapse to one auth middleware, one secret, one payload
   shape. Nothing else can be tested until authentication works end to end.
2. **CRIT-1** — wrap `auth` in `asyncHandler`.
3. **CRIT-5** — stop returning stack traces.
4. **CRIT-2** — implement the timer check (~8 lines).
5. **CRIT-3** — write the three dashboard counters on submit.
6. **CRIT-4** — key answers by `questionId`, grade by exact set match; extract a
   `ScoreCalculator` with the §9.1 tests while you are there.
7. **MAJ-3** pass mark from the quiz, **MAJ-6** attempt status, **MAJ-2** N+1, **MAJ-7** envelope.
8. **MAJ-5** un-mute TS diagnostics, **MAJ-4** make the hashing test real.
