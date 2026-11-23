# Student Grading & Evaluation Spreadsheet Agent Guide

> **Audience:** Mentors and AI review agents grading cohort projects and updating the `Elevate Node <Cohort> Evaluation` Google Sheet.
> **Companions:** [`standards/README.md`](./standards/README.md) · [`GEMINI.md`](./GEMINI.md) · [`readme-guide.md`](./readme-guide.md)

---

## 1. Overview & Pre-Grading Prerequisites

Never populate or update the Evaluation spreadsheet from guesswork, commit counts, or existing template placeholder numbers. Every grade in the sheet must come from **verified repository evidence** and match the cohort's companion evaluation document (`<cohort>/feedback/students-evaluation.md`).

### Required Inputs Before Starting

| Input | Purpose | Source |
| :--- | :--- | :--- |
| **Master Roster Sheet ID** | Authoritative student list (`Name`, `Email`, `Phone`, `Github handle`, `Team`, `Status`) | `Elevate Node <Cohort> - Students data` |
| **Evaluation Sheet ID** | Target grading workbook (`Project 1..3`, `Daily Meetings`, `Team Balance & Leaderboard`, `Students Sync`) | `Elevate Node <Cohort> Evaluation` |
| **Team Repository URLs** | Source code, branch history, tests, and deliverables to audit | GitHub org repos (`<app-name>-team-<letter>`) |
| **Project Requirements** | Functional rules (`BR-###`, user stories, runtime mandates) | `docs/project-*-*/requirements.md` |
| **Coding Standards** | Technical compliance rules (`CC`, `TS`, `JT`, `RS`, `MA`, and `MS` for Project 3) | [`docs/standards/`](./standards/README.md) |

### Step 1A: Reconcile the Master Roster (`Students Data` → `Students Sync`)

Always read the master roster first and compare it against `'Students Sync'` in the Evaluation workbook:

```bash
gsheets readonly read <STUDENTS_DATA_ID> "'Students Data'!A1:G26"
gsheets readonly read <EVALUATION_ID> "'Students Sync'!A1:G26"
```

1. **Trust `'Students Data'` over `'Teams Assignment'` or stale `'Students Sync'` rows.** Check if `'Teams Assignment'` uses a truncated `IMPORTRANGE` or formula range (for example, omitting the bottom row of `'Students Data'`) and note any mismatch in `students-evaluation.md`.
2. **Sync all real student rows into `'Students Sync'!A5:G24`** (20 template slots total):
   - Write every enrolled student (`Name`, `Email`, `Phone`, `Github handle`, `Team`, `Status`, `Notes`).
   - Clear unused trailing slots (`A<n>:G24`) to `""` so no ghost template names remain.
   - Keep **GitHub handles as plain text** (never `@handle`, never `[handle](https://...)` markdown hyperlinks).
   - **Phone number pitfall:** Prefix any phone number starting with `+` (e.g., `+20 115...`) with a leading single quote (`'+20 115...`) so Google Sheets treats it as text instead of parsing `+20...` as a formula (`#ERROR!`).

---

## 2. Actual-Code Contribution Audit Protocol (Max 70 Points)

**Hard Rule:** Grade contributions strictly by **actual meaningful backend code authored**, **never by commit count**. One squashed PR commit with 900 lines of clean TypeScript and 700 lines of passing unit tests outscores 15 trivial commits or formatting churn.

### Step 2A: Inspect All Branches & Authorship Metrics

Clone each team repository locally and run an authorship audit across `main` (and check remote branches for unmerged work):

```bash
git -C <repo> log --branches --remotes --not main --oneline
git -C <repo> shortlog -sne --all
```

Use `--no-merges --numstat` with a unit-separator (`\x1f`) format to separate backend source TypeScript, backend tests, scripts/config, lockfiles, and out-of-scope frontend directories:

