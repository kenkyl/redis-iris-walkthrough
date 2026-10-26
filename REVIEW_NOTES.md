# Review notes

Feedback from a walkthrough of `notebooks/learn_iris_101.py`. Each note says what was seen, why it happens, and what this PR changes. Delete this file before merging.

## Summary of changes

| # | Note | Change |
|:--|:--|:--|
| 1 | Markdown doesn't render until cells run | `pyproject.toml`: `auto_instantiate = true` |

---

### 1. Markdown doesn't render until cells run

- **Seen:** On a fresh `marimo edit`, every cell is stale, so the prose stays blank until you run it.
- **Cause:** marimo 0.24.2 defaults `runtime.auto_instantiate` to `false` (`marimo/_config/config.py`), even though its docs say `true`.
- **Change:** `auto_instantiate = true` under `[tool.marimo.runtime]` in `pyproject.toml`. Project config beats user config, so this applies to every reader, including you if your own config turns it off.
- **Side effects:** Opening the notebook now runs the settings check, the imports, the embedding model download (first open only) and a read-only `FT.INFO`. Writes, model calls and hosted-service actions still wait for Create, Ask, Seed, Clean up or a chat message. Two cells show existing data on open:
  - Chapter 1's three search steps show results if articles from an earlier Create haven't expired yet (they have a one-hour TTL).
  - Chapter 4's memory table lists what the store already holds for Priya and Tom.
