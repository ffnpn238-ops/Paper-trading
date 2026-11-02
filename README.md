# Paper Trader - day-trading simulator with Grok bots

A paper-trading environment for testing several trading bots (Grok-powered or your own) against the **same simulated market**.
**Simulation only: there is no brokerage, no live-trading code, and no way to place a real order.**
All prices are **synthetic** (a seeded random-walk model with fictional tickers) and are labelled as such in the UI. They are not real market data.

## Run it

Requires Node.js 18.17+ (no `npm install` needed - zero dependencies).

```bash
cd paper-trader
npm start            # -> http://localhost:3000
npm test             # unit tests (market, broker, risk, schema, bots, server proxy)
```

Manual paper trading and the rule-based bot work immediately, with no API key.

### Enable Grok bots (xAI API key)

1. Create a key at <https://console.x.ai>.
2. Provide it **to the server process only**, either way:
   - `cp .env.example .env`, then edit `.env`: `XAI_API_KEY=your-key` (`.env` is git-ignored), or
   - environment variable: `XAI_API_KEY=your-key npm start` (PowerShell: `$env:XAI_API_KEY="your-key"; npm start`).
3. Restart. The header pill shows **xAI: key set**; click it to test the connection.

The key is read from the server's environment, used only in the `Authorization` header of server-to-xAI requests, redacted from logs/errors, and never sent to the browser, saved in bot settings, or written to disk by the app.

| Variable | Default | Purpose |
|---|---|---|
| `XAI_API_KEY` | - | Required for Grok bots |
| `PORT` / `HOST` | `3000` / `127.0.0.1` | Listen address (keep loopback) |
| `XAI_BASE_URL` | `https://api.x.ai/v1` | API base |
| `XAI_DEFAULT_MODEL` | `grok-4.7` | Model used when a bot has none |
| `XAI_TIMEOUT_MS` | `45000` | Per-request timeout |
| `XAI_MAX_REQUESTS_PER_MINUTE` / `_PER_HOUR` | `20` / `300` | Server-side cap across all bots |
| `ALLOW_REMOTE_HOSTS` | unset | Set `1` to allow non-localhost Host headers (not recommended) |

**xAI integration (checked against docs.x.ai on 2026-11-02):** the server calls the **Responses API** (`POST /v1/responses`, the endpoint xAI recommends; Chat Completions is marked legacy) with `input` messages, `text.format = {type:"json_schema", strict:true}` for structured output, `max_output_tokens`, and `store:false`. Default model `grok-4.7`; type any other model name in a bot's settings. If xAI changes the API, the only file to update is `server.js`.

## Using the simulator

- **Data modes** (header): *Replay* scenarios / custom seed = identical prices every time (repeatable, deterministic, including slippage and latency). *Live simulation* = new random seed on each reset; the seed is shown so you can re-run it as a custom replay.
- **Controls:** Run/Pause, **Step candle** (one 1-minute candle = 6 ticks of 10 sim-seconds), Run to close (fast-forward; bots do not act), speed, Reset. A running sim pauses itself at the close of each session.
- **Chart:** 1m / 5m / 15m / 30m candles, forming candle updates every tick, SMA20, your average price, stop-loss/take-profit, working orders and fill markers. Mouse wheel zooms.
- **Order ticket:** market / limit / stop, with optional stop-loss and take-profit (Manual account). Positions tab: set SL/TP or close.
- **Tabs:** positions, open orders, order history, **rejected orders with the reason**, bot audit log, performance (leaderboard + equity curves).
- **Settings:** starting capital, commissions, fees, slippage, latency, partial-fill liquidity, borrow rate, margin, flatten-at-close, and all risk limits.

### Execution model

