# TypeScript Rule Catalog (`TS-###`)

Applies to all TypeScript in the repo — `src/**`, `*.config.ts`, `server.ts`, and test
files (test-specific allowances are called out per rule).

Read [`README.md`](./README.md) for severity meanings, the verify-before-report protocol,
and the false-positive rules.

**Bands**

| Band | Topic |
| --- | --- |
| `TS-0xx` | Compiler configuration and project setup |
| `TS-1xx` | Type escapes (`any`, assertions, `!`, `@ts-ignore`) |
| `TS-2xx` | Modelling types |
| `TS-3xx` | Function signatures and generics |
| `TS-4xx` | `null` / `undefined` handling |
| `TS-5xx` | Express, Mongoose, and library typing |
| `TS-6xx` | Modules, imports, and naming conventions |

---

## TS-0xx — Compiler configuration

### TS-001 — `strict` not enabled
**Severity:** critical · **Applies to:** `tsconfig.json`
**Detect:** `"strict"` missing or `false`, or individual strict flags disabled
(`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`).

```jsonc
// Bad
{ "compilerOptions": { "target": "ES2022", "module": "NodeNext" } }

// Good
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

**Why:** Without `strict`, TypeScript is a syntax dialect, not a type system —
`undefined` flows freely and every rule below becomes unenforceable.
**Not a violation when:** Never for `src/**`. A relaxed override scoped to a migration
folder is acceptable if it is documented and shrinking.

---

### TS-002 — Missing `noUncheckedIndexedAccess`
**Severity:** major · **Applies to:** `tsconfig.json`
**Detect:** Flag absent while the code indexes arrays/records and dereferences the result
directly (`req.params.id.trim()`, `rows[0].name`).
**Why:** Without it, `arr[0]` is typed as present even when the array is empty — the single
most common source of runtime `undefined` in this codebase's shape of code.
**Not a violation when:** Enabling it is queued as tracked work and `src/**` currently
guards indexed reads explicitly.

---

### TS-003 — Type errors suppressed at the build level
**Severity:** critical · **Applies to:** `package.json`, `tsconfig.json`, CI config
**Detect:** `transpile-only`, `--transpileOnly`, `isolatedModules` used to skip checking,
`skipLibCheck` masking real project errors, or no `type-check` script anywhere.

```jsonc
// Bad
"dev": "ts-node-dev --transpile-only src/index.ts"   // and no separate type-check step

// Good
"dev": "ts-node-dev --respawn --transpile-only src/index.ts",
"type-check": "tsc --noEmit",
"prebuild": "npm run type-check"
```

**Why:** `--transpile-only` in dev is fine for speed; the violation is having *no* place
where the compiler actually runs. Then type errors ship.
**Not a violation when:** A `tsc --noEmit` script exists and CI or the build runs it.

---

### TS-004 — Compiler target or module resolution mismatched with runtime
**Severity:** major · **Applies to:** `tsconfig.json`
**Detect:** `"module": "CommonJS"` with `.js` ESM-style import specifiers; `"NodeNext"`
without extensions on relative imports; `"target"` below `ES2020` on a modern Node runtime.
**Why:** Mismatch produces errors that only appear at runtime, usually as
`ERR_MODULE_NOT_FOUND` after a successful build.
**Not a violation when:** A bundler or loader demonstrably reconciles the difference.

---

## TS-1xx — Type escapes

### TS-101 — `any` on a boundary
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `: any` on an exported signature, an interface field, a middleware-augmented
request, or a shared util. Grep hint: `:\s*any\b|<any>|as any|\bany\[\]`
Escalate to **critical** on an auth/authorization boundary.

```ts
// Bad
declare global { namespace Express { interface Request { user?: any } } }

// Good
export interface AuthenticatedUser {
  id: string;
  email: string;
  role: Role;
}
declare global { namespace Express { interface Request { user?: AuthenticatedUser } } }
```

**Why:** One `any` on `req.user` disables checking for every downstream consumer — the
type system stops warning exactly where the security decisions are made.
**Not a violation when:** It is a genuinely dynamic third-party payload with no shape —
then use `unknown` and narrow (see TS-102).

---

### TS-102 — `unknown` not used where the shape is unproven
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `any` used for JSON payloads, `catch (error: any)`, or parsed external data.

```ts
// Bad
catch (error: any) { res.status(error.status || 500).json({ message: error.message }); }

// Good
catch (error: unknown) { next(error); }        // typed AppError narrowing happens in one place

// or, when handling locally:
catch (error: unknown) {
  if (error instanceof AppError) throw error;
  throw new InternalError("unexpected failure", { cause: error });
}
```

**Why:** `unknown` forces a narrowing step, which is the check `any` skipped.
**Not a violation when:** The value is immediately passed through untouched.

---

### TS-103 — Implicit `any` from an untyped parameter or callback
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** Callback parameters annotated `any` because the source is untyped:
`(q: any) => ...`, `(a: any) => ...` inside `.map`/`.filter`. Almost always caused by an
untyped upstream query result.

```ts
// Bad
const snapshot = questions.map((q: any) => ({ id: q._id, text: q.text }));

// Good
const questions: Question[] = await this.questionRepository.findByQuizId(quizId);
const snapshot = questions.map((question) => ({ id: question.id, text: question.text }));
```

**Why:** The `any` on the callback is a symptom — the real defect is an untyped repository
return. Fix the source and the callbacks type themselves.
**Not a violation when:** The upstream type is genuinely unavailable and the callback
narrows immediately.

---

### TS-104 — Type assertion used to silence the compiler
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `as SomeType` on a value the compiler cannot verify; `as unknown as T` in
production code; `<T>value` casts. Grep hint: `as unknown as|\bas [A-Z]\w+`

```ts
// Bad
const payload = jwt.verify(token, env.JWT_SECRET) as AuthenticatedUser;

// Good
const decoded = jwt.verify(token, env.JWT_SECRET);
const payload = accessTokenPayloadSchema.parse(decoded);   // validates, then types
```

**Why:** An assertion is a promise to the compiler with nothing behind it. If the promise
is wrong, the failure lands far from the cast.
**Not a violation when:** Narrowing a known literal (`as const`), or in test files when
building a partial mock (see `JT-206`).

---

### TS-105 — Non-null assertion (`!`)
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `value!`, `obj!.field`, `map.get(k)!`. Grep hint: `!\.\w|!\)|!;|!,`
Escalate to **critical** when the asserted value came from a database lookup.

```ts
// Bad
const user = await this.userRepository.findByEmail(email);
user!.forgetPasswordOtp = otp;
await user!.save();

// Good
const user = await this.userRepository.findByEmail(email);
if (!user) throw new NotFoundError(`user ${email}`);
user.forgetPasswordOtp = otp;
```

**Why:** `!` deletes exactly the check `strictNullChecks` exists to demand. A missing row
becomes `Cannot read properties of null` in production instead of a 404.
**Not a violation when:** Existence is guaranteed structurally one or two lines earlier
(`map.set(k, v)` immediately before `map.get(k)!`), or in `env`-validated config where
absence throws at boot.

---

### TS-106 — `@ts-ignore` / `@ts-expect-error` / `eslint-disable` without a reason
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** Any suppression comment with no explanation on the same or previous line.

```ts
// Bad
// @ts-ignore
req.user = decoded;

// Good
// @ts-expect-error upstream @types/multer omits the `size` field added in 1.4.5 (see #318)
file.size;
```

**Why:** An unexplained suppression is permanent — nobody can safely remove it later.
Prefer `@ts-expect-error`: it fails the build once the underlying issue is fixed.
**Not a violation when:** A reason and, ideally, an issue link are present.

---

### TS-107 — `Function`, `Object`, `{}`, or `object` as a type
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Those exact annotations. Grep hint: `:\s*(Function|Object|\{\})\b`

```ts
// Bad
function withRetry(operation: Function, attempts: number) { ... }

// Good
function withRetry<T>(operation: () => Promise<T>, attempts: number): Promise<T> { ... }
```

**Why:** These accept nearly everything and describe nothing — an `any` in disguise.
**Not a violation when:** `object` is used precisely to exclude primitives.

---

## TS-2xx — Modelling types

### TS-201 — Domain shape declared inline and repeatedly
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** The same inline object type written in two or more signatures.

```ts
// Bad
function create(input: { title: string; diplomaId: string; durationMinutes: number }) {}
function update(id: string, input: { title: string; diplomaId: string; durationMinutes: number }) {}

// Good
export interface QuizInput { title: string; diplomaId: string; durationMinutes: number }
function create(input: QuizInput): Promise<Quiz>;
function update(id: string, input: Partial<QuizInput>): Promise<Quiz>;
```

**Why:** A named type is the vocabulary of the module; inline shapes drift apart silently.
**Not a violation when:** The shape is used once and is local to that function.

---

### TS-202 — Types duplicated instead of derived
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** A hand-written type that restates a Zod/Joi schema, a Mongoose schema, or
another type, with fields kept in sync by hand.

```ts
// Bad
export const createQuizSchema = z.object({ title: z.string(), durationMinutes: z.number() });
export interface CreateQuizDto { title: string; durationMinutes: number }   // drifts

// Good
export const createQuizSchema = z.object({ title: z.string(), durationMinutes: z.number() });
export type CreateQuizDto = z.infer<typeof createQuizSchema>;
```

**Why:** One source of truth. Derived types cannot drift; hand-copied ones always do.
**Not a violation when:** The API type deliberately differs from the persistence type —
that divergence is intentional and should be explicit (see `CC-801`).

---

### TS-203 — String literals where a union or enum belongs
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `status: string`, `role: string` on domain types; comparisons to bare literals
scattered across modules.

```ts
// Bad
interface Attempt { status: string }
if (attempt.status === "in_progress") ...     // "inprogress" typo compiles fine

// Good
export const AttemptStatus = { InProgress: "in_progress", Submitted: "submitted", Graded: "graded" } as const;
export type AttemptStatus = (typeof AttemptStatus)[keyof typeof AttemptStatus];
interface Attempt { status: AttemptStatus }
```

**Why:** A union makes typos compile errors and gives exhaustiveness checking for free.
Prefer a `const` object + derived union over a numeric `enum`: it has no runtime surprises
and serialises as the literal string.
**Not a violation when:** The value is genuinely open-ended free text.

---

### TS-204 — Union not discriminated
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A union of object types with no shared literal tag, forcing callers to probe
for fields (`if ("error" in result)`).

```ts
// Bad
type GradeResult = { score: number } | { error: string };

// Good
type GradeResult =
  | { kind: "graded"; score: number }
  | { kind: "rejected"; reason: RejectionReason };

switch (result.kind) {
  case "graded": return result.score;
  case "rejected": return handleRejection(result.reason);
}
```

**Why:** A discriminant makes narrowing mechanical and enables the exhaustiveness check in
`TS-205`.
**Not a violation when:** The union is of primitives or the branches are structurally
disjoint in a way the compiler already narrows.

---

### TS-205 — No exhaustiveness check on a closed union
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `switch` over a union with a `default` that silently returns, and no `never`
assertion.

```ts
// Bad
switch (role) {
  case Role.Student: return studentView();
  case Role.Admin: return adminView();
  default: return studentView();          // a new Role silently behaves as a student
}

// Good
switch (role) {
  case Role.Student: return studentView();
  case Role.Admin: return adminView();
  default: {
    const unhandled: never = role;
    throw new Error(`unhandled role: ${String(unhandled)}`);
  }
}
```

**Why:** The compiler tells you every place to update when the union grows — the whole
point of modelling it as a union.
**Not a violation when:** The default is a deliberate, documented fallback for an
open-ended external value.

---

### TS-206 — Mutable where it should be readonly
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Exported config objects, constant arrays, and DTO types that callers mutate;
missing `as const` on literal maps.

```ts
// Bad
export const ROLES = ["student", "admin"];             // string[], mutable, no literals

// Good
export const ROLES = ["student", "admin"] as const;    // readonly ["student", "admin"]
export type Role = (typeof ROLES)[number];
```

**Why:** `as const` gives literal types *and* immutability in one keyword; mutable shared
constants are an action-at-a-distance bug waiting to happen.
**Not a violation when:** The collection is genuinely a mutable buffer.

---

### TS-207 — Optional and nullable used interchangeably
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `field?: string | null` where only one of the two states is real; a schema with
`default: null` typed as optional.
**Why:** Three states (`present`, `undefined`, `null`) where the domain has two forces every
consumer to handle a case that never occurs.
**Not a violation when:** Both states are meaningful and documented ("never set" vs
"explicitly cleared").

---

## TS-3xx — Function signatures and generics

### TS-301 — Exported function without an explicit return type
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Exported functions, service methods, and repository methods relying on
inference. Grep hint: `export (async )?function \w+\([^)]*\)\s*\{`

```ts
// Bad
export async function getQuizForStudent(id: string) { ... }

// Good
export async function getQuizForStudent(id: string): Promise<StudentQuizView> { ... }
```

**Why:** An explicit return type is a contract: it catches accidental widening (a leaked
`any`, a stray `undefined` branch) at the definition instead of at some distant call site.
**Not a violation when:** The function is a short local arrow or an inline callback.

---

### TS-302 — Async function not returning a `Promise` type
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `async` functions annotated with a non-`Promise` type, or `Promise<any>`.
**Why:** `Promise<any>` erases the awaited type everywhere the result flows.
**Not a violation when:** `Promise<void>` — that is correct and should be explicit.

---

### TS-303 — Overly clever generics
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Type parameters used once; conditional/mapped types that a plain interface
would express; generic wrappers with a single concrete instantiation.

```ts
// Bad
function findOne<T extends Record<string, unknown>, K extends keyof T>(model: Model<T>, key: K, value: T[K]): Promise<T | null>

// Good
findByEmail(email: string): Promise<User | null>
```

**Why:** A type parameter used once is a `any` with ceremony. Generics should carry a
relationship between inputs and outputs.
**Not a violation when:** The generic genuinely relates parameters to the return type
(`function first<T>(items: T[]): T | undefined`).

---

### TS-304 — Public signature typed to the persistence layer
**Severity:** major · **Applies to:** `src/**/*.service.ts`, shared types
**Detect:** Service or controller signatures using `HydratedDocument<T>`, `Document`,
`ObjectId`, or a Mongoose model type.

```ts
// Bad
async getProfile(id: ObjectId): Promise<HydratedDocument<IUser>>

