---
name: x-stock-sentiment
description: Scrape one or more X (Twitter) accounts over a recent window, classify each stock ticker they mention as bullish or bearish, rank tickers by repetition and exaggeration, find cross-account agreement/conflict, and write a dated analysis note into the Obsidian vault. Use when the user wants to analyze trader/finance X accounts for stock sentiment, e.g. "read the last two weeks of @The_RockTrading and @pdicarlotrader and tell me what's bullish/bearish".
---

# X Stock Sentiment Analyzer

Read recent tweets from finance/trader X accounts, derive per-ticker bullish/bearish
sentiment, and produce a structured Obsidian note.

## Inputs (ask the user only if not given)

- **accounts** — X handles to analyze. Default: `@The_RockTrading`, `@pdicarlotrader`.
- **window** — how far back. Default: **14 days** (last two weeks).
- **vault** — Obsidian vault path. Default: `/Users/amir.n/Documents/Obsidian Vault`.
  Output folder: `<vault>/Trading/X Sentiment/`.
- **output mode** — dated note per run (default) vs. single rolling note.

Compute the cutoff date = today − window. Only tweets with a timestamp **on or after**
the cutoff count. Today's date is available from the session context.

## Tooling

Uses the **Playwright MCP** browser (persistent Chrome profile, already logged in).
Load these tools via ToolSearch if not present: `mcp__playwright__browser_navigate`,
`mcp__playwright__browser_evaluate`, `mcp__playwright__browser_press_key`,
`mcp__playwright__browser_snapshot`, `mcp__playwright__browser_click`,
`mcp__playwright__browser_wait_for`.

**Large results:** `browser_evaluate` output over ~60k chars is rejected. When returning
many tweets, pass a `filename` (e.g. `rock_tweets.json`) so the result is written to a
file, then query it with `jq` via Bash instead of returning it inline. Strip `engagement`
and collapse whitespace (`text.replace(/\n+/g,' ').trim()`) to keep it compact. Clean up
these temp JSON files when done.

## Procedure

### 1. Collect tweets per account

For each account:

1. Navigate to `https://x.com/<handle>`.
2. Extract tweets by running `browser_evaluate` against the DOM rather than parsing the
   a11y snapshot — it's faster and gives you the `datetime` attribute. Use a function like:

   ```js
   () => {
     const out = [];
     document.querySelectorAll('article').forEach(a => {
       const t = a.querySelector('time');
       const txtEl = a.querySelector('[data-testid="tweetText"]');
       const socy = a.querySelector('[role="group"]')?.getAttribute('aria-label') || '';
       if (!t) return;
       out.push({
         iso: t.getAttribute('datetime'),
         text: txtEl ? txtEl.innerText : '',
         engagement: socy,           // "N replies, N reposts, N likes, ..."
         isPinned: !!a.querySelector('[data-testid="socialContext"]')
                     ?.innerText?.includes('Pinned'),
       });
     });
     return out;
   }
   ```

