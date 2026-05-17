# AI News & Intelligence Harness

A fully automated geopolitical and market intelligence briefing system powered by 15 parallel AI agents, live data feeds, and a multi-layer truth verification pipeline.

**Live demo → [View the Architecture Presentation](https://mgritzbach.github.io/ai-intelligence-harness)**

---

## What it does

Twice a week (Wednesday + Sunday at 07:30) the system automatically:

1. **Sets a coverage window** — Wednesday covers Mon→Wed, Sunday covers Thu→Sun
2. **Fetches live market data** via Python `yfinance` (EUR/USD, Brent, S&P 500, 10Y UST, Gold) — zero model memory used for any figure
3. **Loads the previous briefing** to enable story deduplication (NEW / Developing / Ongoing / STALE)
4. **Launches 14 low-token research agents in parallel** — one per region/topic:
   - 🌍 Europe · 🇺🇸 USA · 🇷🇺 Russia · 🇨🇳 China · 🌏 Asia · 🕌 Middle East
   - 🌍 Africa · 🌎 Latin America · 🇺🇦 Ukraine · 🌐 UN/International
   - 💻 Tech · ⚔️ Defense · 💰 Economy · 🌱 Climate
5. **Collects all outputs** — each story carries `article_date`, `key_quote`, `source_url`, `second_source_url`, `tier`
6. **Runs a higher-token verification agent** that independently re-fetches every tier1 story and applies 4 binary checks (URL live, date within window, verbatim quote found, second source confirmed)
7. **Deduplicates** against previous briefing
8. **Writes the full briefing** — all regional sections, verified market snapshot, verification log
9. **Saves as Markdown + PDF** to a designated folder
10. **Creates a Google Calendar event** with summary and file links

---

## Architecture

```
[Scheduler] → [Live Market Data] → [14× Low-token Agents] → [Higher-token Verifier] → [Publish]
               Python yfinance      All in parallel           4 checks per tier1        MD+PDF+Cal
```

### The 6 Hard Rules (every low-token agent)

| Rule | Description |
|------|-------------|
| R1 | No numbers from memory — every figure must come from a fetched URL |
| R2 | Verify every URL by fetching — dead links = story rejected |
| R3 | No stories from training data — all must originate from live web search |
| R4 | Extract article date from the page — stories outside coverage window rejected |
| R5 | Include one verbatim quote — proves the agent actually read the source |
| R6 | Two sources for tier1 stories — single-source top stories are auto-downgraded |

### The 4 Verification Checks (higher-token agent, tier1 only)

| Check | Description | Fail action |
|-------|-------------|-------------|
| A | URL live & relevant | REJECT |
| B | Date within coverage window | REJECT |
| C | Verbatim quote found on page | REJECT |
| D | Second source confirmed | DOWNGRADE tier1→tier2 |

### Verdicts

- **VERIFIED** — all four checks pass
- **DOWNGRADE** — tier1→tier2 (missing second source)
- **REJECT** — dead URL, stale date, or quote not found
- **VERIFIED-WARN** — passes all checks, date not found on page
- **PASS-SECONDARY** — primary URL down but secondary confirms

---

## Cost model

| Architecture | Cost/run | Cost/month (2×/wk) |
|---|---|---|
| Original 1 agent | $0.80 | $6.40 |
| 5-agent parallel | $2.20 | $17.60 |
| 14× low-token only | $4.50 | $36 |
| **14 low-token + higher-token verifier ✓** | **$6.00** | **$48** |
| All higher-token agents | $17.00 | $136 |

The selected architecture delivers **65% cost saving** vs. all higher-token while applying the more capable model only where judgment matters most — the verification audit.

---

## Replication

### Prerequisites

- Claude subscription ($100+/month recommended for parallel agent rate limits)
- Python 3.12+ with `yfinance` (`pip install yfinance`)
- Google Calendar API access (optional, for calendar events)
- A scheduler that can run the briefing prompt (Claude Cowork, Claude Code, or [Recursive Harness Builder](https://github.com/Breedoon/recursive-harness-builder))

### Step 1 — Copy the orchestrator prompt

The briefing is driven by a single structured prompt with 10 steps. Key sections to replicate:

**Coverage window logic:**
```
Wednesday run: WINDOW_START = last Thursday, WINDOW_END = today (Wed)
Sunday run:    WINDOW_START = last Thursday, WINDOW_END = today (Sun)
```

**Live market data (Step 2):**
```python
import yfinance as yf, json, datetime

tickers = {
    "EUR/USD":       "EURUSD=X",
    "BRENT CRUDE":   "BZ=F",
    "S&P 500":       "^GSPC",
    "10Y UST YIELD": "^TNX",
    "GOLD":          "GC=F"
}

result = {}
for label, sym in tickers.items():
    try:
        fi = yf.Ticker(sym).fast_info
        price      = round(fi.last_price, 4)
        prev_close = round(fi.previous_close, 4)
        pct_change = round((price - prev_close) / prev_close * 100, 2)
        result[label] = {"price": price, "pct_change": pct_change,
                         "source": f"Yahoo Finance / yfinance ({sym})"}
    except Exception as e:
        result[label] = {"price": "UNAVAILABLE", "error": str(e)}

print(json.dumps(result, indent=2))
```

**Agent JSON schema (each of the 14 agents returns):**
```json
{
  "region": "REGION NAME",
  "stories": [
    {
      "country": "Country or countries",
      "headline": "One-line headline",
      "what_happened": "2–3 factual sentences. No numbers unless from verified URL.",
      "significance_level": "tier1 | tier2 | tier3",
      "article_date": "YYYY-MM-DD",
      "key_quote": "\"Exact verbatim sentence copied word-for-word from the article body.\"",
      "source_url": "https://... [URL you fetched and confirmed]",
      "source_name": "Outlet name",
      "second_source_url": "https://... for tier1 only",
      "second_source_name": "Second outlet, or none-found"
    }
  ]
}
```

### Step 2 — Set the schedule

```
Cron: 30 7 * * 0,3
```
Runs at 07:30 on Sunday (0) and Wednesday (3).

### Step 3 — Adapt to your stack

The system is model-agnostic. The pattern is:
- **Low-token models** for the 14 parallel research agents (fast, cheap, web-search capable)
- **Higher-token model** for the single verification agent (stronger reading comprehension)
- **Orchestrator** handles market data, scheduling, dedup, writing, and file output

Works with [Claude Agent SDK](https://docs.anthropic.com/), [Recursive Harness Builder](https://github.com/Breedoon/recursive-harness-builder), or any framework that supports parallel agent spawning.

---

## Related

- [Recursive Harness Builder](https://github.com/Breedoon/recursive-harness-builder) — the local runtime this system is conceptually based on; adds unlimited recursion depth, inter-agent messaging, hooks, and Telegram UI
- [Claude Agent SDK](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)

---

## License

MIT — use freely, attribution appreciated.

