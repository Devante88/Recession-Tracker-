# Recession Tracker Build Pack v2.0

**Status:** Spec ready for execution
**Architecture:** GitHub Pages + GitHub Actions nightly refresh
**Scope:** Production-grade, MVP-runnable, portfolio-visible
**Date:** May 2026

---

## 0. What this is and what changed from v1

This is a recession monitoring web app. It pulls macroeconomic and microeconomic indicators from FRED, normalizes them to 0–100 recession risk scores, aggregates into five layers and one composite score, and publishes a dashboard. Live monitoring only. No backtest.

**What changed from v1:**

The browser no longer talks to FRED. GitHub Actions does. On a schedule, the runner fetches FRED server-side, runs the scoring engine in Node, and commits JSON snapshots into the `data/` directory. GitHub Pages serves the static front-end which reads those JSON files from its own origin. This solves CORS structurally (no proxy needed) and keeps the FRED API key in GitHub Secrets where it belongs.

The backtest module is removed entirely. History is built up by accumulation: each scheduled run appends a new snapshot to `data/history.json`. After a few weeks you have a real history; after a year you have an annual chart drawn from actual model output, not synthetic noise.

Three registry corrections from v1: `AMTMNO` replaced with `NEWORDER` (the v1 spec misidentified it as ISM PMI New Orders), JOLTS Job Openings threshold lowered from 7M to 5.5M (the v1 threshold triggers green permanently at current levels), and Baa-10Y credit spread threshold raised from 2.5 to 3.0 (v1 was too tight against historical credit-stress periods).

---

## 1. Architecture

```
┌──────────────────────────┐
│   GitHub Actions runner  │
│   (scheduled cron)       │
│                          │
│   • Reads FRED_API_KEY   │
│     from Secrets         │
│   • Fetches all series   │
│   • Runs scoring engine  │
│   • Writes data/*.json   │
│   • Commits to repo      │
└──────────┬───────────────┘
           │ git commit
           ▼
┌──────────────────────────┐
│   GitHub repo            │
│                          │
│   data/current.json      │
│   data/history.json      │
└──────────┬───────────────┘
           │ GitHub Pages serves
           ▼
┌──────────────────────────┐
│   Browser (any user)     │
│                          │
│   • Loads index.html     │
│   • Fetches data/*.json  │
│     from same origin     │
│   • Renders dashboard    │
│                          │
│   No API key. No CORS.   │
└──────────────────────────┘
```

**Schedule:** Twice weekly (Tuesday and Saturday at 13:00 UTC). Most series are monthly. Daily series like the yield curve update on business days. Twice-weekly captures fresh daily prints without hammering FRED.

---

## 2. Repository structure

```
recession-tracker/
├── .github/workflows/
│   ├── refresh.yml             # Scheduled FRED fetch
│   └── test.yml                # CI: run tests on push
├── data/
│   ├── current.json            # Latest snapshot
│   └── history.json            # Append-only snapshot history
├── scripts/
│   └── fetch.mjs               # The Actions entrypoint
├── src/
│   ├── registry.mjs            # Indicator definitions
│   ├── scoring.mjs             # Pure scoring functions
│   └── fred.mjs                # FRED API client
├── tests/
│   ├── scoring.test.mjs        # Pure function tests
│   └── registry.test.mjs       # Schema + weight validation
├── public/                     # GitHub Pages serves this directory
│   ├── index.html
│   ├── app.mjs
│   └── styles.css
├── SKILL.md                    # Agent Skills framework entry
├── README.md
├── package.json
└── .gitignore
```

---

## 3. Build sequence

Four sessions. Each ends with passing tests and an atomic commit. Do not start session N+1 until session N's QA gate passes.

| Session | Scope | QA gate |
|---------|-------|---------|
| 1 | Scaffolding, registry, registry tests | `node --test tests/registry.test.mjs` passes; all weights sum to documented totals |
| 2 | Scoring engine, scoring tests | `node --test` passes all tests; coverage of normalizeIndicator, computeLayerScores, computeCompositeScore, alertState |
| 3 | FRED client, fetch script, Actions workflows | Workflow runs successfully in GitHub Actions; `data/current.json` written with valid schema |
| 4 | Front-end dashboard | Dashboard loads on GitHub Pages, displays composite score, layer cards, indicator table, history chart (empty on first run is expected) |

---

## 4. Session 1: Scaffolding + Registry

### 4.1 Files to create

**`package.json`**

```json
{
  "name": "recession-tracker",
  "version": "1.0.0",
  "type": "module",
  "private": true,
  "engines": {
    "node": ">=20"
  },
  "scripts": {
    "test": "node --test tests/",
    "fetch": "node scripts/fetch.mjs"
  }
}
```

No dependencies. Node 20+ has native fetch, native test runner, native ESM.

**`.gitignore`**

```
node_modules/
.env
.DS_Store
*.log
```

**`src/registry.mjs`**