```bash
python3 - << 'EOF'
import subprocess
from collections import defaultdict

repo = "/path/to/team-repo"
log = subprocess.check_output(
    ["git", "-C", repo, "log", "main", "--no-merges", "--numstat", "--format=COMMIT:%H\x1f%an\x1f%ae\x1f%s"],
    text=True,
)
authors = defaultdict(lambda: {"commits": 0, "emails": set(), "src_add": 0, "src_del": 0, "test_add": 0, "test_del": 0, "fe_add": 0, "files": set()})
cur = None
for line in log.splitlines():
    if line.startswith("COMMIT:"):
        _, sha, an, ae, _ = (["COMMIT:"] + line[7:].split("\x1f", 3))
        cur = authors[an]
        cur["commits"] += 1
        cur["emails"].add(ae)
    elif line.strip() and cur is not None:
        parts = line.split("\t")
        if len(parts) != 3:
            continue
        a, d, f = parts
        add, dele = (int(a) if a.isdigit() else 0), (int(d) if d.isdigit() else 0)
        if f.endswith(("package-lock.json", "pnpm-lock.yaml", "yarn.lock")) or "node_modules/" in f:
            continue
        if f.startswith("frontend/"):
            cur["fe_add"] += add
        elif f.endswith((".spec.ts", ".test.ts")):
            cur["test_add"] += add; cur["test_del"] += dele; cur["files"].add(f)
        elif f.endswith(".ts") and ("/src/" in f or f.startswith("src/")):
            cur["src_add"] += add; cur["src_del"] += dele; cur["files"].add(f)

for an, st in sorted(authors.items(), key=lambda kv: -(kv[1]["src_add"] + kv[1]["test_add"])):
    print(f"{an} ({', '.join(sorted(st['emails']))}): commits={st['commits']}, src=+{st['src_add']}/-{st['src_del']}, tests=+{st['test_add']}/-{st['test_del']}, fe=+{st['fe_add']}, files={len(st['files'])}")
EOF
```

### Exclusions & Attribution Rules

- **Always exclude** from code volume: lockfiles, `node_modules/`, `dist/`, `coverage/`, merge commits, and automated formatter-only sweeps.
- **Out-of-scope frontend code** (`frontend/`, React/Vite, CSS): acknowledge in notes if built, but **do not** award backend contribution points for frontend line counts or let it substitute for missing backend modules.
- **Unmapped or alternate Git identities:** Match commit emails/names carefully against `'Students Data'` (students sometimes commit with slightly misspelled GitHub handles or personal emails). If a student has **0 commits across all branches** and no authored files, score them `0`.

### Contribution Scoring Rubric (0–70 Points)

Evaluate each student's authored files directly against the project requirements and the 5 standards catalogs (`CC`, `TS`, `JT`, `RS`, `MA`):

| Weight | Criterion | What to Verify |
| :---: | :--- | :--- |
| **40%** | **Scope & Slice Completeness** | Did the student deliver their entire assigned vertical slice(s) (routes, validation, controller, service, repository, model) or leave endpoints/layers missing? |
| **30%** | **Business Rules & Technical Complexity** | Correctness on `BR-###` / sprint rules, concurrency guards, aggregations, streaming I/O, worker threads, transactions, and security invariants. |
| **30%** | **Code Quality, Modularity & Tests** | Compliance with `CC-###`, `TS-###`, `JT-###`, `RS-###`, `MA-###`; presence and rigor of unit/integration tests on happy paths, error paths, and edge cases. |

| Score Band | Typical Profile |
| :---: | :--- |
| **62–70** | Built shared core/foundation + complex domain module(s), strong test suite, high standards compliance with only minor/localized findings. |
| **52–61** | Delivered full assigned module cleanly with unit tests and sound business logic; a few major/minor architectural or testing gaps. |
| **38–51** | Delivered core slice with noticeable architectural shortcuts, missing edge-case/integration tests, or incomplete sub-features. |
| **15–37** | Minimal or incomplete slice (e.g., skipped repository layer, 0 unit tests, broken route contracts or core business rules). |
| **0** | Non-contributing (`0` real code authored across all branches) or inactive student. |

---

## 3. Deliverables Matrix (Max 35 Points) & Dropped Items

Section 2 of each Project tab (`Rows 29–40`) scores team-level deliverables across 10 standard items (`3.5 pts × 10 = 35.0` by default):

1. `1. GitHub Repository` — Check `<app-name>-team-<letter>` lowercase `kebab-case` naming standard (e.g., `online-exam-team-a`, `exam-app-team-b`), branch/PR hygiene, and `.gitignore`. Deduct `0.5–1.0` for naming standard violations or broken branching.
2. `2. Readme File` — Engineer setup guide, prerequisites, env vars, run/test commands ([`readme-guide.md`](./readme-guide.md)). Do not give full credit for a copied requirements brief or missing root `README.md`.
3. `3. Environment Example` — `.env.example` documenting all required keys with safe placeholders and zero real secrets.
4. `4. API Documentation` — Swagger/OpenAPI, Postman collection, `.http` requests, or complete endpoint contract docs matching implemented routes (`RS-602`).
5. `5. Basic Error Handling` — Central `AppError` / exception filter, status mapping, uniform `{ success, error: { code, message } }` envelope, no stack/driver leaks (`CC-402`, `CC-405`, `RS-401`).
6. `6. Database Schema` — Proper Mongoose/ORM models, compound & partial unique indexes, `select: false` on secrets, migrations/seeds.
7. `7. Project Demo` — Live working demo or walkthrough.
8. `8. Code Organization` — Vertical slices, controller → service → repository separation, `MA-###` module boundaries, linter/static checks.
9. `9. Deployment` — Live deployment link.
10. `10. Docker Setup` — `Dockerfile` & `docker-compose.yml`.

