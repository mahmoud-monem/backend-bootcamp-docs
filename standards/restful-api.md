# RESTful API Rule Catalog (`RS-###`)

Applies to route modules (`*.routes.ts`, `*.router.ts`), controllers, the app bootstrap, and
any code that builds an HTTP response.

Read [`README.md`](./README.md) for severity meanings, the verify-before-report protocol,
and the false-positive rules.

**Bands**

| Band | Topic |
| --- | --- |
| `RS-0xx` | Resource naming and URL design |
| `RS-1xx` | HTTP methods and semantics |
| `RS-2xx` | Status codes |
| `RS-3xx` | Request design |
| `RS-4xx` | Response design |
| `RS-5xx` | Authentication, authorization, transport |
| `RS-6xx` | Versioning, headers, documentation |

**Reference contract for this codebase** (from the project brief §8.B):

```jsonc
// success
{ "success": true, "data": { } }

// error
{ "success": false, "error": { "code": "EXPIRED_TIMER", "message": "Quiz submission expired (timer exceeded)" } }
```

---

## RS-0xx — Resource naming and URL design

### RS-001 — Verb in the URL path
**Severity:** major · **Applies to:** `*.routes.ts`, `*.router.ts`
**Detect:** Path segments that are actions rather than resources: `/getQuizzes`,
`/createUser`, `/quiz/delete/:id`, `/update-profile`.
Grep hint: `router\.(get|post|put|patch|delete)\(['"]/(get|create|update|delete|list|add|remove)`

```ts
// Bad
router.get('/getAllQuizzes', ...);
router.post('/createQuiz', ...);
router.delete('/deleteQuiz/:id', ...);

// Good
router.get('/quizzes', ...);
router.post('/quizzes', ...);
router.delete('/quizzes/:id', ...);
```

**Why:** The HTTP method is already the verb. Repeating it in the path doubles the vocabulary
a client must learn and breaks caching and tooling that key on method + path.
**Not a violation when:** The segment names a genuine sub-resource or a state transition that
is not CRUD — see `RS-004`.

---

### RS-002 — Resource collections not plural nouns
**Severity:** minor · **Applies to:** `*.routes.ts`, app bootstrap
**Detect:** Singular collection mounts: `/api/quiz`, `/api/user`, `/api/attempt`; mixed
plurality across modules in one app.

```ts
// Bad
app.use('/api/v1/quiz', quizRouter);
app.use('/api/v1/question', questionRouter);
app.use('/api/quizzes', quizRouter);      // and plural elsewhere in the same app

// Good
app.use('/api/v1/quizzes', quizRouter);
app.use('/api/v1/questions', questionRouter);
```

**Why:** `/quizzes` is the collection, `/quizzes/:id` is one member. Singular mounts make
`/quiz/:id` read as "the quiz's id-th field" and force the reader to check every route to
learn the convention.
**Not a violation when:** The resource is a genuine singleton for the caller (`/api/me`,
`/api/profile`, `/api/health`).

---

### RS-003 — Hierarchy not expressed as sub-resources
**Severity:** minor · **Applies to:** `*.routes.ts`
**Detect:** Parent identity carried in a query string or body where it belongs in the path:
`GET /questions?quizId=x`, `POST /submit` with `quizId` in the body.

```ts
// Bad
router.get('/questions', ...);              // quizId arrives as ?quizId=
router.post('/submit/:quizId', ...);        // action-first path

// Good
router.get('/quizzes/:quizId/questions', ...);
router.post('/quizzes/:quizId/attempts/:attemptId/submission', ...);
```

**Why:** The path should express containment. It makes authorization checks obvious (the
parent id is right there) and lets routers mount the child router on the parent.
**Not a violation when:** The child is addressable independently of any parent and the query
parameter is a filter, not an identity (`GET /attempts?status=submitted`).

---

### RS-004 — Non-CRUD state transition modelled badly
**Severity:** minor · **Applies to:** `*.routes.ts`
**Detect:** State changes crammed into `PUT /resource/:id` with a magic body field
(`{ action: "start" }`), or exposed as a top-level verb route (`POST /startQuiz`).

