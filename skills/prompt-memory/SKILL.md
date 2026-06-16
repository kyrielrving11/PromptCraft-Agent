---
name: prompt-memory
description: >
  Workspace-anchored prompt memory management. This skill provides
  checkpoint.py (save prompt contexts to .promptcraft/prompt_vault.json with
  Git-style immutable version log) and hydrate.py (semantic search with
  keyword overlap scoring, version rollback, and version history listing).
  Use when prompt-craft or prompt-review needs to persist or load prompt
  history. ALL persistence is file-based — no host memory API, no database.
---

# Prompt Memory Management

This skill manages the `.promptcraft/prompt_vault.json` workspace-anchored memory file
through two deterministic scripts. No host memory API is used — all state lives in the
project working directory as human-readable, editable JSON.

## Scripts

### `scripts/checkpoint.py` — Save a prompt checkpoint

Executed after prompt-craft or prompt-review produces a prompt. Appends to the vault
without overwriting previous versions. Manages `version_tag`, `is_active`, and
`parent_version` automatically.

When to call:
- After Step 4 of prompt-craft's workflow (new or versioned save).
- After prompt-review produces an improved version.
- After prompt-craft's Step 5c (execution feedback write-back) — append feedback as a
  new version using `--version-of`, with `importance: "REFERENCE"`.

### `scripts/hydrate.py` — Load and filter prompt history

Executed at the start of a new prompt-craft session. Performs keyword-overlap semantic
search against the vault and returns only `is_active` versions of matching tasks.
By default returns compact results (metadata only, ~500 tokens). Use `--full` to
include the complete generated prompt text for reuse.

**Query Expansion (best practice):**

hydrate.py uses Jaccard similarity — it matches exact tokens only. "审计权限" and
"access control" have zero overlap despite being semantically identical. Before
calling hydrate.py, expand the query with cross-language synonyms and related
technical terms:

```
# Poor (single-language, narrow):
hydrate.py --query "审计 ERC-20 合约的权限控制逻辑"

# Good (expanded with cross-language synonyms):
hydrate.py --query "审计 ERC-20 合约的权限控制逻辑 access control authorization ownership mint burn permission check ownable"
```

The expanded query creates token overlap with vault entries in either language,
dramatically improving recall without changing the scoring algorithm.

**Query response structure:**

```json
{
  "status": "ok",
  "query": "<search text>",
  "auto_full_threshold": 0.75,
  "global_entries": [...],   // GLOBAL entries — always returned regardless of query match
  "results": [...],          // Top-k scored entries (excluding those already in global_entries)
  "total_active_tasks": 42
}
```

**`global_entries`** contains all active entries whose `summary.importance` is `"GLOBAL"`.
These represent cross-task long-term constraints and are returned unconditionally —
they ignore `--task-id` / `--skill` filters and are not subject to the top-k cutoff.
The caller MUST inject GLOBAL entries' `hard_constraints`, `summary.key_decisions`,
`summary.hard_constraints_added`, and `summary.summary_text` into every session context.

Each entry in both groups carries:
- `"global": true/false` — whether it came from the GLOBAL pool.
- `"auto_full": true/false` — whether the full prompt was auto-injected (score > threshold).

Also supports:
- `--rollback-to <v1>` to switch the active version for a task.
- `--list-versions` to inspect a task's full version chain.
- `--full` to include `generated_prompt` (complete text) in search results.

## When to Use This Skill

Load `prompt-memory` alongside `prompt-craft` or `prompt-review`. The scripts are
deterministic — execute them directly rather than loading them into the AI's context.

## Reference

- `references/vault-schema.md` — Full vault JSON structure with field descriptions.