```js
// Indicator registry. Each entry defines a FRED series and how to score it.
//
// Fields:
//   name         Human-readable label
//   fred_id      FRED series identifier
//   layer        One of: financial_lead | labor | inflation | real_economy | micro
//   frequency    daily | weekly | monthly | quarterly
//   direction    direct (higher = good)  |  inverse (higher = bad)
//   weight       Within-layer weight (layer weights sum within layer)
//   threshold    Numeric trigger level, or null for z-score normalization
//   category     macro | micro
//   description  Human note

export const REGISTRY = [
  // ─── Financial leading indicators ─────────────────────────────────────
  { name: "Yield Curve 10Y-3M", fred_id: "T10Y3M", layer: "financial_lead", frequency: "daily", direction: "direct", weight: 0.35, threshold: 0.0, category: "macro", description: "Spread between 10Y and 3M Treasury yields. Inversion historically precedes recession." },
  { name: "Yield Curve 10Y-2Y", fred_id: "T10Y2Y", layer: "financial_lead", frequency: "daily", direction: "direct", weight: 0.20, threshold: 0.0, category: "macro", description: "Spread between 10Y and 2Y Treasury yields. Classic recession signal." },
  { name: "Baa-10Y Credit Spread", fred_id: "BAA10YM", layer: "financial_lead", frequency: "monthly", direction: "inverse", weight: 0.25, threshold: 3.0, category: "macro", description: "Corporate spread as a credit-stress proxy. Threshold calibrated to widening regimes." },
  { name: "Chicago Fed NFCI", fred_id: "NFCI", layer: "financial_lead", frequency: "weekly", direction: "inverse", weight: 0.20, threshold: 0.5, category: "macro", description: "Broad financial conditions index. Positive = tighter than average." },

  // ─── Labor ─────────────────────────────────────────────────────────────
  { name: "Unemployment Rate", fred_id: "UNRATE", layer: "labor", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "macro", description: "Headline U3 unemployment. Z-score normalized." },
  { name: "Sahm Rule (Real-Time)", fred_id: "SAHMREALTIME", layer: "labor", frequency: "monthly", direction: "inverse", weight: 0.25, threshold: 0.5, category: "macro", description: "Confirmatory (not leading): triggers at 0.5 when unemployment 3mo avg rises 0.5pp above its 12mo low." },
  { name: "Initial Jobless Claims", fred_id: "ICSA", layer: "labor", frequency: "weekly", direction: "inverse", weight: 0.20, threshold: 300000, category: "macro", description: "Fast labor deterioration signal. Threshold = sustained recessionary level." },
  { name: "Payroll Employment", fred_id: "PAYEMS", layer: "labor", frequency: "monthly", direction: "direct", weight: 0.20, threshold: null, category: "macro", description: "Nonfarm payroll trend. Z-score normalized." },
  { name: "JOLTS Quits Rate", fred_id: "JTSQUR", layer: "labor", frequency: "monthly", direction: "direct", weight: 0.15, threshold: 2.0, category: "micro", description: "Worker confidence proxy. Falling quits = labor market cooling." },

  // ─── Inflation ─────────────────────────────────────────────────────────
  { name: "CPI All Items YoY", fred_id: "CPIAUCSL", layer: "inflation", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "macro", description: "Headline CPI. Z-score normalized over trailing window." },
  { name: "Core CPI YoY", fred_id: "CPILFESL", layer: "inflation", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "macro", description: "Core CPI. Z-score normalized." },
  { name: "PCE Price Index", fred_id: "PCEPI", layer: "inflation", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "macro", description: "Fed's preferred inflation measure. Z-score normalized." },
  { name: "Fed Funds Rate", fred_id: "FEDFUNDS", layer: "inflation", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "macro", description: "Policy rate level. Z-score normalized." },
  { name: "Avg Hourly Earnings", fred_id: "CES0500000003", layer: "inflation", frequency: "monthly", direction: "inverse", weight: 0.20, threshold: null, category: "micro", description: "Wage pressure signal. Z-score normalized." },

  // ─── Real economy ──────────────────────────────────────────────────────
  { name: "Real GDP", fred_id: "GDPC1", layer: "real_economy", frequency: "quarterly", direction: "direct", weight: 0.25, threshold: null, category: "macro", description: "Real output. Z-score on QoQ change." },
  { name: "Industrial Production", fred_id: "INDPRO", layer: "real_economy", frequency: "monthly", direction: "direct", weight: 0.20, threshold: null, category: "macro", description: "Goods-sector output. Z-score normalized." },
  { name: "Retail Sales", fred_id: "RSAFS", layer: "real_economy", frequency: "monthly", direction: "direct", weight: 0.20, threshold: null, category: "macro", description: "Consumer demand. Z-score normalized." },
  { name: "Real Income ex Transfers", fred_id: "W875RX1", layer: "real_economy", frequency: "monthly", direction: "direct", weight: 0.20, threshold: null, category: "macro", description: "Earned household income. NBER coincident series." },
  { name: "Housing Starts", fred_id: "HOUST", layer: "real_economy", frequency: "monthly", direction: "direct", weight: 0.15, threshold: null, category: "micro", description: "Rate-sensitive leading housing signal." },

  // ─── Micro ─────────────────────────────────────────────────────────────
  { name: "Manufacturers New Orders", fred_id: "NEWORDER", layer: "micro", frequency: "monthly", direction: "direct", weight: 0.20, threshold: null, category: "micro", description: "Census new orders ex-defense. Goods-sector proxy (not ISM PMI; that's gated)." },
  { name: "Bank Lending Standards (C&I)", fred_id: "DRTSCILM", layer: "micro", frequency: "quarterly", direction: "inverse", weight: 0.20, threshold: 20.0, category: "micro", description: "Net % of banks tightening C&I loan standards. Survey." },
  { name: "Consumer Credit Delinquency", fred_id: "DRCCLACBS", layer: "micro", frequency: "quarterly", direction: "inverse", weight: 0.20, threshold: 3.0, category: "micro", description: "Credit card delinquency rate. Household stress." },
  { name: "Small Business Optimism", fred_id: "NFIBOPTMI", layer: "micro", frequency: "monthly", direction: "direct", weight: 0.20, threshold: 95.0, category: "micro", description: "NFIB Small Business Optimism Index." },
  { name: "JOLTS Job Openings", fred_id: "JTSJOL", layer: "micro", frequency: "monthly", direction: "direct", weight: 0.20, threshold: 5500000, category: "micro", description: "Employer demand. Threshold calibrated to recessionary trough range." }
];

export const LAYER_WEIGHTS = {
  financial_lead: 0.30,
  labor: 0.25,
  inflation: 0.15,
  real_economy: 0.20,
  micro: 0.10
};

export function getFredIds() {
  return REGISTRY.map(x => x.fred_id);
}

export function getIndicatorsByLayer(layer) {
  return REGISTRY.filter(x => x.layer === layer);
}

export function validateLayerWeights() {
  const sums = {};
  for (const item of REGISTRY) sums[item.layer] = (sums[item.layer] || 0) + item.weight;
  const result = { valid: true, layers: {} };
  for (const [layer, sum] of Object.entries(sums)) {
    const rounded = Number(sum.toFixed(4));
    result.layers[layer] = { sum: rounded, ok: Math.abs(rounded - 1.0) < 0.01 };
    if (!result.layers[layer].ok) result.valid = false;
  }
  return result;
}

export function validateCompositeWeights() {
  const sum = Object.values(LAYER_WEIGHTS).reduce((a, b) => a + b, 0);
  return { sum: Number(sum.toFixed(4)), ok: Math.abs(sum - 1.0) < 0.01 };
}
```

