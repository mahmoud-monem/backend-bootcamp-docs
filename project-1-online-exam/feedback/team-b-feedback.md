# Team B — Code Review Feedback

**Reviewed:** `online-exam/team-b` · 57 TypeScript files, 4 spec files
**Standards applied:** [`clean-code.md`](../../standards/clean-code.md) · [`typescript.md`](../../standards/typescript.md) · [`testing-jest.md`](../../standards/testing-jest.md) · [Project Requirements](../requirements.md)

---

## 1. Summary

Good instincts on structure — feature modules, a base repository, DTOs, validation schemas,
and the only team that implemented the 10-second grace period exactly as specified. The
problems are concentrated in two places, and both are serious: **the score percentage is
computed from the wrong denominator**, and **the repository layer returns `any`**, which has
propagated 77 `any` annotations through the services and disabled type checking on the code
paths that matter most.

| Area | Verdict |
| --- | --- |
| Architecture & layering | Good module split; repository generics broken |
| Type safety | Weak — 77 `any`, 17 non-null assertions, `any` leaking from the base repo |
| Requirements coverage | Scoring rules incorrect; timer correct |
| Testing | Present but shallow; the required multi-choice cases are missing |
| Security | Password hash returned to the client on login |

---

## 2. Requirements coverage

| Requirement | Status | Evidence |
| --- | --- | --- |
| M1 Register / Login / Forgot / Reset | ✅ | `modules/auth/auth.service.ts` |
| M2 Dashboard: passed, fastest, correct | ⚠️ | `modules/dashboard/dashboard.service.ts:70+` — computed by looping all attempts in Node rather than aggregating |
| M2 Quiz catalog | ✅ | `modules/quiz/quiz.service.ts` |
| M3 Start attempt, answers hidden | ✅ | `quiz.service.ts:215-223` — snapshot omits `correct` |
| M3 Server-side timer + 10s grace | ✅ | `quiz.service.ts:287-292` — exactly per spec |
| M3 Single/multi scoring | ❌ | `utils/score-calculator.ts:11-32` (§3, CRIT-1 and CRIT-2) |
| M4 Result summary + review | ⚠️ | submit returns counts; no `passed` flag is computed or stored |
| §8.A.2 Env validated at startup | ❌ | `config/config.ts:6-10` (§3, CRIT-3) |
| §8.A.3 AppError + global handler | ⚠️ | exists, but `console.log(err)` and no `success` envelope (§3, MAJ-4) |
| §8.A.4 Validation middleware | ✅ | `common/middlewares/validation.ts` + per-module `*.validation.ts` |
| §8.A.5 Async errors reach `next` | ⚠️ | controllers `try/catch → next`, but services swallow via `.catch()` (§3, CRIT-4) |
| §8.B.4 Response envelope | ⚠️ | success responses use `{success, data}` ✅; error handler returns bare `{message}` ❌ |
| §9.1 `ScoreCalculator` util + spec | ⚠️ | file exists; 3 tests, none of the required multi-choice cases |
| §9.2 AuthService tests | ✅ | `modules/auth/auth.service.spec.ts` |
| §9.3 QuizService tests | ⚠️ | `modules/quiz/tests/quiz.service.spec.ts` — no timer-expiry test, no double-submit test |
| §9.4 Controller tests | ❌ | none |

---

## 3. Findings

### Critical

**CRIT-1 — `modules/quiz/utils/score-calculator.ts:25-32` — requirement §7.1 (critical): the
score percentage is divided by the number of *selected options*, not the number of quiz
questions.**

```ts
export function calculateScorePercentage(correct_answers: number, incorrect_answers: number) {
  const total = correct_answers + incorrect_answers
  if (total === 0) return 0
  return (correct_answers / total) * 100
}
```

`total` is whatever the student happened to submit. A student who answers **one** question
out of ten, correctly, scores **100%**. Your own test locks the behaviour in:
`score-calculator.spec.ts:39` asserts `calculateScorePercentage(2, 1)` is 66.67% — three
selections, not three questions.

