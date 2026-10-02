# FinAlly PLAN.md Review (Second Pass)

**Date:** 2026-10-01
**Scope:** `planning/PLAN.md` (sections 1–13), checked against the market data code in `backend/app/market/`, `backend/pyproject.toml`, `.gitignore`, `.claude/skills/cerebras/SKILL.md`, and `planning/MARKET_DATA_SUMMARY.md` / `planning/archive/`.

## Summary

The plan is a solid, readable product spec. The earlier review (Section 13) correctly found most of the contract gaps. **None of the 32 Section 13 items have been resolved in the plan body yet.** Sections 1–12 are unchanged, so agents reading them as the source of truth will still build against the old, ambiguous text (for example, §6 still describes a per-ticker SSE event and §10 still offers "Lightweight Charts or Recharts").

This pass found about 40 **new** issues. The most serious ones are:

1. **Volume path collision in Docker.** The runtime volume is mounted at `/app/db`, which will probably hide the schema files in `backend/db/`.
2. **Removing a watchlist ticker deletes its price.** Both data sources already do this, so a held position can no longer be valued (an existing item, now confirmed in code).
3. **Simulator prices are not stable across restarts or re-adds.** Unknown tickers get a new random price, which makes P&L for held positions jump wildly.
4. **The LLM call blocks the event loop.** The prescribed LiteLLM snippet is synchronous, so a chat call would freeze SSE and every other request while it runs.
5. **No auth while listening on all interfaces.** Anyone on the LAN (or the internet, if deployed to the cloud) can trade and spend the OpenRouter key.
6. **The `.gitignore` ignores `lib/`.** That silently drops a typical Next.js `src/lib/` folder, and it doesn't ignore `finally.db`, `node_modules/` or `out/`.
7. **There is no way to load chat history.** It is persisted, but the plan defines no endpoint to read it.
8. **The Dockerfile targets Node 20, which reached end of life in April 2026.**

Severity: **High** = agents will likely build parts that don't fit or that are broken. **Medium** = visible bug or rework likely. **Low** = polish.

---

## 1. Status of Section 13 (Previous Review)

| Item(s) | Status | Note |
|---|---|---|
| 1 (daily change baseline) | Open | The code still has only `price`/`previous_price`. `archive/MASSIVE_API.md` plans to read `day.previous_close`, but `massive_client.py` only reads `last_trade`, so the design doc and the code have drifted apart. |
| 2 (SSE shape) | Open | §6 is still wrong. The code sends `data: {"AAPL": {...}, ...}`. |
| 3 (ticker validation) | Open | Confirmed: `_add_ticker_internal` seeds `random.uniform(50, 300)` for any string. |
| 4 (pricing non-watchlist holdings) | Open, **confirmed as a bug in code** | Both `SimulatorDataSource.remove_ticker` and `MassiveDataSource.remove_ticker` call `cache.remove(ticker)`. |
| 5–32 | Open | No changes to sections 1–12. |

**N1. [S] High — Fold the decisions into the plan body, and track status.** Section 13 is a list of questions, not decisions. Each item should be resolved by editing the relevant section (§6, §7, §8, …) and marking it "Resolved → §x" in Section 13. Otherwise agents get two conflicting sources of truth.

**N2. [S] Low — Keep review material out of the always-loaded context.** `CLAUDE.md` includes `PLAN.md` in full in every agent's context. Moving the review lists to `planning/REVIEW.md` (this file), and keeping only the decisions in `PLAN.md`, keeps context small and authoritative.

---

## 2. Market Data (Plan vs. Implemented Code)

**N3. [C] High — Simulator prices are not stable across restarts or re-adds.** The simulator keeps no state between runs. On restart:
- Known tickers reset to `SEED_PRICES`.
- Unknown tickers (for example, a user-added `PYPL`) get a **new random price between $50 and $300**.

A position bought at $80 could be worth $290 after a restart, or after a remove and re-add. That breaks P&L, the snapshot chart and the heatmap. Options:
- (a) On startup, seed each held ticker's price from the last trade price, or from `positions.avg_cost`.
- (b) Persist the last known prices in SQLite.
- (c) Restrict the simulator to a fixed allowlist with seed prices (this pairs with item 3).

The plan should also say whether a jump in portfolio value on restart is acceptable.