3. Scroll and re-extract inside a **single async `browser_evaluate`** that loops:
   `grab()` the DOM, `window.scrollBy(0, 3000)`, `await sleep(~1300ms)`, merge into a
   `Map` keyed by `iso + text`. Stop when the oldest non-pinned tweet is **older than the
   cutoff**, or after ~4–6 scrolls with no new tweets. High-volume accounts (30–50
   posts/day) may need 40–60 scrolls to cover 14 days — set the cap accordingly.
   **Ignore pinned tweets** whose real post date is outside the window. (Note: X
   virtualizes the DOM, so you must accumulate *during* one continuous scroll pass — a
   fresh `browser_evaluate` after the page is already scrolled only sees what's on screen.)
4. Keep only tweets with `iso >= cutoff`. Note the count kept and the oldest/newest
   dates covered so you can report actual coverage.

**Rate-limit fallback (common):** the profile timeline often returns *"Something went
wrong. Try reloading."* after heavy scraping. When that happens, use the **search-live
timeline** instead, which is more robust: navigate to
`https://x.com/search?q=from%3A<handle>&f=live`. It supports the same DOM extraction and
also accepts date bounds (`since:`/`until:`) and cashtag filters — used below.

Reposts/retweets of others count as endorsement of that content; quote-tweets — judge by
the author's own added text. If an account posts images/charts with little text, note it
as "chart-only, sentiment inferred from caption/cashtags".

### 2. Extract tickers & classify sentiment

For every kept tweet:

- Find tickers: `$` cashtags (`\$[A-Z]{1,5}`) first; also map obvious company names to
  tickers (Micron→MU, Nvidia→NVDA, Palantir→PLTR, etc.) when unambiguous.
- Classify sentiment toward **each** ticker in that tweet as `bullish`, `bearish`, or
  `neutral`, using context — not just keywords:
  - Bullish cues: breakout, base building, accumulate, target/upside, "loading",
    "generational", "to the moon", long, calls, buy, higher.
  - Bearish cues: breakdown, top, overvalued, short, puts, "avoid", "decay", "sell",
    lower, distribution.
  - A tweet may be bullish on one ticker and bearish on another — classify per ticker.
- Score **exaggeration** 0–3 per mention (hype/hyperbole intensity), from language like
  ALL CAPS, "generational wealth", "insane", "10x", "guaranteed", excessive emojis/🚀,
  round-number moonshots. Neutral analytical framing = 0.

### 3. Aggregate per account

For each account, build a ticker table:

| Ticker | Mentions | Net sentiment | Bull/Bear split | Avg exaggeration | Sample dates |

- **Net sentiment** = sign of (bullish mentions − bearish mentions).
- Rank by **Mentions** (repetition) desc, then avg exaggeration desc.
- Bucket into: **Conviction plays** (mentions ≥3, low-mid exaggeration),
  **Hype flags** (avg exaggeration ≥2), **One-offs** (single mention).

### 4. Cross-account comparison

Intersect tickers mentioned by ≥2 accounts:

- **✅ Agreement — both bullish**: highlight (both accounts net-bullish). These are the
  strongest shared signals.
- **⚠️ Conflict — one bullish / one bearish**: highlight prominently (disagreement).
- **➖ Mixed/neutral**: one has a stance, other neutral — list but don't highlight.

### 5. Write the Obsidian note

Write to `<vault>/Trading/X Sentiment/X Sentiment <YYYY-MM-DD>.md` (dated mode) or
`<vault>/Trading/X Sentiment/X Sentiment.md` (rolling mode). Create the folder if needed.

Use this structure (Obsidian-flavored markdown; use `> [!note]` callouts and `==highlight==`
for the flagged tickers):

```markdown
---
generated: <YYYY-MM-DD>
window: <cutoff> → <today>
accounts: [<handles>]
tweets_analyzed: <n>
---

# X Stock Sentiment — <date range>

> [!warning] Not financial advice. Sentiment is auto-derived from public tweets; verify before acting.

## 🔑 Highlights

> [!success] Both bullish (shared conviction)
> - ==$TICK== — @a (Nx bull), @b (Mx bull)

> [!danger] Conflict — bull vs bear
> - ==$TICK== — @a bullish (Nx) vs @b bearish (Mx)

## Per-account breakdown
### @handle  (<kept> tweets, <oldest>→<newest>)
<ticker table>
**Conviction plays:** …  **Hype flags:** …

## Shared tickers matrix
| Ticker | @a | @b | Verdict |

## Method & caveats
- Coverage actually retrieved, any gaps, exaggeration scoring note.
```

### 6. Report back

In chat, give the user: the note path, the ✅ agreement list, the ⚠️ conflict list, and
top repeated tickers per account. Keep it tight.

## Caveats to always surface

- X may rate-limit / lazy-load; report the real date range covered, not the target.
- Sentiment/company-name mapping is heuristic — flag low-confidence calls.
- Always include the "not financial advice" callout.