Fix — the denominator must be the quiz's question count (or total points):

```ts
export function calculateScorePercentage(correctQuestions: number, totalQuestions: number) {
  return totalQuestions === 0 ? 0 : (correctQuestions / totalQuestions) * 100
}
```

and in `quiz.service.ts:363`, pass `questions.length` from the quiz, not `total_wrong_answers`.

**CRIT-2 — `modules/quiz/utils/score-calculator.ts:11-20` — requirement §7.1 (critical):
multi-choice questions receive partial credit.**

```ts
for (const selectedAnswer of selectedAnswers) {
  const matchedAnswer = answerSheet.find(a => String(a.id) === String(selectedAnswer))
  if (!matchedAnswer) continue
  if (matchedAnswer.correct) correct_answers += 1
  else incorrect_answers += 1
}
```

The counter increments per *selected option*. Spec §7.1: *"Award full points **only** if the
selected options exactly match the correct options (all correct options selected, zero
incorrect options selected)."* On a question with correct options {A, B}, selecting only A
currently scores as one correct answer.

Fix — grade per question, exact set match:

```ts
export function isQuestionCorrect(selectedIds: string[], correctIds: string[]): boolean {
  if (selectedIds.length !== correctIds.length) return false
  const correct = new Set(correctIds)
  return selectedIds.every((id) => correct.has(id))
}
```

Then add the three required cases from §9.1: full match, partial selection, extra incorrect
selection.

**CRIT-3 — `config/config.ts:6-10` — TS-504, CC-702 (critical): environment variables are
coerced, not validated — and `PORT` uses a bitwise OR.**

```ts
export const PORT: number = Number(process.env.PORT) | 3000
export const JWT_SECRET: string = String(process.env.JWT_SECRET)
```

Two separate defects:

1. `|` is bitwise OR, not `||`. With `PORT=4000` the expression evaluates to `4024`
   (`0xFA0 | 0xBB8`), so the server listens on a port nobody configured. It only *looks*
   right because `NaN | 3000 === 3000`.
2. `String(undefined)` is the string `"undefined"`. With `JWT_SECRET` unset, every token in
   the system is signed with the literal secret `"undefined"` — a trivially forgeable token,
   and no error is raised anywhere.

Fix: validate with a schema (you have `joi` installed) and fail at boot.

**CRIT-4 — `modules/quiz/quiz.service.ts:160-165, 185-188, 203-208, 258-263, 269-276, 302-309, 317-324, 384-387`
— CC-503, CC-401 (critical): `.catch()` handlers that swallow the rejection and let the
function continue with `undefined`.**

```ts
const quiz: HydratedQuizDoc = await this._quizRepo
  .findDocumentById({ id: quizId })
  .catch((err: any) => {
    console.error('startQuiz: quiz lookup failed', err)
    InternalSererErrorException()
  })
```

The annotation `: HydratedQuizDoc` claims the value is always present. The `.catch` callback
returns `void`. This only compiles because `baseRepo` returns `HydratedDocument<I | any>`
(see MAJ-1), which is `any`. The exception helper does throw at runtime, so today you get
lucky — but TypeScript is checking nothing on any of these lines.

Fix: delete the `.catch()` blocks. Your controllers already forward to `next(error)`, and
the global handler maps the status. If you want the log line, log in the error middleware
where every failure passes.

**CRIT-5 — `modules/auth/auth.service.ts:78-81` — CC-801 (critical): login returns the full
Mongoose user document, including the password hash.**

```ts
return {
  token,
  user,        // whole document: password, verified, otp fields, __v
}
```

Confirmed by your own test — `auth.service.spec.ts:49-53` asserts `user: mockUser` where
`mockUser.password` is the bcrypt hash. Add a mapper:

```ts
const toUserResponse = (user: HydratedUserDoc) => ({
  id: user.id, firstName: user.firstName, email: user.email, role: user.role,
})
```

### Major