```ts
// Bad
router.put('/quizzes/:id', ...);            // body: { action: 'start' }
router.post('/startQuiz/:id', ...);

// Good
router.post('/quizzes/:id/attempts', ...);        // starting a quiz creates an attempt
router.post('/attempts/:id/submission', ...);     // submitting creates a submission
```

**Why:** Most "actions" are the creation of a resource. Naming that resource makes the state
machine visible and gives the client something to `GET` afterwards.
**Not a violation when:** The action genuinely has no resource
(`POST /auth/password-reset-requests` is fine; so is a documented `/:id/start` sub-path used
consistently).

---

### RS-005 — Inconsistent path casing or separators
**Severity:** minor · **Applies to:** `*.routes.ts`, app bootstrap
**Detect:** `camelCase` or `snake_case` in paths, or mixed styles:
`/api/v1/quizAttempt`, `/api/quiz_attempts`, `/api/quizAttempts` beside `/api/quizzes`.

```ts
// Bad
app.use('/api/v1/quizAttempt', quizAttemptRouter);

// Good
app.use('/api/v1/quiz-attempts', quizAttemptRouter);
```

**Why:** URLs are case-sensitive on the path. `kebab-case` is the web convention and avoids
the class of bug where `/quizAttempt` works locally and 404s behind a normalising proxy.
**Not a violation when:** The whole API uses one alternative convention consistently.

---

### RS-006 — Routes mounted outside the API prefix
**Severity:** minor · **Applies to:** app bootstrap
**Detect:** Some routers mounted under `/api` and others at the root.

```ts
// Bad
app.use('/api/quiz', quizRouter);
app.use(authRouter);              // auth endpoints land on /login, /register

// Good
app.use('/api/v1/quizzes', quizRouter);
app.use('/api/v1/auth', authRouter);
```

**Why:** A single prefix is what lets a reverse proxy, rate limiter, or auth gateway target
the API as one unit.
**Not a violation when:** The route is deliberately outside the API surface (`/health`,
`/metrics`, static assets).

---

## RS-1xx — HTTP methods and semantics

### RS-101 — Wrong method for the operation
**Severity:** major · **Applies to:** `*.routes.ts`
**Detect:** `GET` routes that create or mutate state; `POST` used to read; `GET` handlers
that write to the database.

```ts
// Bad
router.get('/quizzes/:id/start', ...);      // creates an attempt row
router.post('/quizzes/list', ...);          // reads only

// Good
router.post('/quizzes/:id/attempts', ...);
router.get('/quizzes', ...);
```

**Why:** `GET` is expected to be safe. Browsers, proxies, crawlers, and link prefetchers will
issue it without a user action — a `GET` that starts a timed attempt can be triggered by a
preview thumbnail.
**Not a violation when:** Never for state-changing `GET`. Read-only `POST` is acceptable only
when the query is too large for a URL, and should be documented.

---

### RS-102 — Non-idempotent handler behind an idempotent method
**Severity:** major · **Applies to:** `*.routes.ts`, controllers
**Detect:** `PUT` or `DELETE` handlers that fail or duplicate on retry: `PUT` that appends to
an array, `DELETE` that 404s on the second call after succeeding on the first.
**Why:** Clients, proxies, and job runners retry `PUT`/`DELETE` on timeout. Non-idempotent
behaviour turns one retry into two attempts or a spurious error.
**Not a violation when:** `POST` — it is explicitly not idempotent.

---

### RS-103 — `PUT` and `PATCH` used interchangeably
**Severity:** minor · **Applies to:** `*.routes.ts`, controllers
**Detect:** `PUT` handlers that apply a partial update (`$set` of whatever keys arrived), or
`PATCH` handlers that replace the whole document.

```ts
// Bad
router.put('/profile', ...);        // body: { firstName } only — a partial update

// Good
router.patch('/profile', ...);      // partial update
router.put('/quizzes/:id', ...);    // full replacement, all required fields validated
```

**Why:** `PUT` means "make the resource equal to this representation". A `PUT` treated as a
patch silently keeps fields the client believes it removed.
**Not a violation when:** The API documents `PUT` as upsert-replace and validates the full
representation.

---

