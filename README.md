# 🦥 SlothBotanist

Pantry-only herbal elixirs. Tell it what is left in your kitchen (or photograph an unknown herb) and it writes two drinks, a slow **📜 Heritage Elixir** and a quick **🌿 Modern Refresh**. Every claim cites a real search result, and the API reports whether anything outside your pantry slipped in.

A local open-weights Qwen model writes the recipes. **SerpApi is the grounding layer**: the model never invents sources and never gets context that SerpApi did not return.

## How SerpApi is used (each engine has a different job)

| Stage | Engine | Job |
|---|---|---|
| Identify | `google_lens` | Photo of an unknown herb → candidate names that join the pantry |
| Method | `google`, `duckduckgo` | Current preparation methods (two independent indexes) |
| Evidence | `google_scholar` | Research behind the herbs |
| Heritage | `google_books`, `google_patents` | Traditional herbals and historical extraction methods |
| Safety | `google_news` | Interaction warnings and recalls, which feed the Cautions line |
| Modern | `google_trends_trending_now`, `youtube` | What functional drinks people are making now, and how |
| UI | `google_images` | Visuals shown to the user, never sent to the model |
| Reproducibility | Search Archive API | Each recipe stores its `search_id`s so it can be replayed exactly |
| Extras | `extract`, Locations API, Account API | Read an old manuscript page, localise searches, show credit usage |

Each engine gets its own query and its own correct parameters (YouTube uses `search_query`; Trends Trending Now uses `geo`). See `build_engine_plan` in `apothecary_core.py`.

## Allergy-safe recipes

Send `"allergies": ["tree nuts", "ragweed", "kiwi"]` to `/api/brew`. Allergies are enforced **in code, not by the model**:

1. Free text is mapped to allergen groups (unknown words such as "kiwi" become custom blocks).
2. Pantry items containing an allergen are removed *before* search queries and the prompt are built; the response lists what was blocked and why. Plant milks are not treated as dairy, and nutmeg is not treated as a tree nut.
3. Pollen and latex allergies add **cross-reactivity cautions** (for example ragweed and chamomile), and the safety engines (Scholar, News) search for allergy evidence with that allergy in the query so the cautions carry citations.
4. The finished recipe is scanned for any allergen term. One hit triggers a single regeneration; if it still appears, the recipe is **withheld** (`status: "withheld"`) rather than shown.

This lowers risk but does not remove it. The app never claims a recipe is "safe", and the notice tells users to check with a clinician. Allergies are not stored or logged.

## Weekly safety scan and measurable findings

`weekly_scan.py` runs one `google_news` search per herb on a watchlist, caches each response, and prints a quantified headline such as *"N of M news listings across K herbs carry a safety signal"*. Run it weekly from cron. `GET /api/stats` reports cached searches, unique listings and per-engine counts.

- Requests carry `_sasl=1` and `_saslop=1` (disable with `SERPAPI_SASL=0`). If SerpApi ever rejects those flags, the request is retried **once** without them.
- **429s are never retried.** The engine enters a cooldown, the cache is used if available, and the scan moves on. If every engine is rate limited the brew answers HTTP 429 with a plain message.
- **401 (bad key) stops further requests.** After a rejected key the server sends no more searches until restarted, serves cached results if it has them, and answers HTTP 401 with a plain message. `/api/health` then reports the key as not usable.
- SerpApi's own cache is used by default; set `SERPAPI_NO_CACHE=1` to force live results.

## Why the output can be trusted

1. **Citations are checked.** The model cites `[S1]`, `[S2]`; any id that does not exist is stripped (`resolve_citations`).
2. **Zero-shopping is measured, not hoped for.** `pantry_violations` returns known ingredients in the output that are not in your pantry. Empty list = constraint held.
3. **Safety layer.** News and Scholar results feed a Cautions line, wording avoids cure claims, and a safety notice is returned with every brew.
4. **Per-request dedupe.** No global state leaks between users.
5. **Resilience.** Parallel `asyncio.gather` calls, cooldowns only on real rate limits, and an on-disk cache so the demo survives a dead network (`SLOTH_OFFLINE=1`).

## Run it

```bash
pip install -r requirements.txt
export SERPAPI_KEY=...
export HUGGINGFACE_MODEL_ID=Qwen/Qwen2.5-3B-Instruct   # 1.5B default; bigger is better
uvicorn main:app --reload
```

Herb photos: Google Lens needs a public URL, so run a tunnel (`ngrok http 8000`) and set `PUBLIC_BASE_URL`, or pass `image_url` instead of a file.

```bash
curl -X POST localhost:8000/api/brew -H 'content-type: application/json' \
  -d '{"pantry_items":["ginger","lemon peel","cinnamon stick","dried mint"],"wellness_goal":"soothing evening drink"}'
```

Optional env: `SERPAPI_ZERO_TRACE=1` (enterprise plans only), `SLOTH_OFFLINE=1`, `SLOTH_SKIP_MODEL=1`.

## Seasonality

The Modern Refresh uses `google_trends` (TIMESERIES, last 12 months) for the user's own herbs. The response includes `seasonality` with a peak month and 12 monthly averages per herb, ready to chart, and the peak is passed to the model as context ("interest in elderberry peaks in October"). Data with fewer than 6 distinct months is ignored.

## Evaluation

With the server running: `python eval_brews.py`. It runs 10 sample pantries (including allergy cases) and writes `eval_report.json` with engines succeeded, sources found, pantry violations, items blocked, withheld count and latency. Allergy cases fail the run if a blocked item appears in the finished recipe.

## Front end

`static/` holds a plain HTML, CSS and JavaScript interface with no build step. FastAPI serves it, so after `uvicorn main:app` open <http://localhost:8000>.

| Page | Uses |
|---|---|
| Brew it | `POST /api/brew`: shelf, goal, allergies, results with notes, seasonality chart and the search crew |
| Identify | `POST /api/scan-herb`: photo or picture link, then add the match to your shelf |
| Pages | `GET /api/extract`: read the text of a historical page |
| Ledger | `GET /api/stats`, `/api/telemetry`, `/api/archive/{id}` |
| Settings | `GET /api/locations` and your place for localised search |

Your shelf, allergies and place are saved only in the browser (localStorage). A banner at the top uses `GET /api/health` to warn when the model or SerpApi key is missing. All server text is escaped before it is shown, and `/api/telemetry` strips the API key and email from the account record.

## Tests

`python -m pytest test_core.py` (the core module has no network or model dependency).

> Educational only, not medical advice.
