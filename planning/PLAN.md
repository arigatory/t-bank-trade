# FinAlly — AI Trading Workstation (T-Invest Edition)

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data from the **T-Invest API**, lets users trade through a **T-Invest sandbox account**, and integrates an LLM chat assistant that can analyze positions and execute real orders on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot — but the trades actually go through a real broker's sandbox.

This is the capstone project for an agentic AI coding course. It is built end-to-end by coding agents (Claude Code) demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents coordinate through files in `planning/`.

### Why T-Invest

- Real market data, real exchange behaviour, real order lifecycle — without real money
- Official, well-documented **C# SDK** (`Tinkoff.InvestApi`) covering instruments, market data streaming, orders, portfolio, and sandbox management
- Single gRPC endpoint, one token, free
- Sandbox contour (`sandbox-invest-public-api.tbank.ru:443`) is identical in shape to prod, so the same code works against both

## 2. User Experience

### First Launch

The user sets `TINVEST_TOKEN` and `OPENROUTER_API_KEY` in `.env`, runs a single Docker command (or a provided start script), and a browser opens to `http://localhost:8000`. No login, no signup. On first start the backend:

1. Opens a fresh sandbox account via `SandboxService.OpenSandboxAccount`
2. Tops it up with 1 000 000 ₽ via `SandboxService.SandboxPayIn`
3. Persists the returned `accountId` in local SQLite
4. Seeds a default watchlist of Russian blue-chip tickers

They immediately see:

