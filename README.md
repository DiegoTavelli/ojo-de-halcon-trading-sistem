# Autonomous Trading System — Binance Futures

A production trading system built on NestJS that monitors market conditions,
detects setups, executes orders and manages positions autonomously.
Three specialized agents handle different market regimes — all running inside
a single process with isolated capital pools per strategy.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     NestJS Motor (PM2)                          │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  ColibriService │  │  HalconService  │  │  MantisService │  │
│  │  Scalping 15m   │  │  Swing 1h/4h    │  │  Short pumps   │  │
│  │  LIMIT entries  │  │  MARKET entries │  │  Score ≥ 7     │  │
│  └────────┬────────┘  └────────┬────────┘  └───────┬────────┘  │
│           │                    │                    │           │
│           └──────────┬─────────┘                    │           │
│                      ▼                              ▼           │
│  ┌───────────────────────────────┐  ┌──────────────────────┐   │
│  │      CapitalStateService      │  │   TpWatcherService   │   │
│  │  Isolated pools per strategy  │  │   TP Guard + Cascade │   │
│  │  Real PnL from Binance trades │  │   Watcher 60s        │   │
│  └───────────────────────────────┘  └──────────────────────┘   │
│                                                                 │
│  ┌───────────────────────────────┐  ┌──────────────────────┐   │
│  │       PriceService            │  │  HTTP API :7331      │   │
│  │  exchangeInfo cache 24h       │  │  /colibri/signal     │   │
│  │  tick/step precision all syms │  │  /mantis/activate    │   │
│  │  calcSL · calcTP · calcQty    │  │  /tp-watcher/start   │   │
│  └───────────────────────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
            ┌─────────────────────────┐
            │    Binance USDⓈ-M       │
            │  REST + Algo Order API  │
            │     Hedge Mode          │
            └─────────────────────────┘
```

---

## Agents

### Colibrí — High-frequency scalping in lateral markets

Detects BB touches on 15m candles filtered by SMC 1h structure and BTC 4h bias.
Uses LIMIT orders inside the spread for maker fee (0.02% vs 0.04% taker).

**Entry flow:**

1. External scanner (`colibri-scanner`, PM2) detects BB touch on 15m
2. POSTs signal to `/colibri/signal` with direction, symbol, score
3. Motor applies per-symbol production filter (TypeScript) — same logic in backtest and live
4. If approved: LIMIT entry 0.1% in favor, 15s fill window
5. On fill: SL + TP via Algo Order API

**Filter methodology:**

- Backtest: `colibri-smcbb-backtest.cjs` generates 365d mechanical trades → JSON export
- Validation: `populate.ts` applies production TypeScript filter to mechanical trades
- Rule: no hard veto with N < 30 — small samples are noise, not signal
- Month/hour bonuses removed if N < 30 per bucket
- Each filter documents N, PnL avoided, and rationale per veto

**Validated symbols (365d):**

| Symbol   | Trades | WR    | PnL vs Mechanical |
| -------- | ------ | ----- | ----------------- |
| SOLUSDT  | 195    | 68.7% | +$16              |
| AVAXUSDT | 191    | 65.4% | +$35              |
| LINKUSDT | 248    | 65.7% | +$25              |
| DOTUSDT  | 227    | 60.8% | +$41              |
| NEARUSDT | 189    | 71.4% | +$50              |

**State machine per symbol:**

```
idle → pendingLimit (LIMIT in flight, 15s)
     → open (position live, SL+TP placed)
     → idle (TP/SL hit or reconciler close)
