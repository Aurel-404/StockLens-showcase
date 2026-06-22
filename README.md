# StockLens — AI-Powered Stock Analysis Platform

> A full-stack SaaS web application that generates in-depth, AI-powered stock analyses on demand.  
> Built as a personal project to explore LLM integration, product design, and modern web architecture.

> **Note:** The source code is kept private. This repository serves as a public project showcase.

---

## Live Product

StockLens is a deployed, production SaaS with real users, Google OAuth login, and a Stripe subscription tier.

---

## Screenshots

**Dashboard** — watchlist with live prices, buy signals, and recently analyzed tickers
![Dashboard](screenshots/dashboard)

**Stock Analysis** — Investment Scorecard (deterministic, from Yahoo Finance) + Price & Fundamentals chart
![Analysis](screenshots/analysis)

**Sector Comparison** — full peer table with revenue growth, margins, valuation multiples, moat scores, and analyst consensus
![Sector](screenshots/sector)

**Analysis Database** — shared cache browser; all fresh analyses are available to every user
![Analysis Database](screenshots/database)

**Fair Value** — DCF model + relative sector valuation with current price position indicator
![Fair Value](screenshots/fairvalue)

---

## What It Does

StockLens lets you analyze any publicly traded stock by selecting which analytical sections you want. The app automatically routes each request to the right AI model, fetches live market data from Yahoo Finance, and returns a structured, sourced report.

### The 13 Analysis Sections

| Section | Source | Description |
|---|---|---|
| **Overview** | Claude Sonnet | Company description, sector, market position |
| **SWOT** | Claude Sonnet | Strengths, Weaknesses, Opportunities, Threats |
| **Risks** | Sonnet + web search | 3–5 key risks with severity ratings |
| **Fair Value** | Claude Sonnet | DCF model + relative peer valuation |
| **Price Scenarios** | Sonnet + web search | Bull / Base / Bear 12-month targets |
| **Revenue Breakdown** | Sonnet + web search | Segment revenue chart (last 6 quarters) |
| **Guidance Accuracy** | Sonnet + web search | Management credibility scorecard |
| **Earnings Calendar** | Yahoo Finance | Next earnings date, recent beat/miss history |
| **Insider Trading** | Yahoo Finance | Insider transactions, institutional ownership |
| **Analyst Targets** | Yahoo Finance | Consensus rating, 12-month price targets |
| **Metrics** | Yahoo Finance | Revenue, margins, P/E, EV/EBITDA, FCF |
| **Scorecard** | Yahoo Finance | Deterministic quality scores (Growth, Valuation, etc.) |
| **Sector Compare** | Sonnet + web search | Full peer comparison table with Best-Of verdict |

---

## Key Technical Features

### Section-Based AI Routing
Each analysis section is mapped to the cheapest model tier that produces reliable output:

- **`none`** — deterministic sections (Metrics, Analyst Targets, Earnings Calendar) are populated server-side directly from Yahoo Finance; no AI call is made at all
- **`haiku`** — lightweight summaries (Changes Since, Portfolio Intelligence)
- **`sonnet`** — deep analytical sections (SWOT, Fair Value, Risks, Scenarios)

This means a request for only "Metrics + Analyst Targets" makes zero Anthropic API calls and returns instantly from Yahoo Finance.

### Group-Level Web Search Caps

Sections are bucketed into groups with hard limits on Anthropic's `web_search` tool:

| Group | Sections | Max searches |
|---|---|---|
| A | Overview, SWOT, Fair Value | 0 |
| B | Risks, Scenarios, Guidance Accuracy, Revenue Breakdown | 4 |
| C | Earnings Calendar, Insider Trading | 1 |
| D | Metrics (Yahoo-only) | 0 |

The effective budget is `min(sum of selected caps, group cap)` — selecting only *Risks* gives max 2 searches, not the full group cap of 4. This makes cost predictable and bounded.

