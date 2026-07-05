# WordCup-Analyst

AI-powered FIFA World Cup match-day briefings. You ask one question — *"Give me a briefing on Brazil's next match"* — and three specialist agents gather stats, scout a key player, and pull news in **parallel**. A synthesizer merges their findings into one readable preview. Runs on free-tier APIs.

---

## How it works

```
User query
    ↓
Supervisor — routes the question, resolves team + next fixture once
    ↓  parallel (LangGraph Send)
┌──────────────┬──────────────┬──────────────┐
│ matchup_agent│ player_agent │  news_agent  │
│ form/tables  │ key scorer   │ RSS + Tavily │
└──────────────┴──────────────┴──────────────┘
    ↓ fan-in
Synthesizer — 70b model writes the briefing (8b fallback if budget spent)
```

| Piece | Role |
|-------|------|
| **Supervisor** | Picks which agents run; looks up opponent and fixture once |
| **Matchup agent** | Form comparison + group standings |
| **Player agent** | Leading scorer + bio from TheSportsDB |
| **News agent** | RSS headlines + optional Tavily search |
| **Synthesizer** | Weaves sections into one cohesive briefing |

**Two-model split:** agents use the fast 8b Groq model (large free budget); only the final write-up uses 70b. This keeps quality high without burning the daily token limit.

**Data sources:** [football-data.org](https://www.football-data.org/) (stats), BBC/Guardian RSS (news), [TheSportsDB](https://www.thesportsdb.com/) (player bios), [Tavily](https://tavily.com/) (optional web search).

---

## Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) (recommended) or a regular `venv`
- Free API keys (no credit card):
  - [Groq](https://console.groq.com/keys) — `GROQ_API_KEY` **required**
  - [football-data.org](https://www.football-data.org/client/register) — `FOOTBALL_DATA_TOKEN` **required**
  - [Tavily](https://app.tavily.com) — `TAVILY_API_KEY` optional (RSS still works without it)

---

## Clone & setup

```bash
git clone <your-repo-url>
cd WordCup-Analyst
```

**1. Create `.env` from the example:**

```bash
cp .env_example .env
```

Edit `.env` and paste your keys:

```env
GROQ_API_KEY=your_groq_key
FOOTBALL_DATA_TOKEN=your_football_token
TAVILY_API_KEY=          # leave empty if you don't have one
```

**2. Install dependencies:**

With **uv** (recommended):

```bash
uv venv
uv pip install -r requirements.txt
```

With **venv + pip**:

```bash
python -m venv .venv
source .venv/bin/activate        # Git Bash / macOS / Linux
# .venv\Scripts\activate         # Windows PowerShell
pip install -r requirements.txt
```

**3. Confirm imports:**

```bash
PYTHONPATH=. uv run python -c "import langgraph, langchain_groq, httpx; print('deps ok')"
```

Expected: `deps ok`

> **Windows note:** In PowerShell use `$env:PYTHONPATH = "."` instead of `PYTHONPATH=.`.  
> The verify scripts below handle imports automatically — no `PYTHONPATH` needed.

---

## Verify scripts (step-by-step checks)

Run these after setup to confirm each layer works. None of them need `PYTHONPATH=.` — they set the import path automatically.

| Script | What it tests |
|--------|----------------|
| `verify_client.py` | Step 3 — football-data.org client |
| `verify_player.py` | Step 8 — player scout agent |
| `verify_run.py` | Full pipeline — supervisor → agents → synthesizer |

### Step 3 — Football data client

```bash
uv run python verify_client.py
```

Do **not** paste the PDF one-liner as a single line — Python needs line breaks between statements. Use the script above instead.

Expected:

```
Brazil id: 764
```

### Step 8 — Player scout agent

```bash
uv run python verify_player.py
```

Expected: 2–3 sentences scouting a real Brazilian player (name varies with live data).

### End-to-end — Full workflow (`verify_run.py`)

```bash
uv run python verify_run.py
uv run python verify_run.py "Give me a briefing on Brazil's next match"
uv run python verify_run.py "Brief me on France's next match" --verbose
```

Runs the complete graph (supervisor → three parallel agents → synthesizer), prints pass/fail checks, and exits `0` on success. Add `--verbose` to see agent tool calls.

---

## Run the full briefing (`app/main.py`)

The main CLI prints the briefing directly (no pass/fail checks). Requires `PYTHONPATH=.` on Git Bash / macOS / Linux.

```bash
PYTHONPATH=. uv run python app/main.py "Give me a briefing on Brazil's next match"
```

**Git Bash:**

```bash
PYTHONPATH=. uv run python app/main.py "Give me a briefing on Brazil's next match"
```

**PowerShell:**

```powershell
$env:PYTHONPATH = "."
uv run python app/main.py "Give me a briefing on Brazil's next match"
```

Example output shape:

```
NEXT MATCH: Brazil vs Norway - 2026-07-05

### Brazil vs Norway - 2026-07-05
...
**Form & Matchup**: ...
**Key Player**: ...
**News & Storylines**: ...
```

Verbose mode (`--verbose` flag):

```bash
PYTHONPATH=. uv run python app/main.py "Brief me on France's next match" --verbose
```

> **Tip:** Use `verify_run.py` to test the pipeline with pass/fail output. Use `app/main.py` for the clean reader-facing briefing.

---

## Project layout

```
app/
├── config.py          # API keys, two-model factories
├── state.py           # Shared graph state + Finding reducer
├── graph.py           # LangGraph wiring (parallel fan-out)
├── main.py            # CLI entry point
├── data/
│   ├── client.py      # football-data.org (async + 90s cache)
│   ├── news.py        # RSS + Tavily
│   └── sportsdb.py    # Player bios
└── agents/
    ├── supervisor.py  # Router + fixture resolution
    ├── matchup.py     # Form & standings agent
    ├── player.py      # Key player scout
    ├── news.py        # News & storylines agent
    ├── synthesizer.py # Final briefing writer
    ├── runner.py      # Hand-rolled ReAct loop
    └── tools.py       # Per-agent tool factories
verify_client.py       # Step 3 smoke test
verify_player.py       # Step 8 smoke test
verify_run.py          # End-to-end workflow test (--verbose optional)
```

---

## Free-tier limits to know

| Limit | Effect |
|-------|--------|
| Groq 70b daily budget (~100k tokens) | Synthesis falls back to 8b prose when exhausted |
| Groq TPM per model | Parallel agents use 8b; backoff retries on 429 |
| football-data 10 req/min | 90-second HTTP cache reduces duplicate calls |
| TheSportsDB demo key | Per-player lookup works; bulk squad lists are truncated |

The architecture stays the same on paid tiers — only keys and limits change.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ModuleNotFoundError: No module named 'app'` | Set `PYTHONPATH=.` or use the `verify_*.py` scripts |
| `HTTP 403` from football-data | Check `FOOTBALL_DATA_TOKEN` in `.env` |
| `SyntaxError` on inline `-c` command | Use multiline quotes or the verify scripts |
| `SyntaxError` on `python -c "import asynciofrom ..."` | One-line paste — use `verify_client.py` or a multiline `-c` block |
| Empty news section | Normal without Tavily; add `TAVILY_API_KEY` for richer results |

