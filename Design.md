# Finance Game — Architecture Design Spec
**Date:** 2026-04-25
**Status:** Approved

---

## 1. Overview

A web-based multiplayer finance management game where players build businesses, invest in a shared financial market, and navigate a dynamic macro-economy. The shared market is the core differentiator: player trades affect prices for all participants.

**Target scale:** Small indie launch — dozens to hundreds of concurrent players.
**Pacing:** Active real-time (tick-based, ~5 seconds per tick = 1 in-game week).
**Economy model:** Simplified models that borrow real financial concepts without full mathematical rigor.

---

## 2. Architecture Overview

**Pattern:** Client-Server, Server-Authoritative Modular Monolith.

All game math runs on the backend. The frontend is responsible for UI and user input only — it never computes financial outcomes.

**Stack:**
| Layer | Technology |
|---|---|
| Frontend | React + TypeScript + Zustand + Chart.js + Socket.io client + Vite |
| Backend | NestJS (Node.js + TypeScript) |
| Real-time | Socket.io |
| Primary DB | PostgreSQL |
| Live cache | Redis |
| Auth | JWT (stateless, included in Socket.io handshake) |

---

## 3. Frontend

### State Management — Three Zustand Stores

- **`marketStore`** — live asset prices, order book, player portfolio positions
- **`businessStore`** — player's businesses, revenue/expense per tick, upgrade state
- **`gameStore`** — macro indicators (inflation, interest rate, GDP), current tick number, leaderboard snapshot, active events

### Tick Handling

The frontend subscribes to a `tick` WebSocket event. Each tick payload contains updated market prices, macro indicators, business financials for the tick, active events, and the leaderboard. The UI re-renders reactively from store updates — no polling.

### Key Screens

- **Dashboard** — net worth trend (Chart.js line chart), macro indicators panel, active events banner
- **Market** — live price table with sparklines, buy/sell panel, portfolio positions
- **Business** — business card per owned business, sliders for price/staff/marketing, revenue breakdown chart
- **Leaderboard** — real-time ranked list by net worth, updated each tick
- **Admin Panel** — protected route (`/admin`), visible to admin-role users only; covers event management, macro controls, player management

### Socket.io Connection Lifecycle

- Connect on login with JWT in the Socket.io handshake
- On disconnect/reconnect, client calls `GET /game/state` to receive a full state snapshot before resuming live updates

---

## 4. Backend Modules (NestJS)

Each engine is a NestJS module. All inter-engine communication is via NestJS dependency injection — no network hops.

### MacroEngine

Runs once per tick. Maintains four global indicators: GDP growth rate, inflation rate, interest rate, consumer confidence index. Each indicator follows a bounded random walk with a slow sine-wave trend (simulating business cycles). The EventManager can apply step-change modifiers to any indicator. Outputs a `MacroSnapshot` consumed by MarketEngine and BusinessEngine.

### MarketEngine

Runs once per tick. Reads `MacroSnapshot`. Maintains a price series for each asset class: stocks, bonds, real estate.

**Price update formula per asset:**
```
newPrice = prevPrice × (1 + drift + noise)
```
- `drift` — derived from macro factors (e.g. rising interest rates suppress tech stocks, increase bond yields)
- `noise` — small bounded random term
- **Player market impact** — trades queued in `tick_actions_queue` (Redis list) during the previous tick are popped at the start of step 3, processed simultaneously, and their signed deltas (`tradeSize / assetMarketCap`) are applied to the price before broadcasting. Orders that arrive mid-tick are validated immediately but deferred to the queue — never applied to a tick already in progress.

### BusinessEngine

Runs once per tick per active business. Computes revenue using a demand function:
```
revenue = baseRevenue × demandMultiplier × (1 / priceElasticity)
```
- `demandMultiplier` — driven by consumer confidence index from MacroEngine
- Player-controlled inputs: price point, staff count, marketing spend
- `priceElasticity` is a fixed constant per business type (e.g. retail = 1.5, tech startup = 0.8, real estate = 0.4) — higher elasticity means revenue is more sensitive to the player's price point setting
- Supports three business types at launch: **Retail**, **Tech Startup**, **Real Estate Venture** — each with distinct base revenue, cost structure, and elasticity constant

Outputs net profit/loss for the tick, written to the transaction ledger.

### EventManager

Evaluated once per tick. Maintains a library of named events with weighted probabilities. Each event applies temporary multiplier modifiers to MacroEngine outputs for a defined duration in ticks. Examples: "Tech Boom" (+20% tech stock drift, 10 ticks), "Recession" (-30% consumer confidence, 20 ticks), "Supply Shock" (+15% inflation, 8 ticks).

Fired events are broadcast to all clients immediately via Socket.io.

### LeaderboardService

After each tick, recalculates net worth for all active players:
```
netWorth = cashBalance + Σ(positionSize × currentPrice) + Σ(businessValuation)
// businessValuation = EMA(netProfit, α=0.1) × 52
// EMA smooths rapid slider changes; α=0.1 makes valuation "sticky" over ~10 ticks
```
Broadcasts the top 100 players by net worth to the `market` room. Players outside the top 100 receive their personal rank in their individual room payload only — keeping the broadcast payload size bounded regardless of player count.

### AdminModule

Protected by an `AdminGuard` (role check on JWT). Exposes:
- Manually trigger or cancel any event
- Nudge macro indicator values live
- View all active player sessions
- Ban a player (disconnect + flag in DB)
- Reset a player's game state

### GameGateway

The Socket.io server. Responsibilities:
- Authenticate connections via JWT handshake
- Join players to personal room + global `market` room; admins also join `admin` room
- Receive and validate inbound player actions (buy/sell orders, business setting changes); validated orders are pushed to `tick_actions_queue` in Redis — never applied mid-tick
- Broadcast tick payload to the `market` room after each tick completes
- Rate-limit inbound actions per player