**N4. [C] Medium — Ticker case handling differs between the data sources.** `MassiveDataSource.add_ticker` upper-cases and strips its input. `SimulatorDataSource`/`GBMSimulator` do not, and neither does `start()` in either source. Specify one place for normalization: the API/service layer, before calling the data source. Add a test for it.

**N5. [C] Medium — The Massive poll interval and market hours.** §6 says the interval is "configurable", but `factory.py` never passes `poll_interval`. It is hard-coded to 15 seconds, and no environment variable exists. Proposed fix: add `MASSIVE_POLL_INTERVAL` to §5. Also document two behaviors:
- Outside US market hours, `last_trade` is static. Prices won't flash, and the timestamp is the time of the last trade, not the time of the poll.
- `start()` makes a blocking first poll before the app finishes starting. With a bad key or no network, startup waits on the HTTP timeout. Consider a short timeout, or run the first poll in the background.

**N6. [Q] Medium — Massive tier entitlements.** The plan assumes the snapshot endpoint works on the free tier (5 calls/min). Confirm that the free plan includes `/v2/snapshot/...`; some Polygon plans limit snapshots or real-time data to paid tiers. If it isn't included, the plan needs a fallback (for example, previous-close aggregates) or a documented requirement for a paid tier.

**N7. [C] Medium — A Massive ticker that is valid in format but doesn't exist.** If the snapshot returns nothing for a ticker, it stays on the watchlist with no price forever. This is the Massive-mode counterpart of item 3. Decide whether the add endpoint validates by doing a synchronous snapshot lookup, or whether the UI shows "no data".

**N8. [S] Medium — Bug in the SSE router factory.** In `stream.py`, `router` is a **module-level** `APIRouter`. `create_stream_router()` registers `/prices` on it each time it's called, so a second call (for example, one app per test via a fixture) adds a duplicate route. Create the router inside the factory. There are also **no tests for `stream.py`**; `MARKET_DATA_SUMMARY.md` lists none. Add at least one test with an async client that reads the first event.

**N9. [S] Low — SSE robustness details to specify:**
- No keepalive comment is sent when the cache version is unchanged, so on Massive there are 15-second gaps. Some proxies and App Runner close idle streams; send `: ping\n\n` every ~15 seconds.
- No data event is sent until the cache is non-empty. The frontend should not treat "connected but no data" as an error.
- Every client gets **every** ticker in the cache, which is right once positions are tracked (item 4). The frontend must ignore, or separately handle, stream tickers that aren't in its watchlist.

**N10. [C] Low — The timestamp format differs between SSE and the database.** SSE/`PriceUpdate` uses Unix seconds (float), while §7 uses ISO strings. Pick one convention per layer and state it in the API contract. Recommendation: ISO 8601 UTC with `Z` in REST and the database, and Unix seconds in SSE (convenient for Lightweight Charts). Also state that all stored timestamps are UTC.

**N11. [C] Low — `rich` is a runtime dependency.** It is only used by `market_data_demo.py`. Move it to the `dev` extra to keep the image small.

---

## 3. Database & Data Model

**N12. [C] High — Seed only on creation, not when tables are empty.** §4 and §7 say to seed "if the file doesn't exist or is empty". If a user removes all 10 watchlist tickers, an "empty table" check reseeds them on the next start. Specify: seed only when the schema is first created (for example, track it with `PRAGMA user_version` or a `meta` table). This also gives you a hook for future migrations.

**N13. [C] Medium — The schema contradicts §7's own rule.** §7 says "all tables include a `user_id` column", but `users_profile` has only `id`. Either state the exception explicitly or rename the column. Also specify:
- Whether there are foreign keys (`positions.user_id → users_profile.id`), and whether `PRAGMA foreign_keys=ON` is set.
- Indexes, at least `portfolio_snapshots(user_id, recorded_at)`, `trades(user_id, executed_at)` and `chat_messages(user_id, created_at)`.

**N14. [S] Medium — Record where each trade came from.** Add a `source` column to `trades` (`"manual"` | `"ai"`), and optionally a `chat_message_id`. It costs little, helps the LLM ("what did you buy for me earlier?"), helps the UI, and helps with debugging.

