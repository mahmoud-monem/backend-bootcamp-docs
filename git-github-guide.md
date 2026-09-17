# Git, GitHub & Version Control — Engineer's Handbook

> **Audience:** backend engineers on the bootcamp projects (`online-exam/team-*`, DOCURA)
> **Read time:** ~45 minutes · keep it open as a reference afterwards
> **Companion:** [`standards/README.md`](./standards/README.md) — the rule catalogs your code is
> reviewed against. This document is about the *workflow* around the code, not the code itself.
> **Not covered:** CI/CD pipelines and GitHub Actions workflows. Everything here is something a
> human does or agrees on.

---

## What this document is for

You already know `git add`, `git commit`, `git push`. That is not the hard part.

The hard part is working on the same codebase as four other people for three months without
producing a history nobody can read, a `main` branch nobody trusts, and merge conflicts that eat
a whole afternoon. That is a set of *habits*, and this document is those habits with the reasoning
attached, so you can tell when a habit applies and when it does not.

**Contents**

| Part | Topic |
| --- | --- |
| 1 | Why any of this matters |
| 2 | The mental model: what git actually stores |
| 3 | Commits — the unit of work |
| 4 | Branching — how work is separated |
| 5 | Keeping a branch healthy — merge, rebase, conflicts |
| 6 | Pull requests — how work is proposed |
| 7 | Code review — how work is accepted |
| 8 | GitHub mechanics — issues, templates, protection |
| 9 | Releases, tags and versions |
| 10 | Secrets and recovery |
| 11 | The whole workflow, end to end |
| 12 | Checklists, common mistakes, exercises |

---

# Part 1 — Why any of this matters

## 1.1 Four ideas that generate everything else

Almost every practice in this document follows from one of these four. When a rule feels arbitrary,
trace it back here.

### History is documentation

In six months, the Slack thread is gone, the person who wrote the code has moved to another team,
and the ticket says "fix the timer thing". What remains is `git log`, `git blame`, and the pull
request discussion. Those three are your only record of *why the code is the way it is*.

This is why commit messages matter more than they seem to. You are not writing them for your
reviewer today. You are writing them for whoever is debugging at 2am next March — quite possibly you.

### Small and often beats big and rare

Everything gets worse faster than linearly with size:

| If you double… | This gets… |
| --- | --- |
| Commit size | Harder to revert cleanly, harder to bisect |
| Branch lifetime | Exponentially more conflict-prone |
| Pull request size | Reviewed *worse*, not just slower — reviewers start skimming |

A 400-line PR gets real review. A 2,000-line PR gets "LGTM". The second one is not faster; it is
review theatre.

### `main` is always releasable

Anything merged into `main` is something you are willing to ship. Not "ship eventually" —
ship now, today, without checking with anyone.

This one sounds strict and is actually liberating: if `main` is always good, you can cut a release
at any moment, roll back to any commit, and onboard someone by telling them "clone `main`, it works".
The cost is that unfinished work has to live somewhere else — behind a feature flag, or on a branch.

### Shared history is immutable

Once a commit exists on a branch that someone else has pulled, you do not rewrite it. You add a
new commit that corrects it.

Rewriting means every teammate's local copy now points at commits that no longer exist. Their next
push either fails confusingly or *resurrects the old history*. This is the single most common way
a team loses work.

## 1.2 What "good" looks like

A healthy repository, described from the outside:

- `git log --oneline` on `main` reads like a changelog. One line per change, each meaningful.
- No commit on `main` is broken. You can check out any of them and the tests pass.
- Every change traces back to an issue that says what it was supposed to do.
- Branches live for hours or days, never weeks.
- Nobody is afraid to pull.

---

# Part 2 — The mental model: what git actually stores

Most git confusion comes from a wrong mental model, so it is worth thirty seconds of precision.

## 2.1 A commit is a snapshot, not a diff

A commit stores the *complete state* of the tree, plus a pointer to its parent(s), plus author,
date and message. Git shows you diffs, but it stores snapshots. This is why checking out an old
commit is instant and why commits are cheap.

```text
A ──► B ──► C          each arrow points to the PARENT
                       C knows about B; B does not know about C
```

Because a commit is identified by the SHA-1/SHA-256 hash of its content *and its parent*, changing
anything about a commit changes its ID — and the ID of every commit after it. That is the whole
reason rewriting shared history is dangerous. It is not a policy choice; it is arithmetic.

## 2.2 A branch is a movable label

A branch is a 40-character file containing a commit ID. That is all. `main` is not a container of
commits; it is a sticky note that says "the latest commit is C".

```text
A ──► B ──► C ◄── main
             ╲
              D ──► E ◄── feat/142-quiz-timer
```

Consequences worth internalising:

- Creating a branch costs nothing. Create them freely.
- Deleting a branch deletes a label, not commits.
- "The branch is 3 commits behind `main`" just means the two labels point at different places.

## 2.3 The three places your work lives