**`tests/registry.test.mjs`**

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { REGISTRY, LAYER_WEIGHTS, validateLayerWeights, validateCompositeWeights, getFredIds, getIndicatorsByLayer } from '../src/registry.mjs';

test('registry has expected layers', () => {
  const layers = new Set(REGISTRY.map(x => x.layer));
  assert.deepEqual([...layers].sort(), ['financial_lead', 'inflation', 'labor', 'micro', 'real_economy']);
});

test('every indicator has required fields', () => {
  const required = ['name', 'fred_id', 'layer', 'frequency', 'direction', 'weight', 'category', 'description'];
  for (const item of REGISTRY) {
    for (const field of required) {
      assert.ok(item[field] !== undefined, `${item.name} missing ${field}`);
    }
  }
});

test('direction values are valid', () => {
  for (const item of REGISTRY) {
    assert.ok(['direct', 'inverse'].includes(item.direction), `${item.name} has bad direction ${item.direction}`);
  }
});

test('frequency values are valid', () => {
  for (const item of REGISTRY) {
    assert.ok(['daily', 'weekly', 'monthly', 'quarterly'].includes(item.frequency), `${item.name} bad frequency`);
  }
});

test('layer weights sum to 1.0 within each layer', () => {
  const result = validateLayerWeights();
  assert.equal(result.valid, true, `Layer weights invalid: ${JSON.stringify(result.layers)}`);
});

test('LAYER_WEIGHTS sums to 1.0', () => {
  const result = validateCompositeWeights();
  assert.equal(result.ok, true, `Composite weights sum = ${result.sum}`);
});

test('getFredIds returns array of strings', () => {
  const ids = getFredIds();
  assert.ok(Array.isArray(ids));
  assert.ok(ids.every(x => typeof x === 'string'));
});

test('getIndicatorsByLayer filters correctly', () => {
  const labor = getIndicatorsByLayer('labor');
  assert.ok(labor.length > 0);
  assert.ok(labor.every(x => x.layer === 'labor'));
});

test('no duplicate FRED IDs', () => {
  const ids = getFredIds();
  assert.equal(new Set(ids).size, ids.length, 'Duplicate FRED IDs detected');
});
```

### 4.2 QA gate for session 1

Run `node --test tests/registry.test.mjs`. All tests must pass. Commit message: `feat(registry): initial indicator registry with layer/composite weight validation`.

### 4.3 Build prompt for executing LLM

```text
Create the following files for a recession tracker repository:
- package.json with the exact content I provide
- .gitignore
- src/registry.mjs with the exact REGISTRY array and helper functions I provide
- tests/registry.test.mjs with the exact test cases I provide

Do not modify the indicator list, weights, or thresholds. Do not add explanations. Output the file contents only, clearly labeled by filename.
```

---

## 5. Session 2: Scoring engine

### 5.1 Files to create

**`src/scoring.mjs`**

```js
// Pure scoring functions. No I/O. Heavily tested.
//
// Pipeline:
//   raw FRED series → normalizeIndicator → score in [0, 1]
//   indicators → computeLayerScores → 5 layer scores in [0, 100]
//   layer scores → computeCompositeScore → 1 composite in [0, 100]
//   composite → alertState → GREEN | YELLOW | RED

import { LAYER_WEIGHTS } from './registry.mjs';

/**
 * Logistic squashing function. Maps real line → [0, 1].
 * @param {number} x
 * @param {number} k Steepness. Higher k = sharper transition.
 */
export function logistic(x, k = 4) {
  return 1 / (1 + Math.exp(-k * x));
}

/**
 * Compute mean and standard deviation of a numeric array.
 */
export function meanStd(values) {
  if (!values.length) return { mean: 0, std: 1 };
  const mean = values.reduce((a, b) => a + b, 0) / values.length;
  const variance = values.reduce((a, b) => a + (b - mean) ** 2, 0) / values.length;
  const std = Math.sqrt(variance) || 1;
  return { mean, std };
}

/**
 * Normalize one indicator's full series into recession risk scores in [0, 1].
 *
 * @param {Array<{date: string, value: number}>} series Time-ordered observations.
 * @param {object} indicator Registry entry.
 * @returns {{latest: object|null, score: number, series: Array}}
 */
export function normalizeIndicator(series, indicator) {
  if (!Array.isArray(series) || !series.length) {
    return { latest: null, score: 0, series: [] };
  }

  const { threshold, direction } = indicator;
  let normalized;

  if (threshold !== null && threshold !== undefined) {
    // Threshold-based: distance from threshold, scaled by |threshold|
    const denom = Math.abs(threshold) || 1;
    normalized = series.map(p => {
      const raw = direction === 'direct'
        ? (threshold - p.value) / denom    // direct: lower than threshold = bad = high score
        : (p.value - threshold) / denom;   // inverse: higher than threshold = bad = high score
      return { ...p, score: clamp01(logistic(raw)) };
    });
  } else {
    // Z-score normalization against the full window
    const { mean, std } = meanStd(series.map(x => x.value));
    normalized = series.map(p => {
      const z = (p.value - mean) / std;
      const raw = direction === 'direct' ? -z : z;
      return { ...p, score: clamp01(logistic(raw)) };
    });
  }

  return {
    latest: normalized[normalized.length - 1],
    score: normalized[normalized.length - 1]?.score ?? 0,
    series: normalized
  };
}