### RS-104 — Missing method returns the wrong error
**Severity:** minor · **Applies to:** app bootstrap
**Detect:** No catch-all 404 handler, so an unknown path falls through to the framework's
HTML error page; or an unmatched method on a known path returns 404 instead of 405.
**Why:** A client hitting `DELETE /api/quizzes` should learn the method is unsupported, not
that the collection does not exist.
**Not a violation when:** A JSON 404 catch-all exists; 405 handling is a refinement, not a
requirement.

---

## RS-2xx — Status codes

### RS-201 — Success status does not match the operation
**Severity:** major · **Applies to:** controllers
**Detect:** `201` on a read, `200` on a creation, `200` on a delete with no body.
Grep hint: `res.status(200).json` inside a `create*` handler.

```ts
// Bad
res.status(200).json({ success: true, data: attempt });     // in startAttempt (a creation)

// Good
res.status(201).json({ success: true, data: attempt });     // resource created
res.status(200).json({ success: true, data: quizzes });     // read
res.status(204).send();                                     // deleted, no body
```

**Why:** Status is the part of the response a client can branch on without parsing. Brief
§8.B.3 names `201` for registration and for starting an attempt specifically.
**Not a violation when:** The project consistently returns `200` with a body for deletes and
documents it.

---

### RS-202 — Everything returns 200, including failures
**Severity:** critical · **Applies to:** controllers, error middleware
**Detect:** `res.status(200).json({ success: false, ... })`; error paths that omit
`res.status(...)` entirely (Express defaults to 200).

```ts
// Bad
res.json({ success: false, message: 'Quiz not found' });    // HTTP 200

// Good
res.status(404).json({ success: false, error: { code: 'QUIZ_NOT_FOUND', message: 'Quiz not found' } });
```

**Why:** Every HTTP client, retry policy, monitor, and load balancer decides on the status
code. A 200-with-error is invisible to all of them.
**Not a violation when:** Never.

---

### RS-203 — Client errors reported as 500
**Severity:** major · **Applies to:** controllers, error middleware
**Detect:** A catch-all that maps every failure to 500; validation failures returning 500;
`next(new AppError(error.message, 500))` wrapping a typed domain error (see `CC-407`).

```ts
// Bad
} catch (error: any) { next(new CustomError(error.message, 500)); }

// Good
} catch (error: unknown) { next(error); }   // middleware maps NotFoundError -> 404, etc.
```

**Why:** 5xx means "the server is broken" and should page someone. Mislabelling a bad request
as 5xx destroys error budgets and hides real incidents.
**Not a violation when:** The failure genuinely is server-side.

---

### RS-204 — 401 and 403 confused
**Severity:** minor · **Applies to:** middleware, controllers
**Detect:** Missing/invalid token returning 403; an authenticated user lacking a role
returning 401.
**Why:** 401 means "authenticate and retry" — it invites the client to refresh a token. 403
means "authenticated, still not allowed" — retrying is pointless. Brief §8.B.3 defines both.
**Not a violation when:** The API deliberately returns 404 instead of 403 to avoid disclosing
that a resource exists — document that choice.

---

### RS-205 — Domain conflicts flattened to 400
**Severity:** minor · **Applies to:** controllers, services
**Detect:** Duplicate email, double submission, or "attempt already in progress" returning
400 rather than 409; expired-timer submissions returning 500.
**Why:** 409 tells the client the request was well-formed but conflicts with current state —
a different remedy than fixing the payload.
**Not a violation when:** The API documents a flat 400 for all client errors and returns a
machine-readable `error.code` that distinguishes them.

---

### RS-206 — Status code magic numbers
**Severity:** minor · **Applies to:** controllers, middleware
**Detect:** Bare numeric literals at call sites where the project has a status constant
module. Grep hint: `res.status(4\d\d)|res.status(5\d\d)|new AppError\([^,]+, *\d{3}\)`
**Why:** Named constants (`statusCode.NotFound`) make a wrong code visible in review.
**Not a violation when:** No constants module exists and the project is consistent.

---

## RS-3xx — Request design

### RS-301 — Input reaches a handler unvalidated
**Severity:** critical · **Applies to:** `*.routes.ts`
**Detect:** A route with a body, params, or query and no validation middleware in the chain.
Pairs with `CC-206`.