**MAJ-1 — `DataBase/repos/base.repo.ts:29, 45, 58` — TS-101, TS-502 (major): the base
repository declares `Promise<HydratedDocument<I | any> | null>`.**

`I | any` collapses to `any`. Every repository call in the project therefore returns `any`,
which is the root cause of the 77 `any` annotations, the `(q: any) =>` callbacks in
`quiz.service.ts:215-231`, and CRIT-4 compiling at all. Fixing this one line surfaces the
rest as real compiler errors:

```ts
async findAllDocuments(...): Promise<HydratedDocument<I>[]>
async findOneDocument(...): Promise<HydratedDocument<I> | null>
```

Note `findAllDocuments` also declares a *single* document where `find()` returns an array.

**MAJ-2 — `DataBase/repos/base.repo.ts:40` — CC-109 (major): a destructured parameter is
renamed and then never used, silently dropping `options`.**

```ts
async findOneDocument({
  filter,
  projection,
  options: QueryOptions          // renames `options` to a local named QueryOptions
}: { ... }) {
```

`options` is accepted by the signature but discarded — sort, limit, and populate silently do
nothing on `findOneDocument`. Rename to `options` and use it, or remove it from the type.

**MAJ-3 — `common/errors/message.error.ts:3-32` — TS-204, CC-108 (major): exception helpers
are typed `void`, so they never narrow.**

```ts
export function NotFoundException(message = 'not found', statusCode = 404) {
  throw new appError(message, statusCode)
}
```

Because the declared return type is inferred as `void` (not `never`), this compiles:

```ts
if (!quiz) { NotFoundException('Quiz not found') }
quiz.time                                       // quiz is still possibly null to TS
```

Two options — annotate `: never`, which makes TypeScript treat the call as terminal:

```ts
export function NotFoundException(message = 'not found', statusCode = 404): never {
  throw new appError(message, statusCode)
}
```

…or, clearer, make them factories and `throw` at the call site: `throw NotFound('Quiz not found')`.
Either way, `dashboard.service.ts:41-43` and `auth.service.ts:60,63` (`return NotFoundException(...)`)
stop reading as if they returned a value.

**MAJ-4 — `common/errors/global.error.handler.ts:10-14` — CC-605, CC-405, §8.B.4 (major).**

```ts
const statusCode = err instanceof appError ? err.statusCode : 500;
console.log(err)
res.status(statusCode).json({ message: err.message })
```

Three problems: `console.log` instead of a logger (`CC-605`); `err.message` is returned for
unknown 500s, leaking internal messages (`CC-405`); and the body is `{message}`, not the
required `{success: false, error: {code, message}}`.

**MAJ-5 — `modules/quiz/quiz.service.ts:226-243` — CC-601, CC-604 (major): `gradingSheet` is
computed and thrown away, and the code that used it is commented out.**

```ts
const gradingSheet = questions.map((q: any) => ({ ... }))
// await this._snapShotRepo.create({ ... })
```

The consequence is architectural, not cosmetic: without the snapshot, `submitQuiz` grades
against *live* question data (`:298-324`). If an admin edits a question or its answers while
a student is mid-attempt, the student is graded against rules that did not exist when they
started. Either finish the snapshot path or delete `quizSnapShots.repo.ts` and the dead
variable — and note the trade-off for your mentor.

**MAJ-6 — `modules/quiz/quiz.service.ts` — requirement §4 Module 4 (major): no pass/fail
determination.**

`submitQuiz` returns `score_percentage`, `correct_answers`, `incorrect_answers`,
`time_finished`, `time_spent`. Requirement M4 asks for pass/fail status computed against the
quiz's passing threshold. Nothing compares the score to a threshold.

**MAJ-7 — `modules/quiz/tests/score-calculator.spec.ts` — JT-703, requirement §9.1 (major):
the three required multi-choice cases are absent.**

Present: mixed correct/incorrect count, zero-total, percentage maths. Required by §9.1 and
missing: multi-choice **full match**, **partial selection**, **extra incorrect selection**.
Those three tests would have caught CRIT-2 before review.