function clamp01(x) {
  return Math.max(0, Math.min(1, x));
}

/**
 * Aggregate normalized indicators into layer scores in [0, 100].
 * Within each layer, weights are applied as defined in the registry.
 */
export function computeLayerScores(indicators) {
  const grouped = {};
  for (const item of indicators) {
    if (!grouped[item.layer]) grouped[item.layer] = [];
    grouped[item.layer].push(item);
  }
  const scores = {};
  for (const [layer, items] of Object.entries(grouped)) {
    const weightSum = items.reduce((a, b) => a + b.weight, 0) || 1;
    const weighted = items.reduce((a, b) => a + (b.score || 0) * b.weight, 0);
    scores[layer] = Number(((weighted / weightSum) * 100).toFixed(1));
  }
  return scores;
}

/**
 * Aggregate layer scores into a single composite in [0, 100] using LAYER_WEIGHTS.
 */
export function computeCompositeScore(layerScores) {
  let weighted = 0;
  let totalWeight = 0;
  for (const [layer, score] of Object.entries(layerScores)) {
    const w = LAYER_WEIGHTS[layer] || 0;
    weighted += score * w;
    totalWeight += w;
  }
  return Number((weighted / (totalWeight || 1)).toFixed(1));
}

/**
 * Three-state alert. Thresholds are configuration, not validated empirically.
 * Documented in README as judgment calls pending future calibration.
 */
export function alertState(score) {
  if (score >= 60) return 'RED';
  if (score >= 30) return 'YELLOW';
  return 'GREEN';
}

/**
 * Top-level: take normalized indicators, produce the full snapshot payload.
 */
export function buildSnapshot(normalizedIndicators, asOfDate) {
  const layerScores = computeLayerScores(normalizedIndicators);
  const compositeScore = computeCompositeScore(layerScores);
  return {
    as_of: asOfDate || new Date().toISOString().slice(0, 10),
    generated_at: new Date().toISOString(),
    composite: {
      score: compositeScore,
      alert: alertState(compositeScore)
    },
    layers: Object.fromEntries(
      Object.entries(layerScores).map(([layer, score]) => [
        layer,
        { score, alert: alertState(score), weight: LAYER_WEIGHTS[layer] }
      ])
    ),
    indicators: normalizedIndicators.map(x => ({
      name: x.name,
      fred_id: x.fred_id,
      layer: x.layer,
      category: x.category,
      latest_value: x.latest?.value ?? null,
      latest_date: x.latest?.date ?? null,
      score: Number(((x.score || 0) * 100).toFixed(1)),
      alert: alertState((x.score || 0) * 100),
      threshold: x.threshold,
      direction: x.direction,
      frequency: x.frequency
    }))
  };
}
```

**`tests/scoring.test.mjs`**

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import {
  logistic, meanStd, normalizeIndicator,
  computeLayerScores, computeCompositeScore, alertState, buildSnapshot
} from '../src/scoring.mjs';

test('logistic returns 0.5 at zero', () => {
  assert.equal(logistic(0), 0.5);
});

test('logistic is bounded in [0, 1]', () => {
  for (const x of [-100, -10, -1, 0, 1, 10, 100]) {
    const y = logistic(x);
    assert.ok(y >= 0 && y <= 1, `logistic(${x}) = ${y} out of bounds`);
  }
});

test('meanStd on simple array', () => {
  const { mean, std } = meanStd([1, 2, 3, 4, 5]);
  assert.equal(mean, 3);
  assert.ok(Math.abs(std - Math.sqrt(2)) < 0.001);
});

test('meanStd on empty array does not crash', () => {
  const { mean, std } = meanStd([]);
  assert.equal(mean, 0);
  assert.equal(std, 1);
});

test('normalizeIndicator: threshold-based inverse direction (value > threshold = high score)', () => {
  const indicator = { threshold: 0.5, direction: 'inverse' };
  const series = [
    { date: '2025-01-01', value: 0.0 },   // below threshold = low risk
    { date: '2025-02-01', value: 1.0 }    // above threshold = high risk
  ];
  const { series: scored } = normalizeIndicator(series, indicator);
  assert.ok(scored[0].score < 0.5, 'value below threshold should score low');
  assert.ok(scored[1].score > 0.5, 'value above threshold should score high');
});

test('normalizeIndicator: threshold-based direct direction (value < threshold = high score)', () => {
  const indicator = { threshold: 0.0, direction: 'direct' };  // mimics yield curve
  const series = [
    { date: '2025-01-01', value: 0.5 },    // positive = healthy = low risk
    { date: '2025-02-01', value: -0.5 }    // inverted = recession signal = high risk
  ];
  const { series: scored } = normalizeIndicator(series, indicator);
  assert.ok(scored[0].score < 0.5, 'positive yield curve = low risk');
  assert.ok(scored[1].score > 0.5, 'inverted yield curve = high risk');
});

test('normalizeIndicator: z-score normalization (no threshold)', () => {
  const indicator = { threshold: null, direction: 'inverse' };
  const series = Array.from({ length: 20 }, (_, i) => ({
    date: `2025-${String(i + 1).padStart(2, '0')}-01`,
    value: i  // monotone increasing
  }));
  const { score } = normalizeIndicator(series, indicator);
  // Latest value is the max of the series → above mean → inverse direction → high score
  assert.ok(score > 0.7, `expected high score, got ${score}`);
});

test('normalizeIndicator: empty series returns zero score', () => {
  const { score, latest } = normalizeIndicator([], { threshold: 0, direction: 'direct' });
  assert.equal(score, 0);
  assert.equal(latest, null);
});

test('computeLayerScores aggregates within layer', () => {
  const indicators = [
    { layer: 'labor', weight: 0.5, score: 0.8 },
    { layer: 'labor', weight: 0.5, score: 0.2 }
  ];
  const scores = computeLayerScores(indicators);
  assert.equal(scores.labor, 50.0);
});

test('computeCompositeScore weights by LAYER_WEIGHTS', () => {
  const layerScores = {
    financial_lead: 100, labor: 0, inflation: 0, real_economy: 0, micro: 0
  };
  const composite = computeCompositeScore(layerScores);
  // financial_lead weight = 0.30, so composite = 30
  assert.equal(composite, 30.0);
});

test('alertState thresholds', () => {
  assert.equal(alertState(0), 'GREEN');
  assert.equal(alertState(29.9), 'GREEN');
  assert.equal(alertState(30), 'YELLOW');
  assert.equal(alertState(59.9), 'YELLOW');
  assert.equal(alertState(60), 'RED');
  assert.equal(alertState(100), 'RED');
});

test('buildSnapshot produces full payload schema', () => {
  const indicators = [
    { name: 'X', fred_id: 'X', layer: 'labor', category: 'macro', weight: 1.0,
      score: 0.5, latest: { date: '2025-01-01', value: 5 },
      threshold: null, direction: 'inverse', frequency: 'monthly' }
  ];
  const snap = buildSnapshot(indicators, '2025-05-18');
  assert.equal(snap.as_of, '2025-05-18');
  assert.ok(snap.composite);
  assert.ok(snap.layers);
  assert.ok(Array.isArray(snap.indicators));
  assert.equal(snap.indicators.length, 1);
});
```