### Multi-Turn AI Loop with Hard Caps
```
User request
    │
    ▼
buildAIPlanForSections()
  ├─ model tier (none / haiku / sonnet)
  ├─ webSearchMaxUses (hard cap passed to Anthropic)
  └─ maxTurns (derived from sections)
    │
    ├── tier = "none" ──► Yahoo-only response (0 tokens)
    │
    └── tier = haiku/sonnet
            │
            ▼
     Multi-turn loop (up to maxTurns)
       ├─ AI responds with text or tool_use
       ├─ If tool_use → execute web_search → continue
       └─ If end_turn → parse JSON → break
            │
            ▼
     Inject Yahoo fundamentals server-side
            │
            ▼
     Save to shared Supabase cache
            │
            ▼
     Fire-and-forget: logAnalysis() + trackUsage()
```

### Shared Result Cache
All fresh analyses are saved to a Supabase-backed shared cache and served to all users. Popular tickers like AAPL or NVDA load in under a second — no AI call needed. Cache entries include a timestamp so users know when the data was generated.

### Post-Split Price Awareness
The current stock price and shares outstanding from Yahoo Finance are injected into every AI prompt as "ground truth" (labeled explicitly as post-split Yahoo data). This prevents the model from using stale training-data prices — e.g., BKNG after its 24-for-1 split in January 2025 ($4,000 → ~$170).

### Sector Classification Pipeline
When you enter a ticker for sector comparison:
1. Yahoo Finance GICS data is checked first (fast, no AI)
2. For ambiguous industries (e.g. "Internet Content & Information" which covers both Google and Booking.com), a one-shot Haiku call reads the business description and picks the correct canonical sector
3. The result is cached for 30 days with a versioned key — bumping the version invalidates stale classifications

### Generation Lock
A Supabase-backed lock prevents duplicate concurrent sector generations that would waste tokens. If a generation for a sector is already in progress, subsequent requests return a 503 without starting a new AI call. The lock expires automatically after 5 minutes.

### Admin Cost Dashboard
Every request — cache hit or fresh generation — is logged to an `analysis_log` table with full metadata (ticker, model, tokens, actual web searches used, estimated cost). An admin-only tab in the UI shows daily cost, model tier breakdown, section frequency, and the 60 most recent requests. The API endpoint returns 403 for any non-admin email — the UI tab is in addition to, not instead of, server-side gating.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router, TypeScript) |
| Styling | Tailwind CSS, Framer Motion |
| AI | Anthropic Claude — `claude-sonnet-4-6`, `claude-haiku-4-5` |
| Market data | Yahoo Finance (`yahoo-finance2`) |
| Database | Supabase (PostgreSQL) |
| Auth | NextAuth.js — Google OAuth |
| Payments | Stripe (Checkout + Customer Portal + Webhooks) |
| Charts | Recharts |
| Deployment | Vercel |

---

## Infrastructure

- **Rate limiting** — tiered daily request limits (free vs. Pro) enforced server-side with per-user counters in Supabase
- **Global budget kill-switch** — hard daily spend ceiling across all users; blocks new AI calls when exceeded
- **Prompt caching** — system prompt cached at the Anthropic layer (1-hour TTL), reducing input token costs by ~80% on warm runs
- **Batch pre-warming** — Vercel Cron job submits analyses for popular tickers to Anthropic's Batch API overnight, so the shared cache is warm by market open

---

## Development Approach

This project was built using **Claude Code** (Anthropic's AI coding assistant) as a primary development tool alongside standard development practices.

**What AI helped with:**
- Architectural decisions — section routing, group-based search caps, generation lock design, deterministic scorecard logic
- Boilerplate-heavy work — Stripe webhook handling, NextAuth configuration, Supabase schema and query patterns
- Debugging edge cases — post-split price data issues, Yahoo Finance GICS mis-tagging, prompt caching stability
- Iterating on AI prompts for each analysis section to get structured, reliable JSON output

**What I (the developer) owned:**
- All product decisions — what sections to build, what the UX should feel like, what makes a good analysis
- All architectural trade-offs — e.g. choosing shared cache over per-user cache, deciding on group-based search caps vs per-section caps
- Feature scope, prioritization, and the overall vision
- Final review and approval of every implementation

I treat AI-assisted development the same way I would treat using Stack Overflow, documentation, or a senior colleague for code review — it accelerates the work, but the decisions and understanding remain with the engineer.

---

*Built by Aurel Genda · 2024–2025*
