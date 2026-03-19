---
name: scaffold-exercises
description: Create exercise directory structures with sections, problems, solutions, and explainers that pass linting. Use when user wants to scaffold exercises, create exercise stubs, or set up a new course section.
---

# Scaffold Exercises

Create exercise directory structures that pass `pnpm ai-hero-cli internal lint`, then commit with `git commit`.

## Directory naming

- **Sections**: `XX-section-name/` inside `exercises/` (e.g., `01-retrieval-skill-building`)
- **Exercises**: `XX.YY-exercise-name/` inside a section (e.g., `01.03-retrieval-with-bm25`)
- Section number = `XX`, exercise number = `XX.YY`
- Names are dash-case (lowercase, hyphens)

## Exercise variants

Each exercise needs at least one of these subfolders:

- `problem/` - student workspace with TODOs
- `solution/` - reference implementation
- `explainer/` - conceptual material, no TODOs

When stubbing, default to `explainer/` unless the plan specifies otherwise.

## Required files

Each subfolder (`problem/`, `solution/`, `explainer/`) needs a `readme.md` that:

- Is **not empty** (must have real content, even a single title line works)
- Has no broken links

When stubbing, create a minimal readme with a title and a description:

```md
# Exercise Title

Description here
```

If the subfolder has code, it also needs a `main.ts` (>1 line). But for stubs, a readme-only exercise is fine.

## Workflow

1. **Parse the plan** - extract section names, exercise names, and variant types
2. **Create directories** - `mkdir -p` for each path
3. **Create stub readmes** - one `readme.md` per variant folder with a title
4. **Run lint** - `pnpm ai-hero-cli internal lint` to validate
5. **Fix any errors** - iterate until lint passes

## Lint rules summary

The linter (`pnpm ai-hero-cli internal lint`) checks:

- Each exercise has subfolders (`problem/`, `solution/`, `explainer/`)
- At least one of `problem/`, `explainer/`, or `explainer.1/` exists
- `readme.md` exists and is non-empty in the primary subfolder
- No `.gitkeep` files
- No `speaker-notes.md` files
- No broken links in readmes
- No `pnpm run exercise` commands in readmes
- `main.ts` required per subfolder unless it's readme-only

## Moving/renaming exercises

When renumbering or moving exercises:

1. Use `git mv` (not `mv`) to rename directories - preserves git history
2. Update the numeric prefix to maintain order
3. Re-run lint after moves

Example:

```bash
git mv exercises/01-retrieval/01.03-embeddings exercises/01-retrieval/01.04-embeddings
```

## Example: stubbing from a plan

Given a plan like:

```
Section 05: Memory Skill Building
- 05.01 Introduction to Memory
- 05.02 Short-term Memory (explainer + problem + solution)
- 05.03 Long-term Memory
```

Create:

```bash
mkdir -p exercises/05-memory-skill-building/05.01-introduction-to-memory/explainer
mkdir -p exercises/05-memory-skill-building/05.02-short-term-memory/{explainer,problem,solution}
mkdir -p exercises/05-memory-skill-building/05.03-long-term-memory/explainer
```

Then create readme stubs:

```
exercises/05-memory-skill-building/05.01-introduction-to-memory/explainer/readme.md -> "# Introduction to Memory"
exercises/05-memory-skill-building/05.02-short-term-memory/explainer/readme.md -> "# Short-term Memory"
exercises/05-memory-skill-building/05.02-short-term-memory/problem/readme.md -> "# Short-term Memory"
exercises/05-memory-skill-building/05.02-short-term-memory/solution/readme.md -> "# Short-term Memory"
exercises/05-memory-skill-building/05.03-long-term-memory/explainer/readme.md -> "# Long-term Memory"
```

## Anvil Verification Loop

All code changes go through this loop. Use `session_store` for all SQL. Never create local DB files.

### Git Hygiene (before every task)