### Re-Weighting When Deliverable Items Are Dropped

When the mentor instructs dropping specific items for a project (for example, dropping **7. Project Demo**, **9. Deployment**, and **10. Docker Setup** in Project 1):

1. Set the dropped rows (`C36:F36`, `C38:F38`, `C39:F39`) to `0` and record `"Dropped for Project <N> — weight redistributed across <K> active items (<35/K> pts each)"` in `Evaluator Notes` (`G`).
2. Re-weight each of the `K` remaining active items equally so their sum equals **35.0 Max** (e.g., `7` active items × `5.0` pts = `35.0` Max).
3. Keep Row 40 (`=SUM(C30:C39)` .. `=SUM(F30:F39)`) intact.
4. Link each student's `Deliverables (Max 35)` cell (`G6:G25`) dynamically to their team's Row 40 total while forcing `0` for non-contributing (`F=0`) or unassigned students:

```excel
=IF(F6=0, 0, IF(D6="Team A", $C$40, IF(D6="Team B", $D$40, IF(D6="Team C", $E$40, IF(D6="Team D", $F$40, 0)))))
```

---

## 4. Attendance (Max 20) & Presentation (Max 25) Rules

- **Attendance (`E6:E25`):**
  - **Standard mode:** Pull from `'Daily Meetings & Sprints'!AB6:AB25` once sprint standups are tracked.
  - **Contributing-student credit mode** (when instructed to award attendance to all contributors): set `20` for every student who contributed real code (`F > 0`), and `0` for non-contributing or inactive students (`F = 0`).
- **Presentation (`H6:H25`) & Dynamic Percentage (`J6:J25`):**
  - If Presentation / Demo is graded, enter `0–25` in `H<r>` and use `=I<r>/150`.
  - If Presentation / Demo is **dropped or pending**, leave `H<r>` **blank** (`""`) and set `J<r>` so active points are divided by **125** (`20 + 70 + 35`) automatically without penalizing students for the dropped 25 points:

```excel
=IF(A6="", "", IF(H6="", I6/125, I6/150))
```

---

## 5. Evaluation Workbook Cell Map & Cleaning Unfinished Tabs

Every Evaluation workbook shares six tabs. Whenever you grade one project, **clean any template placeholder data** on ungraded tabs so no fake numbers or `#DIV/0!` errors pollute the leaderboard.

| Tab | Active Ranges | Cleaning & Formula Rules |
| :--- | :--- | :--- |
| **`Students Sync`** | `A5:G24` | Sync exact rows from `'Students Data'`; clear unused rows `A<n>:G24` to `""`. Escape leading `+` on phones (`'+20...`). |
| **`Project 1/2/3`** | Section 1: `E6:K25`, Avg `E26:K26`<br>Section 2: `C30:G39`, Sum `C40:F40` | Unused student rows (`20–25`): clear `E:H` & `K` to `""`; wrap `I<r>` in `=IF(A<r>="", "", SUM(E<r>:H<r>))` and `J<r>` in `=IF(A<r>="", "", IF(H<r>="", I<r>/125, I<r>/150))`.<br>Row 26 (`Cohort Average`): use `=AVERAGEIFS(E6:E25, $D$6:$D$25, "Team*", $F$6:$F$25, ">0")` on graded projects so `0`-contribution/inactive rows do not skew active cohort means.<br>**Ungraded future project tabs:** clear `E6:H25`, `K6:K25`, `C30:G39` to `""`; guard `I<r>` with `=IF(OR(A<r>="", COUNTA(E<r>:H<r>)=0), "", SUM(E<r>:H<r>))`, `J<r>` with `=IF(I<r>="", "", I<r>/150)`, and Row 26 with `=IFERROR(AVERAGE(<col>6:<col>25), "")`. |
| **`Daily Meetings & Sprints`** | Attendance: `E6:X25`<br>Summary: `Y6:AC26` | When unrecorded, clear fake `1`/`0` cells in `E6:X25` and notes in `AC6:AC25` to `""`. Guard `Y`–`AB`:<br>• `Y<r>`: `=IF(OR(A<r>="", COUNT(E<r>:X<r>)=0), "", SUM(E<r>:X<r>))`<br>• `Z<r>`: `=IF(OR(A<r>="", COUNT(E<r>:X<r>)=0), "", COUNT(E<r>:X<r>))`<br>• `AA<r>`: `=IF(OR(Z<r>="", Z<r>=0), "", Y<r>/Z<r>)`<br>• `AB<r>`: `=IF(AA<r>="", "", ROUND(AA<r>*20, 1))`<br>• `Z26`: `=IF(COUNT(Z6:Z25)=0, "", MAX(Z6:Z25))` (never bare `MAX`, which returns `0` on empty strings). |
| **`Team Balance & Leaderboard`** | Section 1: `C6:J10`<br>Section 2: `A14:L34` | Wrap every `AVERAGEIF` (`C6:E9`) in `=IFERROR(..., "")` so empty teams (`Team C`, `Team D`) do not emit `#DIV/0!`.<br>Compute dynamic active denominator in `G<r>`: `=IF(F<r>="", "", F<r>/(IF(C<r>="",0,125)+IF(D<r>="",0,150)+IF(E<r>="",0,150)))` (adjust `125` vs `150` per project).<br>In Section 2 (`J14:L33`):<br>• `A<r>`: `=IF(OR(B<r>="", J<r>=""), "", RANK(J<r>, $J$14:$J$33))`<br>• `J<r>`: `=IF(B<r>="", "", SUM(D<r>, F<r>, H<r>))`<br>• `K<r>`: `=IF(OR(B<r>="", J<r>=""), "", IFERROR(AVERAGE(E<r>, G<r>, I<r>), ""))`<br>• `L<r>`: `=IF(K<r>="", "", IFS(K<r>>=0.9, "Distinction (Tier 1)", K<r>>=0.8, "Merit (Tier 2)", K<r>>=0.7, "Pass (Tier 3)", TRUE, "Needs Support"))` |