```

`pendingLimit` flag prevents the reconciler from resetting the lock during the
15-second fill window — avoids false "position not found" resets.

---

### Ojo de Halcón — Autonomous swing trading cycle

Detects technical setups on 15m + 1h timeframe using BB + SMC Order Block filters.
Handles the full trade lifecycle: detection → entry → monitoring → exit.

**Entry flow:**

1. Cron 5min watches for setup conditions
2. On setup forming: escalates to cron 30s
3. MARKET entry + SL/TP/Trailing Stop via Algo Order API
4. Launches `scripts/monitor/monitor.cjs` — standalone Node.js process, zero tokens
   between alerts. Queries Binance directly every 5min via REST.
5. Claude Code Monitor listens to stdout — activates only on `ALERT:LOSS_EXIT`
   or `ALERT:PROFIT_EXIT`

**Scoring by conviction:**

```
Score 8-9  → margin ~$114  risk $12
Score 10-11 → margin ~$190  risk $20
Score 12+   → margin ~$286  risk $30
Formula: margin = risk_target / (leverage × sl_pct)
```

---

### Mantis — Short on pump exhaustion

Waits for the confirmed top of a pumped asset (50–300%) and enters SHORT at the
exact moment of exhaustion. Unlike Halcón (technical setup), Mantis operates on
narrative pumps and uses a pre-entry scanner instead of market structure signals.

**Entry conditions (score ≥ 7 required):**

- RSI 1h > 72
- 2 consecutive red 15m candles
- Declining volume on bounce
- Funding rate positive (> 0.01%)
- Bearish divergence on 15m confirmed

**Critical rules:**

- No entry without `ENTRY_SIGNAL` from scanner — even if the trader requests it
- Stop loss always at `pump_high × 1.025` — never percentage from entry
  (percentage SL gets hit by normal volatility spikes on pumped assets)
- Second position valid only if token makes a new ATH above the previous entry

**Pre-entry mandatory research (3 layers):**

1. Pump profile: recent listing, market cap, concentrated volume
2. Narrative: active sector, Twitter/Telegram, call channels
3. Structural risk: concentrated wallets, sustained funding, real product

---

## Capital Management

### Isolated pools per strategy

Each strategy has an independent capital pool in `capital-state.json`.
No cross-strategy borrowing — a loss in Colibrí does not affect Halcón's budget.

```json
{
  "strategies": {
    "colibri": {
      "capital_usdt": 250,
      "op_size_usdt": 50,
      "min_available_usdt": 30,
      "available_usdt": 255.31,
      "pnl_net_usdt": 5.31
    },
    "halcon": {
      "capital_usdt": 50,
      "op_size_usdt": 25,
      "min_available_usdt": 16,
      "available_usdt": 50,
      "pnl_net_usdt": 0
    }
  }
}
```

`available_usdt = capital_usdt + pnl_net_usdt − margin_in_use_usdt`

If `available_usdt < min_available_usdt` → `preTradeCheck` blocks new entries
automatically. Losses reduce the operating budget in real time.

### Real PnL reconciliation

When a position closes externally (SL/TP hit on Binance while motor was down),
the reconciler fetches the actual realized PnL before updating capital state:

```
reconciler detects position gone from Binance
→ GET /fapi/v1/userTrades?symbol=X&startTime=openedAt
→ sum(realizedPnl) across all closing trades
→ capitalState.onClose(strategy, symbol, realPnl, 'RECONCILER_CLOSE')
```

Without this, external closes registered `pnl=0` — balance drops were
misclassified as "withdrawals" and `pnl_net` drifted from reality.

### Withdrawal auto-detection

```
every 2min: updateWalletBalance(binanceBalance)
  expected = Σ(capital + pnl_net) − withdrawn_usdt
  diff = binanceBalance − expected

  if diff < −$5 AND pnl_net unchanged since last cycle:
    → auto-register as withdrawal (money sent to spot)

  if diff < −$5 AND pnl_net changed:
    → ALERT:CAPITAL:DISCREPANCY (loss not fully explained)

  if diff > +$5:
    → ALERT:CAPITAL:DISCREPANCY (unexpected surplus)
