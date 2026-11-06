# flightcheck spec

## 1. Purpose

flightcheck is an MCP server that gives any agent reliable, structured flight data, so the agent can answer open-ended trip-planning questions. The tools return facts and do exact arithmetic. The agent chooses which tools to call, combines their output with its own knowledge and web search, and makes the judgment.

Typical questions the agent should be able to handle with these tools:

- "When is it cheapest to fly BCN to LIS in December?"
- "Where can I go for a weekend in January under 100 EUR?"
- "Which of these flights don't need a visa for me, including layovers?"
- "What does this trip cost with a carry-on and one checked bag?"
- "Is it cheaper to fly into Porto and train to Lisbon, or fly direct?"
- "Two one-ways or a return?"
- "Is this price good or should I wait?"
- "Tell me when it drops under 80."

The list is illustrative. The design goal is that new kinds of questions need no new code, because the tools expose everything the data source knows and the agent fills the gaps.

flightcheck also has a CLI and scheduled watches that run without any agent.

## 2. Principles

1. **Tools return facts, the agent judges.** No tool decides whether a flight is good, needs a visa, or is worth booking.
2. **Expose everything.** Every field Kiwi returns reaches the agent through `get_itinerary`, either mapped to a typed field or kept under `extras`. Nothing is silently dropped.
3. **Unknown is explicit.** A fact the data source does not provide is marked `unknown`, never guessed or left out. This tells the agent what to look up itself (for example a carry-on fee).
4. **Arithmetic in code.** Totals, per-person costs, per-night costs and differences are computed by a tool, not by the model.
5. **Compact by default, detail on demand.** Search tools return short tables with quote ids. Full detail comes from `get_itinerary`.
6. **Bounded cost.** Every tool reports the Kiwi requests it used. A budget stops runaway exploration, and a cache lets the agent revisit results for free.
7. **No LLM inside flightcheck.** flightcheck never calls a model. All reasoning happens in the MCP client.

## 3. Non-goals

- Booking, payment or seat selection.
- Built-in knowledge of visa rules, baggage policies, airport transfers, weather or events. The agent looks these up.
- Currency conversion. Comparisons require one currency, and the agent converts extras before passing them in.
- A web dashboard (possible later, see section 14).
- Any data source other than Kiwi's official MCP server.
- Hosting for other users.

## 4. Context and constraints

- Google Flights has no public API. Amadeus Self-Service shut down on 17 July 2026. Kiwi's Tequila API is invitation-only for new partners, and Skyscanner's API is limited to established businesses.
- The only flight data source is Kiwi's official MCP server at [`https://mcp.kiwi.com`](https://mcp.kiwi.com). It is free and needs no key. Scraping [kiwi.com](http://kiwi.com), Skyscanner or Google Flights is out of scope.
- One Kiwi call with flexibility on covers ±3 days around a date. A 30-day one-way window takes about 5 calls.
- Kiwi returns a curated set of a few dozen results per call. Rate limits are not published. Which details it returns (segments, baggage, self-transfer flags) is not documented and is established by the probe (PLAN 2.0).
- Kiwi sells self-transfer itineraries, where the traveller collects bags and may pass immigration at the layover. This affects visas and missed-connection risk, so the flag must reach the agent if Kiwi provides it.
- There is no fallback source. If Kiwi changes or blocks its server, flight tools stop working until the backend is updated. This is an accepted risk for v1.

## 5. Architecture

flightcheck uses MCP in two roles.

```
agent (Claude Code, Orca, any model)          cron / terminal
   │ reasoning, web search                        │
   │ MCP, our server                               │
   ▼                                               ▼
mcp_server.py  (tools, resources, budget)      cli.py, watch.py
   └────────────────────┬──────────────────────────┘
                        ▼
   scan.py   compare.py   airports.py   profile.py
        │                                   
        ▼                                   
   backends.py ──────► store.py (history, cache, watches)
   Kiwi MCP client
        │ MCP, Kiwi's server
        ▼
   mcp.kiwi.com (search-flight)

```