### 5.2 QA gate for session 2

Run `node --test`. All session 1 and session 2 tests must pass. Commit message: `feat(scoring): pure scoring functions with full test coverage`.

### 5.3 Build prompt for executing LLM

```text
Create src/scoring.mjs and tests/scoring.test.mjs with the exact content I provide. Use Node's built-in test runner (no external test framework). Functions must be pure (no I/O, no side effects). Output file contents only.
```

---

## 6. Session 3: FRED client + fetch script + Actions workflows

### 6.1 Files to create

**`src/fred.mjs`**

```js
// FRED API client. Server-side only (runs in GitHub Actions, not browser).

const FRED_BASE = 'https://api.stlouisfed.org/fred/series/observations';

/**
 * Fetch a FRED series and return time-ordered observations.
 * @param {string} fredId
 * @param {string} apiKey
 * @param {object} opts
 * @returns {Promise<Array<{date: string, value: number}>>}
 */
export async function fetchFredSeries(fredId, apiKey, opts = {}) {
  const limit = opts.limit ?? 2000;  // ~5 years of daily, much more for monthly
  const url = `${FRED_BASE}?series_id=${fredId}&api_key=${apiKey}&file_type=json&sort_order=desc&limit=${limit}`;
  const res = await fetch(url);
  if (!res.ok) {
    throw new Error(`FRED fetch failed for ${fredId}: HTTP ${res.status}`);
  }
  const json = await res.json();
  const observations = json.observations || [];
  return observations
    .filter(x => x.value !== '.')
    .map(x => ({ date: x.date, value: Number(x.value) }))
    .filter(x => Number.isFinite(x.value))
    .reverse();  // ascending order
}

/**
 * Fetch all series with simple sequential calls and a small delay to be polite.
 * FRED's rate limit is generous (120 req/min) but sequential keeps us safe.
 */
export async function fetchAllSeries(fredIds, apiKey, delayMs = 200) {
  const results = {};
  for (const id of fredIds) {
    try {
      results[id] = await fetchFredSeries(id, apiKey);
    } catch (err) {
      console.error(`Error fetching ${id}: ${err.message}`);
      results[id] = [];
    }
    if (delayMs > 0) await new Promise(r => setTimeout(r, delayMs));
  }
  return results;
}
```

**`scripts/fetch.mjs`**

```js
// Entrypoint for the GitHub Actions runner.
// Reads FRED_API_KEY from env, fetches all series, computes the snapshot,
// writes data/current.json, appends to data/history.json.

import fs from 'node:fs/promises';
import path from 'node:path';
import { REGISTRY } from '../src/registry.mjs';
import { fetchAllSeries } from '../src/fred.mjs';
import { normalizeIndicator, buildSnapshot } from '../src/scoring.mjs';

const DATA_DIR = path.join(process.cwd(), 'data');

async function main() {
  const apiKey = process.env.FRED_API_KEY;
  if (!apiKey) {
    console.error('FRED_API_KEY not set');
    process.exit(1);
  }

  console.log(`Fetching ${REGISTRY.length} series...`);
  const fredIds = REGISTRY.map(x => x.fred_id);
  const rawData = await fetchAllSeries(fredIds, apiKey);

  const normalized = REGISTRY.map(indicator => {
    const series = rawData[indicator.fred_id] || [];
    const result = normalizeIndicator(series, indicator);
    return { ...indicator, ...result };
  });

  const asOf = new Date().toISOString().slice(0, 10);
  const snapshot = buildSnapshot(normalized, asOf);

  await fs.mkdir(DATA_DIR, { recursive: true });
  await fs.writeFile(
    path.join(DATA_DIR, 'current.json'),
    JSON.stringify(snapshot, null, 2)
  );

  // Append to history (compact form: just date + composite + layer scores)
  const historyPath = path.join(DATA_DIR, 'history.json');
  let history = [];
  try {
    const raw = await fs.readFile(historyPath, 'utf-8');
    history = JSON.parse(raw);
  } catch {
    history = [];
  }

  const historyEntry = {
    date: snapshot.as_of,
    composite: snapshot.composite.score,
    alert: snapshot.composite.alert,
    layers: Object.fromEntries(
      Object.entries(snapshot.layers).map(([k, v]) => [k, v.score])
    )
  };

  // Replace today's entry if it exists (idempotent), else append
  const idx = history.findIndex(x => x.date === historyEntry.date);
  if (idx >= 0) history[idx] = historyEntry;
  else history.push(historyEntry);

  await fs.writeFile(historyPath, JSON.stringify(history, null, 2));

  console.log(`Composite: ${snapshot.composite.score} (${snapshot.composite.alert})`);
  console.log(`Layers:`, snapshot.layers);
  console.log(`History entries: ${history.length}`);
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

**`.github/workflows/refresh.yml`**

```yaml
name: Refresh FRED Data

