┌─────────────────────────────────────────────────┐
│                  NestJS Backend                  │
│                                                  │
│  ┌──────────────┐      ┌──────────────────────┐  │
│  │ MantisService│      │  TpWatcherService    │  │
│  │              │      │  (TP Guard + Cascade)│  │
│  │ Scanner 1min │      │  Watcher 60s         │  │
│  │ Entry signal │─────▶│  Activates on zone   │  │
│  └──────────────┘      └──────────────────────┘  │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │           HTTP API (port 7331)           │    │
│  │  POST /mantis/activate                   │    │
│  │  POST /tp-watcher/start                  │    │
│  │  POST /tp-watcher/cascade/:symbol        │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
│
▼
┌─────────────────────┐
│   Binance Futures   │
│   REST + Algo API   │
│   USDⓈ-M Futures   │
└─────────────────────┘



---

## Agents

### Mantis — Short on pump exhaustion
Monitors pumped assets and waits for confirmed exhaustion before entering SHORT.
Evaluates entry conditions every minute using a scoring system (0–12):

- RSI 1h > 72
- 2 consecutive red 15m candles
- Declining volume on bounce
- Positive funding rate

Entry triggers only when score ≥ 7 with confirmed bearish divergence.
Structural stop loss placed above the pump high — never percentage-based.

### TP Watcher — Post-entry position manager
Runs every 60 seconds silently. Activates when price is within 8% of the TP target.
Reads momentum snapshot (RSI 15m, volume ratio, funding) and recommends trailing
stop callback rate. Cancels fixed TP and places trailing stop automatically.
Activates cascade mode via HTTP to the motor on confirmation.

### Ojo de Halcón — Full autonomous cycle
Detects technical setups on 15m timeframe using BB + SMC filters.
Escalates from 5min polling to 30s when setup is forming.
Executes MARKET entry + SL/TP/Trailing Stop via Algo Order API automatically.
Launches a Node.js monitor script that queries Binance directly every 5min —
zero token consumption between alerts.

---

## Tech Stack

- **Runtime:** Node.js + TypeScript
- **Framework:** NestJS
- **Exchange:** Binance USDⓈ-M Futures (REST + Algo Order API)
- **Architecture:** Multi-agent, event-driven, single NestJS process
- **Process manager:** PM2

---

## Key Technical Decisions

**Single process architecture**
All agents run inside one NestJS process managed by PM2.
No inter-process communication overhead — services communicate via direct injection.
`fs.watch` watcher lives inside the motor at 0% CPU when idle.

**Algo Order API for SL/TP/Trailing Stop**
Standard `/fapi/v1/order` endpoint returns error -4120 for conditional orders.
All stop loss, take profit and trailing stop orders go through `/fapi/v1/algoOrder`
using `triggerPrice` instead of `stopPrice`.

**Structural stop loss**
Stop loss is always placed at `pump_high × 1.025` — never percentage-based from entry.
Percentage-based SL gets hit by normal volatility spikes on pumped assets.

**Hedge Mode**
Account runs in Hedge Mode — all orders require explicit `positionSide: LONG` or `SHORT`.
Allows simultaneous long and short positions on the same symbol.

**Zero-token monitoring**
Monitor script runs as a standalone Node.js process querying Binance directly.
Only activates on specific alert patterns in stdout — not on every tick.

---

## Project Structure

├── src/
│   ├── mantis/
│   │   ├── mantis.service.ts
│   │   └── mantis.controller.ts
│   ├── tp-watcher/
│   │   ├── tp-watcher.service.ts
│   │   └── tp-watcher.controller.ts
│   └── app.module.ts
├── scripts/
│   ├── monitor/
│   │   └── monitor.cjs
│   └── utils/
│       └── current-time.cjs
└── docs/



---

## Environment

| BINANCE_ENV | Endpoint |
|---|---|
| `testnet` | testnet.binancefuture.com |
| `production` | fapi.binance.com |

---

*Private repository — code available upon request.*