- **MCP client.** [`backends.py`](http://backends.py) calls Kiwi's `search-flight` from Python. Requests are deterministic and cost no tokens.
- **MCP server.** `mcp_[server.py](http://server.py)` is the primary interface. It exposes the tools in section 7.6 to any agent.
- **Agent.** Picks tools based on the request, reads the profile, uses its own web search for anything outside flight data, and answers.

Example, "flights to Marrakech in March with a carry-on, no visa issues".

1. `get_profile`, for passport countries and home airports.
2. `scan_cheapest_dates` BCN to RAK in March.
3. `get_itinerary` on the top 5 quote ids, for carriers, layover countries, baggage and self-transfer flags.
4. Web search, for each carrier's carry-on fee where baggage is `unknown`, and for entry and transit rules for the passport.
5. `compare_trips` with the fees as extras, for exact totals.
6. Answer with totals, the visa reasoning with sources, and booking links.

## 6. Users

Single user (me), through an agent or on the command line.

## 7. Functional requirements

### 7.1 Data model

- **F1** A `SearchRequest` has origin, destination, departure date, optional return date, adults, children, infants, cabin (`economy`, `premium-economy`, `business`, `first`), optional max stops, a `flexible` flag and a preferred currency.
- **F2** A `Quote` has
  - `id`, unique and stable while cached
  - origin, destination, depart date, return date
  - total price and currency for all passengers
  - `legs`, outbound and optional return, each a list of `Segment`
  - `layovers`, derived from consecutive segments
  - `bags`, with `carry_on` and `checked`, each `included`, a price, or `unknown`
  - `self_transfer`, true, false or `unknown`
  - booking link
  - `extras`, a dict holding every Kiwi field not mapped above
  - `fetched_at`
- **F3** A `Segment` has carrier code and name, operating carrier if different, flight number, departure and arrival airport (IATA), local departure and arrival times, duration, and cabin if given.
- **F4** A `Layover` has airport, city, country, duration, whether it spans a night, and whether it changes airport.
- **F5** Any field Kiwi does not provide is `None` in Python and `"unknown"` in tool output. Derived fields (layovers, total duration, airport countries) are computed by flightcheck.

### 7.2 Search and scan

- **F6** "No flights" returns an empty list. Network errors, schema changes and parse failures raise `BackendError`.
- **F7** The backend declares `flex_days` for one-way and `flex_days_round_trip`. These describe Kiwi's own date flexibility, either 0 or Kiwi's fixed range (3). They are not a user setting. The user's flexibility is the scan window. Kiwi declares 3 for one-way and 0 for round trip until round-trip behaviour is verified.
- **F8** Max stops and currency are passed to Kiwi only if its tool schema has matching parameters. Otherwise max stops is enforced by the scan filter and quotes keep Kiwi's currency.
- **F9** A scan spec has lists of origins and destinations, a departure window, an optional nights range (absent means one-way), optional allowed weekdays, passengers, cabin, max stops, currency and a request limit.
- **F10** The scan expands the spec into requests ordered by departure date, using the backend's flexibility `f`.
  - `f > 0` splits the window into consecutive blocks of `2f + 1` days, the span one flexible request covers, and requests each block's middle date with flexibility on. A short final block still gets one request.
  - `f = 0` gives one request per (origin, destination, depart, return) combination.
  - Requests beyond the limit are not sent, and the result reports truncation.
- **F11** The scan drops quotes outside the spec (departure outside the window, a disallowed weekday, a trip length outside `nights`, more stops than allowed).
- **F12** The scan keeps the cheapest quote per actual (origin, destination, depart, return) combination and sorts by price, then date. All in-scope quotes, not only the cheapest, go to the cache and to `on_quote`.
- **F13** A failed request is retried once after a backoff, then recorded as failed. The scan continues.
- **F14** `run_scan` is async. Requests share one Kiwi session and run through `asyncio.gather` with a semaphore (default 2) and a randomised delay (default about 2 s). Sync entry points wrap it with [`asyncio.run`](http://asyncio.run).
- **F15** `run_scan` accepts an optional `on_quote` function, called once per in-scope quote. The CLI and watches pass `store.add` to save history. The scan never touches the database itself.

### 7.3 Cache and budget

- **F16** Every quote returned by any tool is cached with its full detail for `FLIGHTCHECK_CACHE_TTL` (default 30 min). A search whose exact request is cached is answered from the cache without calling Kiwi.
- **F17** `get_itinerary` and `compare_trips` read only from the cache. An expired or unknown id returns an error telling the agent to search again.
- **F18** The MCP server keeps a request budget per server process, `FLIGHTCHECK_REQUEST_BUDGET` (default 60 Kiwi requests). A tool whose estimated cost exceeds the remaining budget refuses with the estimate and does not call Kiwi. Cached answers cost nothing.
- **F19** Every tool result that may call Kiwi includes `kiwi_requests` (used by this call) and `budget_remaining`.

### 7.4 Airports and profile

- **F20** [`airports.py`](http://airports.py) resolves IATA codes to name, city, country code, coordinates and time zone from an offline dataset, and finds airports within a radius of another airport or coordinates. No network.
- **F21** The profile lives in `~/.flightcheck/profile.yaml` (overridable with `FLIGHTCHECK_PROFILE`). Fields are `passports` (country codes), `home_airports`, `currency`, default passengers, and `preferences`, a list of free-text notes for the agent (for example "avoid departures before 07:00"). flightcheck stores the profile and never interprets `preferences`.

### 7.5 Comparison

- **F22** [`compare.py`](http://compare.py) computes trip totals from cached quotes and extras. A trip is a label, one or more quote ids (for multi-leg or split trips), passenger count, optional nights, and extras. An extra has a label, amount, currency and `per` (`trip`, `person`, `person_leg` or `night`).
- **F23** The result per trip has the flight total, each extra expanded, the grand total, per-person and per-night totals if nights are given, and the difference to the cheapest trip. Trips are sorted by grand total.
- **F24** If amounts in one comparison use more than one currency, the tool returns an error listing them. It never converts.
- **F25** The result notes any input quote whose baggage or self-transfer status is `unknown`, so the agent knows the comparison may be incomplete.

### 7.6 MCP tools

All tools take IATA codes and ISO dates, and default currency and passengers from the profile.


| Tool                                                                                                                                             | Purpose                                                           | Calls Kiwi |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------- | ---------- |
| `get_profile`                                                                                                                                    | passports, home airports, currency, preferences                   | no         |
| `airport_info(code_or_query)`                                                                                                                    | name, city, country, time zone                                    | no         |
| `nearby_airports(code, radius_km=150)`                                                                                                           | alternative airports, with distance                               | no         |
| `search_flights(origins, destinations, depart, return_date?, flexible=false, max_stops?, top=10)`                                                | itineraries for specific dates, cheapest first                    | yes        |
| `scan_cheapest_dates(origins, destinations, window_start, window_end, min_nights?, max_nights?, weekdays?, max_stops?, max_requests=20, top=10)` | cheapest dates in a window                                        | yes        |
| `scan_destinations(origins, destinations, window_start, window_end, min_nights?, max_nights?, weekdays?, max_stops?, max_requests=30)`           | best fare per destination, for "where can I go" questions         | yes        |
| `estimate_requests(...)`                                                                                                                         | Kiwi requests a scan would cost, same arguments as the scan tools | no         |
| `get_itinerary(quote_id)`                                                                                                                        | full detail per F2 to F4, including `extras`                      | no         |
| `compare_trips(trips)`                                                                                                                           | exact totals per F22 to F25                                       | no         |
| `price_history(origin, destination, days=30)`                                                                                                    | daily lows from stored history                                    | no         |
| `list_watches`, `add_watch(...)`, `remove_watch(name)`                                                                                           | manage scheduled watches                                          | no         |


- **F26** Search and scan tools return a compact table (quote id, route, dates, price, stops, total duration, carriers, baggage status, self-transfer) plus `kiwi_requests`, `budget_remaining`, `searched`, `failed` and `truncated`.
- **F27** The profile is also exposed as an MCP resource `flightcheck://profile`.
- **F28** Server instructions tell the agent to
  - read the profile first and respect its preferences
  - call `estimate_requests` before broad scans and keep within the budget
  - call `get_itinerary` before judging any specific flight
  - look up anything the tools mark `unknown` or do not cover (baggage fees, entry and transit rules, ground transport, local costs) with its own web search, and cite sources
  - use `compare_trips` for every total or comparison instead of mental arithmetic
  - say when prices are from cache and how old they are, and that fares and rules should be confirmed before booking
- **F29** Tool descriptions state cost (Kiwi requests or none) in their first line.
- **F30** Transports are stdio (default) and streamable HTTP on `127.0.0.1:8765/mcp`, overridable with `FLIGHTCHECK_HOST` and `FLIGHTCHECK_PORT`.
- **F31** Works with `mcp` 2.x (`from mcp.server.mcpserver import MCPServer`) and falls back to 1.x (`from mcp.server.fastmcp import FastMCP`).

### 7.7 Kiwi connection

- **F32** The backend connects to `FLIGHTCHECK_KIWI_URL` (default [`https://mcp.kiwi.com`](https://mcp.kiwi.com)) as an MCP client over streamable HTTP, using the `mcp` SDK.
- **F33** Opening a session sends `initialize`, `notifications/initialized` and `tools/list`. The backend checks that `search-flight` exists and every argument it sends is in the schema, and records which optional arguments exist for F8. A failed check raises `BackendError` naming what is missing.
- **F34** One session per scan or tool call, reused across its requests and closed at the end.

### 7.8 Watches

- **F35** Watches live in `~/.flightcheck/watches.yaml`. Each has a unique `name`, `from`, `to`, either a fixed `window: {start, end}` or a rolling `days_ahead: [a, b]`, optional `nights`, `weekdays`, `max_stops`, passengers, `cabin`, `max_requests`, `alert_below` and `alert_drop_pct` (default 10). `add_watch` and `remove_watch` edit this file and validate it.
- **F36** A `days_ahead` window is recalculated from the current date on every run. A window start of today or earlier moves to tomorrow. A fixed window whose end is today or earlier is skipped with a one-line note.
- **F37** Running watches executes each scan, stores in-scope quotes tagged with the watch name, and alerts when the cheapest fare is at or below `alert_below`, or at least `alert_drop_pct` percent under the baseline. The baseline is the lowest stored price for that watch among quotes whose departure falls inside the current window and is later than today, read before the run writes new rows. With no baseline, only `alert_below` applies.
- **F38** Watches run without an agent and do not use the MCP request budget. They have their own `max_requests` per watch.

### 7.9 History and alerts

- **F39** Quotes from scans and watches are stored in SQLite at `~/.flightcheck/history.db`, overridable with `FLIGHTCHECK_DB`.
- **F40** History for a route groups rows by the day of `fetched_at` and returns each day's minimum price with its dates, newest first.
- **F41** Alerts print to stderr, and POST to `FLIGHTCHECK_NTFY` (ntfy topic, `Title` header) and `FLIGHTCHECK_WEBHOOK` (`{"text": ...}`) when set. A failed delivery is logged and never fails the run.

### 7.10 CLI

```
flightcheck scan ORIGIN DEST --start YYYY-MM-DD --end YYYY-MM-DD
            [--nights 3 | 3-5] [--days thu,fri] [--max-stops N] [--adults N]
            [--cabin ...] [--max-requests N] [--top N] [--json]
flightcheck itinerary QUOTE_ID
flightcheck watch [--only NAME]
flightcheck history ORIGIN DEST
flightcheck mcp [--http]

```

- **F42** `scan` prints the number of requests before starting.
- **F43** Exit code 0 on success, 1 on `BackendError`.

## 8. Kiwi mapping notes

- Tool arguments as documented by third parties. Verify with `tools/list`.
  - `trip_type` `"one-way"` or `"round-trip"`
  - `origin`, `destination` city name or airport code. Always pass IATA codes.
  - `departure_date`, `return_date` in `YYYY-MM-DD`
  - `adults`, `children`, `infants`
  - `cabin_class` `"economy"`, `"premium_economy"`, `"business"` or `"first"`. Map `premium-economy` to `premium_economy`.
  - `flexibility` boolean for ±3 days
- The response format is not documented. PLAN 2.0 records real responses before any parsing code is written. The mapping to F2 to F4 is written here after the probe, with every unmapped field listed so it is clear what lands in `extras`.
- Client imports for `mcp` 1.x are below. Confirm the 2.x equivalents in PLAN 2.0.

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

```

## 9. Data model

```sql
-- price history, one row per in-scope quote per scan, never updated
CREATE TABLE quotes (
  id INTEGER PRIMARY KEY,
  fetched_at TEXT NOT NULL,      -- UTC ISO-8601
  watch TEXT,                    -- NULL outside watches
  origin TEXT, destination TEXT, depart TEXT, return_date TEXT,
  price REAL, currency TEXT, carriers TEXT,  -- carriers is a JSON array
  stops INTEGER, duration_min INTEGER, link TEXT
);
CREATE INDEX ix_route ON quotes (watch, origin, destination, depart, return_date);

-- full quotes for get_itinerary and compare_trips, pruned after the TTL
CREATE TABLE quote_cache (
  quote_id TEXT PRIMARY KEY,
  request_key TEXT NOT NULL,     -- normalised SearchRequest, for cache hits
  fetched_at TEXT NOT NULL,
  quote_json TEXT NOT NULL       -- full Quote including extras
);
CREATE INDEX ix_cache_request ON quote_cache (request_key, fetched_at);

```

## 10. Configuration


| Variable                               | Purpose                                          |
| -------------------------------------- | ------------------------------------------------ |
| `FLIGHTCHECK_KIWI_URL`                 | Kiwi MCP endpoint override                       |
| `FLIGHTCHECK_DB`                       | history and cache database path                  |
| `FLIGHTCHECK_PROFILE`                  | profile path                                     |
| `FLIGHTCHECK_WATCHES`                  | watches file path                                |
| `FLIGHTCHECK_CACHE_TTL`                | cache lifetime in minutes, default 30            |
| `FLIGHTCHECK_REQUEST_BUDGET`           | Kiwi requests per MCP server process, default 60 |
| `FLIGHTCHECK_NTFY`                     | ntfy topic URL                                   |
| `FLIGHTCHECK_WEBHOOK`                  | generic JSON webhook                             |
| `FLIGHTCHECK_HOST`, `FLIGHTCHECK_PORT` | MCP HTTP bind address                            |


The ntfy topic and webhook URL are secrets. They come only from the environment. The profile is personal data and stays in the local file.

## 11. Non-functional requirements

- Python 3.10+, managed with uv. Python version pinned in `.python-version`.
- Project created with `uv init --package`, console script `flightcheck`.
- Runtime dependencies `mcp`, `requests`, `pyyaml` and an offline airport dataset package (chosen in PLAN 1.E). Dev dependencies `pytest` and `pytest-asyncio` in the `dev` group.
- `uv.lock` is committed. Everything runs through `uv sync` and `uv run`.
- Tests run offline. No test touches the network.
- Tool outputs stay small. A search or scan result with `top=10` is under 2,000 tokens.

## 12. Error handling


| Case                                              | Behaviour                                                  |
| ------------------------------------------------- | ---------------------------------------------------------- |
| Kiwi unreachable at session start                 | `BackendError`, tool error or CLI exit 1                   |
| Protocol mismatch, tool missing or schema changed | `BackendError` naming what is missing                      |
| A single request fails                            | Retried once, then counted as failed. The scan continues   |
| Estimated cost over budget                        | Tool refuses with the estimate, no Kiwi call               |
| Unknown or expired quote id                       | Error telling the agent to search again                    |
| Mixed currencies in `compare_trips`               | Error listing the currencies                               |
| Unknown IATA code                                 | Error from `airport_info`, empty result from Kiwi searches |
| Fixed watch window over                           | Watch skipped with a note                                  |
| Alert endpoint down                               | Logged, run continues                                      |


## 13. Acceptance criteria

1. `uv run pytest` passes offline.
2. A 30-day one-way `scan_cheapest_dates` uses 5 Kiwi requests and returns no quote outside the window.
3. Repeating the same search within the TTL uses 0 requests.
4. `get_itinerary` on a returned id shows segments, layover countries, baggage and self-transfer status, with `unknown` where Kiwi gave nothing, and an `extras` field.
5. `compare_trips` with two trips and per-person extras returns totals that match a hand calculation, and refuses mixed currencies.
6. A tool call whose estimate exceeds the budget is refused without a Kiwi request.
7. Running a watch twice with a fake backend that drops the price by 20% triggers exactly one drop alert. A stored low for a past departure date does not suppress it.
8. With `claude mcp add flightcheck -- uv run --directory <repo> flightcheck mcp`, Claude Code answers each question in `evals/[questions.md](http://questions.md)` using the tools, with any failures traced to a tool gap or a model error in `evals/[results.md](http://results.md)`.
9. With `FLIGHTCHECK_NTFY` set, a matching watch delivers a push notification.

## 14. Later, not in v1

- HTML dashboard over the history DB.
- Rail and bus data for multimodal comparisons.
- An agent that reviews watches and suggests changes.
- An agent that judges watch alerts before they are sent.
- Alert de-duplication window.
- A second flight data source, if Kiwi proves unreliable.

## 15. Open questions

- What exactly does Kiwi return per itinerary? Segments, baggage allowance and prices, self-transfer flags, fare details?
- Does Kiwi accept a city, country or "anywhere" as destination? If not, the agent supplies candidate cities to `scan_destinations`.
- Does `flexibility` apply to both legs of a round trip, and is the range really ±3 days with every date returned?
- Does the tool accept currency or stops parameters?
- What rate limits does [`mcp.kiwi.com`](http://mcp.kiwi.com) enforce, and is 60 requests per server process a good default budget?
- Do curated results ever hide the cheapest fare? Check a sample against [kiwi.com](http://kiwi.com) by hand.
- Keep all itineraries in history, or only the cheapest per combination, once the DB grows?