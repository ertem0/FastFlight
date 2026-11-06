# [AGENTS.md](http://AGENTS.md)

Instructions for any coding agent working in this repo. [`SPEC.md`](http://SPEC.md) is the source of truth for behaviour. [`PLAN.md`](http://PLAN.md) holds the task order. If they disagree, follow [`SPEC.md`](http://SPEC.md) and flag the conflict.

## Project

flightcheck is an MCP server that gives agents structured flight data from Kiwi, so they can answer open-ended trip-planning questions. The tools return facts and exact arithmetic. The agent using the tools does the reasoning and looks up anything else on the web. There is also a CLI and scheduled watches.

flightcheck uses MCP twice. It is an MCP client of Kiwi ([`backends.py`](http://backends.py)) and an MCP server for agents (`mcp/`). flightcheck itself never calls an LLM.

## Layout

```
pyproject.toml
uv.lock
.python-version
src/flightcheck/
  models.py      SearchRequest, Quote, Segment, Layover (no I/O)
  backends.py    Backend protocol, BackendError, KiwiBackend (MCP client)
  scan.py        request planning, async run_scan, filter, merge
  compare.py     trip totals and comparisons (pure functions)
  airports.py    offline airport lookup and radius search
  profile.py     user profile load and validation
  store.py       SQLite history, quote cache, watch baseline
  notify.py      stderr, ntfy, webhook
  watch.py       watches file, alert rules
  cli.py         argparse entry point
  mcp/
    server.py    server object, instructions, transports
    search.py    search and scan tools, budget
    detail.py    profile, airports, itinerary, compare, history tools
    watches.py   watch tools
scripts/
  kiwi_probe.py  records real Kiwi responses as fixtures
  smoke.py       one live search
tests/
  fakes.py
  fixtures/kiwi/
evals/
  questions.md   trip-planning questions and what a good answer contains
  results.md     latest eval run
profile.example.yaml
watches.example.yaml

```

Dependencies flow one way. `models` imports nothing from the package. `backends`, `store`, `airports`, `profile` and `compare` import only `models`. `scan` imports `models` and `backends`. `watch`, `cli` and `mcp/` sit on top. Never import `cli` or `mcp/` from lower modules.

## Commands

```bash
uv sync                          # create .venv from uv.lock, dev group included
uv run pytest -q
uv run flightcheck --help
uv run flightcheck mcp           # manual MCP check
uv run scripts/smoke.py          # one live Kiwi search

```

## Tool design rules

These matter more than usual, because the tools are the product and an agent is the user.

- Tools return facts. Never add a field or behaviour that judges (good deal, visa required, recommended). That belongs to the agent.
- Never drop data from Kiwi. Map it to a typed field or keep it in `extras`. A backend test asserts no field is lost.
- Never guess. Missing data is `None` in Python and `"unknown"` in tool output.
- Do arithmetic in code. If a tool output invites the agent to add or compare numbers, consider whether `compare_trips` should do it.
- Keep search output compact. Full detail goes through `get_itinerary`.
- Tool names, argument names and descriptions are prompts. Write them for a model reading them cold. The first line of every description states its cost in Kiwi requests.
- Any change to a tool's name, arguments or output updates the server instructions if relevant, the README, and is followed by an eval rerun (PLAN 4.2).
- Error messages tell the agent what to do next ("search again", "narrow the window", "convert extras to EUR").

## Rules

- Kiwi is used only through its official MCP server at [`mcp.kiwi.com`](http://mcp.kiwi.com). Do not scrape [kiwi.com](http://kiwi.com), Skyscanner or Google Flights, and do not call Kiwi's internal website APIs. Do not add data sources without a spec change.
- Do not add built-in visa, baggage-policy or similar rule data. The agent looks those up.
- Do not write Kiwi parsing code from guesses. Build it from the fixtures recorded by `scripts/kiwi_[probe.py](http://probe.py)` (PLAN 2.0).
- Tests never touch the network. Use `tests/[fakes.py](http://fakes.py)` for anything above [`backends.py`](http://backends.py). `KiwiBackend` tests parse the recorded fixtures through a stubbed MCP session.
- Do not run live scans larger than 7 requests while developing. Kiwi's rate limits are unknown and a block stops the whole tool.
- Never write ntfy topics, webhook URLs or a real profile into code, tests, fixtures or commits.
- Use uv for everything. Add dependencies with `uv add` (or `uv add --dev`), never by hand-editing `pyproject.toml` or calling `pip`. Run code and tests with `uv run`. Commit `uv.lock` with any dependency change. Ask before adding a dependency not listed in [`SPEC.md`](http://SPEC.md) section 11.
- Kiwi-specific details stay inside [`backends.py`](http://backends.py).
- Kiwi returns dates and itineraries outside the request. The scan filter (SPEC F11) is the only place that decides what is in scope.
- `flex_days` is Kiwi's fixed flexibility, either 0 or 3. It is not a user setting.
- Scanning is async (SPEC F14). Do not mix threads with the MCP client session.
- `mcp` 2.x renamed `FastMCP` to `MCPServer` under `mcp.server.mcpserver`. Keep the server import fallback for 1.x. Client imports for 2.x are confirmed in PLAN 2.0.
- Dates are [`datetime.date`](http://datetime.date) internally and ISO strings at the edges. Segment times are local times with the airport's time zone available from [`airports.py`](http://airports.py).
- Prices are totals for all passengers, in the currency Kiwi returns.

## Style

- Python 3.10+, type hints on public functions, `from __future__ import annotations`.
- Small modules, plain functions and dataclasses, no frameworks.
- Docstrings say what a function does and any cost or side effect.
- User-facing text (README, CLI help, tool descriptions, server instructions) is plain and short. No em dashes, no semicolons, no colons before full clauses, no filler phrasing.

## Definition of done for a task

1. Code matches the referenced requirement IDs in [`SPEC.md`](http://SPEC.md).
2. New behaviour has an offline test.
3. `uv run pytest -q` passes.
4. README updated if a command, flag, env var or tool changed.
5. The task's checkbox in [`PLAN.md`](http://PLAN.md) is ticked with a one-line note if anything diverged.

## Working in parallel

Tasks in a [`PLAN.md`](http://PLAN.md) phase marked "parallel" touch separate files and can run in separate worktrees. Tasks that edit [`models.py`](http://models.py) or the `Backend` protocol run alone, because everything depends on them. MCP tool groups register through a `register(server)` function per module, so they do not edit each other's files. If you need to change a shared interface, stop and note it in [`PLAN.md`](http://PLAN.md) instead of changing it quietly.

## When unsure

Leave a `TODO(spec):` comment with the question, add it to [`SPEC.md`](http://SPEC.md) section 15, and carry on with the narrowest reasonable choice.