```

---

## Precision Service

`PriceService` centralizes all price/quantity calculations using Binance
`exchangeInfo` (cached 24h). Eliminates hardcoded `toFixed(4)` across strategies.

```typescript
// One source of truth for all symbols
const meta = await this.price.getMeta('AVAXUSDT');
// → { tickSize: 0.01, stepSize: 0.1, priceDecimals: 2, qtyDecimals: 1, ... }

const slPrice  = await this.price.calcSL('AVAXUSDT', entry, sl_pct, 'SHORT');
const tpPrice  = await this.price.calcTP('AVAXUSDT', entry, tp_pct, 'SHORT');
const qty      = await this.price.calcQty('AVAXUSDT', capital, leverage, entry);
const limit    = await this.price.calcLimitEntry('AVAXUSDT', price, 0.1, 'SHORT');
```

---

## Key Technical Decisions

**Single process architecture**
All agents run inside one NestJS process managed by PM2.
No inter-process communication — services communicate via dependency injection.
`fs.watch` watcher lives inside the motor at 0% CPU when idle.

**Algo Order API for SL/TP/Trailing Stop**
`/fapi/v1/order` returns error -4120 for conditional orders in Hedge Mode.
All stops, take profits and trailing stops use `/fapi/v1/algoOrder` with
`triggerPrice` (not `stopPrice`).

**Hedge Mode**
Account runs in Hedge Mode — all orders require `positionSide: LONG` or `SHORT`.
Enables simultaneous long and short on the same symbol across strategies.

**LIMIT vs MARKET entries**

- Colibrí uses LIMIT (maker fee 0.02% vs taker 0.04%) — scalping edge matters
- Halcón uses MARKET — swing timeframe, entry precision less critical than certainty
- Both use MARKET for SL/TP to guarantee execution

**Zero-token monitoring**
Monitor script (`monitor.cjs`) runs standalone, queries Binance every 5min.
Claude Code activates only on specific stdout patterns — not on every tick.

**Dual environment**
Controlled via `BINANCE_ENV` in `.mcp.json`:

| Value          | Endpoint                  |
| -------------- | ------------------------- |
| `testnet`    | testnet.binancefuture.com |
| `production` | fapi.binance.com          |

---

## Project Structure

```
src/
├── colibri/
│   ├── colibri.service.ts          # signal handling, LIMIT entry, reconciler
│   ├── colibri.config.ts           # COLIBRI_SYMBOLS registry
│   └── filters/
│       ├── sol.filter.ts           # per-symbol production filter
│       ├── avax.filter.ts
│       ├── link.filter.ts
│       ├── dot.filter.ts
│       └── near.filter.ts
├── halcon/
│   ├── halcon.service.ts           # BB+SMC detection, MARKET entry, reconciler
│   ├── halcon.config.ts
│   └── halcon.types.ts
├── mantis/
│   ├── mantis.service.ts           # pump scanner, scoring, entry
│   └── mantis.controller.ts
├── tp-watcher/
│   ├── tp-watcher.service.ts       # TP Guard, cascade mode
│   └── tp-watcher.controller.ts
├── capital/
│   └── capital-state.service.ts   # isolated pools, preTradeCheck, reconciler
├── price/
│   └── price.service.ts            # exchangeInfo cache, precision calculations
├── executor/
│   └── executor.service.ts         # Binance REST client, getRealizedPnl
└── app.module.ts

scripts/
├── backtesting/pilar2-direccional/
│   ├── colibri/
│   │   ├── colibri-smcbb-backtest.cjs   # mechanical backtest (365d)
│   │   └── populate.ts                  # applies production filter to trades
│   └── halcon/
│       └── halcon-smcbb-backtest.cjs
├── monitor/
│   └── monitor.cjs                      # standalone position monitor
└── utils/
    └── current-time.cjs                 # session/funding time helper

data/
├── capital-state.json       # live capital tracking per strategy
├── colibri-state.json       # open positions per symbol
├── halcon-state.json
└── live-operations/         # per-operation logs
```

---

*Private repository — code available upon request.*
