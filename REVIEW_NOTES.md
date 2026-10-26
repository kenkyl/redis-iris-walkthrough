# Review notes

Feedback from a walkthrough of `notebooks/learn_iris_101.py`. Each note says what was seen, why it happens, and what this PR changes. Delete this file before merging.

## Summary of changes

| # | Note | Change |
|:--|:--|:--|
| 1 | Markdown doesn't render until cells run | `pyproject.toml`: `auto_instantiate = true` |
| 2 | Clean up everything is slow on a shared database | `crs_cleanup`: one pipelined `FT.INFO` round trip |
| 3 | LangCache cleanup forgets entries after a kernel restart | None (suggestion only) |
| 4 | Readers have to guess which LLM provider and model to use | `README.md`, `.env.example`: say where the key goes, the default model, and how to switch |
| 5 | Chapter 1 search results appear before the reader acts | None (suggestion only) |
| 6 | "Try it" boxes don't stand out from the prose | `theme-redis.css`: tint, thicker edge, bolder title |
| 7 | No guide for readers going through it on their own | `README.md`: "How to use the notebook"; `.env.example`: `REDIS_URL` examples |
| 8 | Few visuals | Architecture page and README overview; a diagram in the prologue and each chapter |
| 9 | No way to check you ran everything before cleanup | None (suggestion only) |
| 10 | Clean up everything gives no progress while it runs | None (suggestion only) |
| 11 | A LangCache outage shows only an error type | `lcs_answer`: the error shows LangCache's `detail` |

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

### 5. Chapter 1 search results appear before the reader acts

- **Seen:** As soon as **Create** finishes, steps 2 to 4 (`KNN`, `VECTOR_RANGE`, the year filter) all show results, before the reader has done anything in those steps. It reads as though a click was skipped.
- **Cause:** By design, these steps have no button. They rerun whenever the index has articles and the dropdown or a slider changes. Note 1's auto-run adds one case: opening the notebook within an hour of an earlier Create.
- **Suggestion (no change in this PR):** Give each step a **Search** button that stays on after its first press, so the slider is still live afterwards: `mo.ui.button(value=0, on_click=lambda n: n + 1, label="Search")` plus `mo.stop(not button.value, hint)` in `rs_knn`, `rs_floor`, and `rs_filter`, and a short line in each "Try it". Avoid `mo.ui.run_button` here: it's only on for the run its press triggered, so moving the slider afterwards would hide the results. A lighter option is to start the step 2 dropdown empty, so all three steps wait for a pick.

### 6. "Try it" boxes don't stand out from the prose

- **Seen:** The boxes that ask the reader to act look much like the text around them. Only the small title is styled.
- **Change:** `notebooks/theme-redis.css` gives `.admonition.success` (used only for "Try it") a light tint of its green, a thicker left edge, slightly larger text, and a bold title. No notebook code changes.

### 7. No guide for readers going through it on their own

- **Seen:** Without a live presenter, a reader has to work out the layout, which buttons do what, and why results can vanish (expiry).
- **Change:** a short "How to use the notebook" section in `README.md`: the column layout, buttons, "Try it" boxes, hidden cells, restarting, expiry, and cleanup. `.env.example` now shows `REDIS_URL` examples for Redis Cloud, TLS, and local, noting that Chapter 3's hosted service can't reach a local database.

### 8. Few visuals

- **Seen:** The notebook is all text and tables. Many readers learn faster from a picture of the moving parts, and the chapters look alike at a glance.
- **Change:**
  - `docs/architecture/`: an architecture model and page, made with the visualizer plugin for Claude Code. It shows the app, your Redis, the three Iris services, the LLM, which credential goes where, and where state lives, plus three sequence diagrams (one support turn, cache-aside, creating a surface). The README gets an overview in a new "Architecture" section after the intro. To change it, edit `architecture.json` and re-render; don't edit the Mermaid by hand.
  - Notebook: a small Mermaid flow in the prologue and after each chapter's opening story, drawn with `mo.mermaid` (marimo 0.24.2 shows mermaid code blocks in `mo.md` as plain code). Each title cell's `mo.md` is split in two around the diagram. No other text changes.
- **Please check:** The diagrams were drawn from the notebook's own text and request lists. Confirm the flows, especially that LangCache and Agent Memory keep their indexes in the reader's database (seen in `FT._LIST` on a test database).

### 9. No way to check you ran everything before cleanup

- **Seen:** At the end of Chapter 4, a reader working alone can't easily tell whether every step worked or what Clean up everything is about to remove. Presenting live, you can't tell whether the audience is in step either.
- **What already exists:** settings that stop the notebook if missing, a hint in each step when its dependency is missing ("Press **Create** in step 1 first"), and each step's own status line. So a check after every chapter would repeat these. One check at the end would not.
- **Suggestion (no change in this PR):** One read-only "Check your work" cell just before **Clean up everything**: a checklist built from state the notebook already has.
  - Chapter 1: the index exists and holds articles (`FT.INFO`).
  - Chapter 2: answers stored this session (`len(lcs_entries)`).
  - Chapter 3: a surface exists and lists its tools (`crs_surface()`).
  - Chapter 4: memories for Priya and Tom, and at least one chat turn (`ams_memories`, `ams_log`).

  Each line gets ✅ or "not yet: <the step to go back to>". Rerun after cleanup, the same cell confirms everything is gone, like a teardown check. That's about 20 lines with no new requests.

### 10. Clean up everything gives no progress while it runs

- **Seen:** After the press, the reader sees only marimo's hourglass until the summary appears, with no sign of what's happening or whether it's working.
- **Cause:** `am_cleanup` runs four steps one after another (Context Retriever, Redis Search, LangCache, Agent Memory) and draws output only at the end.
- **Suggestion (no change in this PR):** Wrap the steps in `with mo.status.spinner(title="Cleaning up everything") as _spinner:` and call `_spinner.update(subtitle="<service>: <what it's deleting>")` before each one. The spinner disappears at the end and the existing summary stays. That's about 15 lines in one cell. The same pattern fits Chapter 3's **Create** and Chapter 4's **Seed**.

### 11. A LangCache outage shows only an error type

- **Seen:** Chapter 2's **Ask** failed with `LangCache answered 424 (/errors/resource-unavailable) to the search`. A direct search got the same answer several times, then worked again with nothing changed.
- **Cause:** The database behind the LangCache service was briefly unavailable. The error body says so in `title` and `detail` ("Cache Database Error", "Cache database is unavailable"), but the notebook printed only `type`.
- **Change:** `lcs_answer` puts `detail` in the error, falling back to `type`: `LangCache answered 424 to the search: Cache database is unavailable`.
- **Suggestion:** One line in the README's how-to: "If LangCache answers 424, its database is unavailable. Wait a minute and press Ask again."