```text
working tree  ──git add──►  staging area  ──git commit──►  repository
(your files)                (index)                        (history)
```

`git status` tells you where everything currently is. `git add -p` (patch mode) lets you stage
*part* of a file — this is the tool that makes atomic commits possible when you accidentally did
two things at once.

## 2.4 Local and remote are separate repositories

`origin/main` is your *local cache* of where `main` was on the server the last time you fetched.
It is not live.

| Command | Does |
| --- | --- |
| `git fetch` | Update your cache of the remote. Touches nothing else. **Always safe.** |
| `git pull` | `fetch` + merge (or rebase) into your current branch. Changes your files. |
| `git push` | Send your commits to the remote. |

When something looks wrong, `git fetch` first. It cannot hurt you, and half the time the confusion
was a stale cache.

---

# Part 3 — Commits: the unit of work

## 3.1 What makes a commit atomic

**A commit is atomic when reverting it removes exactly one idea and nothing else.**

That is the whole test. Apply it before you commit.

```text
Not atomic — one commit, four ideas
  "quiz module"   (+2,400 / -900 across 38 files)
  contains: a new feature, a rename, a formatting pass, a dependency bump

Atomic — four commits, four ideas
  feat(quiz): add Quiz and Question schemas
  feat(quiz): add QuizRepository with findById and create
  feat(quiz): expose POST /quizzes
  test(quiz): cover quiz creation validation
```

Why bother? Four concrete payoffs:

1. **Review.** Your reviewer reads four small, coherent stories instead of one confusing one.
2. **Revert.** If the route is wrong but the schema is fine, you revert one commit.
3. **Bisect.** `git bisect` finds the commit that broke something. With atomic commits it hands you
   a 20-line suspect. With one giant commit it hands you the whole feature.
4. **Blame.** `git blame` on a line gives you the commit whose message explains *that line*.

Two things that break atomicity most often, and their fixes:

| Problem | Fix |
| --- | --- |
| You changed two unrelated things in one file | `git add -p` — stage only the hunks for the first idea |
| You already committed a mess | `git rebase -i` and split it, before you push |

## 3.2 The commit message

We use **Conventional Commits**. The format is not decoration — it is what lets tools derive
version numbers and changelogs from history, and it is what makes `git log --oneline` scannable.

```text
<type>(<scope>)<!>: <subject>

<body — why, not what>

<footer — BREAKING CHANGE:, Refs: #123, Co-Authored-By:>
```

| Type | Use for |
| --- | --- |
| `feat` | New user-visible capability |
| `fix` | Bug fix |
| `refactor` | Behaviour-preserving restructure |
| `perf` | Performance change, behaviour preserved |
| `test` | Adding or fixing tests only |
| `docs` | Documentation only |
| `style` | Formatting only, no change in meaning |
| `build` | Build system, dependencies, tooling |
| `chore` | Housekeeping with no `src/` impact |
| `revert` | Reverting a previous commit |

The `!` and the `BREAKING CHANGE:` footer both mean the same thing: this change breaks callers.

### The subject line

Three rules, all mechanical:

- **Imperative mood.** The subject completes *"Applied, this commit will …"*.
  `add`, `fix`, `remove` — not `added`, `fixing`, `removes`.
- **72 characters or fewer.** Longer gets truncated everywhere it is displayed.
- **No trailing period.** It is a title, not a sentence.

```text
Bad:  Added validation and also fixed the controller and renamed things.
Bad:  update stuff
Bad:  WIP
Good: feat(auth): validate refresh tokens against the revocation list
```

If you cannot write a subject under 72 characters without the word "and", your commit is not atomic.
The message is telling you something about the commit — listen to it.

### The body: why, not what

The diff already shows *what* changed. Nobody needs it restated in prose. What the diff cannot
show is the reasoning: the bug report, the constraint, the approach you tried first and abandoned.

```text
fix(quiz): expire submissions using server clock, not client timestamp

Clients could submit after the deadline by sending a stale `submittedAt`.
The timer is now derived from `attempt.startedAt + quiz.durationMinutes`
and compared against `Date.now()` on the server.

Rejected alternative: signing the client timestamp. It still trusts the
client's clock, and the signature adds a key to rotate for no benefit.

Refs: #142
```

That body took ninety seconds to write and will save someone an hour.

## 3.3 What never belongs in a commit

- **Commented-out code.** Git is the archive. Deleted code is recoverable by anyone who wants it;
  commented code rots silently and misleads readers who assume it is there for a reason.
- **Debug leftovers.** `console.log('here')`, `debugger`, `.only(` in a spec file.
  `.only` is the dangerous one — it silently disables every other test in the file.
- **Formatting mixed with logic.** A 3-line fix inside a 400-line reformat is unreviewable.
  Commit the reformat separately (`style:`) so reviewers can skip it.
- **Generated files.** `dist/`, `coverage/`, `node_modules/`. They create phantom conflicts and
  bury the real diff.