// Good
async getProfile(id: string): Promise<UserProfile>
```

**Why:** The document type carries Mongoose methods, `__v`, and mutability into layers that
should never see them — and makes those layers untestable without Mongoose.
**Not a violation when:** The file is the repository or a mapper.

---

### TS-305 — Overloads or optional parameters used to fake two functions
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Multiple optional parameters where certain combinations are invalid and only a
comment says so.
**Why:** Make illegal states unrepresentable — two named functions, or a discriminated
input union, beat a signature that documents its own invalid combinations.
**Not a violation when:** The optionality is genuinely orthogonal.

---

## TS-4xx — `null` / `undefined`

### TS-401 — Possibly-absent value dereferenced without narrowing
**Severity:** critical · **Applies to:** `src/**/*.ts`
**Detect:** A `T | null` / `T | undefined` used directly, typically enabled by `!` (TS-105)
or an assertion (TS-104). Also `req.params.id` and `req.query.x` used without a check when
`noUncheckedIndexedAccess` is off.

```ts
// Bad
const quiz = await this.quizRepository.findById(req.params.id as string);
return quiz.questions.length;

// Good
const quizId = req.params.id;
if (!quizId) throw new BadRequestError("quiz id is required");
const quiz = await this.quizRepository.findById(quizId);
if (!quiz) throw new NotFoundError(`quiz ${quizId}`);
return quiz.questions.length;
```

**Why:** This is the single most common runtime crash in Express + Mongoose services.
**Not a violation when:** Narrowing happens earlier in the same scope in a way the compiler
accepts.

---

### TS-402 — `||` used for defaulting where `??` is meant
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `||` defaulting on values that can legitimately be `0`, `""`, or `false`.

```ts
// Bad
const limit = Number(req.query.limit) || 10;    // ok here, but:
const showAnswers = options.showAnswers || true;  // false is silently overridden

