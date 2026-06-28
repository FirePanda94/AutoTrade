# AutoTrade

An autonomous AI trading floor where four LLM-powered traders compete on a simulated stock market. Each trader is an [OpenAI Agents](https://github.com/openai/openai-agents-python) agent wired to multiple [MCP](https://modelcontextprotocol.io/) servers for accounts, market data, web research, and persistent memory. A separate web dashboard reads portfolio state and activity logs in real time.

## Overview

Four named traders run on a schedule. On each cycle they research the market, decide whether to buy or sell, execute trades through MCP tools, and log their reasoning. Traders alternate between **finding new opportunities** and **rebalancing** their existing portfolio.

| Trader | Style | Default model |
|--------|-------|---------------|
| Warren Patience | Long-term, cautious | GPT 5.4 mini |
| George Bold | Aggressive | GPT 5.4 mini |
| Ray Systematic | Rules-based | GPT 5.4 mini |
| Cathie Crypto | High-growth / thematic | GPT 5.4 mini |

Each trader starts with **$10,000** cash. Trades apply a small bid–ask spread (0.2%).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Trading floor scheduler                   │
│              backend/trading_floor.py                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ Warren  │      │ George  │  …   │ Cathie  │   OpenAI Agents
    └────┬────┘      └────┬────┘      └────┬────┘
         │                │                │
         └────────────────┼────────────────┘
                          │ MCP (stdio)
    ┌─────────────────────┼─────────────────────┐
    ▼                     ▼                     ▼
 accounts_server    push_server          market data
 (buy/sell/balance) (notifications)   Massive API or simulator
                          │
              researcher tools (per trader)
         fetch · Tavily search · libSQL memory
                          │
                          ▼
                   accounts.db (SQLite)
                          │
                          ▼
              FastAPI read-only API (main.py)
                          │
                          ▼
              Vite frontend (frontend/)
```

The trading engine **writes** to `accounts.db`. The HTTP API and dashboard **only read** from it, so the frontend can run independently of the agent loop.

## Prerequisites

- [Python 3.13+](https://www.python.org/)
- [uv](https://docs.astral.sh/uv/) (recommended package manager)
- [Node.js 18+](https://nodejs.org/) and npm
- `npx` (bundled with Node) — used to launch Tavily and memory MCP servers at runtime

## Setup

### 1. Clone and install Python dependencies

From the project root:

```bash
uv sync
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```env
# Required
OPENAI_API_KEY=sk-...

# Recommended — powers the researcher's web search MCP server
TAVILY_API_KEY=tvly-...

# Optional — live US market data via Massive; without it, prices are simulated
MASSIVE_API_KEY=

# Optional tuning
RUN_EVERY_N_MINUTES=60
RUN_EVEN_WHEN_MARKET_IS_CLOSED=false
USE_MANY_MODELS=false
```

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | Yes | Powers all trader and researcher agents |
| `TAVILY_API_KEY` | Recommended | Web search for the researcher MCP server |
| `MASSIVE_API_KEY` | No | Live market data; omit to use the built-in price simulator |
| `RUN_EVERY_N_MINUTES` | No | Minutes between trading cycles (default: `60`) |
| `RUN_EVEN_WHEN_MARKET_IS_CLOSED` | No | Run cycles even when the US market is closed (default: `false`) |
| `USE_MANY_MODELS` | No | Assign a different OpenAI model to each trader (default: `false`) |

### 3. Install frontend dependencies

```bash
cd frontend
npm install
cd ..
```

## Running

You need **two or three processes** depending on whether you want the dashboard.

### Start the trading floor

This is the main engine. It launches MCP servers and runs all four traders on the configured schedule:

```bash
uv run python -m backend.trading_floor
```

On first run, trader accounts are created automatically in `accounts.db`.

### Start the API (for the dashboard)

In a second terminal, from the project root:

```bash
uv run uvicorn main:app --port 8000
```

Endpoints:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/traders` | List all traders |
| `GET` | `/api/traders/{name}` | Portfolio, holdings, transactions, P&amp;L |
| `GET` | `/api/traders/{name}/logs` | Recent activity log lines |
| `GET` | `/api/market` | Market data source and open/closed status |

### Start the frontend

In a third terminal:

```bash
cd frontend
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). The Vite dev server proxies `/api` requests to the FastAPI backend on port 8000.

## MCP servers

Each trader connects to several stdio MCP servers:

**Trader tools**

| Server | Module | Purpose |
|--------|--------|---------|
| Accounts | `backend.accounts_server` | Balance, holdings, buy/sell, strategy |
| Push | `backend.push_server` | Post-trade notifications (stub by default) |
| Market | Massive MCP or `backend.market_server` | Share price lookup |

**Researcher tools** (sub-agent, one memory DB per trader)

| Server | Package | Purpose |
|--------|---------|---------|
| Fetch | `mcp-server-fetch` | Fetch web pages |
| Search | `tavily-mcp` | Tavily web search |
| Memory | `mcp-memory-libsql` | Persistent knowledge graph (`memory/{name}.db`) |

When `MASSIVE_API_KEY` is set, market data comes from the [Massive MCP server](https://github.com/massive-com/mcp_massive). Otherwise, prices are generated deterministically by `backend/market_simulator.py` so the project runs without external market credentials.

## Project structure

```
AutoTrade/
├── main.py                  # FastAPI read-only HTTP API
├── backend/
│   ├── trading_floor.py     # Scheduler — entry point for the engine
│   ├── traders.py           # Agent definitions and run loop
│   ├── mcp_servers.py       # MCP server configuration
│   ├── accounts.py          # Account model and trade logic
│   ├── accounts_server.py   # Accounts MCP server
│   ├── market.py            # Price lookup (Massive or simulator)
│   ├── market_server.py     # Market MCP server (simulator mode)
│   ├── market_simulator.py  # Deterministic simulated prices
│   ├── push_server.py       # Push notification MCP server
│   ├── database.py          # SQLite persistence (accounts.db)
│   ├── templates.py         # Agent prompts
│   └── tracers.py           # Trace logging to SQLite
├── frontend/                # Vite + TypeScript dashboard
│   └── src/
├── accounts.db              # Created at runtime
└── memory/                  # Per-trader MCP memory databases
```

## Development

Build the frontend for production:

```bash
cd frontend
npm run build
npm run preview
```

Reset a trader account by deleting their row from `accounts.db`, or by calling the account reset logic in `backend/accounts.py` from a Python shell.

## Disclaimer

This project is for **educational and experimental purposes only**. Autonomous agents placing trades — even on simulated accounts — can behave unpredictably. Do not connect this to real brokerage accounts or use it as financial advice.
