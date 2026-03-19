---
name: git-guardrails-claude-code
description: Set up Claude Code hooks to block dangerous git commands (push, reset --hard, clean, branch -D, etc.) before they execute. Use when user wants to prevent destructive git operations, add git safety hooks, or block git push/reset in Claude Code.
---

# Setup Git Guardrails

Sets up a PreToolUse hook that intercepts and blocks dangerous git commands before Claude executes them.

## What Gets Blocked

- `git push` (all variants including `--force`)
- `git reset --hard`
- `git clean -f` / `git clean -fd`
- `git branch -D`
- `git checkout .` / `git restore .`

When blocked, Claude sees a message telling it that it does not have authority to access these commands.

## Steps

### 1. Ask scope

Ask the user: install for **this project only** (`.claude/settings.json`) or **all projects** (`~/.claude/settings.json`)?

### 2. Copy the hook script

The bundled script is at: [scripts/block-dangerous-git.sh](scripts/block-dangerous-git.sh)

Copy it to the target location based on scope:

- **Project**: `.claude/hooks/block-dangerous-git.sh`
- **Global**: `~/.claude/hooks/block-dangerous-git.sh`

Make it executable with `chmod +x`.

### 3. Add hook to settings

Add to the appropriate settings file:

**Project** (`.claude/settings.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous-git.sh"
          }
        ]
      }
    ]
  }
}
```

**Global** (`~/.claude/settings.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-dangerous-git.sh"
          }
        ]
      }
    ]
  }
}
```

If the settings file already exists, merge the hook into existing `hooks.PreToolUse` array — don't overwrite other settings.

### 4. Ask about customization

Ask if user wants to add or remove any patterns from the blocked list. Edit the copied script accordingly.

### 5. Verify

Run a quick test:

```bash
echo '{"tool_input":{"command":"git push origin main"}}' | <path-to-script>
```

Should exit with code 2 and print a BLOCKED message to stderr.

[include](../git-guardrails-claude-code/scripts/)

## Anvil Verification Loop

All code changes go through this loop. Use `session_store` for all SQL. Never create local DB files.

### Git Hygiene (before every task)

1. Run `git status --porcelain`. If uncommitted changes unrelated to current task exist, surface them and ask: "Commit them now" / "Stash them" / "Ignore and proceed".
2. Run `git rev-parse --abbrev-ref HEAD`. If on `main` or `master` for any meaningful change, ask: "Create branch for me" / "Stay on main" / "I'll handle it".

### Verification Ledger

At the start of every task, generate a `task_id` slug (e.g. `git-guardrails-setup`). Create the ledger:

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