**N15. [C] Medium — Snapshots taken when prices are missing.** At startup in Massive mode, or for a held ticker with no cached price (see item 4), what value does a snapshot use for that position? Options: skip the snapshot, value the position at `avg_cost`, or value it at the last trade price. Without a rule, the P&L chart will show spurious drops to roughly the cash balance.

**N16. [C] Low — Location of the schema files and packaging.** §4 places the schema and seed in `backend/db/`, but `pyproject.toml` packages only `app/` (`[tool.hatch.build.targets.wheel] packages = ["app"]`). Put the schema in `backend/app/db/` so it's importable and loaded with `importlib.resources`. This also fixes N32.

---

## 4. API Contract Gaps (New)

**N17. [C] High — Missing read endpoints.**
- **`GET /api/chat` (history).** Chat is persisted, but after a page refresh the chat panel has no way to load it. Specify a limit and the order.
- **`GET /api/portfolio/trades`.** Trade history is stored but never exposed. That is useful for a "recent fills" list and for E2E assertions.
- Optional: `GET /api/prices` (a one-shot snapshot of the cache), so the frontend can render prices before the first SSE event arrives.

**N18. [C] Medium — One error envelope for all endpoints.** Item 7 covers trades only. Define one JSON error shape for every endpoint (`{"error": {"code", "message"}}`) and a shared code list (`INVALID_TICKER`, `PRICE_UNAVAILABLE`, `INSUFFICIENT_CASH`, `INSUFFICIENT_SHARES`, `INVALID_QUANTITY`, `LLM_UNAVAILABLE`, …). Map FastAPI's default 422 validation errors to the same shape, so the frontend has a single error path.

**N19. [Q] Low — Dollar-amount orders.** Users will say "buy $1,000 of NVDA". Should the LLM convert that to shares using the price in its context, or should the trade API accept `notional`? Recommendation: keep the API share-based, and tell the LLM in the system prompt to convert using the provided prices.

---

## 5. LLM / Chat

**N20. [C] High — A synchronous LLM call blocks the event loop.** The cerebras skill uses synchronous `litellm.completion`. Called from an `async def` FastAPI route, it freezes the event loop for the whole call: SSE stops, the status dot may change, and the snapshot task stalls. Specify one of two options:
- Use `litellm.acompletion` (preferred).
- Declare the chat route with a plain `def`, so FastAPI runs it in a thread pool.

Add a timeout (for example, 30 seconds).

**N21. [C] High — Make the structured output schema strict-mode compatible.** §9 marks `trades` and `watchlist_changes` as optional. Strict JSON-schema structured outputs (OpenAI-style, as passed through OpenRouter) generally require every property to be `required` and `additionalProperties: false`. Provider support for Cerebras may be stricter (for example, limited keyword support). Specify a Pydantic model where:
- Both arrays are **required** (and may be empty).
- `side` is `Literal["buy","sell"]` and `action` is `Literal["add","remove"]`.
- Quantity rules (> 0) and ticker format are enforced **server-side**, not in the schema.

Also decide whether OpenRouter may fall back to a non-Cerebras provider (`allow_fallbacks`).

**N22. [C] Medium — The order in which actions run.** When one response contains both watchlist changes and trades, which runs first? Recommendation: watchlist adds first, so a newly added ticker gets a price before the trade. With Massive, though, an added ticker has no price until the next poll, so "add PYPL and buy 5" will always fail the trade in Massive mode. Document this, or fetch a one-off price on add.

**N23. [S] Medium — Prompt injection and output handling.** Users (or ticker names) can inject instructions. The risk is low with fake money, but specify:
- LLM text is rendered as plain text or sanitized Markdown, never as raw HTML (no `dangerouslySetInnerHTML`).
- A maximum user message length (for example, 2,000 characters).
- A cap on how much history is replayed (item 15).

**N24. [C] Low — Context size and cost.** §9 step 1 includes "watchlist with live prices". Specify a compact context format (one line per ticker) and an approximate token budget, so prompts stay fast on every request.

---

## 6. Frontend

**N25. [C] High — Development workflow across two origins.** The plan covers only the production setup (one origin, no CORS). During development, `next dev` (port 3000) and uvicorn (port 8000) are separate origins. Next.js `rewrites` are unsupported with `output: 'export'`, and dev-server proxying tends to buffer SSE. Specify one approach:
- (a) Allow CORS from `http://localhost:3000` in dev mode only, and use `NEXT_PUBLIC_API_BASE`.
- (b) Always develop against the built export served by FastAPI.