- **Secrets.** See [Part 10](#part-10--secrets-and-recovery). This one is not a style issue.

Noise commits (`wip`, `oops`, `fix typo`, `address review`) are **fine while you are working** —
that is what a private branch is for. They just must not survive into `main`, which is handled
either by an interactive rebase before review, or automatically by squash-merge (Part 5).

## 3.4 Repository hygiene

Set these up once, at the start of the project.

```gitignore
# .gitignore
node_modules/
dist/
coverage/
*.tsbuildinfo
.env
.env.*
!.env.example
.DS_Store
```

```gitattributes
# .gitattributes — prevents "every line changed" diffs across Windows/macOS/Linux
* text=auto eol=lf
*.png binary
```

```bash
# Make sure your commits are attributed to you, not to `root` or a hostname
git config user.name  "Your Name"
git config user.email "you@example.com"
```

Also commit a `.env.example` with every variable your code reads and **no real values**.
A new machine must be runnable without asking a teammate which variables exist.

```bash
# .env.example
PORT=3000
MONGO_URI=mongodb://localhost:27017/exam
JWT_SECRET=replace-me
JWT_EXPIRES_IN=15m
```

If generated files are *already* tracked, adding them to `.gitignore` is not enough — git keeps
tracking what it already tracks:

```bash
git rm -r --cached node_modules dist coverage
git commit -m "chore: stop tracking generated files"
```

---

# Part 4 — Branching: how work is separated

## 4.1 The strategies, compared

You will meet all four of these in the wild. Knowing why each exists is more useful than
memorising one.

| Strategy | Shape | Fits when | Costs you |
| --- | --- | --- | --- |
| **Trunk-based** | `main` + short-lived `feat/*`, merged in 1–3 days | Small team, one deployable version, continuous delivery | Requires discipline: small slices, feature flags |
| **GitHub Flow** | `main` + feature branches, deploy on merge | Web services deployed straight from `main` | No support for maintaining old versions |
| **Git Flow** | `main` + `develop` + `release/*` + `hotfix/*` | Installed/versioned software with several supported versions in the field | Heavy. `develop` drifts from `main`; integration is slow |
| **Release branches** | `main` + `release/x.y` cut at feature freeze | You must patch old versions | Back-porting every fix to every supported branch |

**What we use on these projects: trunk-based with short-lived branches.**

`main` is the single source of truth. Every branch is one issue wide and lives less than three days.
`release/*` exists only when a version genuinely has to be frozen. We choose it because it *punishes
large batches* — which is the exact habit the rest of this document is trying to build. Git Flow
would let a `develop` branch quietly accumulate a month of unintegrated work, and that month of
conflict is the thing we are trying to avoid.

A note on a pattern you may have seen: **permanent `staging` / `qa` / `production` branches that
get merged into each other to "promote" code**. Avoid it. The same change ends up existing as
several different commits, cherry-picks drift apart, and "what exactly is on staging?" becomes
unanswerable. An environment should point at a *tag or a commit*, not at a branch.

## 4.2 Branch naming

```text
<type>/<issue-id>-<short-kebab-summary>

feat/142-quiz-timer-expiry
fix/158-refresh-token-revocation
refactor/171-extract-attempt-repository
docs/180-readme-quick-start
chore/190-bump-mongoose-8
hotfix/201-null-pointer-on-grade
```

Same `<type>` vocabulary as commits. Two payoffs: `git branch -a` becomes a readable work queue,
and the issue number links the code back to the requirement that motivated it forever.

Branches named `mo`, `test2`, `new`, `dev-final-final` tell nobody anything, including you, in two days.

### Hyphen or slash before the summary?

You will see both of these in the wild, and both are legitimate:

```text
feat/142-quiz-timer-expiry      one slash  — type is the namespace
feat/142/quiz-timer-expiry      two slashes — type and issue are both namespaces
```

Git refs are paths, so any depth is allowed. The difference is mostly cosmetic, but there are
two real mechanics worth knowing before your team picks one.

**The cost of the extra slash.** A git ref cannot be both a file and a directory, so a branch
name can never be a path prefix of another branch name:

```text
$ git branch feat/142/quiz-timer && git branch feat/142
fatal: cannot lock ref 'refs/heads/feat/142':
       'refs/heads/feat/142/quiz-timer' exists; cannot create 'refs/heads/feat/142'
```

So `feat/142/…` permanently reserves `feat/142` as a namespace — you can never have a plain
`feat/142` branch in that repository. That is rarely something you want, so the cost is small.
Note this is not a slash-versus-hyphen problem: it is symmetric, and `feat/158-x` equally blocks
`feat/158-x/more`. It is a reason to be **consistent across the team**, because mixing the two
forms on the same issue number is what actually produces this error.

**The gain of the extra slash.** It groups several branches under one issue, which is exactly the
stacked-branch case in [4.5](#45-stacked-branches-for-genuinely-large-work):

```text
feat/171/repository
feat/171/service
feat/171/routes
```

**Arguments that do not actually differentiate the two**, despite being the ones people usually
give:

| Claim | Reality |
| --- | --- |
| "Slashes let you glob by issue" | `git branch --list 'feat/142-*'` works exactly as well as `'feat/142/*'` |
| "Slashes break Docker tags / k8s labels" | Those cannot contain `/` at all, so both forms get flattened — and to the *same* string, `feat-142-quiz-timer` |
| "Slashes break the GitHub UI / push / delete / completion" | They do not. All identical |

Two genuine minor annoyances with deeper paths: `git worktree add ../$branch` creates nested
directories, and a few terminal tools display only the last path segment, which drops the issue
number out of view.

**What we use, and why:** `<type>/<issue-id>-<short-kebab-summary>`. Because the one-branch-per-issue
rule ([4.3](#43-one-branch-one-concern-three-days)) means the issue level normally has exactly one
child, so the second namespace groups nothing. Use `<type>/<issue-id>/<part>` for a stacked chain,
where there really are several branches under one issue. Either way, **pick one per repository and
stick to it** — consistency matters more here than the choice does.

## 4.3 One branch, one concern, three days

**Rule of thumb: if a branch is older than ~3 working days or larger than ~400 lines of production
code, it has become a problem regardless of how good the code is.**

Divergence cost is not linear. A three-day branch merges cleanly. A three-week branch means every
file you touched was also touched by someone else, and you now get to resolve conflicts in code
you have forgotten.

"But the feature is big" — then split it *vertically*, not horizontally. Not "all the schemas,
then all the services, then all the routes" (nothing works until the last one lands), but a thin
slice that works end to end, then another:

```text
Bad split (horizontal)              Good split (vertical)
  1. all 6 Mongoose schemas           1. create + read one quiz, end to end
  2. all 6 repositories               2. add questions to a quiz
  3. all 6 services                   3. start an attempt
  4. all the routes                    4. submit and grade an attempt
  nothing usable until step 4          something usable after step 1
```

## 4.4 Feature flags: how to merge unfinished work safely

When a feature genuinely cannot ship in three days, ship it **disabled** instead of leaving it
unmerged. The code integrates continuously; the behaviour stays off.

```ts
// Merged early, inert in production, zero divergence cost
if (features.isEnabled('adaptive-grading')) {
  return this.adaptiveGrader.grade(attempt);
}
return this.legacyGrader.grade(attempt);
```

Delete the flag and the dead branch of the `if` as soon as the feature is on everywhere. A flag
that lives forever is just a long-lived branch that learned to hide inside `main`.

## 4.5 Stacked branches for genuinely large work

When one change really needs 1,500 lines, build a chain where each branch is based on the previous
one and each pull request is reviewable alone:

```text
main
 └── feat/171-attempt-repository        PR #1 — data access
      └── feat/172-attempt-service      PR #2 — domain rules, based on #1
           └── feat/173-attempt-routes  PR #3 — HTTP layer, based on #2
```

If the whole stack belongs to **one** issue rather than three, this is the case where the
two-slash naming form earns its keep ([4.2](#hyphen-or-slash-before-the-summary)) — the issue
number becomes the namespace and the stack reads as one unit:

```text
main
 └── feat/171/repository        PR #1 — data access
      └── feat/171/service      PR #2 — domain rules, based on #1
           └── feat/171/routes  PR #3 — HTTP layer, based on #2
```

Merge bottom-up, rebasing the rest of the stack after each merge. Say in every PR description
which PR it is based on, or your reviewer will think PR #2 contains PR #1's changes as well.

## 4.6 Cleaning up

```bash
git branch -r --merged origin/main   # what is safe to delete
git fetch --prune                    # drop local refs to deleted remote branches
```

Turn on "automatically delete head branches" in the repository settings and this mostly takes care
of itself. Deleting a merged branch loses nothing — the commits are in `main`.

---

# Part 5 — Keeping a branch healthy: merge, rebase, conflicts

## 5.1 Merge and rebase do the same job differently

Both integrate `main` into your branch. They differ in what history they leave behind.

```text
Starting point
  A ── B ── C ◄── main
        ╲
         D ── E ◄── feat/142

git merge main                        git rebase main
  A ── B ── C ──── M ◄── feat/142       A ── B ── C ── D' ── E' ◄── feat/142
        ╲         ╱
         D ── E ──                      D and E are REWRITTEN as D' and E'
                                        (new SHAs — new commits)
  Nothing rewritten. Extra commit.      Linear. History rewritten.
```

**Merge** preserves history exactly and adds a merge commit. **Rebase** replays your commits on
top of the new base, producing a straight line — and new commit IDs.

## 5.2 The golden rule of rebase

> **Rebase only commits that are yours alone. Never rebase a branch someone else is working from.**

Rebase gives every replayed commit a new ID. Anyone who had the old commits now holds a branch
pointing at commits that no longer exist on the remote. Their next `git pull` either fails in a
confusing way or merges the old and new history together, duplicating everything.

So, in practice:

| Situation | Do this |
| --- | --- |
| Updating **your own** branch from `main` | `git rebase origin/main` — keeps the PR diff clean |
| Updating a branch **two people share** | `git merge origin/main` |
| Integrating a finished branch into `main` | Through a pull request, never locally |

## 5.3 Merge strategy at the pull request

GitHub offers three merge buttons. Pick per branch type, write the choice down, and stop
re-deciding it:

| Merging… | Use | Result on `main` |
| --- | --- | --- |
| `feat/*`, `fix/*`, `chore/*` | **Squash and merge** | One clean commit per PR |
| `release/*`, `hotfix/*` | **Merge commit** (`--no-ff`) | Release boundaries stay visible |

With squash-merge, **the PR title becomes the commit message on `main`** — so the PR title must be
a valid Conventional Commit subject. This is also why `wip` commits inside your branch are
harmless: they never reach `main`.

## 5.4 Conflicts

A conflict means two branches changed the same lines. Git is not broken and you did nothing wrong.

```bash
git fetch origin
git rebase origin/main
# CONFLICT (content): Merge conflict in src/modules/quiz/quiz.service.ts

# 1. Open the file. Resolve by READING BOTH SIDES and writing what is correct now.
# 2. Stage the resolved file
git add src/modules/quiz/quiz.service.ts
# 3. Continue
git rebase --continue

# Lost? This always gets you back to before you started:
git rebase --abort
```

**The dangerous shortcut to avoid:** `git checkout --ours` / `--theirs` on a whole file, or
"resolving" by deleting the other side. That silently reverts a teammate's work, and it is
invisible in review because the diff just looks like your version of the file.

Resolve hunk by hunk. Re-run the tests after a non-trivial resolution. Mention in the PR that you
resolved a conflict in file X and how.

One config worth setting once — it remembers how you resolved a conflict and reapplies it if the
same conflict appears again during a long rebase:

```bash
git config --global rerere.enabled true
```

## 5.5 Force-push: when, and how

Sometimes you legitimately need to rewrite your own published branch (you rebased, or squashed
your noise commits). Do it safely:

```bash
# Bad — overwrites whatever is on the remote, including a teammate's push you never saw
git push --force

# Good — refuses if the remote moved since your last fetch
git push --force-with-lease
```

`--force-with-lease` is the same command with a seatbelt. Make it your reflex; there is no case
where you want the version without the check.

Never force-push `main` or `release/*`. Branch protection should make that impossible — see
[Part 8](#part-8--github-mechanics).

## 5.6 Undoing things on a shared branch

> **Warning:** `reset --hard` discards commits and uncommitted work, and combined with a
> force-push it rewrites history other people already have. On a shared branch, use `revert`.

```bash
# On a SHARED branch (main, release/*): add a commit that undoes the change
git revert <sha>

# On YOUR OWN unpublished branch: rewriting is fine
git reset --hard HEAD~1
```

`revert` is not a defeat. A revert commit in the history is an honest record that something was
tried and withdrawn, which is exactly what future readers need.

---

# Part 6 — Pull requests: how work is proposed

## 6.1 Size is the single biggest factor in review quality

Target **under ~400 changed lines of production code and under ~20 files**, excluding lockfiles
and generated code.

This is not bureaucracy. Past roughly that size, reviewers stop reading and start scanning, and a
scanned PR is an unreviewed PR that has an approval on it. If your PR is bigger, split it
(Part 4.3) or stack it (Part 4.5).

## 6.2 The description is the PR

Your reviewer has no context. They do not know the bug report, the constraint, or the two
approaches you rejected. The description is where you hand them all of that so they can review the
*decision*, not just the syntax.

Commit this as `.github/pull_request_template.md` and it appears pre-filled in every new PR:

```markdown
## What
One paragraph: what this changes, in user or API terms.

## Why
The problem, bug report, or requirement. Link the issue: Closes #142

## How
Notable design decisions, and anything a reviewer would otherwise have to
reverse-engineer. Alternatives considered and why they were rejected.

## Testing
How you verified it. Commands run, cases covered, what is intentionally
not covered and why.

## Risk / rollback
Migrations, breaking changes, feature flags, how to undo this.

## Screenshots / API samples
Request and response bodies for new or changed endpoints.
```

An empty description, or the template with the sections still unfilled, means the reviewer has to
ask you everything in comments — which costs a day of round-trips for something that took you two
minutes to write down.

Always link the issue (`Closes #142`). It is the only durable connection between the code and the
requirement it was supposed to satisfy.

## 6.3 Draft PRs early, ready PRs late

Open a **draft** pull request as soon as you have a direction. It shows your teammates what you
are doing, invites course correction while changing course is still cheap, and gives you a place
to think out loud.

Mark it **Ready for review** only when you believe it is mergeable. "Ready" is a claim, and
reviewers allocate real time based on it.

## 6.4 Review your own PR first

Before you request review: open your own PR on GitHub and read every line of the diff as if
someone else wrote it.

You will find debug logs, a stray file, a rename you meant to undo, a function you left half
refactored. Reviewers find these too — but each one costs a full round-trip. Finding them yourself
costs five minutes.

While you are there, leave comments on the parts you already know will raise questions:
*"this cast is needed because the Mongoose types for `findOneAndUpdate` are wrong before v9"*.
You are steering the review toward what actually matters.

## 6.5 Things that must be called out explicitly

A reviewer can miss these in a diff, and missing them is expensive:

- **Breaking changes** — renamed or removed fields, changed status codes, changed response shape.
  Use a `BREAKING CHANGE:` footer and say it in the description.
- **New or changed environment variables** — update `.env.example` and the README in the same PR.
- **Migrations** — and how to roll them back.
- **New dependencies** — say why, in one sentence. Dependencies are the most common supply-chain
  entry point and the least-read part of any diff.

---

# Part 7 — Code review: how work is accepted

Review is the highest-leverage hour in your week. It is where defects are cheapest to fix and
where the team's shared understanding of the codebase actually gets built.

## 7.1 As the reviewer

**Read in this order.** It stops you from nit-picking the naming of a function that should not exist.

1. **The description and the linked issue.** What is this supposed to do?
2. **The tests.** What does the author believe the behaviour is? Would these tests fail if the
   implementation were wrong?
3. **The structure.** Are the responsibilities in the right layer? Does anything import across a
   boundary it should not? (The [`clean-code.md`](./standards/clean-code.md) catalog is the reference.)
4. **The details.** Naming, edge cases, error handling, async correctness.

**Say what kind of comment you are making.** Without this the author cannot tell what blocks the
merge, and reviewers end up accidentally blocking on taste:

| Prefix | Means | Blocks merge? |
| --- | --- | --- |
| `blocking:` | Correctness, security, data loss, broken layering | Yes |
| `question:` | I do not understand this; the answer may or may not change code | Until answered |
| `suggestion:` | A better approach — author's call | No |
| `nit:` | Style or naming preference | No |
| `praise:` | This is good; keep doing it | No |

`praise:` is not filler. Review that only ever finds fault teaches people to fear review.

**Comment on the code, not the person.** State the problem, then propose a fix or ask a question:

```text
Bad:  You clearly didn't think about concurrency here.

Good: blocking: two concurrent submissions can both pass this check —
      `findOne` followed by `save` is not atomic. `findOneAndUpdate` with a
      filter on `status: 'in_progress'` would make it a single operation.
```

**Approving means something.** It means *"I understand this change and I accept it in `main`"*.
You co-own it from that moment. A one-minute approval on a 600-line PR is not politeness, it is
a transfer of risk to whoever debugs it later.

**Resolve your own threads.** The reviewer who opened a conversation decides whether the reply
settled it — not the author.

**Start within one working day.** A blocked teammate outranks your own feature work. If you cannot
review, say so immediately so someone else picks it up.

## 7.2 As the author

- **Answer every comment**, even the ones you disagree with. "Good point, but X, because Y" is a
  complete answer. Silence reads as agreement or as dismissal, and neither is what you meant.
- **Push new commits during review.** Do not force-push — it destroys the "changes since your last
  review" diff and your reviewer has to start over. Squash-merge cleans it all up at the end anyway.
- **Disagreement is normal.** If a reviewer is wrong, say so with reasoning. If you are both sure,
  escalate to a third person rather than trading comments for a day.
- **Pushback is about the code.** A reviewer finding a bug in your PR is the system working exactly
  as designed.

## 7.3 Critical paths need two reviewers

Authentication, authorization, payments, grading, and data migrations get **two approvals**.
These are the places where a bug is not a bug report, it is an incident. `CODEOWNERS`
([Part 8.3](#83-codeowners)) can enforce this automatically.

---

# Part 8 — GitHub mechanics

## 8.1 Issues: where the requirement lives

The issue holds the *requirement*; the pull request holds the *implementation*. If the issue is
missing, there is no way to judge whether the PR is correct — only whether it is tidy.

A usable issue has testable acceptance criteria:

```markdown
## Context
Students can submit an attempt after the timer expires by replaying an old request.

## Acceptance criteria
- [ ] Submission after `startedAt + durationMinutes` returns 409 `EXPIRED_TIMER`
- [ ] Expiry is computed from the server clock only
- [ ] Unit test covers the boundary: exactly at the deadline is accepted
```

Those three checkboxes are also your definition of done, your test list, and your PR description's
"What" section. Writing them costs five minutes and saves an argument.

## 8.2 Templates and labels

Commit these under `.github/`:

```text
.github/
  pull_request_template.md
  ISSUE_TEMPLATE/
    bug_report.md
    feature_request.md
```

Keep labels few and meaningful: `bug`, `feature`, `chore`, `blocked`, `needs-discussion`,
`good-first-issue`. Twenty labels means nobody uses any of them.

## 8.3 `CODEOWNERS`

```text
# .github/CODEOWNERS
*                       @team-leads
/src/modules/auth/      @security-reviewers
/docs/standards/        @team-leads
```

GitHub automatically requests review from the owner of every path you touched. It routes review to
people with context, instead of relying on someone remembering who knows the auth module.

## 8.4 Branch protection on `main`

Configure this on day one, before there is anything to protect. Minimum:

- Require a pull request before merging — at least **1 approval** (2 on critical paths).
- **Dismiss stale approvals** when new commits are pushed.
- **Require conversation resolution** before merging.
- **Block force-pushes and deletions** on `main` and `release/*`.
- Require linear history (if you chose squash-merge).
- **Apply the rules to administrators too** — a rule you can click past is a suggestion.

Without this, "we always use PRs" lasts until the first Friday afternoon.

---

# Part 9 — Releases, tags and versions

## 9.1 Tag every release

```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0
```

Use `-a` (annotated): it stores a tagger, a date and a message, and it is what GitHub Releases
builds on. A tag is the only stable name for "what is in production right now" — branches move,
tags do not.

## 9.2 Semantic versioning, and why commit types matter

`MAJOR.MINOR.PATCH`:

| Bump | When | Triggered by |
| --- | --- | --- |
| MAJOR | Breaking change to your public interface | `feat!:` or a `BREAKING CHANGE:` footer |
| MINOR | New, backward-compatible capability | `feat:` |
| PATCH | Backward-compatible bug fix | `fix:` |

This is the payoff for Conventional Commits: the version bump and the changelog are *derivable*
from the history. Nobody has to remember what changed.

## 9.3 Changelog

Keep a `CHANGELOG.md`, grouped by `Added` / `Changed` / `Fixed` / `Removed` / `Security`, or
generate it from your commit types. Either way it is committed and reviewable, so a wrong entry
can be caught in review like any other mistake.

## 9.4 Hotfixes

```text
1. Branch from the affected TAG, not from main:  git switch -c hotfix/201-grade-npe v1.4.0
2. Fix. Add a test that fails without the fix.
3. PR, review, merge into the release branch. Tag v1.4.1.
4. MERGE THE FIX BACK INTO main.
```

Step 4 is the one people forget, and forgetting it means the next release silently reintroduces
the bug you just fixed in production.

---

# Part 10 — Secrets and recovery

## 10.1 A committed secret is a leaked secret

Git history is permanent. Every clone, every fork and every CI cache has a copy. Deleting the file
in a later commit changes nothing — the old commit still contains the value, and anyone can read it.

```ts
// Never
export const MONGO_URI = 'mongodb+srv://admin:S3cret!@cluster0.mongodb.net/exam';

// Always
export const MONGO_URI = requireEnv('MONGO_URI');
```

> **Security warning — the order of these steps matters.** Purging history *before* rotating the
> credential leaves a live secret in every existing clone, fork and cache, while giving everyone
> the false impression that the problem is handled.
>
> If you push a secret:
>
> 1. **Rotate the credential immediately.** Treat it as compromised from the moment it was pushed.
> 2. **Revoke the old value** at the provider so the leaked copy is dead.
> 3. Only then remove it from history — `git filter-repo --invert-paths --path .env` — and ask
>    GitHub Support to purge cached views.
> 4. Force-push, having told the whole team first. Everyone re-clones.
> 5. Add the path to `.gitignore` and enable secret scanning so it cannot recur.
>
> Steps 3–5 are hygiene. Step 1 is the fix.

Enable GitHub's secret scanning and push protection on the repository. It catches the common token
formats before the push lands.

## 10.2 Almost nothing is ever lost

Git keeps things far longer than people expect. Learn these four and stop re-cloning in a panic:

```bash
git reflog                              # every position HEAD has had — recover "deleted" commits
git restore --source=<sha> -- <path>    # bring one file back from any commit
git revert <sha>                        # undo a public commit safely (Part 5.6)
git bisect start / bad / good           # binary-search for the commit that introduced a bug
```

`git reflog` is the one that saves you. A "lost" commit after a bad reset or rebase is almost
always sitting in the reflog, and `git switch -c recovered <sha>` brings it back.

## 10.3 Local settings worth doing once

```bash
git config --global pull.ff only        # never create an accidental merge commit on pull
git config --global push.default current
git config --global rerere.enabled true # remember conflict resolutions
git config --global alias.lg "log --oneline --graph --decorate --all"
```

A `commit-msg` hook (for example `commitlint` with `husky`) checks your commit format locally,
before the message ever becomes history. It is a tool on your machine, not a pipeline, so it fits
here: the earlier a mistake is caught, the cheaper it is.

---

# Part 11 — The whole workflow, end to end

This is the loop. Everything in the previous ten parts exists to keep this loop cheap.

```bash
# 1 — Start from a fresh main
git switch main
git pull --ff-only origin main

# 2 — One branch per issue, named by convention (Part 4.2)
git switch -c feat/142-quiz-timer-expiry

# 3 — Commit small and often; imperative, typed subject (Part 3)
git add src/modules/quiz/quiz.service.ts src/modules/quiz/quiz.service.spec.ts
git commit -m "fix(quiz): expire submissions using the server clock"

# 4 — Stay close to main while you work (Part 5.2)
git fetch origin
git rebase origin/main          # your branch, still private — rebase is safe here

# 5 — Publish
git push -u origin feat/142-quiz-timer-expiry

# 6 — Open a DRAFT pull request early. Fill the template (Part 6.2).
#     Self-review the diff (Part 6.4). Mark ready when you believe it is mergeable.

# 7 — Answer review with NEW commits, never a force-push (Part 7.2)
git commit -m "fix(quiz): use findOneAndUpdate to close the submission race"
git push

# 8 — Squash-merge from the GitHub UI (Part 5.3).
#     The PR title becomes the commit on main — make it a valid Conventional Commit.

# 9 — Clean up
git switch main && git pull --ff-only origin main
git branch -d feat/142-quiz-timer-expiry
git fetch --prune
```

**Definition of done for a branch.** All of these, not most of them:

- [ ] Every acceptance criterion on the issue is met.
- [ ] Tests exist for the new behaviour and pass locally.
- [ ] The diff contains nothing unrelated to the issue.
- [ ] The PR description explains *why*, not just *what*.
- [ ] Docs, `.env.example` and the API reference are updated in this same PR.
- [ ] Every review thread is resolved by the person who opened it.

---

# Part 12 — Checklists, mistakes, exercises

## 12.1 Before every commit

- [ ] Does this commit contain exactly one idea?
- [ ] Can I write the subject in under 72 characters without "and"?
- [ ] Is the type (`feat`/`fix`/`refactor`/…) correct?
- [ ] Does the body explain *why*?
- [ ] Any `console.log`, `debugger`, `.only(`, commented-out code, or `.env` in this diff?

## 12.2 Before requesting review

- [ ] Under ~400 lines of production code?
- [ ] Rebased on current `main`?
- [ ] Description filled in, issue linked?
- [ ] I have read my own diff line by line?
- [ ] Tests for the new behaviour exist?
- [ ] Breaking changes, new env vars and migrations called out?

## 12.3 The mistakes that cost the most

| Mistake | What it costs | Instead |
| --- | --- | --- |
| Committing `.env` or a token | Credential rotation, possibly an incident | `.env.example`; rotate first if it happens (Part 10.1) |
| A three-week branch | Days of conflict resolution in forgotten code | Vertical slices, feature flags (Part 4.3–4.4) |
| `git push --force` on a shared branch | Teammates lose commits | `--force-with-lease`, and never on `main` (Part 5.5) |
| Committing `node_modules` / `dist` | Unreviewable diffs, phantom conflicts | `.gitignore` + `git rm -r --cached` (Part 3.4) |
| "LGTM" on a 900-line PR | The defect ships, with your name on the approval | Split the PR; review properly (Part 7.1) |
| Resolving a conflict by taking your whole file | You silently reverted someone's work | Resolve hunk by hunk, re-run tests (Part 5.4) |
| No issue, no acceptance criteria | Nobody can tell whether the PR is correct | Write the criteria first (Part 8.1) |
| Hotfix merged to the release branch only | The bug returns in the next release | Merge back into `main` (Part 9.4) |
| Docs updated "in a follow-up ticket" | The docs are now wrong, indefinitely | Same PR as the code |

## 12.4 Exercises on your own project repository

1. Run `git log --oneline -30` on your team's repo. How many subjects tell you what changed without
   opening the diff? That percentage is your history's real quality score.
2. Find your largest merged PR. Count the changed lines. Write down how you would have split it
   into three vertical slices.
3. Take a commit that does two things and split it: `git rebase -i`, `edit`, `git reset HEAD~`,
   then stage and commit the two halves separately with `git add -p`.
4. Deliberately create a conflict: change the same line on two branches, then rebase one onto the
   other and resolve it. Do it once on purpose so the first real one is not stressful.
5. Configure branch protection on `main` with the settings in Part 8.4 and confirm a direct push
   is rejected.
6. Add `.github/pull_request_template.md` and `.github/ISSUE_TEMPLATE/bug_report.md` to your repo
   and open one issue and one PR through them.

## 12.5 Agree these as a team, and write them in `CONTRIBUTING.md`

Deciding once removes a recurring argument. Put the answers in `CONTRIBUTING.md` and link it from
your README:

- Branching strategy and branch naming pattern.
- Merge strategy per branch type, and **who clicks merge** (recommended: the author, after approval —
  they are the one who can react if something breaks).
- Required approvals, and which paths need two.
- Comment severity prefixes (Part 7.1).
- Expected review response time, and what to do when the reviewer is unavailable.
- What "done" means for a branch (Part 11).