1. Run `git status --porcelain`. If uncommitted changes unrelated to current task exist, surface them and ask: "Commit them now" / "Stash them" / "Ignore and proceed".
2. Run `git rev-parse --abbrev-ref HEAD`. If on `main` or `master` for any meaningful change, ask: "Create branch for me" / "Stay on main" / "I'll handle it".

### Verification Ledger

At the start of every task, generate a `task_id` slug (e.g. `scaffold-exercises-memory`). Create the ledger:

```sql
CREATE TABLE IF NOT EXISTS anvil_checks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL,
    phase TEXT NOT NULL CHECK(phase IN ('baseline', 'after', 'review')),
    check_name TEXT NOT NULL,
    tool TEXT NOT NULL,
    command TEXT,
    exit_code INTEGER,
    output_snippet TEXT,
    passed INTEGER NOT NULL CHECK(passed IN (0, 1)),
    ts DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Rule: Every verification step must be an INSERT. If the INSERT didn't happen, the verification didn't happen.**

### Recall (before planning)

Query session history for files you're about to change:

```sql
-- database: session_store
SELECT s.id, s.summary, s.branch, sf.file_path, s.created_at
FROM session_files sf JOIN sessions s ON sf.session_id = s.id
WHERE sf.file_path LIKE '%{filename}%' AND sf.tool_name = 'edit'
ORDER BY s.created_at DESC LIMIT 5;
```

### Baseline Capture (before changing code)

Run IDE diagnostics and build/test baseline. INSERT all results with `phase = 'baseline'`.

### Verification (after changes)

Run all applicable tiers:

- **Tier 1**: IDE diagnostics (`ide-get_diagnostics`), syntax/parse check
- **Tier 2**: Build, type checker, linter, tests (detect from ecosystem files)
- **Tier 3** (if no runtime signal from Tier 2): import/load test or smoke script

INSERT every result with `phase = 'after'`.

### Adversarial Review

**Medium tasks** (no 🔴 files): One code-review subagent with model `gpt-5.3-codex`.

**Large tasks or 🔴 files** (auth/crypto/payments/data deletion/schema migration): Three reviewers in parallel:
- `gpt-5.3-codex`
- `gemini-3-pro-preview`
- `claude-opus-4.6`

INSERT each verdict with `phase = 'review'` and `check_name = 'review-{model_name}'`.

Risk classification:
- 🟢 New tests, docs, config, additive changes
- 🟡 Modifying business logic, function signatures, DB queries
- 🔴 Auth/crypto/payments, data deletion, schema migrations, public API surface

### Evidence Bundle

Present after all verification INSERTs are complete:

```
## 🔨 Evidence Bundle

**Task**: {task_id} | **Size**: S/M/L | **Risk**: 🟢/🟡/🔴

### Baseline
| Check | Result | Command | Detail |

### Verification
| Check | Result | Command | Detail |

### Regressions
{Checks that went passed=1 → passed=0. If none: "None detected."}

### Adversarial Review
| Model | Verdict | Findings |

**Changes**: [each file and what changed]
**Confidence**: High / Medium / Low
**Rollback**: `git checkout HEAD -- {files}`
```

### Learn (store_memory)

After verification:
1. Working build/test command discovered → `store_memory` immediately
2. Codebase pattern found in existing code → `store_memory`
3. Reviewer caught something your verification missed → `store_memory`
4. Fixed a regression → `store_memory` (file + what went wrong)

### Commit (after presenting)

1. `git rev-parse HEAD` → store as `{pre_sha}`
2. `git add -A`
3. Commit with concise subject + body. Include trailer: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`
4. Tell user: `✅ Committed on \`{branch}\`: {short_message}` and rollback command.

### Documentation Lookup

When unsure about a library/framework:
1. `context7-resolve-library-id` with library name
2. `context7-query-docs` with resolved ID and your question

Do this BEFORE guessing at API usage.