**MAJ-8 — `modules/quiz/tests/quiz.service.spec.ts:29-35` — JT-306 (major): the first test
asserts on Mongoose schema internals, not on your code.**

```ts
const answersVirtual = QuestionModel.schema.virtuals.answers
expect(answersVirtual.options.ref).toBe('answers')
```

This tests that Mongoose stored the config you passed it. It cannot fail for any reason a
user would care about. Requirement §9.3 asks instead for: timer expiry, score calculation
with pass/fail, and double-submission rejection — none of which are covered.

### Minor

- **`modules/quiz/quiz.service.ts:163, 186, 204, 261, 274, 307, 322, 385` — CC-605 (minor):**
  `console.error` in a service. `morgan` is installed for HTTP logs; add a real app logger.
- **`quiz.service.ts:246` — TS-105 (minor):** `quizAttempt?.id!` combines optional chaining
  and a non-null assertion on the same expression — the two cancel out and hide the real
  question of whether the attempt exists.
- **`quiz.service.ts:215-231` — TS-103 (minor):** `(q: any)`, `(a: any)` callbacks. These
  resolve themselves once MAJ-1 is fixed.
- **`DataBase/repos/base.repo.ts:31-33` — TS-105 (minor):** `options?.limit!`, `options?.skip!`,
  `options?.sort!` — `?.` then `!` on the same access.
- **`modules/quiz/tests/score-calculator.spec.ts:7` — CC-005 (minor):** imports
  `HydratedAnswerDoc` from `../quiz.model.js` while the implementation imports it from
  `../questions/question.model.js`. Two sources for one type.
- **`common/utils/services/mail/nodemailer.ts:24`, `mail.event.ts:16` — TS-101 (minor):**
  `data: any` on the mail payload boundary.
- **`common/middlewares/authentication.ts:11` — TS-101 (major → fix with MAJ-1):**
  `user?: any` on the Express `Request` augmentation. This is the single most valuable
  `any` to remove: it disables checking on every authorization decision in the app.
- **`DataBase/` vs `config/` vs `common/` — CC-006 (minor):** inconsistent casing across
  sibling folders; `DataBase/` breaks on case-sensitive CI filesystems.

---

## 3b. REST API findings (`RS-###`)

Reviewed against [`restful-api.md`](../../standards/restful-api.md).

**RS-101 (critical) — `modules/dashboard/dashboard.routes.ts:60-66`: the dashboard statistics
endpoint is a `DELETE`, it is admin-only, and it is unreachable.**

```ts
dashboardRouter.delete(
  '/dashboard',
  validate(deleteDiplomaValidation),          // wrong schema — copied from deleteDiploma
  authenticateUser,
  authorizeRole([UserRoleEnum.admin]),        // but the handler reads req.user.id — a student
  dashboardController.getDashBoard,           // a read, behind DELETE
)
```

Four defects in six lines:

1. **`DELETE` performs a read** (RS-101). `DELETE /dashboard/dashboard` returns the caller's
   quiz statistics. Any retry policy or crawler treating DELETE as destructive will behave
   unpredictably, and no cache or client can reason about it.
2. **Unreachable** — `dashboardRouter.delete('/:id', ...)` is registered at `:52`, above this
   one. Express matches in order, so `DELETE /dashboard/dashboard` binds `:id = "dashboard"`
   and calls `deleteDiploma`. The dashboard endpoint has never run.
3. **Wrong guard** (RS-502) — `authorizeRole([admin])` on an endpoint whose handler reads
   `req.user?.id` to build *that user's* dashboard. Requirement §4 Module 2 is a student
   feature; no student can reach it.
4. **Stats returned under `message`** (RS-401) — `dashboard.controller.ts:74-77` puts the
   statistics object in the `message` field instead of `data`.

```ts
dashboardRouter.get('/me/stats', authenticateUser, dashboardController.getDashBoard)
// and in the controller: res.status(200).json({ success: true, data: stats })
```