```ts
// Bad
router.post('/quizzes/:id/attempts', isAuth, quizController.start);

// Good
router.post('/quizzes/:id/attempts', isAuth, validate({ params: quizIdSchema }), quizController.start);
```

**Why:** Unvalidated `req.body` reaching a Mongoose query is how mass assignment and
NoSQL-operator injection happen.
**Not a violation when:** The handler reads no input.

---

### RS-302 — Only the body is validated
**Severity:** major · **Applies to:** validation middleware
**Detect:** A `validate(schema)` helper that inspects `req.body` alone while routes carry
`:id` params and `?page` queries.

```ts
// Bad
const { error } = schema.validate(req.body);

// Good
export const validate = (schemas: { body?: Schema; params?: Schema; query?: Schema }) => ...
```

**Why:** Path parameters flow straight into `Types.ObjectId(...)` and database filters. An
unvalidated `:id` produces a `CastError` surfacing as 500 instead of 400.
**Not a violation when:** Params and query are validated elsewhere in the chain.

---

### RS-303 — Identity taken from the request instead of the token
**Severity:** critical · **Applies to:** controllers, `*.routes.ts`
**Detect:** `req.body.userId`, `req.params.userId`, or `req.query.userId` used to scope data
that belongs to the authenticated user.

```ts
// Bad
const userId = req.body.user?.id;
const history = await quizService.history(userId);       // any caller can pass another id

// Good
const history = await quizService.history(req.user.id);
```

**Why:** The client controls the body. Trusting it for identity is a horizontal-privilege
escalation — one user reads another's attempts.
**Not a violation when:** An admin endpoint takes a target user id *and* enforces the admin
role.

---

### RS-304 — Collection endpoint without pagination
**Severity:** major · **Applies to:** `*.routes.ts`, controllers
**Detect:** `GET` on a collection returning `find({})` with no `limit`/`skip`, or accepting a
client `limit` with no ceiling. Pairs with `CC-803`.

```ts
// Good
const { page, limit } = parsePagination(req.query);   // limit capped at 100
res.status(200).json({ success: true, data: items, meta: { page, limit, total } });
```

**Why:** An unbounded collection endpoint is a denial of service that requires no attacker.
**Not a violation when:** The collection is bounded by design.

---

### RS-305 — Filtering, sorting, and pagination expressed inconsistently
**Severity:** minor · **Applies to:** controllers
**Detect:** `?page`/`?limit` in one module and `?skip`/`?size` in another; sort passed as a
raw Mongo object in a query string.
**Why:** One convention across the API is the difference between a client writing one helper
and writing one per endpoint. A raw sort object in the query string is also an injection
surface.
**Not a violation when:** A single convention is applied everywhere.

---

### RS-306 — Content type not enforced
**Severity:** minor · **Applies to:** app bootstrap, `*.routes.ts`
**Detect:** No `express.json()` size limit; upload routes with no MIME or size validation;
handlers assuming JSON without checking.

```ts
// Good
app.use(express.json({ limit: '100kb' }));
```

**Why:** An unbounded JSON body is a trivial memory-exhaustion vector.
**Not a violation when:** A gateway enforces the limit upstream and that is documented.

---

## RS-4xx — Response design

### RS-401 — Response envelope not consistent across the API
**Severity:** major · **Applies to:** controllers, error middleware
**Detect:** More than one success shape or more than one error shape in the codebase:
`{status:'SUCCESS', quiz}` beside `{status:'success', data}` beside `{msg, stack}`.

```ts
// Bad — three shapes in one project
res.status(200).json({ status: 'SUCCESS', message: 'ok', quiz });
res.status(200).json({ status: 'success', message: 'ok', data: history });
res.status(500).json({ msg: err.message, stack: err.stack });

// Good — one shape, everywhere
res.status(200).json({ success: true, data: quiz });
res.status(404).json({ success: false, error: { code: 'QUIZ_NOT_FOUND', message: 'Quiz not found' } });
```

**Why:** Clients unwrap responses in one place. Every extra shape is a branch in their code
and a bug when a new endpoint picks a fourth.
**Not a violation when:** A documented exception exists (file download, health probe).

---

### RS-402 — Error response carries no machine-readable code
**Severity:** major · **Applies to:** error middleware, error classes
**Detect:** Error bodies containing only `message`. Brief §8.B.4 requires
`error: { code, message }`.

