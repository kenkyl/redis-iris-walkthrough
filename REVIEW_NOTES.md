# Review notes

Feedback from a walkthrough of `notebooks/learn_iris_101.py`. Each note says what was seen, why it happens, and what this PR changes. Delete this file before merging.

## Summary of changes

| # | Note | Change |
|:--|:--|:--|
| 1 | Markdown doesn't render until cells run | `pyproject.toml`: `auto_instantiate = true` |
| 2 | Clean up everything is slow on a shared database | `crs_cleanup`: one pipelined `FT.INFO` round trip |
| 3 | LangCache cleanup forgets entries after a kernel restart | None (suggestion only) |
| 4 | Readers have to guess which LLM provider and model to use | `README.md`, `.env.example`: say where the key goes, the default model, and how to switch |

---

### 1. Markdown doesn't render until cells run

- **Seen:** On a fresh `marimo edit`, every cell is stale, so the prose stays blank until you run it.
- **Cause:** marimo 0.24.2 defaults `runtime.auto_instantiate` to `false` (`marimo/_config/config.py`), even though its docs say `true`.
- **Change:** `auto_instantiate = true` under `[tool.marimo.runtime]` in `pyproject.toml`. Project config beats user config, so this applies to every reader, including you if your own config turns it off.
- **Side effects:** Opening the notebook now runs the settings check, the imports, the embedding model download (first open only) and a read-only `FT.INFO`. Writes, model calls and hosted-service actions still wait for Create, Ask, Seed, Clean up or a chat message. Two cells show existing data on open:
  - Chapter 1's three search steps show results if articles from an earlier Create haven't expired yet (they have a one-hour TTL).
  - Chapter 4's memory table lists what the store already holds for Priya and Tom.

### 2. Clean up everything is slow on a shared database

- **Seen:** Clean up everything sends `FT.INFO` to every index in the database, one at a time.
- **Cause:** `crs_cleanup` (in `cr_lifecycle`) checks each index's prefixes so it can drop leftover Context Retriever indexes. Each check is its own round trip.
- **Change:** one pipelined `FT.INFO` round trip. Same rule for what gets dropped, and still by name only. On a shared database with 118 indexes, the check went from 5.5 s to 0.3 s, with identical results.

### 3. LangCache cleanup forgets entries after a kernel restart

- **Seen:** After a kernel restart, **Clear** and **Clean up everything** report 0 LangCache entries deleted, and earlier entries stay in the cache.
- **Cause:** LangCache can't list entries, so the notebook keeps the IDs it stored in `lcs_entries`, a dict in the kernel (`lc_connect`). A restart empties it.
- **Impact:** Entries expire after 15 minutes. Until then, a fresh session's first **Ask** in Chapter 2 can come back as a hit, not the miss the lesson expects.
- **Suggestion (no change in this PR):** Keep the IDs in a Redis hash, for example `cairn:20261021:langcache`, with the same 15-minute TTL, so a restarted kernel can still delete them. That's about three lines in `lc_connect`. The minimal alternative is one sentence in the cleanup text saying entries from before a restart expire within 15 minutes.

### 4. Readers have to guess which LLM provider and model to use

- **Seen:** The README names three providers but not the variable the key goes in, which model the notebook uses, or how to switch. An Anthropic or Gemini key fails at the first chat turn until you find `AI_MODEL` in the notebook.
- **Cause:** The provider follows the model name (`src/cairn_support/llm.py`), and the model is hard-coded as `AI_MODEL = "gpt-4o-mini"` in the `ai_model` cell. `LLM_API_KEY` is also the only setting in `.env.example` without a comment.
- **Change:** `README.md` and `.env.example` now say: put the key in `LLM_API_KEY`; the default is OpenAI's `gpt-4o-mini`; for Claude or Gemini, set `AI_MODEL` in the notebook.
- **Suggestion (no change in this PR):** Read the model from `.env`, so every setting lives in one place: `AI_MODEL = os.environ.get("AI_MODEL", "gpt-4o-mini")` in `ai_model` (which then takes `os`), plus an optional `AI_MODEL=` in `.env.example`. That also makes the settings panel's "`AI_MODEL` sets which" (`learn_iris_101.py:90`) accurate. Today it reads as if `AI_MODEL` were a `.env` setting.