// Good
const showAnswers = options.showAnswers ?? true;
```

**Why:** `||` triggers on every falsy value, not just absence — the bug appears only for
`0`/`""`/`false` and is easy to miss in review.
**Not a violation when:** Falsy values genuinely should take the default.

---

### TS-403 — Optional chaining used to hide a real absence bug
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `?.` chains ending in a silently-undefined expression on a path where absence
is an error (`user?.email` when a user must exist).
**Why:** `?.` converts a crash into a silent wrong value — worse, not better, when the
value is required.
**Not a violation when:** Absence is legitimately expected and handled right after.

---

## TS-5xx — Express, Mongoose, and library typing

### TS-501 — `Request` augmented ad hoc per file
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** `(req as any).user`, `req as Request & { user: ... }`, or a locally redefined
`AuthRequest` interface repeated in several modules.

```ts
// Bad
const userId = (req as any).user.id;

// Good — src/types/express.d.ts, declared once
declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;
      validated?: { body: unknown; query: unknown; params: unknown };
    }
  }
}
export {};
```

**Why:** One augmentation keeps the auth contract in one place; per-file casts let the two
definitions drift and defeat the check where it matters most.
**Not a violation when:** The project uses an explicit `AuthenticatedRequest` type applied
consistently after the auth middleware.

---

### TS-502 — Mongoose schema and TypeScript interface out of sync
**Severity:** major · **Applies to:** `src/**/models/**`, `src/**/*.model.ts`
**Detect:** `new Schema({...})` with no type parameter, or an interface listing fields the
schema lacks (or vice versa).

```ts
// Bad
const userSchema = new Schema({ email: String, password: String, role: String });
export const UserModel = model("User", userSchema);   // Document<any>