```ts
// Bad
res.status(400).json({ success: false, message: 'Quiz submission expired (timer exceeded)' });

// Good
res.status(400).json({
  success: false,
  error: { code: 'EXPIRED_TIMER', message: 'Quiz submission expired (timer exceeded)' },
});
```

**Why:** Without a stable code, the client must string-match the message — which then cannot
be reworded or localised.
**Not a violation when:** The status code alone fully identifies the condition.

---

### RS-403 — Internal detail in the response body
**Severity:** critical · **Applies to:** error middleware, controllers
**Detect:** `stack`, the raw error object, driver messages, or Mongoose validation objects in
a response. Pairs with `CC-405`.

```ts
// Bad
res.status(err.statusCode || 400).json({ msg: err.message, cause: err.cause, stack: err.stack, err });

// Good
logger.error({ err, path: req.originalUrl }, 'unhandled');
res.status(500).json({ success: false, error: { code: 'INTERNAL_ERROR', message: 'Internal server error' } });
```

**Why:** Stack traces disclose file paths, dependency versions, and internal structure to an
attacker.
**Not a violation when:** Gated behind `NODE_ENV !== 'production'` with an explicit check.

---

### RS-404 — Persistence document returned as the API contract
**Severity:** major · **Applies to:** controllers, services
**Detect:** A Mongoose document or `.lean()` result passed to `res.json` with no mapper;
`password`, `__v`, OTP fields, or `correctAnswer` reaching a client response. Pairs with
`CC-801`.
**Why:** Schema is not API. Without an explicit mapper, every new field is published by
default — and on this project, the first leak is the answer key or the password hash.
**Not a violation when:** A projection provably excludes the sensitive fields *and* the
response type is explicit.

---

### RS-405 — Correct answers exposed before submission
**Severity:** critical · **Applies to:** `modules/quiz/**`, `modules/attempt/**`
**Detect:** A start-attempt or quiz-detail response that includes `isCorrect`,
`correctAnswer`, or `correctOptionIds`; an admin-shaped payload reused on a student route.

```ts
// Good
questions: questions.map((question) => ({
  id: question._id,
  text: question.text,
  type: question.type,
  options: question.options.map((option) => ({ id: option._id, text: option.text })),
})),
```

**Why:** Project-specific and non-negotiable — brief §4 M3: *"Return questions to the user
without revealing the correct options."* Leaking the answer key defeats the product.
**Not a violation when:** The route is admin-only *and* the authorization is enforced at the
router, and the response type is named for that (`AdminQuizResponse`).

---

### RS-406 — Response shape depends on the code path
**Severity:** minor · **Applies to:** controllers
**Detect:** One endpoint returning a bare array in one branch and `{ data, meta }` in another;
`data` sometimes an object and sometimes a string.
**Why:** Clients type responses once per endpoint. A conditional shape forces defensive
parsing everywhere.
**Not a violation when:** The variants are discriminated and documented.

---

## RS-5xx — Authentication, authorization, transport

### RS-501 — Protected route without an auth guard
**Severity:** critical · **Applies to:** `*.routes.ts`
**Detect:** A route touching user-scoped or admin-scoped data with no auth middleware in its
chain. Compare sibling routes in the same file — one guarded, one not, is the common shape.

```ts
// Bad
router.post('/admin/quizzes', isAuth, controller.create);
router.get('/admin/quizzes', controller.listForAdmin);       // no guard, returns answer keys
router.delete('/admin/quizzes/:id', controller.remove);      // no guard

// Good
const adminOnly = [isAuth, authorize(Role.Admin)];
router.use('/admin', ...adminOnly);
```

**Why:** One forgotten guard is the whole breach. Applying it to the subtree makes it
impossible to add an unprotected sibling by accident.
**Not a violation when:** The endpoint is genuinely public (login, register, catalog).

---

### RS-502 — Authentication without authorization
**Severity:** critical · **Applies to:** `*.routes.ts`, controllers
**Detect:** Admin-only operations guarded by `isAuth` alone, with no role check anywhere.
**Why:** `isAuth` proves who the caller is, not that they may act. Any registered student can
then delete a quiz.
**Not a violation when:** Every authenticated user is genuinely permitted.