- A watchlist of 10 default tickers with live-updating prices
- 1 000 000 ₽ in sandbox cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream live** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade; updates come from `MarketDataStream.SubscribeLastPrice`
- **View sparkline mini-charts** — price action beside each ticker, accumulated on the frontend from the SSE stream since page load
- **Click a ticker** to see a larger detailed chart, with optional historical candles loaded via `MarketDataService.GetCandles`
- **Buy and sell shares** — market orders via `SandboxService.PostSandboxOrder` (`OrderType.Market`, `OrderDirection.Buy`/`Sell`), instant fill at last price, no confirmation dialog
- **Monitor their portfolio** — positions, cash, total value and P&L are fetched from `SandboxService.GetSandboxPortfolio`; shown as a treemap sized by weight and colored by P&L, plus a P&L line chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity (lots × lot size), average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat; the backend resolves each ticker to a `FIGI`/`InstrumentUid` via `InstrumentsService.FindInstrument`

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500 ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header — reflects the health of the T-Invest gRPC stream, not just the SSE link
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌────────────────────────────────────────────────────────────┐
│  Docker Container (port 8000)                              │
│                                                            │
│  ASP.NET Core 8 (Minimal API, C#)                          │
│  ├── /api/*                REST endpoints                  │
│  ├── /api/stream/*         SSE streaming                   │
│  └── /*                    Static file serving             │
│                            (Next.js export)                │
│                                                            │
│  Hosted services (BackgroundService):                      │
│  ├── MarketDataStreamWorker  gRPC → price cache            │
│  └── PortfolioSnapshotWorker polls sandbox portfolio       │
│                                                            │
│  Tinkoff.InvestApi (official C# SDK) ──► gRPC ──►          │
│                              sandbox-invest-public-api     │
│                                       .tbank.ru:443        │
│                                                            │
│  SQLite (EF Core) — app-level state only                   │
│    watchlist, chat_messages, portfolio_snapshots           │
│    /app/db/finally.db (volume-mounted)                     │
└────────────────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by ASP.NET Core as static files
- **Backend**: ASP.NET Core 8 Minimal API (C#), uses the official `Tinkoff.InvestApi` NuGet package
- **Database**: SQLite via Entity Framework Core, single file at `/app/db/finally.db`, volume-mounted for persistence
- **Real-time data**: T-Invest `MarketDataStream` (gRPC, long-lived) inside the backend → in-memory price cache → Server-Sent Events to the browser
- **Trading**: all orders, positions, balances live in the T-Invest sandbox — the backend does not maintain its own copy of this state
- **AI integration**: OpenAI-compatible HTTP client to OpenRouter (`openrouter/openai/gpt-oss-120b` via Cerebras), structured outputs for trade/watchlist actions
- **Source of truth**: T-Invest sandbox for anything financial; local SQLite for anything app-specific (watchlist, chat, snapshot history)

### Why These Choices

| Decision | Rationale |
|---|---|
| T-Invest Sandbox over a home-grown simulator | Real broker semantics (order states, margin, commissions, trading hours) with no downside |
| C#/.NET 8 backend | The official `Tinkoff.InvestApi` SDK is C#-first; Ivan works in .NET daily; gRPC is first-class in .NET |
| SSE to the browser (not gRPC-Web) | One-way push is all the UI needs; native `EventSource`; no extra proxy/filters; keeps gRPC concerns server-side |
| Don't mirror positions/trades in SQLite | T-Invest is the source of truth; duplicating it invites drift bugs. We query `GetSandboxPortfolio`/`GetSandboxOperations` on demand |
| SQLite over Postgres | No auth = no multi-user = no need for a database server |
| Single Docker container | One command to run; no docker-compose for production |
| `dotnet publish` into runtime image | Fast, reproducible; standard .NET packaging |
| Market orders only (initial scope) | Eliminates limit-order UX complexity; can be extended later — the SDK already supports all order types |

---

## 4. Directory Structure

```
finally/
├── frontend/                       # Next.js TypeScript project (static export)
├── backend/                        # .NET 8 ASP.NET Core solution
│   ├── FinAlly.sln
│   ├── FinAlly.Api/                # ASP.NET Core project (entry point)
│   │   ├── Program.cs
│   │   ├── Endpoints/              # Minimal API endpoint definitions
│   │   ├── Sse/                    # SSE writer/helpers
│   │   ├── Services/               # Application services (TInvest wrappers, LLM, etc.)
│   │   ├── Workers/                # BackgroundService implementations
│   │   ├── Data/                   # EF Core DbContext, migrations, seed logic
│   │   ├── Models/                 # DTOs and domain records
│   │   └── appsettings.json
│   └── FinAlly.Tests/              # xUnit unit tests
├── planning/                       # Project-wide documentation for agents
│   ├── PLAN.md                     # This document
│   ├── TINVEST_NOTES.md            # SDK quirks, quotation conversion, etc.
│   └── ...
├── scripts/
│   ├── start_mac.sh                # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh
│   ├── start_windows.ps1
│   └── stop_windows.ps1
├── test/                           # Playwright E2E tests + docker-compose.test.yml
├── db/                             # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep
├── Dockerfile                      # Multi-stage (node → dotnet-sdk → aspnet runtime)
├── docker-compose.yml              # Optional convenience wrapper
├── .env                            # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about .NET or T-Invest. It talks to the backend via `/api/*` and `/api/stream/*`. Internal structure is the Frontend Engineer's call.
- **`backend/FinAlly.Api/`** owns all server logic: T-Invest SDK wiring, gRPC streaming, SSE bridging, EF Core, LLM calls. The T-Invest client is registered once via DI and shared across services.
- **`backend/FinAlly.Api/Data/`** contains the `FinAllyDbContext`, entity configurations, and seed logic. The backend runs `Database.Migrate()` (or `EnsureCreated()`) on startup and seeds default watchlist + sandbox account bootstrap.
- **`db/`** at the top level is the runtime volume mount point mapped to `/app/db` in the container.
- **`planning/`** contains the shared contract. Every agent should read `PLAN.md` and `TINVEST_NOTES.md` before touching code.

---

## 5. Environment Variables

```bash
# Required: T-Invest API token (use a sandbox or full-access token)
# Get one at https://www.tbank.ru/invest/settings/api/
TINVEST_TOKEN=t.your-tinvest-token-here

# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: override the T-Invest endpoint (default is sandbox).
# Use "invest-public-api.tbank.ru:443" ONLY if you want prod trading with real money.
TINVEST_ENDPOINT=sandbox-invest-public-api.tbank.ru:443

# Optional: existing sandbox account to reuse. If empty, backend opens a new one on first run.
TINVEST_ACCOUNT_ID=

# Optional: initial sandbox top-up amount in RUB (default 1_000_000)
TINVEST_INITIAL_BALANCE_RUB=1000000

# Optional: deterministic mock LLM responses for E2E tests
LLM_MOCK=false
```

### Behavior

- The backend reads `.env` via `DotNetEnv` on startup (or the container receives it via `docker run --env-file .env`).
- If `TINVEST_TOKEN` is missing → backend fails fast with a clear error at startup.
- If `TINVEST_ACCOUNT_ID` is missing → backend calls `OpenSandboxAccount`, tops it up with `TINVEST_INITIAL_BALANCE_RUB` via `SandboxPayIn`, and stores the account id in SQLite (table `app_state`). Subsequent starts reuse it.
- If the stored account id returns `ACCOUNT_NOT_FOUND` (sandbox accounts are purged after 3 months of inactivity) → the backend opens and seeds a new one, logs the rotation, and continues.
- If `LLM_MOCK=true` → LLM client returns canned structured responses. No OpenRouter call is made.

---

## 6. Market Data

### Source: T-Invest `MarketDataStream`

All live quotes come from the T-Invest gRPC `MarketDataStream.MarketDataStream` bidirectional stream, subscribed via `SubscribeLastPrice` (and optionally `SubscribeOrderBook` / `SubscribeTrades` as a stretch goal).

### `MarketDataStreamWorker` (BackgroundService)

A single hosted service owns the T-Invest stream connection for the whole app:

1. On startup, opens a `MarketDataStream` using `InvestApiClient.MarketDataStream` from the SDK.
2. Reads the current watchlist from SQLite, resolves each ticker to an `InstrumentUid` via `InstrumentsService.FindInstrument`, caches the mapping.
3. Sends a `SubscribeLastPriceRequest` with the full set of instrument UIDs.
4. Consumes stream responses and updates an in-memory `PriceCache` (`ConcurrentDictionary<Figi, PriceTick>`).
5. Reacts to watchlist mutations: exposes `SubscribeAsync(ticker)` / `UnsubscribeAsync(ticker)` that send incremental `SubscribeLastPriceRequest` frames (add/remove).
6. On stream error or disconnect, backs off exponentially and re-subscribes to the full current set. Surfaces the connection state via an `IObservable<StreamHealth>` consumed by SSE clients for the UI health dot.

### `PriceCache`

```csharp
record PriceTick(
    string Ticker,
    string Figi,
    decimal Price,
    decimal? PreviousPrice,
    DateTimeOffset Timestamp
);
```

- Thread-safe in-memory store keyed by `Figi`.
- `T-Invest` delivers quotes as `Quotation { units, nano }` — convert once to `decimal` on ingest and store the decimal form.
- Holds last price + previous price for flash direction.

### SSE Bridge to the Browser

- Endpoint: `GET /api/stream/prices`
- ASP.NET Core writes `text/event-stream` responses; each event is JSON with `{ ticker, figi, price, previousPrice, timestamp, direction }`.
- The endpoint subscribes to `PriceCache`'s change notifications (simple `Channel<T>` fan-out) and flushes events at up to 2 Hz per ticker (coalescing faster upstream updates).
- Client uses the native `EventSource` API, which handles reconnection.
- Browser never talks to T-Invest directly — the backend is the only gRPC consumer.

### Historical Candles (for the main chart)

- Endpoint: `GET /api/candles/{ticker}?interval=1h&from=...&to=...`
- Backed by `MarketDataService.GetCandles` (`CandleInterval.CANDLE_INTERVAL_1_MIN` … `CANDLE_INTERVAL_DAY`).
- Results are cached in-memory with a short TTL (e.g. 30 s for 1-minute candles, 5 min for hourly).

---

## 7. Trading (T-Invest Sandbox)

### Source of Truth

The sandbox account holds cash, positions, and order history. The backend **reads** from it on every relevant request — it does not maintain a parallel ledger. This is the single most important architectural rule of the project.

### Order Placement

`POST /api/portfolio/trade` with body `{ ticker, quantity, side }` maps to:

```csharp
var resolved = await instruments.ResolveAsync(ticker); // Figi + LotSize
var lots     = (int)Math.Round(quantity / resolved.LotSize);

var response = await sandbox.PostSandboxOrderAsync(new PostOrderRequest
{
    InstrumentId = resolved.InstrumentUid,
    Quantity     = lots,
    Direction    = side == "buy" ? OrderDirection.Buy : OrderDirection.Sell,
    AccountId    = accountId,
    OrderType    = OrderType.Market,
    OrderId      = Guid.NewGuid().ToString("N") // idempotency key
});
```

- `quantity` in the API is **shares**, not lots. The backend divides by `LotSize` and validates that the result is a whole number of lots (returning `400` otherwise with a helpful message including the instrument's lot size).
- The `orderId` acts as an idempotency key — the backend generates a fresh GUID per request and persists the mapping so a retried HTTP call doesn't double-execute.
- Response returns the sandbox `PostOrderResponse` projected into an app-friendly DTO (`executedPrice`, `executedQuantityLots`, `executedQuantityShares`, `commission`, `status`).

### Portfolio

`GET /api/portfolio` calls `SandboxService.GetSandboxPortfolio(accountId)` and projects it:

```json
{
  "cashRub": 823450.12,
  "totalValue": 1012345.67,
  "expectedYield": 12345.67,
  "expectedYieldPct": 1.23,
  "positions": [
    {
      "ticker": "SBER",
      "figi": "BBG004730N88",
      "quantityShares": 200,
      "quantityLots": 20,
      "avgPriceRub": 280.15,
      "currentPriceRub": 289.40,
      "unrealizedPnlRub": 1850.00,
      "unrealizedPnlPct": 3.30
    }
  ]
}
```

- `quantity`, `averagePositionPrice`, `currentPrice`, and `expectedYield` come back as T-Invest `Quotation`/`MoneyValue` — convert to `decimal` in the DTO layer.
- `cashRub` is the `PortfolioPosition` with `InstrumentType == "currency"` and `Figi == "RUB000UTSTOM"`.

### Portfolio Snapshots (for the P&L chart)

- A second BackgroundService, `PortfolioSnapshotWorker`, polls `GetSandboxPortfolio` every 30 seconds and on trade completion.
- It writes `(totalValue, recordedAt)` rows into the local `portfolio_snapshots` table.
- `GET /api/portfolio/history` returns these rows for the P&L line chart. This is local app state (T-Invest itself doesn't provide a "historical total value" endpoint).

### Operations History

`GET /api/portfolio/operations?from=&to=` surfaces `GetSandboxOperations` (or `GetSandboxOperationsByCursor` for pagination) for a trades-history panel. In sandbox only `OPERATION_TYPE_BUY` and `OPERATION_TYPE_SELL` are supported in the filter.

### Account Bootstrap

On startup, `AccountBootstrapService` runs once:

```
if TINVEST_ACCOUNT_ID is set and OpenSandboxAccount succeeds on GetAccounts check → use it
else if local SQLite has a cached accountId that is still valid → use it
else → OpenSandboxAccount → SandboxPayIn (RUB, TINVEST_INITIAL_BALANCE_RUB) → persist accountId
```

## 8. Database (local, app-level only)

### SQLite + EF Core

The local database holds **only** data that T-Invest doesn't own: the watchlist, chat history, portfolio snapshots, and a tiny key/value `app_state` table for bootstrap.

Startup runs `context.Database.Migrate()`; migrations live in `backend/FinAlly.Api/Data/Migrations`.

### Schema (EF Core entities)

**AppState** — singleton key/value
- `Key` TEXT PK
- `Value` TEXT
- Known keys: `sandbox_account_id`, `schema_version`

**WatchlistItem**
- `Id` GUID PK
- `UserId` TEXT (default `"default"`)
- `Ticker` TEXT
- `Figi` TEXT
- `InstrumentUid` TEXT
- `ClassCode` TEXT (e.g. `TQBR` for MOEX equities)
- `AddedAt` DateTimeOffset
- Unique index on `(UserId, Ticker)`

**PortfolioSnapshot** — total portfolio value over time
- `Id` GUID PK
- `UserId` TEXT (default `"default"`)
- `TotalValueRub` REAL
- `RecordedAt` DateTimeOffset
- Index on `(UserId, RecordedAt)`

**ChatMessage**
- `Id` GUID PK
- `UserId` TEXT (default `"default"`)
- `Role` TEXT (`user` | `assistant`)
- `Content` TEXT
- `Actions` TEXT (JSON — executed trades and watchlist changes; null for user messages)
- `CreatedAt` DateTimeOffset

**No `positions`, `trades`, or `users_profile` tables** — T-Invest owns that state.

### Default Seed

Default watchlist (Russian blue chips on MOEX):

```
SBER, GAZP, LKOH, GMKN, ROSN, TATN, MGNT, NVTK, PLZL, CHMF
```

On first launch each is resolved via `InstrumentsService.FindInstrument` and persisted with its `Figi`/`InstrumentUid`/`ClassCode`.

---

## 9. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates for all watched tickers |
| GET | `/api/candles/{ticker}?interval=&from=&to=` | Historical candles for the main chart |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Live snapshot from `GetSandboxPortfolio` (cash, positions, P&L) |
| POST | `/api/portfolio/trade` | Execute a market order via `PostSandboxOrder` |
| GET | `/api/portfolio/history` | Local `portfolio_snapshots` rows for the P&L chart |
| GET | `/api/portfolio/operations` | `GetSandboxOperations` projection (trade history) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist with latest prices (joined from `PriceCache`) |
| POST | `/api/watchlist` | Add a ticker (resolves FIGI, subscribes in MarketDataStream) |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker (unsubscribes from stream) |

### Instruments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/instruments/search?q=` | Thin wrapper over `InstrumentsService.FindInstrument` for the add-ticker autocomplete |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a user message; receive a complete JSON response with assistant reply + executed actions |
| GET | `/api/chat/history` | Recent chat messages |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check: reports T-Invest stream health, DB reachability, LLM config |

---

## 10. LLM Integration

### Client

A thin `ILLMClient` in C# wraps an `HttpClient` pointed at OpenRouter's OpenAI-compatible `chat/completions` endpoint. Model: `openrouter/openai/gpt-oss-120b` with the Cerebras provider preference set in the request body.

Auth: `Authorization: Bearer ${OPENROUTER_API_KEY}`.

No SDK dependency — plain JSON over HTTPS keeps the backend slim. Structured outputs are requested via `response_format: { type: "json_schema", json_schema: { ... } }`.

### Chat Flow

When the user sends a message, `POST /api/chat`:

1. Loads portfolio context: cash, positions with P&L (from `GetSandboxPortfolio`), watchlist with live prices (from `PriceCache`), total portfolio value.
2. Loads recent conversation history (last N messages) from `chat_messages`.
3. Builds messages: system prompt + portfolio context + history + new user message.
4. Calls OpenRouter with structured output enforcement.
5. Deserializes the JSON response.
6. Auto-executes every trade in `trades[]` by calling the same `PostSandboxOrder` path used by the manual trade endpoint. Validates each (lot sizing, trading status, sufficient funds) and collects per-trade results.
7. Auto-applies every watchlist change.
8. Persists the assistant message with `Actions` JSON describing what was executed (including failures with their error messages).
9. Returns the full payload to the frontend.

No token-by-token streaming — Cerebras finishes fast enough that a loading indicator is sufficient UX.

### Structured Output Schema

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    { "ticker": "SBER", "side": "buy", "quantity": 10 }
  ],
  "watchlist_changes": [
    { "ticker": "YDEX", "action": "add" }
  ]
}
```

- `message` (required): conversational text shown to the user
- `trades` (optional): auto-executed as market orders. `quantity` is **shares** (backend converts to lots)
- `watchlist_changes` (optional): `action ∈ { "add", "remove" }`

### Auto-Execution

Trades execute automatically — no confirmation dialog. Justified because:

- This is a **sandbox** account with virtual money
- It showcases the agentic AI theme of the course
- Failures (bad ticker, insufficient cash, closed market, lot size mismatch) are folded back into the response so the assistant can narrate them

### System Prompt

FinAlly is prompted as "an AI trading assistant for the T-Invest platform" with guidance to:

- Analyze portfolio composition, risk concentration, P&L, currency exposure
- Suggest trades with concise reasoning
- Execute trades when asked or when the user agrees
- Manage the watchlist proactively (add correlated peers, remove losers the user dismissed)
- Be aware that tickers are MOEX-listed Russian equities by default
- Never invent instruments — if a ticker isn't known, say so
- Always respond with valid structured JSON

### LLM Mock Mode

With `LLM_MOCK=true`, `ILLMClient` returns canned structured responses keyed on the last user message. Used for E2E tests, dev without a key, and CI.

---

## 11. Frontend Design

### Layout

Single-page application with a dense, terminal-inspired layout. The specific component architecture is up to the Frontend Engineer, but the UI includes:

- **Watchlist panel** — grid/table with ticker, current price (flashing), daily change %, and a sparkline accumulated from SSE
- **Main chart area** — larger chart for the selected ticker with candle history from `/api/candles/{ticker}` plus live overlay from SSE
- **Portfolio heatmap** — treemap where each rectangle is a position, sized by weight, colored by P&L
- **P&L chart** — line chart of total portfolio value over time, fed by `/api/portfolio/history`
- **Positions table** — ticker, shares, lots, avg cost, current price, unrealized P&L, % change
- **Trade bar** — ticker field (autocomplete hitting `/api/instruments/search`), quantity in shares, buy / sell buttons; shows the lot size next to the quantity field so the user knows what rounds correctly
- **AI chat panel** — docked/collapsible sidebar; conversation history; loading indicator; inline confirmations for LLM-executed trades and watchlist changes
- **Header** — total portfolio value (live), cash balance (RUB), connection status dot reflecting T-Invest stream health

### Technical Notes

- `EventSource` for `/api/stream/prices`
- Canvas-based charting library for performance (Lightweight Charts or Recharts)
- Price flash: on tick, toggle a CSS class with a green/red background transition, remove after ~500 ms
- All API calls are same-origin (`/api/*`) — no CORS
- Tailwind CSS with a custom dark theme
- All money values display in RUB with proper formatting (`Intl.NumberFormat('ru-RU')`)

---

## 12. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: node:20-slim
  - Copy frontend/
  - npm ci && npm run build (static export → frontend/out)

Stage 2: mcr.microsoft.com/dotnet/sdk:8.0
  - Copy backend/
  - dotnet restore
  - dotnet publish FinAlly.Api -c Release -o /app/publish

Stage 3: mcr.microsoft.com/dotnet/aspnet:8.0
  - Copy /app/publish from stage 2
  - Copy frontend/out into /app/publish/wwwroot from stage 1
  - EXPOSE 8000
  - ENV ASPNETCORE_URLS=http://+:8000
  - ENTRYPOINT ["dotnet", "FinAlly.Api.dll"]
```

ASP.NET Core serves the static frontend (`UseDefaultFiles` + `UseStaticFiles`) and the API on the same port.

### Docker Volume

```bash
docker run --rm \
  -v finally-data:/app/db \
  -p 8000:8000 \
  --env-file .env \
  finally
```

The `db/` folder in the project root → `/app/db` in the container. The backend writes `finally.db` there.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the image if missing or if `--build` is passed
- Runs the container with volume mount, port mapping, `.env` file
- Prints the URL and optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume

**`scripts/start_windows.ps1` / `stop_windows.ps1`** — PowerShell equivalents.

All scripts are idempotent.

---

## 13. Testing Strategy

### Unit Tests (`backend/FinAlly.Tests`, xUnit)

- **T-Invest wrappers**: given recorded gRPC responses (via the SDK's fake/mocked channel), the `TInvestSandboxService` correctly converts `Quotation`/`MoneyValue` to `decimal`, handles lot-size math, maps `OrderExecutionReportStatus` to our DTO
- **Portfolio projection**: `PortfolioDto` assembly from `PortfolioResponse`, RUB cash extraction, P&L math
- **Instrument resolution**: ticker → FIGI caching, ambiguous matches (same ticker on multiple classes), not-found handling
- **Chat structured output parsing**: valid schemas accepted, malformed JSON rejected gracefully, trade validation failures surfaced in the response
- **SSE coalescing**: upstream 20 Hz ticks → downstream ≤ 2 Hz per ticker, correct direction flags
- **API endpoints**: correct status codes, response shapes, error bodies

### Frontend Unit Tests

- Component rendering with mock data
- Price flash animation triggers on price change
- Watchlist CRUD
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (`test/`, Playwright)

Separate `docker-compose.test.yml` spins up the app container plus a Playwright container.

Tests run with:
- `LLM_MOCK=true` (fast, deterministic, offline)
- A pre-configured sandbox token (dedicated for CI) and a dedicated sandbox account id to isolate state

Scenarios:
- Fresh start: default watchlist renders, 1 000 000 ₽ shown, prices streaming within N seconds
- Add and remove a ticker
- Buy shares: cash decreases, position appears (via `GetSandboxPortfolio`), portfolio totals update
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colours, P&L chart has data points after the snapshot worker has run
- AI chat (mocked): message → response → inline trade confirmation → position actually appears in `GetSandboxPortfolio`
- SSE resilience: kill + reconnect the stream, UI health dot transitions red → yellow → green
- T-Invest stream resilience: simulate upstream disconnect (via test toggle), backend re-subscribes, SSE clients continue receiving ticks once reconnected

### What's NOT tested in CI

- Prod T-Invest trading. The test pipeline only ever hits `sandbox-invest-public-api.tbank.ru:443`. There is no code path that defaults to the prod endpoint — it requires an explicit `TINVEST_ENDPOINT` override.

---

## Appendix A — T-Invest SDK references

- SDK repo: https://opensource.tbank.ru/invest/invest-csharp
- SDK samples: https://opensource.tbank.ru/invest/invest-csharp/-/tree/master/Tinkoff.InvestApi.Sample
- C# SDK docs: https://developer.tbank.ru/invest/sdk/faq_csharp/
- Sandbox docs: https://developer.tbank.ru/invest/intro/developer/sandbox/
- Sandbox methods: https://developer.tbank.ru/invest/intro/developer/sandbox/methods
- API overview: https://developer.tbank.ru/invest/intro/intro
- Token: https://developer.tbank.ru/invest/intro/intro/token
- Glossary (for Quotation / MoneyValue / FIGI / InstrumentUid): https://developer.tbank.ru/invest/intro/intro/glossary

## Appendix B — SDK wiring sketch

```csharp
// Program.cs
builder.Services.AddInvestApiClient((_, settings) =>
{
    settings.AccessToken = Environment.GetEnvironmentVariable("TINVEST_TOKEN")
        ?? throw new InvalidOperationException("TINVEST_TOKEN is required");
    // Endpoint defaults to prod in the SDK — override for sandbox:
    settings.Endpoint = Environment.GetEnvironmentVariable("TINVEST_ENDPOINT")
        ?? "sandbox-invest-public-api.tbank.ru:443";
});

builder.Services.AddSingleton<PriceCache>();
builder.Services.AddSingleton<InstrumentResolver>();
builder.Services.AddHostedService<MarketDataStreamWorker>();
builder.Services.AddHostedService<PortfolioSnapshotWorker>();
builder.Services.AddHostedService<AccountBootstrapService>();

builder.Services.AddDbContext<FinAllyDbContext>(opt =>
    opt.UseSqlite($"Data Source={Path.Combine("/app/db", "finally.db")}"));
```

Note: the exact package name / registration helper may differ slightly — verify against the current SDK README at `https://opensource.tbank.ru/invest/invest-csharp` before writing code.

## Appendix C — Quotation conversion

T-Invest numbers come as `Quotation { long units; int nano }` (nano is 1e-9). Every service that touches money/prices must convert once at the boundary:

```csharp
public static decimal ToDecimal(this Quotation q) =>
    q.Units + q.Nano / 1_000_000_000m;

public static decimal ToDecimal(this MoneyValue m) =>
    m.Units + m.Nano / 1_000_000_000m;
```

No `double` anywhere in financial math — always `decimal`.
