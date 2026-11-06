# [PLAN.md](http://PLAN.md)

Build order for flightcheck v1. Requirement IDs refer to [`SPEC.md`](http://SPEC.md). Tasks in a phase marked "parallel" touch separate files and can run in separate worktrees.

## Phase 0. Skeleton (sequential)

- \[ \] **0.1** `uv init --package flightcheck`, pin Python in `.python-version`, `uv add mcp requests pyyaml`, `uv add --dev pytest pytest-asyncio`, console script `flightcheck`, one trivial test. Commit `uv.lock`.
- \[ \] **0.2** [`models.py`](http://models.py) with `SearchRequest`, `Quote`, `Segment`, `Layover` and the bag and self-transfer types, `unknown` handling, `to_dict` for tool output and `to_row` for history. F1 to F5.
- \[ \] **0.3** [`backends.py`](http://backends.py) with the async `Backend` protocol (`flex_days`, `flex_days_round_trip`, `async search`, async context manager for the session) and `BackendError`. F6, F7.
- \[ \] **0.4** `tests/[fakes.py](http://fakes.py)` with `FakeBackend` (`flex_days = 0`) and `FakeFlexBackend` (`flex_days = 3`). Both return full quotes with segments, a mix of known and unknown baggage, some self-transfer itineraries, quotes outside the window and quotes with too many stops.

Exit check. `uv run pytest` green, `uv run python -c "import flightcheck"` works.

## Phase 1. Core (parallel)

- \[ \] **1.A** [`scan.py`](http://scan.py). Request planning with block stepping, async `run_scan`, semaphore, delay, retry, filter, merge, `on_quote`, request limit and truncation. F9 to F15. Tests for one-way, round trip, multiple origins and destinations, weekdays, truncation, failures, 30-day window giving 5 requests, short tail block, filtering, merging, concurrency limit.
- \[ \] **1.B** [`store.py`](http://store.py). History table, quote cache with TTL and request-key lookup, watch baseline, route history. F16, F17, F37 baseline, F39, F40. Tests on a temp DB, including cache hits, expiry and a past-date low that must not count.
- \[ \] **1.C** [`notify.py`](http://notify.py). F41. Tests with [`requests.post`](http://requests.post) monkeypatched.
- \[ \] **1.D** [`compare.py`](http://compare.py). Pure functions, no I/O. F22 to F25. Tests for single and multi-leg trips, every `per` mode, per-night totals, sorting, differences, mixed-currency error, unknown-baggage notes.
- \[ \] **1.E** [`airports.py`](http://airports.py). Pick an offline dataset package with IATA, city, country, coordinates and time zone (check `airportsdata` first), `uv add` it, implement lookup, search by name and radius search. F20. Tests on a few known airports and BCN's neighbours.
- \[ \] **1.F** [`profile.py`](http://profile.py). Load, validate and default the profile, write a commented `profile.example.yaml`. F21. Tests for missing file, defaults and validation errors.
- \[ \] **1.G** Task 2.0 can run here too, since it only touches `scripts/` and `tests/fixtures/`.

## Phase 2. Kiwi backend (sequential)

- \[ \] **2.0** Kiwi probe, before any parsing code. `scripts/kiwi_[probe.py](http://probe.py)` connects to [`mcp.kiwi.com`](http://mcp.kiwi.com), prints server info and the full `tools/list` schema, and saves raw responses to `tests/fixtures/kiwi/` for six calls.
  - one-way and round trip, each with `flexibility` off and on, BCN to LIS about 30 days out
  - one long-haul route with layovers, to see segment and self-transfer data
  - one city or country destination, to test open destinations Confirm the `mcp` 2.x client imports. Write the field mapping into SPEC 8, listing every unmapped field. Answer what you can in SPEC 15.
- \[ \] **2.1** `KiwiBackend` built on the fixtures. Handshake and schema check, argument mapping, full parsing into F2 to F4 with everything unmapped kept in `extras`, layovers derived with countries from [`airports.py`](http://airports.py), `flex_days` from the probe findings, optional parameters per F8. F6 to F8, F32 to F34. Tests parse every fixture, assert no Kiwi field is lost, and cover unreachable server, missing tool, missing argument and malformed response.
- \[ \] **2.2** `scripts/[smoke.py](http://smoke.py)`, one live flexible one-way search printing the cheapest quote and its itinerary. Not part of `pytest`.

Exit check. Smoke script returns a full itinerary on my machine.

## Phase 3. Interfaces (parallel)

- \[ \] **3.A** MCP search tools. `search_flights`, `scan_cheapest_dates`, `scan_destinations`, `estimate_requests`, budget accounting and cache use. F16 to F19, F26, F29. Tests with the fake backends for compact output, budget refusal, cache hits at zero cost, and output size under the F11 limit.
- \[ \] **3.B** MCP detail tools. `get_profile`, the profile resource, `airport_info`, `nearby_airports`, `get_itinerary`, `compare_trips`, `price_history`. F17, F20 to F27. Tests for expired ids, mixed currencies and unknown fields rendered as `"unknown"`.
- \[ \] **3.C** MCP server shell. Server instructions per F28, tool descriptions with cost first, stdio and HTTP, 2.x import with 1.x fallback. F28 to F31. Test by calling `list_tools()` and checking every tool and description.
- \[ \] **3.D** [`watch.py`](http://watch.py) and the watch tools. YAML load and save, rolling and fixed windows, expired windows skipped, alert rules, `list_watches`, `add_watch`, `remove_watch`. F35 to F38. Tests for config parsing, target alert, drop alert across two runs, past-date low ignored, expired window skipped, add and remove round trip.
- \[ \] **3.E** [`cli.py`](http://cli.py). `scan`, `itinerary`, `watch`, `history`, `mcp`. F42, F43. Tests via `main([...])` with the fake backend.

3.A, 3.B and 3.D register tools on the server object from 3.C. Agree the registration pattern in 3.C first (one module per tool group, each exposing a `register(server)` function), then run the rest in parallel.

## Phase 4. Agent evaluation (sequential)

- \[ \] **4.1** Write `evals/[questions.md](http://questions.md)` with at least 15 questions across these kinds, each with what a good answer must contain.
  - when (cheapest dates)
  - where (cheapest destinations, open destinations)
  - which (itinerary trade-offs, layovers, times, self-transfer risk)
  - true cost (carry-on and checked bags, airport transfers)
  - entry rules (visas, transit through layover countries)
  - alternatives (nearby airports, open-jaw, two one-ways vs return, fly plus train)
  - timing (is this price good, history)
  - monitoring (set up a watch from a sentence)
  - preferences (answers that respect the profile)
- \[ \] **4.2** Register the server in Claude Code with `uv run --directory <repo> flightcheck mcp` and run every question. Record in `evals/[results.md](http://results.md)` which tools were called, the Kiwi requests used, and whether the answer met the criteria.
- \[ \] **4.3** For every failure, decide whether it is a tool gap (missing data, awkward output, missing tool) or a model error (bad reasoning, skipped lookup). Fix tool gaps, tighten server instructions for repeated model errors, and rerun. Note each change in the Notes section.
- \[ \] **4.4** Repeat 4.2 with one other client (Orca or a non-Claude model) to check the tools work without Claude-specific behaviour.

## Phase 5. Run it for real (sequential)

- \[ \] **5.1** `profile.example.yaml` and `watches.example.yaml`.
- \[ \] **5.2** README covering install with uv (`uv sync` for development, `uv tool install .` for a global command), what the tools do and cost, how to register the server, example questions, watches, alerts and cron.
- \[ \] **5.3** Container. uv installed, `uv sync --frozen --no-dev`, data directory on the home volume, env vars passed in, MCP HTTP bound to 127.0.0.1 only.
- \[ \] **5.4** Cron or systemd timer running `uv run --directory <repo> flightcheck watch` twice a day, with the absolute path to `uv` and output to `~/.flightcheck/watch.log`.
- \[ \] **5.5** Set `FLIGHTCHECK_NTFY` and confirm a push from a watch with a high `alert_below`.

## Phase 6. After a week of use

- \[ \] Tune concurrency, delay, budget and cache TTL from real usage.
- \[ \] Compare a sample of results with [kiwi.com](http://kiwi.com) by hand.
- \[ \] Add any question types that came up in real use to `evals/[questions.md](http://questions.md)` and rerun.
- \[ \] Decide on the SPEC 14 items.

## Risks


| Risk                                                           | Mitigation                                                                                                            |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Kiwi returns less detail than needed (no baggage, no segments) | probe first, `unknown` fields push the agent to look things up, `extras` keeps whatever exists                        |
| Kiwi changes, blocks or retires its MCP server                 | schema check every session, source isolated in [`backends.py`](http://backends.py), a second source listed in SPEC 14 |
| Agent fans out into too many requests                          | budget per server process, `estimate_requests`, cache, request limits on scan tools                                   |
| Agent does arithmetic in its head and gets totals wrong        | `compare_trips`, server instructions require it, evals check it                                                       |
| Agent states visa or baggage rules from memory                 | server instructions require web lookup with sources, evals check it                                                   |
| Tool output too large for the agent's context                  | compact tables, detail only through `get_itinerary`, size test in 3.A                                                 |
| `mcp` SDK churn                                                | versions locked in `uv.lock`, server import fallback, client imports confirmed in 2.0                                 |


## Notes

Record divergences from the spec and eval-driven changes here as tasks complete.