// Good
export interface UserDocument {
  email: string;
  password: string;
  role: Role;
  isVerified: boolean;
}
const userSchema = new Schema<UserDocument>({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ROLES, default: "student" },
  isVerified: { type: Boolean, default: false },
});
export const UserModel = model<UserDocument>("User", userSchema);
```

**Why:** An untyped model returns `any` from every query, which is where most `any` in this
codebase originates (see TS-103).
**Not a violation when:** The model file derives its interface from the schema via
`InferSchemaType`.

---

### TS-503 — Untyped `.lean()`, `.aggregate()`, or raw driver result
**Severity:** major · **Applies to:** `src/**/*.repository.ts`, `src/**/*.service.ts`
**Detect:** `.aggregate(` or `.lean()` whose result is used without a type parameter or a
mapper.

```ts
// Bad
const rows = await AttemptModel.aggregate([...]);
return rows.map((r: any) => ({ studentId: r._id, average: r.avg }));

// Good
interface AttemptAverageRow { _id: string; avg: number }
const rows = await AttemptModel.aggregate<AttemptAverageRow>([...]);
return rows.map((row) => ({ studentId: row._id, averageScore: row.avg }));
```

**Why:** Aggregation output is where the type system loses the thread; annotate it at the
boundary or every downstream field access is unchecked.
**Not a violation when:** The result is passed straight to a schema parser.

---

### TS-504 — Environment configuration untyped or unvalidated
**Severity:** major · **Applies to:** `src/config/**`
**Detect:** `process.env.X!` or `process.env.X as string` in the config module; no schema
validating required variables at startup. Pairs with `CC-701`.

```ts
// Bad
export const JWT_SECRET = process.env.JWT_SECRET!;

// Good
const envSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
  PORT: z.coerce.number().default(3000),
  MONGO_URI: z.string().url(),
  JWT_SECRET: z.string().min(32),
});
export const env = envSchema.parse(process.env);      // fails loudly at boot
export type Env = z.infer<typeof envSchema>;
```

**Why:** `!` on env turns a missing variable into an undefined secret — JWTs then sign with
`undefined`, which does not throw until much later.
**Not a violation when:** Never for required variables.

---

### TS-505 — Missing or wrong `@types` for a runtime dependency
**Severity:** minor · **Applies to:** `package.json`
**Detect:** A runtime dependency used in TS with no bundled types and no `@types/*`
counterpart; `@types` packages listed under `dependencies`; type packages whose major
version disagrees with the runtime package.
**Why:** Missing types silently degrade a whole import to `any`; mismatched versions
describe an API that is not there.
**Not a violation when:** The package ships its own types.

---

### TS-506 — Two libraries doing one job
**Severity:** minor · **Applies to:** `package.json`, `src/**/*.ts`
**Detect:** Both `bcrypt` and `bcryptjs`; both `joi` and `zod`; two HTTP clients; two
loggers. Grep the import sites to confirm both are actually used.
**Why:** Two validation stacks means two definitions of "valid", and reviewers must know
which one guards a given route.
**Not a violation when:** A migration is in progress and tracked.

---

## TS-6xx — Modules, imports, conventions

### TS-601 — Deep relative import chains
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `../../../` or deeper. Grep hint: `from "(\.\./){3,}`

```ts
// Bad
import { AppError } from "../../../common/errors/app-error.js";

// Good
import { AppError } from "@common/errors/app-error.js";   // paths configured in tsconfig
```

**Why:** Deep chains break on every file move and hide the real dependency direction.
**Not a violation when:** The project has no path aliases configured — then report once at
the config level, not per import.

---

### TS-602 — Inconsistent module syntax
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** `require()` mixed with `import` in the same project; `export default` in some
modules and named exports in others with no rule; missing `.js` extensions under
`module: NodeNext`.
**Why:** Mixed syntax breaks tree-shaking, confuses mocking in Jest, and produces
ESM/CJS interop errors at runtime.
**Not a violation when:** A config file must be CommonJS by tooling requirement.

---

### TS-603 — Type-only import not marked
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Importing a symbol used only in type position without `import type`, when the
project otherwise uses `import type`.
**Why:** Under `isolatedModules`/`verbatimModuleSyntax`, unmarked type imports emit real
runtime imports and can create import cycles.
**Not a violation when:** The project does not use these flags and is consistent.

---

### TS-604 — Circular import between modules
**Severity:** major · **Applies to:** `src/**/*.ts`
**Detect:** A imports B and B imports A, directly or through a barrel `index.ts`.
**Why:** Cycles produce `undefined` at import time — a class that is defined when you read
it and `undefined` when the process starts.
**Not a violation when:** The cycle is type-only and marked with `import type`.

---

### TS-605 — Naming conventions not followed
**Severity:** minor · **Applies to:** `src/**/*.ts`
**Detect:** Deviations from the project convention: `PascalCase` for types/classes/enums,
`camelCase` for values and functions, `UPPER_SNAKE_CASE` for module-level constants,
`kebab-case.ts` (or the project's consistent alternative) for filenames.
**Why:** Consistent casing is how a reader tells a type from a value at a glance without
jumping to the definition.
**Not a violation when:** The identifier mirrors an external API's casing.

---

### TS-606 — Barrel file re-exporting everything
**Severity:** minor · **Applies to:** `src/**/index.ts`
**Detect:** `export * from "./..."` across a whole module, especially one that also holds
side-effectful initialisation.
**Why:** Wildcard barrels create import cycles, defeat tree-shaking, and make every
consumer look like it depends on the entire module.
**Not a violation when:** The barrel is a small, curated public API of named exports.