Otherwise each agent will invent its own setup.

**N26. [C] Medium — Static export constraints.** Specify that the app has no dynamic routes, server actions, route handlers or `next/image` optimization (`images.unoptimized: true`). Also specify `trailingSlash` behavior so FastAPI's static mount resolves `/` → `index.html`. Ties to item 28.

**N27. [C] Medium — Lightweight Charts data rules.** The library requires **strictly ascending, unique** time values (in seconds). A trade snapshot and a periodic snapshot within the same second, or two SSE ticks sharing a timestamp, will throw at runtime. Specify that the frontend dedupes and sorts points, or that the backend guarantees unique `recorded_at` values. (This assumes item 21 is resolved in favor of Lightweight Charts.)

**N28. [S] Medium — Cap the in-memory chart buffers.** Sparklines and the main chart accumulate about 2 points per second per ticker without limit. Cap them (for example, the last 1,000 points per ticker) so the browser's memory doesn't grow over a long session.

**N29. [C] Low — Empty, loading and error states.** Define the UI for:
- No positions (heatmap, positions table, P&L chart with zero or one snapshot).
- No price yet for a ticker.
- Chat unavailable (no API key).
- The trade bar showing a rejected trade (inline message or toast).

**N30. [C] Low — Flash and connection-status semantics.**
- Don't flash on a `"flat"` direction. Simulator ticks rounded to cents are often flat.
- Map `EventSource` states explicitly: `OPEN` → green, `CONNECTING` → yellow, `CLOSED` or no message for more than N seconds → red.

**N31. [C] Low — Pick the frontend test runner.** "React Testing Library or similar" should name a runner (Vitest or Jest) and say how it runs (`npm test`), so CI and agents agree.

---

## 7. Docker, Ops & Repo Hygiene

**N32. [C] High — The volume mount can hide the schema files.** The intended container layout looks like `/app` as the working directory, `backend/` copied into it, and the volume mounted at `/app/db`. If `backend/` is copied to `/app` (so `backend/db/` becomes `/app/db/`), the volume mount **hides the schema and seed files**. Fix: put the schema in `backend/app/db/` (N16), and/or mount the data volume at a separate path such as `/data`, with `DB_PATH=/data/finally.db` (item 26). Write the container layout down explicitly.

**N33. [C] Medium — Named volume or bind mount?** §11 runs `-v finally-data:/app/db` (a named volume) but also says "the `db/` directory in the project root maps to `/app/db`" (a bind mount). These conflict. Pick one, and state that the top-level `db/` is used only for local, non-Docker runs.

**N34. [C] Medium — Node 20 is end-of-life.** Node 20 reached end of life in April 2026. Use `node:22-slim` (or 24 LTS), and pin the Next.js major version.

**N35. [C] Medium — Single worker.** The price cache, simulator and snapshot task live in process memory. Specify `uvicorn ... --workers 1`. With multiple workers, each would run its own simulator with different prices.

**N36. [C] Medium — Docker build details.** Use `uv sync --frozen --no-dev`. Copy `pyproject.toml` and `uv.lock` before the source, for layer caching. Add a `.dockerignore` (`node_modules`, `.next`, `out`, `.venv`, `db/*.db`, `.env`) so secrets and the database aren't baked into the image. The `HEALTHCHECK` (item 30) can't rely on `curl` in `python:slim`; use a Python one-liner, or install curl.

**N37. [S] High — `.gitignore` problems.** The current file is the stock Python template.
- `lib/` is ignored, so a Next.js `frontend/src/lib/` or `frontend/lib/` folder would silently never be committed.
- `db/finally.db` (or `*.db`, `*.db-wal`, `*.db-shm`) is not ignored. The plan claims it is; it only ignores `db.sqlite3`.
- `node_modules/`, `.next/` and `out/` are not ignored.
- `db/.gitkeep` doesn't exist yet.

Fix these before the frontend agent starts. Scope the ignore as `/backend/lib/`, or remove `lib/`.

**N38. [S] Medium — Line endings on Windows.** The user is on Windows. Add a `.gitattributes` with `*.sh text eol=lf` (and `*.ps1 text eol=crlf`). Otherwise `start_mac.sh`, or any shell script copied into an image, breaks on CRLF.