---

## 6. `gsheets` CLI Playbook & Critical Footguns

1. **Always use the `gsheets` CLI** (never web-fetch Google Sheets URLs). Initialize once per environment via the gsheets skill script (`scripts/gsheets --version`), then call bare `gsheets`.
2. **Never run destructive commands** (`delete-sheet`, `delete-rows`, `delete-cols`, or wide `clear`) unless explicitly requested. Write narrow ranges or `""` values to preserve sheet layout, borders, and conditional formatting.
3. **Prefer `gsheets mutate batch -f batch.json`** for multi-range updates and formulas:
   - Each operation object MUST use `"data"` (**never** `"values"`).
   - Example:
     ```json
     [
       { "op": "write", "range": "'Students Sync'!A5:G24", "data": [["..."]] },
       { "op": "write", "range": "'Project 1 (Quiz APP)'!E6:K26", "data": [[20, 59, "..."]] }
     ]
     ```
4. **Comma escaping on inline `mutate write`:** When writing single formulas with `gsheets mutate write`, escape every comma as `\,` (e.g., `'=IF(COUNT(Z6:Z25)=0\, ""\, MAX(Z6:Z25))'`) or use `mutate batch` JSON instead.
5. **Leading `+` escaping:** Phone numbers like `+20 115...` MUST be prefixed with `'` (`'+20 115...`) when written, otherwise Sheets parses them as malformed formulas (`#ERROR!`).
6. **Mandatory post-write verification:** After every `mutate` call, read back all modified tabs with `gsheets readonly read` and confirm **zero** `#ERROR!`, `#DIV/0!`, `#VALUE!`, `#REF!`, or `#N/A` cells and accurate row/team averages.

---

## 7. Companion Markdown Evaluation Report (`<cohort>/feedback/students-evaluation.md`)

Every grading pass must produce or update `<cohort>/feedback/students-evaluation.md` alongside the spreadsheet so scores are permanently auditable:

1. **Evaluation Policy & Adjustments:** Document roster reconciliation notes, dropped deliverables/presentation rules, denominator (`125` vs `150`), and attendance policy.
2. **Cohort Master Scorecard & Ranking:** Table matching the sheet exactly (Rank, Name, plain-text GitHub handle, Git signatures, non-merge commits, backend src/test `+/-` lines, Attendance, Contributions, Deliverables, Presentation if active, Total, Score %, Grade, Status).
3. **Team Deliverables Matrix:** Exact scores per item with concrete evidence and deductions.
4. **Detailed Individual Evaluations:** Grouped by team (`Team A`, `Team B`, ..., `Inactive / Unassigned`), listing owned files/modules, actual-code stats, engineering strengths, and specific rule-ID violations (`CC`/`TS`/`JT`/`RS`/`MA`) on each student's authored code.
