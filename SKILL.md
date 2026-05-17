---
name: daily-intelligence-briefing
description: >
  Generate a daily Morning Intelligence Briefing — a senior analyst-style report
  covering global geopolitics, defense, economics, technology, and AI. Use this
  skill whenever the user asks to run, generate, or schedule a morning briefing,
  intelligence briefing, geopolitical briefing, daily news digest, or
  political-economic daily summary. Also trigger when the user says things like
  "run my briefing", "generate the intel brief", "do the morning brief", or
  "what happened in the world today". Saves output as both Markdown and PDF to
  the workspace folder, then updates or creates a Google Calendar event with
  the top headlines.
---

# Daily Intelligence Briefing

Produces a structured intelligence briefing in analyst format, saved as
**Markdown + PDF**, with a **Google Calendar event** updated with the day's
top 5 headlines. Uses parallel **Haiku** subagents for all web research to
keep token use in the main session minimal.

---

## Design principles

**Token-efficient:** Haiku subagents handle all searching and story extraction.
This session only orchestrates, assembles, and writes. Do not run web searches
yourself.

**Analytical lens:** Signal over noise. Every included item must have a clear
political, economic, military, technological, legal, or strategic implication.
No gossip, tactical military updates without strategic dimension, celebrity
politics, sports, or lifestyle content.

**Source bar:** Reuters, AP, BBC, Bloomberg, Financial Times, WSJ, NYT,
Washington Post, The Economist, Politico, Foreign Policy, Al Jazeera,
Nikkei Asia, South China Morning Post, Der Spiegel, Le Monde, and comparable
national papers of record. Every item must have at least one real source link.
Do not invent URLs.

---

## Step 1 — Get today's date

```bash
date
```

Use the result for the briefing header, file names, and calendar event.

---

## Step 2 — Spawn 5 Haiku research agents in parallel

Launch all five agents **in one message** (one Agent tool call each).
Set `model: "haiku"` on every agent. Each must return only a **JSON object**
— no prose, no commentary. Instruct them explicitly to return JSON only.

---

### Agent A — United States

**Search queries (use WebSearch for each):**
- `"United States politics news today [DATE]"` — White House, Congress, executive orders, elections
- `"US economy Federal Reserve trade tariffs news today [DATE]"` — Fed, inflation, markets, jobs
- `"US foreign policy NATO Ukraine news today [DATE]"` — alliances, aid, summits

**Return JSON:**
```json
{
  "region": "NORTH AMERICA",
  "stories": [
    {
      "country": "UNITED STATES",
      "subsection": "POLITICS & POLICY",
      "headline": "One-line headline in title case",
      "what_happened": "2–3 sentences. Factual. Present tense for ongoing, past for completed.",
      "why_it_matters": "One sentence on the strategic or economic implication.",
      "confidence": "HIGH | MEDIUM | LOW",
      "sources": [{"name": "Reuters", "url": "https://..."}]
    }
  ]
}
```

Include **3–5 stories**. Use `subsection` values: `"POLITICS & POLICY"`,
`"ECONOMY & MARKETS"`, or `"FOREIGN POLICY"`.

---

### Agent B — Europe, NATO & Ukraine

**Search queries:**
- `"Europe politics economy news today [DATE]"` — EU, Germany, France, UK, ECB, elections
- `"NATO defense spending Ukraine war news today [DATE]"` — alliance, frontlines, aid
- `"Russia economy military news today [DATE]"` — sanctions, mobilization, energy revenues

**Return JSON:** Same schema. Use `region` values `"EUROPE"` and
`"RUSSIA & EURASIA"`. Subsections: country names (e.g. `"GERMANY"`,
`"UKRAINE"`, `"RUSSIA"`), or `"NATO & EUROPEAN DEFENSE"`.
Include **4–6 stories** total across both regions.

---

### Agent C — Middle East, Energy & Gulf

**Search queries:**
- `"Middle East Iran Israel war news today [DATE]"` — conflict, diplomacy, nuclear
- `"oil energy price OPEC news today [DATE]"` — supply, Hormuz, prices, OPEC+
- `"Gulf GCC Saudi Arabia UAE news today [DATE]"` — investment, diplomacy, production

**Return JSON:** Same schema. `region`: `"MIDDLE EAST & NORTH AFRICA"`.
Subsections: country names or `"ENERGY & OIL MARKETS"`.
Include **3–5 stories**.

---

### Agent D — Asia-Pacific & Latin America

**Search queries:**
- `"China economy trade military Taiwan news today [DATE]"` — GDP, exports, PLA, US-China
- `"India Japan South Korea Asia Pacific news today [DATE]"` — security, trade, elections
- `"Latin America Brazil Colombia Mexico news today [DATE]"` — politics, elections, economy

**Return JSON:** Same schema. Use `region` values `"ASIA-PACIFIC"` and
`"LATIN AMERICA"`. Subsections: country names.
Include **3–5 stories** total. Omit Latin America if nothing macro-relevant.

---

### Agent E — Technology, AI & Defense Tech

**Search queries:**
- `"artificial intelligence AI regulation policy news today [DATE]"` — EU AI Act, US policy, major launches
- `"US China technology semiconductor defense competition today [DATE]"` — export controls, CHIPS Act
- `"defense technology procurement military AI news today [DATE]"` — drones, autonomy, cyber, space