---

## 5. Data Layer

### PostgreSQL Schema (source of truth)

| Table | Purpose |
|---|---|
| `users` | Auth credentials, role (`player` \| `admin`), created_at |
| `game_sessions` | One row per player's active game — cash balance, current tick |
| `portfolio_positions` | Each player's holdings per asset |
| `businesses` | Player-owned businesses — type, settings, current valuation |
| `transactions` | **Append-only ledger** — every trade, revenue, and expense. Never updated, only inserted. |
| `event_log` | Record of every fired event with tick timestamp and modifier details |

**ACID guarantee on trades:** Buy/sell orders run inside a PostgreSQL transaction. Steps: debit cash → credit position (or reverse) → insert ledger row. Any failure rolls back the entire transaction. No money is created or lost by partial failures.

### Redis (live game state)

- Current market prices for all assets (written each tick, read by all clients)
- Active player sessions and last-known state
- Current `MacroSnapshot`
- Active event modifiers and their remaining tick durations
- Current leaderboard snapshot

Redis is the fast path for everything live. PostgreSQL is the recovery path. A checkpoint job flushes Redis game state to PostgreSQL every 2 minutes.

**Crash recovery:** Trades are always written to PostgreSQL immediately (ACID). Non-trade state (macro indicators, business revenues, tick counter) lives in Redis and may be up to 2 minutes stale after a crash. On server startup, the `RecoveryService` loads the last PostgreSQL checkpoint into Redis, then replays all `transactions` ledger rows with a `created_at` timestamp after the checkpoint to reconstruct accurate cash balances and positions. Macro/event state resets to the last checkpoint — acceptable data loss for non-financial state.

---

## 6. Real-time Communication & Tick System

### Tick Loop

A NestJS `@Cron` job fires every 5 seconds (configurable). Execution order per tick:

1. MacroEngine updates global indicators
2. EventManager evaluates random triggers, applies new event modifiers
3. MarketEngine pops all entries from `tick_actions_queue`, applies player market impact, then computes new prices
4. BusinessEngine computes revenue/loss per active business, writes to ledger
5. LeaderboardService recalculates net worth for all players
6. GameGateway broadcasts single `tick` payload to all clients in the `market` room

### Tick Payload Shape

```typescript
// Broadcast to `market` room (all players)
interface TickBroadcast {
  tick: number;
  macro: MacroSnapshot;
  prices: Record<AssetId, number>;
  activeEvents: GameEvent[];
  leaderboard: { userId: string; netWorth: number }[]; // top 100 only
}

// Sent to each player's personal room
interface PlayerTickUpdate {
  tick: number;
  cash: number;
  positions: Position[];
  businesses: Business[];
  rank: number; // player's global rank (included whether or not they're in top 100)
}
```

### REST API Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/auth/register` | Register new user |
| POST | `/auth/login` | Login, receive JWT |
| GET | `/game/state` | Full state snapshot (used on reconnect) |
| POST | `/market/order` | Submit buy/sell order |
| GET | `/leaderboard` | Current standings |
| POST | `/business/create` | Create a new business |
| PATCH | `/business/:id/settings` | Update business settings |
| POST | `/admin/events/trigger` | Trigger an event (admin) |
| DELETE | `/admin/events/:id` | Cancel an active event (admin) |
| PATCH | `/admin/macro` | Adjust macro indicators (admin) |
| GET | `/admin/sessions` | View all active sessions (admin) |
| POST | `/admin/players/:id/ban` | Ban a player (admin) |
| POST | `/admin/players/:id/reset` | Reset player game state (admin) |

### Rate Limiting

- REST endpoints: NestJS `ThrottlerModule`, global default + stricter limits on `/market/order`
- WebSocket actions: per-player rate limit enforced in `GameGateway` before any engine is touched

---

## 7. Security & Anti-Cheat

- **Server-authoritative:** All financial outcomes computed server-side. Frontend sends intent ("buy 10 shares of X"), backend validates and executes.
- **Server-side validation on every order:** Does the player have sufficient funds? Is the price current? Is the asset tradeable?
- **Append-only ledger:** Full audit trail of all financial movements.
- **JWT auth:** Stateless, included in Socket.io handshake. Tokens expire; refresh flow required.
- **Role guards:** `AdminGuard` on all `/admin` routes and admin Socket.io events.
- **Rate limiting:** Prevents market timing exploitation and server abuse.

---

## 8. What Changed from Original Proposal

| Original | Change | Reason |
|---|---|---|
| Svelte / Vue / React | React only | Decision made, remove optionality |
| Redux / Pinia | Zustand | Lighter, simpler, better suited to game state |
| D3.js | Removed | Chart.js is sufficient; D3 is overkill for a game |
| Generic Express/NestJS/FastAPI/Go | NestJS specifically | Module system maps to engine domains; TypeScript throughout |
| Geometric Brownian Motion | Mean-reverting random walk + macro modifiers | Easier to balance and tune without losing realism feel |
| No player market impact | Added | Core to the shared market mechanic |
| No leaderboard service | Added | Required for competitive multiplayer |
| No admin interface | Added | Necessary for live game management |
| No race condition handling | Added Action Queue pattern | Orders validated immediately, queued in Redis, applied atomically at tick start — never mid-tick |
| Rolling 10-tick average for business valuation | EMA (α=0.1) | Prevents net worth from swinging wildly when players tweak sliders |
| Full leaderboard in tick broadcast | Top 100 broadcast + personal rank in player room | Keeps payload size bounded at scale |
| No crash recovery mechanism | Added RecoveryService | Replays ledger from last checkpoint on startup to restore financial state |
