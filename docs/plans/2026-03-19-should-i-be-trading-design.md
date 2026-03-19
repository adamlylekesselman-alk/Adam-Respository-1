# "Should I Be Trading?" — Design Document

**Date:** 2026-03-19
**Status:** Approved

---

## Overview

A live, auto-refreshing market dashboard that evaluates the current stock market environment and outputs a clear YES / CAUTION / NO trading decision for swing traders. Styled as a neon cyberpunk Bloomberg Terminal.

---

## Deliverable

Single file: `trading-dashboard.html`
Stack: Vanilla HTML/CSS/JS — no dependencies, no build step, no server

---

## Visual Design

### Aesthetic
- Neon cyberpunk Bloomberg Terminal
- Background: `#050a0e` (near-black)
- Primary neon: `#00ff88` (electric green), `#00d4ff` (cyan)
- Warning: `#ffaa00` (amber)
- Danger: `#ff3366` (red)
- Font: `'Courier New', monospace`
- Panel borders: `1px solid rgba(0,255,136,0.3)` with neon box-shadow
- CSS `text-shadow` and `box-shadow` for glow on key values
- Scanline overlay: `repeating-linear-gradient` for CRT feel
- Pulsing dot animation for LIVE indicator
- SVG arc-based animated circular gauges

### Layout (top to bottom)

1. **Top Bar**
   - Scrolling ticker: SPY, QQQ, VIX, DXY, TNX, all 11 sectors — CSS `@keyframes` scroll
   - `● LIVE` pulsing dot + last-updated timestamp ("updated 12s ago")
   - `[↻ Refresh]` button
   - `[SWING] [DAY]` mode toggle

2. **FOMC Alert Banner** (conditional)
   - Glowing amber full-width banner when FOMC within 72 hours

3. **Hero Panel**
   - Left: Decision badge — large text YES / CAUTION / NO with color-matched neon glow
   - Right: Two SVG circular gauges side by side
     - Market Quality Score (0–100%)
     - Execution Window Score (0–100%)

4. **Data Grid** (2×3, last row is 2 wide)
   - Panel 1: Volatility
   - Panel 2: Trend
   - Panel 3: Momentum
   - Panel 4: Breadth
   - Panel 5: Macro/Liquidity
   - Each panel: title, current values with ↑↓→ arrows, color status, score

5. **Sector Heatmap**
   - 11 sector ETFs sorted by 5d return
   - Horizontal gradient bars (green → red)
   - Labels: ticker, name, return %, leader/laggard tag

6. **Scoring Breakdown**
   - One row per category: name, weight, score, weighted pts
   - Neon glow progress bar showing score
   - Total row: Market Quality Score

7. **Terminal Analysis**
   - Monospace text block
   - Typewriter animation on each refresh
   - Generated from score state (rule-based templates, not AI API)

---

## Data Sources

### Yahoo Finance (no key needed)
Endpoint: `https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1d&range=1y`

Tickers fetched:
- `SPY` — S&P 500 ETF
- `QQQ` — Nasdaq ETF
- `^VIX` — CBOE Volatility Index
- `^TNX` — 10-Year Treasury yield
- `DX-Y.NYB` — Dollar Index
- All 11 sectors: `XLK, XLF, XLE, XLV, XLI, XLY, XLP, XLU, XLB, XLRE, XLC`

Data extracted:
- Current price (most recent close)
- 252-day close history (for MA calculations and percentiles)
- Volume history