on:
  schedule:
    # Tuesday and Saturday at 13:00 UTC (08:00 CT)
    - cron: '0 13 * * 2,6'
  workflow_dispatch:  # allow manual trigger

permissions:
  contents: write

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Run tests
        run: node --test tests/

      - name: Fetch FRED data and compute snapshot
        env:
          FRED_API_KEY: ${{ secrets.FRED_API_KEY }}
        run: node scripts/fetch.mjs

      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "data: refresh snapshot $(date -u +%Y-%m-%d)"
            git push
          fi
```

**`.github/workflows/test.yml`**

```yaml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: node --test tests/
```

### 6.2 QA gate for session 3

Configure `FRED_API_KEY` in GitHub repo Secrets. Manually trigger the `Refresh FRED Data` workflow. It must complete green and produce `data/current.json` and `data/history.json` in the repo. Commit message: `feat(pipeline): fetch script + scheduled Actions workflow`.

### 6.3 Build prompt for executing LLM

```text
Create src/fred.mjs, scripts/fetch.mjs, .github/workflows/refresh.yml, and .github/workflows/test.yml with the exact content I provide. Use Node 20 native fetch (no external HTTP libraries). The fetch script must read FRED_API_KEY from process.env. Output file contents only.
```

---

## 7. Session 4: Front-end dashboard

### 7.1 Files to create

**`public/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Recession Tracker</title>
  <link rel="stylesheet" href="styles.css">
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <h1>Recession Tracker</h1>
        <p class="subtitle">Macro + micro indicators from FRED. Updated twice weekly.</p>
      </div>
      <div class="meta">
        <div>As of <span id="asOf">loading...</span></div>
        <div class="muted">Generated <span id="generatedAt">loading...</span></div>
      </div>
    </header>

    <section class="hero">
      <div class="hero-label">Composite Recession Risk</div>
      <div class="hero-score" id="compositeScore">--</div>
      <div class="hero-alert"><span class="badge" id="compositeAlert">--</span></div>
    </section>

    <section class="layers" id="layerCards"></section>

    <section class="panel">
      <h2>Composite History</h2>
      <p class="muted">History accumulates from each scheduled snapshot. Chart populates as data builds.</p>
      <div class="chart-wrap"><canvas id="historyChart"></canvas></div>
    </section>

    <section class="panel">
      <h2>Indicator Detail</h2>
      <div class="filter-row">
        <select id="layerFilter">
          <option value="">All layers</option>
        </select>
        <select id="categoryFilter">
          <option value="">Macro + micro</option>
          <option value="macro">Macro only</option>
          <option value="micro">Micro only</option>
        </select>
      </div>
      <table id="indicatorTable">
        <thead>
          <tr>
            <th>Indicator</th>
            <th>Layer</th>
            <th>Cat</th>
            <th>Latest</th>
            <th>Date</th>
            <th>Threshold</th>
            <th>Score</th>
            <th>Alert</th>
          </tr>
        </thead>
        <tbody id="indicatorTableBody"></tbody>
      </table>
    </section>

    <footer>
      <p class="muted">Data: <a href="https://fred.stlouisfed.org" target="_blank">FRED, Federal Reserve Bank of St. Louis</a>. Source: <a id="repoLink" href="#" target="_blank">GitHub</a>.</p>
    </footer>
  </div>

  <script type="module" src="app.mjs"></script>
</body>
</html>
```

**`public/styles.css`**

```css
:root {
  --bg: #0b1020;
  --panel: #121a2f;
  --panel-2: #1a2340;
  --border: rgba(255, 255, 255, 0.08);
  --text: #eef2ff;
  --text-muted: #8b95b8;
  --accent: #66b3ff;
  --green: #2ddc8c;
  --yellow: #f1c84a;
  --red: #ff7a7a;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Inter', 'Segoe UI', sans-serif;
  background: var(--bg);
  color: var(--text);
  line-height: 1.5;
}

.wrap {
  max-width: 1400px;
  margin: 0 auto;
  padding: 32px 24px;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  margin-bottom: 32px;
  flex-wrap: wrap;
  gap: 16px;
}

h1 {
  margin: 0 0 4px;
  font-size: 32px;
  font-weight: 800;
  letter-spacing: -0.02em;
}

.subtitle { margin: 0; color: var(--text-muted); }

.meta { text-align: right; font-size: 13px; }
.muted { color: var(--text-muted); font-size: 13px; }

.hero {
  background: linear-gradient(135deg, var(--panel), var(--panel-2));
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 32px;
  margin-bottom: 24px;
  text-align: center;
}

.hero-label {
  font-size: 14px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 12px;
}

.hero-score {
  font-size: 88px;
  font-weight: 900;
  line-height: 1;
  margin-bottom: 16px;
  letter-spacing: -0.04em;
}

.layers {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 16px;
  margin-bottom: 24px;
}

.layer-card {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 20px;
}