**N39. [C] Low — The environment variable list is incomplete.** §5 calls `OPENROUTER_API_KEY` "Required", but the app must start without it (mock mode, E2E tests, development). Also list `DB_PATH`, `MASSIVE_POLL_INTERVAL`, `LOG_LEVEL`, and optionally `LLM_MODEL` and `SIM_SEED` (see N42). Commit `.env.example`; it doesn't exist yet.

---

## 8. Security

**N40. [S] High — No auth, but listening on all interfaces.** `-p 8000:8000` publishes on every host interface, so anyone on the same network can trade, read the chat, and spend the OpenRouter key through `/api/chat`. Recommend:
- Start scripts use `-p 127.0.0.1:8000:8000`.
- §11 warns that the cloud deployment stretch goal turns the app into a **public, unauthenticated LLM proxy** on your key. Before deploying, require at least a shared-secret header or basic auth, plus a rate limit on `/api/chat`.

**N41. [S] Low — Never log or return secrets.** State that API keys are never logged, and that the health check and error responses never echo environment values or stack traces.

---

## 9. Testing

**N42. [S] Medium — Make the simulator deterministic for tests.** E2E tests can't assert prices, and unit tests of portfolio math depend on random prices. Add a `SIM_SEED` environment variable (seeding both `random` and `numpy`). Also allow injecting a fixed price for a ticker (for example, a test-only cache setter), so buy/sell P&L can be asserted exactly.

**N43. [C] Medium — E2E isolation and startup ordering.**
- In `docker-compose.test.yml`, the app needs a `healthcheck`, and Playwright needs `depends_on: condition: service_healthy`.
- Use a fresh database per run (no named volume, or `tmpfs`).
- Tests that buy and then sell share one portfolio, so either run them serially in a defined order or use the reset endpoint (item 13).
- Pin the Playwright image version to match `@playwright/test`.

**N44. [S] Medium — No CI for tests.** `.github/workflows/` only contains Claude review workflows. Add a CI job that runs backend `pytest` and `ruff`, frontend unit tests and `npm run build`, and optionally the E2E suite with `LLM_MOCK=true`.

**N45. [S] Low — Add contract tests.** Once `API_CONTRACT.md` exists (item 32), add backend tests that check response shapes against it (for example, Pydantic response models with a snapshot of the OpenAPI schema), so the frontend and backend can't drift silently.

---

## 10. Prioritized Action List

**P0: before parallel frontend and backend work starts**
1. N1: fold the Section 13 decisions into sections 1–12, starting with the Section 13 priority list (items 1, 2, 3, 4, 7, 10, 14, 16, 21, 32).
2. Item 32 + N17 + N18: write `planning/API_CONTRACT.md`, including the chat history and trades endpoints and one shared error envelope.
3. Item 4 + N3: decide the tracked-ticker set and how simulator prices stay stable for held positions. Fix `remove_ticker` evicting held tickers from the cache.
4. N32 + N33 + N16 + item 26: decide the container layout, the volume path, `DB_PATH` and the schema location.
5. N37: fix `.gitignore` (`lib/`, `*.db`, node and Next.js outputs) and add `db/.gitkeep`.
6. N25: decide the development workflow across two origins (CORS in dev, or always use the built export).
7. N20 + N21: use an async or thread-pooled LLM call with a timeout, and a strict-mode-compatible Pydantic schema.

**P1: during implementation**
8. N12: seed only on schema creation, with schema versioning.
9. N40: bind to localhost in the scripts, and warn about cloud deployment.
10. N35, N34, N36: single worker, Node 22, frozen uv sync, `.dockerignore`.
11. N42 + N43 + N44: deterministic simulator seed, E2E isolation and health gating, CI workflow.
12. N22 + N7 + N5: action ordering in chat, missing prices in Massive mode, poll interval environment variable.
13. N8: fix the module-level SSE router and add SSE tests.
14. N27 + N28: Lightweight Charts timestamp rules and buffer caps.
15. N15: snapshot valuation when prices are missing.

**P2: polish**
16. N4, N9, N10, N11, N13, N14, N19, N23, N24, N26, N29, N30, N31, N38, N39, N41, N45, N2.