Calculated client-side:
- 20/50/200-day simple moving averages
- 14-day RSI (Wilder's smoothing)
- 5-day slope (linear regression)
- VIX 1-year percentile rank
- ATR (14-day average true range)

### FRED API (no key needed for public series)
- `DGS10` — 10-Year Treasury yield (cross-reference TNX)

### Derived / Estimated (no API)
- Put/Call ratio: estimated from VIX regime (VIX < 15 → low fear, 15–25 → neutral, > 25 → elevated fear)
- Breadth (% above MAs): calculated across 11 sector ETFs as proxy (true breadth requires paid data)
- Fed stance: inferred from Fed Funds rate trend (rising = hawkish, falling = dovish, flat = neutral) — labeled manually as starting point
- FOMC calendar: hardcoded 2026 FOMC meeting dates, checked against `new Date()` for within-72h flag

---

## Scoring Formulas

All scores are clamped 0–100.

### Volatility Score (weight: 25%)
```
vixLevel  = 100 - clamp(VIX × 2.5, 0, 100)         // VIX 40 → 0
percentile = 100 - VIX_1yr_percentile                // higher pct → worse
slope      = VIX_5d_slope < 0 ? 70 : 30              // falling = good
volatilityScore = vixLevel×0.5 + percentile×0.3 + slope×0.2
```

### Trend Score (weight: 20%)
```
above200 = SPY_price > MA200 ? 40 : 0
above50  = SPY_price > MA50  ? 30 : 0
above20  = SPY_price > MA20  ? 20 : 0
rsiScore = clamp((RSI14 - 30) / 40 × 10, 0, 10)
trendScore = above200 + above50 + above20 + rsiScore
```

### Momentum Score (weight: 25%)
```
sectorBreadth = (sectors_with_positive_5d_return / 11) × 50
rsiMomentum   = clamp((QQQ_RSI - 40) / 30 × 30, 0, 30)
leadLag       = (top3_avg_return - bottom3_avg_return) > 3 ? 20 : 10
momentumScore = sectorBreadth + rsiMomentum + leadLag
```

### Breadth Score (weight: 20%)
```
aboveMA20 = (sectors_above_20d_MA / 11) × 50
aboveMA50 = (sectors_above_50d_MA / 11) × 50
breadthScore = aboveMA20×0.4 + aboveMA50×0.6
```

### Macro Score (weight: 10%)
```
yieldTrend  = TNX_5d_slope > 0 ? 30 : 70             // rising rates = bad
dxyTrend    = DXY_5d_slope > 0 ? 30 : 70             // rising dollar = bad
fedStance   = { dovish: 80, neutral: 60, hawkish: 30 }
macroScore  = yieldTrend×0.35 + dxyTrend×0.35 + fedStance×0.30
```

### Market Quality Score
```
MQS = volatility×0.25 + momentum×0.25 + trend×0.20 + breadth×0.20 + macro×0.10
```

### Decision
```
MQS >= 80 → YES (green glow)
MQS >= 60 → CAUTION (amber glow)
MQS <  60 → NO (red glow)
```

### Execution Window Score (separate, not in MQS weighting)
```
closeVsHigh  = clamp((SPY_close / SPY_intraday_high) × 100, 0, 100)
sectorCount  = (positive_sector_count / 11) × 100
qqqRSI       = clamp(QQQ_RSI, 0, 100)
vixContract  = VIX_5d_slope < 0 ? 80 : 40
trendExt     = clamp(100 - ((SPY_close - MA20) / MA20 × 1000), 0, 100)
EWS = avg(closeVsHigh, sectorCount, qqqRSI, vixContract, trendExt)
```

---

## Mode Toggle

### Swing Mode (default)
- Thresholds as described above

### Day Mode
- VIX multiplier tightened: `VIX × 3` instead of `× 2.5`
- Momentum weight: 30%, Trend weight: 15% (redistributed)
- Decision badge subtitle changes: "Full day trade" / "Scalps only" / "Stay flat"

---

## Terminal Analysis (Rule-Based)

Generated from score state, no external AI API call needed:

```
function generateAnalysis(scores, mode) {
  const parts = []
  if (scores.trend > 70) parts.push("Strong trend structure")
  else if (scores.trend > 50) parts.push("Mixed trend signals")
  else parts.push("Weak or broken trend")

  if (scores.breadth > 70) parts.push("with broad participation")
  else if (scores.breadth > 50) parts.push("with selective participation")
  else parts.push("with narrow or deteriorating breadth")

  // ... etc for volatility, macro, top sectors
  return parts.join(". ") + "."
}
```

---

## Auto-Refresh

```javascript
const REFRESH_INTERVAL = 45 * 1000  // 45 seconds
let lastUpdated = Date.now()

setInterval(async () => {
  showSkeletons()
  await fetchAll()
  score()
  render()
  lastUpdated = Date.now()
}, REFRESH_INTERVAL)

// Update "updated Xs ago" counter every second
setInterval(() => updateTimestamp(), 1000)
```

---

## FOMC Calendar (2026)

Hardcoded dates (next meeting from today triggers alert):
```
Jan 28–29, Mar 18–19, Apr 29–30, Jun 17–18,
Jul 29–30, Sep 16–17, Oct 28–29, Dec 9–10
```

---

## Error Handling

- Each fetch wrapped in try/catch
- Failed fetches show `N/A` with amber color in panel
- If >2 of 5 category scores fail, decision changes to CAUTION with "(data incomplete)" note
- Console warnings logged per failed symbol

---

## File Structure

```
trading-dashboard.html
  <style>          — CSS variables, layout, neon effects, animations
  <body>           — HTML structure (all 7 zones)
  <script>
    CONFIG         — symbols, weights, thresholds, FOMC dates
    DataFetcher    — fetch + normalize functions per source
    Calculator     — MA, RSI, percentile, slope math
    Scorer         — category score + MQS + EWS computation
    UIRenderer     — DOM update functions, SVG gauge drawing
    Analysis       — rule-based text generation
    App            — init, refresh loop, event handlers
```