Bid/ask spread (varies with volatility) - slippage + size impact - commission (per share, minimum per order, charged cumulatively over partial fills) - sell/short regulatory fee - order latency (extra 10-second ticks) - partial fills (an account can take only a % of each tick's volume) - market, limit (fills at the touch when marketable) and stop orders - position-attached stop-loss (market exit, extra slippage) and take-profit (limit exit) - short selling with borrow cost - regular session only (orders rejected when closed; day orders expire and positions flatten at the close unless disabled) - forced liquidation below maintenance margin. Positions are marked at the bid (long) / ask (short), so equity = cash + liquidation value and **net P&L = realized + unrealized - fees**.

Risk checks (every order, manual or bot; each rejection states the rule and numbers): market hours, symbol allowed, quantity, price band, stop/take-profit side and size, no oversell / no accidental flips, shorting allowed, max leverage / buying power, max position % and shares, max order size, risk-per-trade to the stop, require-stop-loss, open-order and daily-order caps, daily loss limit, max drawdown.

## Grok bots

Create bots with **+ New bot**: name, bot type, Grok model, strategy instructions, watchlist, starting capital, risk limits, auto-run cadence and a **max requests per run**. Each bot has its **own account**, and every bot trades the same market. Two samples are pre-created (a rule bot and a Grok bot); settings live in your browser's localStorage and never contain secrets. Auto-run is always off after a page reload.

- **Ask bot** = one manual request. **Auto-run** = one request every N candles, only while the market is open, skipping a candle if the previous request is still pending, stopping itself at the request limit or after repeated errors. **Stop all bots** (header) cancels everything in flight.
- **What a bot sees:** only its own account, its watchlist (last 30 one-minute bars, quote, SMA/RSI/VWAP), its risk limits and its recent orders/rejections.
- **What a bot can do:** return ONE decision matching the strict schema (below). Pipeline: *snapshot -> adapter -> schema validation (server and browser) -> risk checks -> simulated execution*. Unknown fields are rejected; there is no field that can change settings or limits; strategy text and API output are untrusted and only ever rendered as text.
- **Audit log:** per request - trigger, model/latency/tokens, proposed action, reasoning summary, schema result, risk result (with the rejection reason), simulated fill, and the exact snapshot sent.
- No key / API down: the pill shows the status, Grok bots log a clear error, and everything else keeps working.

### Decision schema (`public/core/schema.js`)

```json
{ "action": "buy|sell|short|cover|close|hold",
  "symbol": "ALPH" | null, "quantity": 100 | null,
  "order_type": "market|limit" | null, "limit_price": 48.1 | null,
  "stop_loss": 47.6 | null, "take_profit": 49.2 | null,
  "reason": "short explanation (max 400 chars)" }
```
`buy` opens/adds long, `sell` reduces long, `short` opens/adds short, `cover` reduces short, `close` flattens the symbol's position, `hold` does nothing.

## Add your own bot (adapter interface)

Modules are separate: `core/market.js` (data) - `core/snapshot.js` (what bots see) - `bots/*` (decisions) - `core/schema.js` + `core/executor.js` (decision -> order) - `core/risk.js` (rules) - `core/broker.js` (execution/accounting) - `core/simulation.js` (clock).

An adapter is a plain object (full docs in `public/bots/adapter.js`):

```js
// public/bots/mybot.js
export const myAdapter = {
  id: 'my-bot',                  // stored in saved bot settings
  label: 'My bot',
  needsApiKey: false,            // true => UI warns when the server has no key
  usesModel: false,              // true => shows the model field
  async decide(snapshot, bot, { signal }) {
    // snapshot: compact market + account view (see core/snapshot.js); bot: saved config; signal: AbortSignal
    return { decision: { action: 'hold', symbol: null, quantity: null, order_type: null,
                         limit_price: null, stop_loss: null, take_profit: null, reason: 'No edge' } };
    // optional: meta: { model, usage: {input_tokens, output_tokens}, latencyMs }
  },
};
```
Then add `import { myAdapter } from './mybot.js'; registerAdapter(myAdapter);` to `public/bots/index.js`; it appears in the bot-type list. Throw `new AdapterError(code, message)` on failure. Adapters that need secrets should call a server route (like `grok.js` -> `/api/grok/decide`) rather than hold keys in the browser. `bots/rulebased.js` (SMA cross + RSI, ATR stops, risk-based sizing, no randomness) is the reference implementation.

## Known limitations

- Prices are synthetic. There is no import of real historical data (the `Market` class is the seam to add a CSV/replay source).
- The simulated market ignores your orders: no market impact on prices, and accounts do not compete for the same liquidity.
- Fills use 10-second ticks (mid-price candles); latency is coarse. No intrabar order-book model, halts, auctions, or pre/post-market.
- Stops trigger on quotes, not guaranteed prices; take-profit is a virtual order that becomes a limit order when touched. Order types: market, limit, stop (no trailing/bracket-OCO objects beyond position-attached SL/TP).
- Simple margin model (leverage cap + maintenance liquidation); no PDT rules, taxes, dividends, borrow availability or locates.
- Simulation state is in memory: reload = reset (bot configs/settings persist in localStorage).
- A bot decision is made on a snapshot and executes on arrival; with slow models the market may have moved (risk checks use the current price at arrival).
- Grok output is non-deterministic and costs tokens; use the request limits. Large watchlists increase prompt size.
- Single-user, local-only server (loopback host check + required request header); not hardened for internet exposure.
- Not financial advice; simulated results say nothing reliable about live performance.
