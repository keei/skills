---
name: tdd
description: Test-driven development with red-green-refactor loop. Use when user wants to build features or fix bugs using TDD, mentions "red-green-refactor", wants integration tests, or asks for test-first development.
---

# Test-Driven Development

## Philosophy

**Core principle**: Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_ it does it. A good test reads like a specification - "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means (like querying a database directly instead of using the interface). The warning sign: your test breaks when you refactor, but behavior hasn't changed. If you rename an internal function and tests fail, those tests were testing implementation, not behavior.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" - treating RED as "write all tests" and GREEN as "write all code."

This produces **crap tests**:

- Tests written in bulk test _imagined_ behavior, not _actual_ behavior
- You end up testing the _shape_ of things (data structures, function signatures) rather than user-facing behavior
- Tests become insensitive to real changes - they pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat. Each test responds to what you learned from the previous cycle. Because you just wrote the code, you know exactly what behavior matters and how to verify it.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Workflow

### 1. Planning

Before writing any code:

- [ ] Confirm with user what interface changes are needed
- [ ] Confirm with user which behaviors to test (prioritize)
- [ ] Identify opportunities for [deep modules](deep-modules.md) (small interface, deep implementation)
- [ ] Design interfaces for [testability](interface-design.md)
- [ ] List the behaviors to test (not implementation steps)
- [ ] Get user approval on the plan

Ask: "What should the public interface look like? Which behaviors are most important to test?"

**You can't test everything.** Confirm with the user exactly which behaviors matter most. Focus testing effort on critical paths and complex logic, not every possible edge case.

### 2. Tracer Bullet

Write ONE test that confirms ONE thing about the system:

```
RED:   Write test for first behavior → test fails
GREEN: Write minimal code to pass → test passes
```

This is your tracer bullet - proves the path works end-to-end.

### 3. Incremental Loop

For each remaining behavior:

```
RED:   Write next test → fails
GREEN: Minimal code to pass → passes
```

Rules:

- One test at a time
- Only enough code to pass current test
- Don't anticipate future tests
- Keep tests focused on observable behavior

### 4. Refactor

After all tests pass, look for [refactor candidates](refactoring.md):

- [ ] Extract duplication
- [ ] Deepen modules (move complexity behind simple interfaces)
- [ ] Apply SOLID principles where natural
- [ ] Consider what new code reveals about existing code
- [ ] Run tests after each refactor step

**Never refactor while RED.** Get to GREEN first.

## Checklist Per Cycle

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```

[include](../tdd/tests.md)
[include](../tdd/mocking.md)
[include](../tdd/deep-modules.md)
[include](../tdd/interface-design.md)
[include](../tdd/refactoring.md)

## Anvil Verification Loop

All code changes go through this loop. Use `session_store` for all SQL. Never create local DB files.

### Git Hygiene (before every task)

1. Run `git status --porcelain`. If uncommitted changes unrelated to current task exist, surface them and ask: "Commit them now" / "Stash them" / "Ignore and proceed".
2. Run `git rev-parse --abbrev-ref HEAD`. If on `main` or `master` for any meaningful change, ask: "Create branch for me" / "Stay on main" / "I'll handle it".

### Verification Ledger

At the start of every task, generate a `task_id` slug (e.g. `tdd-user-auth`). Create the ledger:

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
