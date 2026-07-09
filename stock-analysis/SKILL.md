---
name: stock-analysis
description: Analyze a single stock/company named by the user (ticker or company name) and produce a structured research report covering company overview, financials, valuation, price trend, and news/risks. Use when the user names a specific stock or company and asks to analyze, research, or evaluate it.
---

## Scope

Triggers when the user names a specific stock (ticker or company name) and asks for analysis, research, an overview, or an opinion on it. Not for general market commentary or sector-wide analysis with no named company. If the user names multiple stocks, repeat this template once per stock.

If the ticker/company name is ambiguous (common name, multiple exchange listings), ask the user to confirm which company/exchange before proceeding.

## Data sources

No live market-data API is configured in this environment by default. Gather data via WebSearch / WebFetch from public sources: company investor-relations pages, financial data sites (Yahoo Finance, Google Finance, SET SMART for Thai-listed stocks, MarketWatch, Reuters), and recent news outlets.

- Always state the data retrieval date next to any figure.
- Never fabricate numbers. If a figure can't be found, say so explicitly rather than guessing or estimating silently.
- Flag that fetched prices/figures may be delayed and should be verified against a live source before acting on them.
- If the user has a live market-data API or MCP tool available in the session, prefer that over WebSearch.

## Report template

Respond in the user's language, using this structure:

1. **Company overview** — sector/industry, core business, market cap, exchange, recent notable events.
2. **Financials & key ratios** — revenue and net profit trend (last 3–5 periods if available), P/E, P/BV, ROE, D/E, dividend yield. Cite the period each figure is from.
3. **Price trend** — recent price range, 52-week high/low, notable trend if found in sourced data. Do not invent precise technical-indicator values (RSI, MACD, etc.) without a cited source.
4. **Recent news & risk factors** — material news from roughly the last 3–6 months; competitive, regulatory, or macro risks specific to this company.
5. **Summary** — balanced synthesis of strengths vs. risks, ending with an explicit disclaimer that this is informational research, not investment advice, and figures should be verified before acting.

## Rules

- Do not issue a buy/sell/hold recommendation. Present balanced facts and let the user judge. If asked directly for a call, restate the disclaimer and frame any view as opinion, not advice.
- Keep the tone factual and cite where each material figure came from (source name, not full URL dump) so the user can spot-check.