---

### RS-503 — Ownership not checked on a scoped resource
**Severity:** critical · **Applies to:** controllers, services
**Detect:** `GET /attempts/:id` or `POST /attempts/:id/submission` that loads by id alone,
with no `userId` in the filter and no comparison afterwards.

```ts
// Bad
const attempt = await this.repo.findById(attemptId);

// Good
const attempt = await this.repo.findOne({ filter: { _id: attemptId, userId } });
if (!attempt) throw new NotFoundError(`attempt ${attemptId}`);
```

**Why:** Authentication says the caller is a valid user; only an ownership check says the
record is theirs. This is the most common real vulnerability in CRUD APIs.
**Not a violation when:** The route is admin-scoped and the role is enforced.

---

### RS-504 — Credentials or tokens in the URL
**Severity:** critical · **Applies to:** `*.routes.ts`, controllers
**Detect:** `?token=`, `?password=`, `?otp=` in query strings or path segments.
**Why:** URLs land in access logs, proxy logs, browser history, and `Referer` headers.
**Not a violation when:** A single-use, short-lived, hashed reset token in a path is
documented and expires quickly — still prefer the body.

---

### RS-505 — Security headers and rate limiting absent
**Severity:** major · **Applies to:** app bootstrap
**Detect:** No `helmet()`; no rate limiter on `/auth/login`, `/auth/register`, or password
reset; `cors()` with no origin allowlist.

```ts
// Good
app.use(helmet());
app.use(cors({ origin: env.CLIENT_URL, credentials: true }));
app.use('/api/v1/auth', rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }), authRouter);
```

**Why:** Unlimited login attempts make password and OTP brute force free. `cors()` with no
argument reflects any origin.
**Not a violation when:** Enforced at a gateway and documented.

---

### RS-506 — Generic auth failures leak account existence
**Severity:** minor · **Applies to:** `modules/auth/**`
**Detect:** Login returning "user not found" vs "wrong password"; forgot-password returning
404 for an unknown email.
**Why:** Distinct messages let an attacker enumerate registered addresses.
**Not a violation when:** The endpoint is admin-only.

---

## RS-6xx — Versioning, headers, documentation

### RS-601 — No API version in the path
**Severity:** minor · **Applies to:** app bootstrap
**Detect:** Routes mounted at `/api/...` with no `/v1` segment.
**Why:** Without a version segment, the first breaking change breaks every client at once.
**Not a violation when:** Versioning is handled by a header and that is documented.

---

### RS-602 — Endpoints undocumented or documentation drifted
**Severity:** minor · **Applies to:** `README.md`, `docs/`, Postman collections
**Detect:** Routes with no entry in the API documentation; documented endpoints that no
longer exist; a documented response shape that disagrees with the controller.
**Why:** Brief §6 makes the endpoint table a deliverable. A drifted table is worse than none —
it is trusted and wrong.
**Not a violation when:** Generated from the code (OpenAPI) and regenerated in CI.

---

### RS-603 — `Location` header missing on creation
**Severity:** minor · **Applies to:** controllers
**Detect:** `201` responses with no `Location` header pointing at the new resource.

```ts
res.status(201).location(`/api/v1/attempts/${attempt.id}`).json({ success: true, data: attempt });
```

**Why:** It tells the client where the resource now lives without inventing the URL.
**Not a violation when:** The body already carries a canonical URL or the project documents
that it does not use `Location`.

---

### RS-604 — Timestamps and durations in ambiguous units
**Severity:** minor · **Applies to:** controllers, DTOs
**Detect:** Fields named `time`, `duration`, or `takenTime` with no unit, or the same concept
in different units across endpoints (minutes here, seconds there, milliseconds elsewhere).

```ts
// Bad
{ time: 30, takenTime: 4 }

// Good
{ durationMinutes: 30, timeSpentSeconds: 247, submittedAt: '2026-08-03T10:14:00.000Z' }
```

**Why:** Brief §4 specifies fastest completion time **in seconds**. A unitless number is a
guess at the client, and rounding to minutes destroys sub-minute values entirely.
**Not a violation when:** The unit is in the field name or the API documentation defines it
globally.