Register it before `/:id`, or scope the two routers separately.

**RS-002 (major) — `src/app.ts:33`: the `dashboard` module is really diploma CRUD.**
`POST /dashboard`, `GET /dashboard`, `GET /dashboard/:id`, `PATCH /dashboard/:id`,
`DELETE /dashboard/:id` all operate on diplomas (`dashboard.service.ts:16-70`). A reader
cannot guess that from the URL. Mount it at `/api/v1/diplomas` and keep `/dashboard` for the
statistics endpoint above.

**RS-001 (major) — `modules/quiz/quiz.routes.ts:70, 78`: verbs in the path.**

```ts
quizRouter.post('/:id/startQuiz', ...)
quizRouter.post('/:id/submitQuiz', ...)

// Good — the transition creates a resource
quizRouter.post('/:id/attempts', ...)
quizRouter.post('/:id/attempts/:attemptId/submission', ...)
```

**RS-505 (major) — `src/app.ts:25-28`: the rate limiter is commented out.**

```ts
// app.use(rateLimit({ limit: 5, windowMs: 1000 * 60 * 2, message: 'too many requests' }))
```

`express-rate-limit` is installed and the configuration is written — it is just disabled, so
login and OTP endpoints accept unlimited attempts (also `CC-601`, commented-out code).
`helmet()` and a real `corsOptions` object are correctly applied on `:30` — good.

**RS-006 + RS-601 (minor) — `src/app.ts:32-34`: no `/api` prefix, no version.**
Routers mount at `/auth`, `/dashboard`, `/quizzes`.

**RS-402 (major) — `common/errors/global.error.handler.ts:12-14`: error body is `{message}`
only** — no `success` flag and no `code`. Same as MAJ-4.

**Clean:** `quizRouter.use('/:id/questions', questionRouter)` (`quiz.routes.ts:27`) is textbook
sub-resource nesting (RS-003); `validate(...)` + `authenticateUser` + `authorizeRole([...])`
composed per route reads clearly and makes authorization auditable (RS-501, RS-502); `201` used
on creation (RS-201); success responses use `{success: true, data}` — the closest to the brief's
envelope among the five teams.

---

## 4. Done well — keep doing this

- **Timer implementation is exactly right** (`quiz.service.ts:287-292`), including the 10s
  grace period and the comment explaining why it exists. The only team that matched the spec
  line for line.
- **Forged-answer guard** (`:341-352`): submitted answer IDs are checked against the answer
  sheet for that specific question before grading. That is a real "never trust the client"
  control, and only two teams have it.
- **Question ownership check** (`:311-315`): rejects a submission containing questions that
  do not belong to the quiz.
- **Clean module boundaries** — `*.dto.ts`, `*.validation.ts`, `*.routes.ts`, `*.model.ts` per
  feature. This is the vertical-slice structure the brief asks for.
- **Generic `baseRepo`** is the right idea; it needs its type parameters fixed, not replacing.
- **ESM + `jest.unstable_mockModule`** in `auth.service.spec.ts` is handled correctly — that
  is a genuinely awkward corner of Jest.

---

## 5. Priority order

1. **CRIT-1 + CRIT-2** — scoring is the core of the product and is currently wrong in two
   independent ways. Fix the calculator, then write the three §9.1 tests that prove it.
2. **CRIT-3** — `|` → `||`, and validate the env with Joi at boot.
3. **CRIT-5** — stop returning the password hash.
4. **MAJ-1 + MAJ-3** — fix `baseRepo` generics and make the exception helpers `never`. Expect
   a wave of new compiler errors: those are the bugs the `any` was hiding.
5. **CRIT-4** — delete the `.catch()` swallows once MAJ-1 makes them visible.
6. **MAJ-6** pass/fail, **MAJ-4** error envelope + logger, **MAJ-5** snapshot decision.
7. **MAJ-7 + MAJ-8** — replace the schema-internals test with the three §9.3 scenarios.
