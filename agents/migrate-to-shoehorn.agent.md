---
name: migrate-to-shoehorn
description: Migrate test files from `as` type assertions to @total-typescript/shoehorn. Use when user mentions shoehorn, wants to replace `as` in tests, or needs partial test data.
---

# Migrate to Shoehorn

## Why shoehorn?

`shoehorn` lets you pass partial data in tests while keeping TypeScript happy. It replaces `as` assertions with type-safe alternatives.

**Test code only.** Never use shoehorn in production code.

Problems with `as` in tests:

- Trained not to use it
- Must manually specify target type
- Double-as (`as unknown as Type`) for intentionally wrong data

## Install

```bash
npm i @total-typescript/shoehorn
```

## Migration patterns

### Large objects with few needed properties

Before:

```ts
type Request = {
  body: { id: string };
  headers: Record<string, string>;
  cookies: Record<string, string>;
  // ...20 more properties
};

it("gets user by id", () => {
  // Only care about body.id but must fake entire Request
  getUser({
    body: { id: "123" },
    headers: {},
    cookies: {},
    // ...fake all 20 properties
  });
});
```

After:

```ts
import { fromPartial } from "@total-typescript/shoehorn";

it("gets user by id", () => {
  getUser(
    fromPartial({
      body: { id: "123" },
    }),
  );
});
```

### `as Type` → `fromPartial()`

Before:

```ts
getUser({ body: { id: "123" } } as Request);
```

After:

```ts
import { fromPartial } from "@total-typescript/shoehorn";

getUser(fromPartial({ body: { id: "123" } }));
```

### `as unknown as Type` → `fromAny()`

Before:

```ts
getUser({ body: { id: 123 } } as unknown as Request); // wrong type on purpose
```

After:

```ts
import { fromAny } from "@total-typescript/shoehorn";

getUser(fromAny({ body: { id: 123 } }));
```

## When to use each

| Function        | Use case                                           |
| --------------- | -------------------------------------------------- |
| `fromPartial()` | Pass partial data that still type-checks           |
| `fromAny()`     | Pass intentionally wrong data (keeps autocomplete) |
| `fromExact()`   | Force full object (swap with fromPartial later)    |

## Workflow

1. **Gather requirements** - ask user:
   - What test files have `as` assertions causing problems?
   - Are they dealing with large objects where only some properties matter?
   - Do they need to pass intentionally wrong data for error testing?

2. **Install and migrate**:
   - [ ] Install: `npm i @total-typescript/shoehorn`
   - [ ] Find test files with `as` assertions: `grep -r " as [A-Z]" --include="*.test.ts" --include="*.spec.ts"`
   - [ ] Replace `as Type` with `fromPartial()`
   - [ ] Replace `as unknown as Type` with `fromAny()`
   - [ ] Add imports from `@total-typescript/shoehorn`
   - [ ] Run type check to verify

## Anvil Verification Loop

All code changes go through this loop. Use `session_store` for all SQL. Never create local DB files.

### Git Hygiene (before every task)

1. Run `git status --porcelain`. If uncommitted changes unrelated to current task exist, surface them and ask: "Commit them now" / "Stash them" / "Ignore and proceed".
2. Run `git rev-parse --abbrev-ref HEAD`. If on `main` or `master` for any meaningful change, ask: "Create branch for me" / "Stay on main" / "I'll handle it".

### Verification Ledger

At the start of every task, generate a `task_id` slug (e.g. `migrate-shoehorn-auth`). Create the ledger:

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