.layer-card h3 {
  margin: 0 0 8px;
  font-size: 13px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.layer-score {
  font-size: 36px;
  font-weight: 800;
  margin-bottom: 8px;
}

.layer-weight {
  font-size: 12px;
  color: var(--text-muted);
}

.panel {
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 24px;
}

.panel h2 {
  margin: 0 0 8px;
  font-size: 18px;
}

.chart-wrap {
  height: 320px;
  margin-top: 16px;
}

.filter-row {
  display: flex;
  gap: 12px;
  margin: 16px 0;
}

select {
  background: var(--panel-2);
  color: var(--text);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 14px;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 8px;
  font-size: 14px;
}

th, td {
  text-align: left;
  padding: 10px 12px;
  border-bottom: 1px solid var(--border);
}

th {
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  font-size: 11px;
  letter-spacing: 0.06em;
}

.badge {
  display: inline-block;
  padding: 4px 10px;
  border-radius: 999px;
  font-weight: 700;
  font-size: 11px;
  letter-spacing: 0.05em;
}

.GREEN { background: rgba(45, 220, 140, 0.15); color: var(--green); }
.YELLOW { background: rgba(241, 200, 74, 0.15); color: var(--yellow); }
.RED { background: rgba(255, 122, 122, 0.15); color: var(--red); }

footer {
  margin-top: 48px;
  padding-top: 24px;
  border-top: 1px solid var(--border);
  text-align: center;
}

footer a { color: var(--accent); }

@media (max-width: 1024px) {
  .layers { grid-template-columns: repeat(2, 1fr); }
  .hero-score { font-size: 64px; }
}

@media (max-width: 600px) {
  .layers { grid-template-columns: 1fr; }
  .hero { padding: 20px; }
  .hero-score { font-size: 48px; }
  table { font-size: 12px; }
  th, td { padding: 8px; }
}
```

**`public/app.mjs`**

```js
// Front-end dashboard. Reads static JSON, renders everything.

const LAYER_NAMES = {
  financial_lead: 'Financial Leading',
  labor: 'Labor',
  inflation: 'Inflation',
  real_economy: 'Real Economy',
  micro: 'Micro / Business'
};

const LAYER_COLORS = {
  financial_lead: '#66b3ff',
  labor: '#ff9966',
  inflation: '#f1c84a',
  real_economy: '#2ddc8c',
  micro: '#c084fc'
};

let currentSnapshot = null;
let historyData = [];

async function loadData() {
  const [currentRes, historyRes] = await Promise.all([
    fetch('../data/current.json'),
    fetch('../data/history.json')
  ]);

  if (!currentRes.ok) throw new Error('No snapshot found. Run the GitHub Action once to generate data.');

  currentSnapshot = await currentRes.json();
  historyData = historyRes.ok ? await historyRes.json() : [];

  render();
}

function render() {
  renderHeader();
  renderComposite();
  renderLayers();
  renderHistoryChart();
  renderIndicators();
  wireFilters();
  setRepoLink();
}

function renderHeader() {
  document.getElementById('asOf').textContent = currentSnapshot.as_of;
  document.getElementById('generatedAt').textContent = new Date(currentSnapshot.generated_at).toLocaleString();
}

function renderComposite() {
  document.getElementById('compositeScore').textContent = currentSnapshot.composite.score.toFixed(1);
  const badge = document.getElementById('compositeAlert');
  badge.textContent = currentSnapshot.composite.alert;
  badge.className = `badge ${currentSnapshot.composite.alert}`;
}

function renderLayers() {
  const container = document.getElementById('layerCards');
  container.innerHTML = Object.entries(currentSnapshot.layers)
    .map(([layer, data]) => `
      <div class="layer-card">
        <h3>${LAYER_NAMES[layer] || layer}</h3>
        <div class="layer-score">${data.score.toFixed(1)}</div>
        <div><span class="badge ${data.alert}">${data.alert}</span></div>
        <div class="layer-weight">Weight: ${(data.weight * 100).toFixed(0)}%</div>
      </div>
    `).join('');
}

function renderHistoryChart() {
  const ctx = document.getElementById('historyChart').getContext('2d');
  new Chart(ctx, {
    type: 'line',
    data: {
      labels: historyData.map(x => x.date),
      datasets: [
        {
          label: 'Composite',
          data: historyData.map(x => x.composite),
          borderColor: '#66b3ff',
          backgroundColor: 'rgba(102, 179, 255, 0.1)',
          borderWidth: 2.5,
          tension: 0.3,
          fill: true
        }
      ]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: {
        legend: { labels: { color: '#eef2ff' } }
      },
      scales: {
        x: { ticks: { color: '#8b95b8' }, grid: { color: 'rgba(255,255,255,0.05)' } },
        y: {
          min: 0, max: 100,
          ticks: { color: '#8b95b8' },
          grid: { color: 'rgba(255,255,255,0.05)' }
        }
      }
    }
  });
}

function renderIndicators() {
  const layerFilter = document.getElementById('layerFilter').value;
  const categoryFilter = document.getElementById('categoryFilter').value;

  const filtered = currentSnapshot.indicators.filter(x =>
    (!layerFilter || x.layer === layerFilter) &&
    (!categoryFilter || x.category === categoryFilter)
  );

  const body = document.getElementById('indicatorTableBody');
  body.innerHTML = filtered.map(x => `
    <tr>
      <td><strong>${x.name}</strong><div class="muted" style="font-size:11px">${x.fred_id}</div></td>
      <td>${LAYER_NAMES[x.layer] || x.layer}</td>
      <td>${x.category}</td>
      <td>${formatValue(x.latest_value)}</td>
      <td>${x.latest_date || '-'}</td>
      <td>${x.threshold !== null ? x.threshold : 'z-score'}</td>
      <td>${x.score.toFixed(1)}</td>
      <td><span class="badge ${x.alert}">${x.alert}</span></td>
    </tr>
  `).join('');
}

function formatValue(v) {
  if (v === null || v === undefined) return '-';
  if (Math.abs(v) >= 1e6) return (v / 1e6).toFixed(2) + 'M';
  if (Math.abs(v) >= 1e3) return (v / 1e3).toFixed(2) + 'K';
  return v.toFixed(2);
}

function wireFilters() {
  const layerSelect = document.getElementById('layerFilter');
  Object.entries(LAYER_NAMES).forEach(([k, v]) => {
    const opt = document.createElement('option');
    opt.value = k;
    opt.textContent = v;
    layerSelect.appendChild(opt);
  });
  layerSelect.addEventListener('change', renderIndicators);
  document.getElementById('categoryFilter').addEventListener('change', renderIndicators);
}

function setRepoLink() {
  // Replace with actual repo URL after first commit
  const host = window.location.hostname;
  if (host.endsWith('.github.io')) {
    const [user] = host.split('.');
    const repo = window.location.pathname.split('/')[1];
    document.getElementById('repoLink').href = `https://github.com/${user}/${repo}`;
  }
}

loadData().catch(err => {
  document.querySelector('.wrap').innerHTML = `
    <div class="panel">
      <h2>No data yet</h2>
      <p>${err.message}</p>
      <p class="muted">Run the GitHub Action manually (Actions tab → Refresh FRED Data → Run workflow) to generate the first snapshot.</p>
    </div>`;
});
```

### 7.2 QA gate for session 4

Enable GitHub Pages on the repo (Settings → Pages → Source: deploy from `main` branch, root). Wait for deployment. Visit the published URL. The page must:

1. Load without errors.
2. Display the composite score, layer cards, and indicator table from `data/current.json`.
3. Show the history chart (will be empty/sparse on first run, this is expected).
4. Filter the indicator table by layer and category.

Commit message: `feat(ui): polished dashboard with filters and history chart`.

### 7.3 Build prompt for executing LLM

```text
Create public/index.html, public/app.mjs, and public/styles.css with the exact content I provide. The app must:
- Use vanilla JS modules (no React, no build step)
- Read data from ../data/current.json and ../data/history.json
- Use Chart.js from CDN
- Be responsive and render correctly on mobile
Output file contents only.
```

---

## 8. SKILL.md (Agent Skills framework entry)

```markdown
# Recession Tracker

## Trigger conditions
- User asks to build a recession monitoring dashboard
- User wants to track FRED indicators with automated refresh
- User wants a server-side scheduled data pipeline with static front-end

## Process
1. Scaffold repo per spec (registry + tests)
2. Build pure scoring engine (TDD)
3. Build FRED pipeline (fetch + Actions workflow)
4. Build static dashboard

## Acceptance criteria
- `node --test` passes all suites
- GitHub Action completes green
- `data/current.json` and `data/history.json` exist with valid schema
- GitHub Pages site loads and renders

## Anti-patterns
- Browser direct FRED fetch (CORS will block it)
- Synthetic backtest (intellectually fraudulent)
- API key in client code
- Mixing data fetch with UI rendering
```

---

## 9. README.md

```markdown
# Recession Tracker

A macro + micro recession monitoring dashboard. Pulls 24 indicators from
FRED on a scheduled cadence, scores each into a 0-100 risk score, aggregates
into 5 layers and 1 composite, and publishes a dashboard via GitHub Pages.

## Architecture

GitHub Actions fetches FRED data twice weekly, computes the composite score
in Node, commits JSON snapshots to the repo. GitHub Pages serves the static
front-end which reads those JSON files. No CORS issues, no API key in
the browser.

## Setup

1. Get a free FRED API key from https://fred.stlouisfed.org/docs/api/api_key.html
2. Fork or clone this repo
3. Add `FRED_API_KEY` to repo Secrets (Settings → Secrets and variables → Actions)
4. Enable GitHub Pages (Settings → Pages → deploy from `main` branch, `/` root, set Pages folder to `/public`)
5. Manually trigger the "Refresh FRED Data" workflow once to generate initial data
6. Visit your GitHub Pages URL

## Local development

```bash
node --test tests/                    # run tests
FRED_API_KEY=xxx node scripts/fetch.mjs  # generate data locally
```

Then open `public/index.html` via a local server (e.g. `python -m http.server 8000`)
because ES modules won't load from `file://`.

## Limitations and honest disclosures

- **Composite weights are doctrinal, not optimized.** The 5-layer weights
  (financial 30%, labor 25%, real economy 20%, inflation 15%, micro 10%)
  reflect a judgment about which signal classes lead vs lag, not an
  empirical optimization against NBER chronology.
- **Alert thresholds (60 RED, 30 YELLOW) are calibrated by intuition.**
  Without a backtest, these are not validated. Treat as relative signals.
- **Series have different release lags.** Daily series are ~1 day fresh,
  monthly are 1-2 months stale, quarterly are 3-4 months stale. The
  composite blends these and surfaces each indicator's actual observation
  date.
- **No backtest.** This was an intentional architectural choice. History
  accumulates from real scheduled runs over time.

## Indicator registry

24 indicators across 5 layers. See `src/registry.mjs` for the full list,
weights, and thresholds.
```

---

## 10. Final QA acceptance criteria

Before declaring the build complete:

1. All tests pass: `node --test tests/`
2. Both workflows (refresh, test) have run green at least once.
3. `data/current.json` exists with composite, layers, and indicators populated.
4. `data/history.json` exists with at least one entry.
5. GitHub Pages site loads, renders all sections, and filters work.
6. README documents setup, architecture, and limitations honestly.
7. No FRED API key anywhere in the public-facing files.

Tag the release `v1.0.0` and move on.

---

## 11. Post-MVP improvements (deferred)

- Recession shading on the history chart (FRED `USREC` series).
- Sparkline per indicator in the table.
- Email digest via GitHub Actions when the composite crosses 30 or 60.
- CSV export of `history.json`.
- Replace doctrinal weights with empirically-tuned weights once 12+ months
  of history exists.
- Add a procurement-specific layer (oil price, freight costs, supplier
  PMIs) for energy-sector use cases.