**Return JSON:** Same schema. `region`: `"TECHNOLOGY & AI"`.
Subsections: `"AI & FRONTIER TECH"`, `"DEFENSE TECHNOLOGY"`, or
`"SEMICONDUCTORS & SUPPLY CHAINS"`. Include **3–4 stories**.

---

## Step 3 — Collect and merge agent outputs

Wait for all 5 agents. Parse JSON from each. If an agent returns malformed
JSON, extract what you can. Merge all `stories` arrays into one list, grouped
by region.

---

## Step 4 — Write the briefing

Using the merged stories, write the full briefing in **clean Markdown** following
the structure below. Do not copy agent output verbatim — synthesize, deduplicate
across agents, and write in analyst prose. Add the `Relevance to me` and
`Confidence / uncertainty` fields yourself, drawing on the agent's `confidence`
value and your own assessment.

---

### Required output structure

```
# [YYYY-MM-DD] | Daily Intelligence Brief

*Prepared: [Full date] | Automated Intelligence System*

---

## Section 1: Executive Summary

8–15 bullet points. Each must include: what happened, why it matters (one
sentence), source link.

---

## Section 2: Regional Breakdown

Regions in order:
1. North America
2. Latin America
3. Europe
4. Middle East and North Africa
5. Sub-Saharan Africa
6. Asia-Pacific

Within each region, use ### for region heading, #### for country/subregion.

For each development:
- **Headline:** One-line title
- **What happened:** 2–3 factual sentences.
- **Why it matters:** One sentence on the strategic/economic implication.
- **Relevance to me:** 1–2 sentences tailored to the user's focus areas.
- **Confidence / uncertainty:** HIGH / MEDIUM / LOW + brief note.
- **Source:** [Outlet](URL)

---

## Section 3: Strategic Themes

5–10 cross-cutting themes. 2–4 sentences each on what changed and why it
matters. Examples: NATO burden-sharing, AI competition, energy supply shock,
Fed policy trajectory, China-Taiwan escalation, sanctions and trade.

---

## Section 4: Watchlist

5–10 forward-looking signposts for the next 24–72 hours. Specific, not generic.

---

## Section 5: Bottom Line

- What matters most today
- What matters most for [user's primary focus region]
- What matters most for [user's secondary focus]
- What matters most for defense and technology
```

---

## Step 5 — Save Markdown file

Write the completed briefing to:

```
[WORKSPACE]/[YYYY-MM-DD]_Daily_Intelligence_Brief.md
```

---

## Step 6 — Generate PDF

Write and run a Python script using `reportlab` to convert the Markdown to a
formatted PDF. Save the PDF to the same workspace folder as:

```
[YYYY-MM-DD]_Daily_Intelligence_Brief.pdf
```

Key formatting guidelines for the PDF:
- Title in large dark blue, subtitle in grey italic
- `##` section headers: white text on dark blue background bar
- `###` / `####` subheadings: dark blue bold
- Body text: 9pt justified, dark charcoal
- Bullets: indented, 9pt
- Horizontal rules between major sections
- Margins: 0.75 inch all sides, Letter page size

Strip all Markdown link syntax `[text](url)` — render as plain text in the PDF
(links are not clickable in PDF output). Convert `**bold**` to `<b>` tags for
ReportLab Paragraph objects.

If `reportlab` is not installed:
```bash
pip install reportlab --break-system-packages -q
```

Confirm PDF creation succeeded before proceeding.

---

## Step 7 — Update Google Calendar

1. Search today's calendar for an event titled `"Morning Intelligence Briefing"`
   (typically around 8:00 AM).
2. If found, **update** its description with:
   - Date
   - One-line summary of the day's top theme
   - Top 5 headlines with source links
   - Note that full briefing files are saved to the workspace folder
3. If not found, **create** an event for today at 8:00 AM titled
   `"🌍 Morning Intelligence Briefing"` with the same description content.
4. If the Calendar connector is not available or the update fails, state this
   clearly in the output — do not fabricate success.

---

## Step 8 — Final output to user

Report:
- ✅ Markdown saved: `[filename].md`
- ✅ PDF saved: `[filename].pdf`
- ✅ Calendar event updated/created (or ❌ with reason if failed)
- The day's **top theme** in one sentence
- The **top 5 headlines** as a bulleted list with source links

---

## Story selection bar (instruct each agent)

Include a story only if it clears at least one bar:
- Moves or could move Western financial markets
- Shifts political calculus of a G7 or major G20 government
- Affects Western trade flows, energy supply, or sanctions regimes
- Geopolitical development with direct economic or security consequences
- Technology or AI development with Western regulatory, competitive, or
  defense implications

**Hard exclude:** local crime, celebrity politics, sports, lifestyle content,
and purely tactical military updates with no strategic dimension.

---

## Confidence level definitions

| Level  | Meaning |
|--------|---------|
| HIGH   | Confirmed by 2+ major independent outlets |
| MEDIUM | Single credible source or analytical assessment |
| LOW    | Preliminary, unconfirmed, or contested reporting |

---

## Failure handling

- Source inaccessible → replace with best credible alternative; note the
  substitution.
- Agent returns no usable output → note the gap; do not pad with low-quality
  stories.
- PDF generation fails → save Markdown only; report the failure.
- Calendar update fails → report clearly; do not invent success.
- Never invent URLs or fabricate source links.
