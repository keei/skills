---
name: setup-pre-commit
description: Set up Husky pre-commit hooks with lint-staged (Prettier), type checking, and tests in the current repo. Use when user wants to add pre-commit hooks, set up Husky, configure lint-staged, or add commit-time formatting/typechecking/testing.
---

# Setup Pre-Commit Hooks

## What This Sets Up

- **Husky** pre-commit hook
- **lint-staged** running Prettier on all staged files
- **Prettier** config (if missing)
- **typecheck** and **test** scripts in the pre-commit hook

## Steps

### 1. Detect package manager

Check for `package-lock.json` (npm), `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), `bun.lockb` (bun). Use whichever is present. Default to npm if unclear.

### 2. Install dependencies

Install as devDependencies:

```
husky lint-staged prettier
```

### 3. Initialize Husky

```bash
npx husky init
```

This creates `.husky/` dir and adds `prepare: "husky"` to package.json.

### 4. Create `.husky/pre-commit`

Write this file (no shebang needed for Husky v9+):

```
npx lint-staged
npm run typecheck
npm run test
```

**Adapt**: Replace `npm` with detected package manager. If repo has no `typecheck` or `test` script in package.json, omit those lines and tell the user.

### 5. Create `.lintstagedrc`

```json
{
  "*": "prettier --ignore-unknown --write"
}
```

### 6. Create `.prettierrc` (if missing)

Only create if no Prettier config exists. Use these defaults:

```json
{
  "useTabs": false,
  "tabWidth": 2,
  "printWidth": 80,
  "singleQuote": false,
  "trailingComma": "es5",
  "semi": true,
  "arrowParens": "always"
}
```

### 7. Verify

- [ ] `.husky/pre-commit` exists and is executable
- [ ] `.lintstagedrc` exists
- [ ] `prepare` script in package.json is `"husky"`
- [ ] `prettier` config exists
- [ ] Run `npx lint-staged` to verify it works

### 8. Commit

Stage all changed/created files and commit with message: `Add pre-commit hooks (husky + lint-staged + prettier)`

This will run through the new pre-commit hooks — a good smoke test that everything works.

## Notes

- Husky v9+ doesn't need shebangs in hook files
- `prettier --ignore-unknown` skips files Prettier can't parse (images, etc.)
- The pre-commit runs lint-staged first (fast, staged-only), then full typecheck and tests

## Anvil Verification Loop

All code changes go through this loop. Use `session_store` for all SQL. Never create local DB files.

### Git Hygiene (before every task)

1. Run `git status --porcelain`. If uncommitted changes unrelated to current task exist, surface them and ask: "Commit them now" / "Stash them" / "Ignore and proceed".
2. Run `git rev-parse --abbrev-ref HEAD`. If on `main` or `master` for any meaningful change, ask: "Create branch for me" / "Stay on main" / "I'll handle it".

### Verification Ledger

At the start of every task, generate a `task_id` slug (e.g. `setup-pre-commit-myrepo`). Create the ledger:

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
