# Finance Game — Frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the React frontend — auth screens, real-time market dashboard, business management, leaderboard, and admin panel — consuming the NestJS backend via REST and Socket.io.

**Prerequisite:** The backend plan (`2026-04-25-finance-game-backend.md`) must be complete and the server must be running at `http://localhost:3000`.

**Architecture:** Vite + React + TypeScript. Three Zustand stores (`marketStore`, `businessStore`, `gameStore`) receive data from Socket.io `tick` and `player_tick` events. REST calls handle auth, order placement, and business management. Chart.js renders the net worth trend line.

**Tech Stack:** React 18 + TypeScript, Vite, Zustand, Socket.io client, Chart.js (react-chartjs-2), React Router v6, Axios

---

## File Map

```
frontend/
  src/
    main.tsx
    App.tsx                         # Router + auth wrapper
    socket.ts                       # Socket.io singleton
    api/
      client.ts                     # Axios instance with JWT header
      auth.ts                       # register(), login()
      game.ts                       # getState()
      market.ts                     # placeOrder()
      business.ts                   # createBusiness(), updateSettings()
      admin.ts                      # triggerEvent(), nudgeMacro(), getSessions(), ban(), reset()
    stores/
      authStore.ts                  # token, userId, role, login/logout actions
      marketStore.ts                # prices, positions, placeOrder action
      businessStore.ts              # businesses, create/update actions
      gameStore.ts                  # macro, tick, leaderboard, activeEvents, playerRank, cash
    types/
      index.ts                      # Shared interfaces matching backend responses
    pages/
      Login.tsx
      Register.tsx
      Dashboard.tsx
      Market.tsx
      Business.tsx
      Leaderboard.tsx
      Admin.tsx
    components/
      Layout.tsx                    # Nav sidebar + outlet
      ProtectedRoute.tsx
      AdminRoute.tsx
      NetWorthChart.tsx             # Chart.js line chart
      MacroPanel.tsx                # Four macro indicator tiles
      EventBanner.tsx               # Active events ticker
      PriceTable.tsx                # Live sortable asset table
      BuySellPanel.tsx              # Order form
      PositionsList.tsx
      BusinessCard.tsx              # Settings sliders + revenue display
      LeaderboardTable.tsx
      admin/
        SessionsTable.tsx
        EventControls.tsx
        MacroControls.tsx
```

---

### Task 1: Project Scaffold

**Files:**
- Create: `frontend/` (Vite + React project)
- Create: `frontend/src/types/index.ts`

- [ ] **Step 1: Scaffold with Vite**

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
```

- [ ] **Step 2: Install dependencies**

```bash
npm install \
  zustand socket.io-client axios \
  react-router-dom \
  chart.js react-chartjs-2 \
  && npm install --save-dev @types/node
```

- [ ] **Step 3: Define shared types**

Create `frontend/src/types/index.ts`:
```typescript
export interface MacroSnapshot {
  gdpGrowth: number;
  inflation: number;
  interestRate: number;
  consumerConfidence: number;
}

export interface AssetPrice {
  id: string;
  price: number;
}

export interface Position {
  assetId: string;
  quantity: number;
}

export interface BusinessRow {
  id: string;
  type: 'retail' | 'tech_startup' | 'real_estate';
  pricePoint: number;
  staffCount: number;
  marketingSpend: number;
  profitEma: number;
}

export interface RankedPlayer {
  userId: string;
  netWorth: number;
  rank: number;
}

export interface ActiveEvent {
  id: string;
  name: string;
  description: string;
  remainingTicks: number;
}

export interface TickBroadcast {
  tick: number;
  macro: MacroSnapshot;
  prices: Record<string, number>;
  leaderboard: RankedPlayer[];
  activeEvents: ActiveEvent[];
  newEvent?: { id: string; name: string };
}

export interface PlayerTickUpdate {
  tick: number;
  cash: number;
  positions: Position[];
  businesses: BusinessRow[];
  rank: number;
}

export interface GameState {
  tick: number;
  macro: MacroSnapshot;
  prices: Record<string, number>;
  leaderboard: RankedPlayer[];
  activeEvents: ActiveEvent[];
  player: {
    cash: number;
    positions: Position[];
    businesses: BusinessRow[];
  };
}
```

- [ ] **Step 4: Delete Vite boilerplate**

```bash
rm src/App.css src/assets/react.svg public/vite.svg
```

Replace `src/index.css` with the full design token system. All components reference these classes — no inline styles anywhere:

```css
/* ===== Design Tokens ===== */
:root {
  --bg-base:     #0f1117;
  --bg-surface:  #1a202c;
  --bg-elevated: #2d3748;
  --bg-hover:    #374151;

  --text-primary:   #e2e8f0;
  --text-secondary: #a0aec0;
  --text-muted:     #718096;

  --accent-blue:       #63b3ed;
  --accent-blue-dark:  #3182ce;
  --accent-green:      #68d391;
  --accent-green-dark: #276749;
  --accent-red:        #fc8181;
  --accent-red-dark:   #9b2c2c;
  --accent-gold:       #f6ad55;

  --border:     1px solid rgba(255, 255, 255, 0.06);
  --radius:     8px;
  --radius-sm:  4px;
  --transition: all 0.15s ease;
}

/* ===== Reset ===== */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

/* ===== Base ===== */
body {
  font-family: system-ui, -apple-system, sans-serif;
  background: var(--bg-base);
  color: var(--text-primary);
  line-height: 1.5;
}

/* ===== Layout ===== */
.layout        { display: flex; height: 100vh; }
.sidebar       { width: 200px; background: var(--bg-surface); padding: 1rem; display: flex; flex-direction: column; gap: 0.25rem; border-right: var(--border); }
.sidebar-logo  { font-weight: 700; font-size: 1.1rem; color: var(--accent-blue); margin-bottom: 1rem; padding: 0.5rem 0.75rem; }
.sidebar-spacer { flex: 1; }
.main-content  { flex: 1; overflow: auto; padding: 1.5rem; }

.nav-link {
  display: block;
  color: var(--text-muted);
  text-decoration: none;
  padding: 0.5rem 0.75rem;
  border-radius: var(--radius-sm);
  transition: var(--transition);
  font-size: 0.9rem;
}
.nav-link:hover                 { color: var(--text-primary); background: var(--bg-elevated); }
.nav-link.active                { color: var(--accent-blue); background: rgba(99, 179, 237, 0.1); }
.nav-link.active--admin         { color: var(--accent-gold); background: rgba(246, 173, 85, 0.1); }

/* ===== Page header ===== */
.page-header   { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; }
.page-title    { font-size: 1.5rem; font-weight: 700; }
.page-subtitle { color: var(--text-muted); font-size: 0.875rem; }

/* ===== Grid ===== */
.grid         { display: grid; gap: 1rem; }
.grid--2      { grid-template-columns: 1fr 1fr; }
.grid--3      { grid-template-columns: repeat(3, 1fr); }
.grid--4      { grid-template-columns: repeat(4, 1fr); }
.grid--auto   { grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); }
.grid--market { grid-template-columns: 1fr 320px; }

/* ===== Cards ===== */
.card {
  background: var(--bg-surface);
  border-radius: var(--radius);
  padding: 1.5rem;
  border: var(--border);
}
.card--sm { padding: 1rem; }
.card--glass {
  background: rgba(26, 32, 44, 0.85);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(99, 179, 237, 0.15);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
}
.card-label { color: var(--text-muted); font-size: 0.7rem; text-transform: uppercase; letter-spacing: 0.06em; margin-bottom: 0.25rem; }
.card-value { font-size: 1.75rem; font-weight: 700; }

/* ===== Stat tiles ===== */
.stat-tile { background: var(--bg-surface); padding: 1rem; border-radius: var(--radius); border: var(--border); transition: border-color 0.2s ease; }
.stat-tile:hover { border-color: rgba(99, 179, 237, 0.25); }

/* ===== Buttons ===== */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.6rem 1.25rem;
  border: none;
  border-radius: var(--radius-sm);
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: var(--transition);
  white-space: nowrap;
}
.btn:disabled { opacity: 0.45; cursor: not-allowed; }

.btn--primary { background: var(--accent-blue-dark); color: white; }
.btn--primary:hover:not(:disabled) { background: #4299e1; transform: translateY(-1px); box-shadow: 0 4px 12px rgba(49, 130, 206, 0.4); }

.btn--success { background: var(--accent-green-dark); color: white; }
.btn--success:hover:not(:disabled) { background: #38a169; transform: translateY(-1px); box-shadow: 0 4px 12px rgba(39, 103, 73, 0.4); }

.btn--danger { background: var(--accent-red-dark); color: white; }
.btn--danger:hover:not(:disabled) { background: #c53030; transform: translateY(-1px); }

.btn--ghost { background: var(--bg-elevated); color: var(--text-secondary); border: 1px solid #4a5568; }
.btn--ghost:hover:not(:disabled) { background: var(--bg-hover); color: var(--text-primary); }

.btn--full { width: 100%; }
.btn--sm   { padding: 0.3rem 0.6rem; font-size: 0.75rem; }
.btn--tab  { flex: 1; border-radius: var(--radius-sm); }
.btn--tab.active--buy  { background: var(--accent-green-dark); color: white; }
.btn--tab.active--sell { background: var(--accent-red-dark); color: white; }

/* ===== Inputs ===== */
.input {
  width: 100%;
  padding: 0.75rem;
  background: var(--bg-elevated);
  border: 1px solid #4a5568;
  color: var(--text-primary);
  border-radius: var(--radius-sm);
  font-size: 1rem;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}
.input:focus { outline: none; border-color: var(--accent-blue); box-shadow: 0 0 0 3px rgba(99, 179, 237, 0.15); }

.input[type="range"] {
  -webkit-appearance: none;
  display: block;
  margin-top: 0.25rem;
  height: 4px;
  border: none;
  padding: 0;
  background: var(--bg-elevated);
  border-radius: 2px;
  cursor: pointer;
}
.input[type="range"]::-webkit-slider-thumb { -webkit-appearance: none; width: 14px; height: 14px; background: var(--accent-blue); border-radius: 50%; cursor: pointer; }

.form-group { display: flex; flex-direction: column; gap: 0.5rem; }
.form-label { color: var(--text-secondary); font-size: 0.8rem; }

/* ===== Tables ===== */
.table { width: 100%; border-collapse: collapse; }
.table th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  color: var(--text-muted);
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  border-bottom: 1px solid var(--bg-elevated);
}
.table td { padding: 0.75rem; }
.table tbody tr { border-bottom: 1px solid rgba(255, 255, 255, 0.03); transition: background 0.1s ease; cursor: default; }
.table tbody tr:hover { background: var(--bg-elevated); }
.table tbody tr.row--active { background: rgba(99, 179, 237, 0.1); }
.table tbody tr.row--self   { background: rgba(99, 179, 237, 0.07); }
.table .col-right { text-align: right; }

/* ===== Price / status colouring ===== */
.price--up   { color: var(--accent-green); }
.price--down { color: var(--accent-red); }
.rank--top3  { color: var(--accent-gold); font-weight: 700; }
.rank--self  { color: var(--accent-blue); }

/* ===== Status messages ===== */
.status-success { color: var(--accent-green); font-size: 0.875rem; }
.status-error   { color: var(--accent-red);   font-size: 0.875rem; }
.status-muted   { color: var(--text-muted);   font-size: 0.875rem; }

/* ===== Event banner ===== */
.event-banner {
  background: linear-gradient(135deg, #744210, #92400e);
  border: 1px solid rgba(246, 173, 85, 0.3);
  padding: 0.75rem 1rem;
  border-radius: var(--radius);
  margin-bottom: 1rem;
  animation: slideDown 0.3s ease;
}
.event-banner__row  { display: flex; justify-content: space-between; align-items: baseline; gap: 1rem; }
.event-banner__name { color: var(--accent-gold); font-weight: 600; }
.event-banner__desc { color: #fbd38d; font-size: 0.875rem; }

/* ===== Auth screen ===== */
.auth-wrapper { max-width: 400px; margin: 18vh auto; }
.auth-title   { font-size: 1.5rem; font-weight: 700; color: var(--accent-blue); margin-bottom: 1.5rem; }
.auth-footer  { margin-top: 1rem; color: var(--text-muted); font-size: 0.9rem; }
.auth-footer a { color: var(--accent-blue); text-decoration: none; }
.auth-footer a:hover { text-decoration: underline; }

/* ===== Business type picker ===== */
.business-picker__card {
  padding: 1rem;
  background: var(--bg-elevated);
  border: 1px solid #4a5568;
  border-radius: var(--radius);
  cursor: pointer;
  text-align: left;
  transition: var(--transition);
  color: var(--text-primary);
}
.business-picker__card:hover:not(:disabled) { border-color: var(--accent-blue); background: var(--bg-hover); transform: translateY(-2px); }
.business-picker__card-title { font-weight: 600; margin-bottom: 0.25rem; }
.business-picker__card-desc  { color: var(--text-muted); font-size: 0.8rem; }

/* ===== Admin panel ===== */
.admin-title { color: var(--accent-gold); }

/* ===== Animations ===== */
@keyframes slideDown {
  from { opacity: 0; transform: translateY(-6px); }
  to   { opacity: 1; transform: translateY(0); }
}
@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}
@keyframes tickFlash {
  0%   { background: rgba(99, 179, 237, 0.12); }
  100% { background: transparent; }
}

.fade-in   { animation: fadeIn 0.2s ease; }
.tick-flash { animation: tickFlash 0.5s ease; }

/* ===== Responsive ===== */
@media (max-width: 768px) {
  .layout        { flex-direction: column; }
  .sidebar       { width: 100%; flex-direction: row; flex-wrap: wrap; }
  .grid--2, .grid--3, .grid--4 { grid-template-columns: 1fr; }
  .grid--market  { grid-template-columns: 1fr; }
}

/* ===== Spacing utilities ===== */
.mb-1 { margin-bottom: 0.5rem; }
.mb-2 { margin-bottom: 1rem; }
.mb-3 { margin-bottom: 1.5rem; }
.mb-4 { margin-bottom: 2rem; }

/* ===== Flex helpers ===== */
.flex-between { display: flex; justify-content: space-between; align-items: center; }
.flex-wrap    { display: flex; flex-wrap: wrap; gap: 0.5rem; }
.flex-col     { display: flex; flex-direction: column; gap: 1rem; }

/* ===== Section heading inside a card ===== */
.section-label { color: var(--text-secondary); font-size: 0.875rem; margin-bottom: 0.75rem; }

/* ===== Chart empty state ===== */
.chart-placeholder { color: var(--text-muted); padding: 2rem; text-align: center; font-size: 0.875rem; }

/* ===== Admin macro nudge row ===== */
.macro-row        { display: flex; align-items: center; gap: 0.5rem; }
.macro-row__label { flex: 0 0 160px; color: var(--text-secondary); font-size: 0.8rem; }
```

- [ ] **Step 5: Commit**

```bash
git init && git add . && git commit -m "chore: scaffold React frontend with Vite, Zustand, Socket.io"
```

---

### Task 2: API Client & Auth API

**Files:**
- Create: `frontend/src/api/client.ts`
- Create: `frontend/src/api/auth.ts`
- Create: `frontend/src/api/game.ts`
- Create: `frontend/src/api/market.ts`
- Create: `frontend/src/api/business.ts`
- Create: `frontend/src/api/admin.ts`

- [ ] **Step 1: Create Axios instance**

Create `frontend/src/api/client.ts`:
```typescript
import axios from 'axios';

export const api = axios.create({
  baseURL: 'http://localhost:3000',
});

export function setAuthToken(token: string | null) {
  if (token) {
    api.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  } else {
    delete api.defaults.headers.common['Authorization'];
  }
}
```

- [ ] **Step 2: Create all API modules**

Create `frontend/src/api/auth.ts`:
```typescript
import { api } from './client';

export const register = (email: string, password: string) =>
  api.post<{ accessToken: string }>('/auth/register', { email, password });

export const login = (email: string, password: string) =>
  api.post<{ accessToken: string }>('/auth/login', { email, password });
```

Create `frontend/src/api/game.ts`:
```typescript
import { api } from './client';
import type { GameState } from '../types';

export const getState = () => api.get<GameState>('/game/state');
```

Create `frontend/src/api/market.ts`:
```typescript
import { api } from './client';

export const placeOrder = (assetId: string, quantity: number, side: 'buy' | 'sell') =>
  api.post<{ queued: boolean }>('/market/order', { assetId, quantity, side });
```

Create `frontend/src/api/business.ts`:
```typescript
import { api } from './client';
import type { BusinessRow } from '../types';

export const createBusiness = (type: string) =>
  api.post<BusinessRow>('/business', { type });

export const updateSettings = (id: string, settings: { pricePoint?: number; staffCount?: number; marketingSpend?: number }) =>
  api.patch<BusinessRow>(`/business/${id}/settings`, settings);

export const listBusinesses = () => api.get<BusinessRow[]>('/business');
```

Create `frontend/src/api/admin.ts`:
```typescript
import { api } from './client';

export const getSessions = () => api.get('/admin/sessions');
export const triggerEvent = (eventId: string) => api.post('/admin/events/trigger', { eventId });
export const cancelEvent = (id: string) => api.delete(`/admin/events/${id}`);
export const nudgeMacro = (modifier: Record<string, number>) => api.patch('/admin/macro', modifier);
export const banPlayer = (userId: string) => api.post(`/admin/players/${userId}/ban`);
export const resetPlayer = (userId: string) => api.post(`/admin/players/${userId}/reset`);
```

- [ ] **Step 3: Commit**

```bash
git add . && git commit -m "feat: add API client modules for all backend endpoints"
```

---

### Task 3: Zustand Stores

**Files:**
- Create: `frontend/src/stores/authStore.ts`
- Create: `frontend/src/stores/gameStore.ts`
- Create: `frontend/src/stores/marketStore.ts`
- Create: `frontend/src/stores/businessStore.ts`

- [ ] **Step 1: Create authStore**

Create `frontend/src/stores/authStore.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { setAuthToken } from '../api/client';

interface AuthState {
  token: string | null;
  userId: string | null;
  role: string | null;
  setAuth: (token: string) => void;
  clearAuth: () => void;
}

function parseJwt(token: string): { sub: string; role: string } {
  const base64 = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/');
  return JSON.parse(atob(base64));
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      userId: null,
      role: null,
      setAuth: (token: string) => {
        const payload = parseJwt(token);
        setAuthToken(token);
        set({ token, userId: payload.sub, role: payload.role });
      },
      clearAuth: () => {
        setAuthToken(null);
        set({ token: null, userId: null, role: null });
      },
    }),
    {
      name: 'auth',
      onRehydrateStorage: () => (state) => {
        if (state?.token) setAuthToken(state.token);
      },
    },
  ),
);
```

- [ ] **Step 2: Create gameStore**

Create `frontend/src/stores/gameStore.ts`:
```typescript
import { create } from 'zustand';
import type { ActiveEvent, MacroSnapshot, RankedPlayer } from '../types';

interface GameState {
  tick: number;
  macro: MacroSnapshot | null;
  leaderboard: RankedPlayer[];
  activeEvents: ActiveEvent[];
  cash: number;
  playerRank: number;
  netWorthHistory: { tick: number; value: number }[];
  seedHistory: (initialNetWorth: number, atTick: number) => void;
  applyTickBroadcast: (data: {
    tick: number;
    macro: MacroSnapshot;
    leaderboard: RankedPlayer[];
    activeEvents: ActiveEvent[];
  }) => void;
  applyPlayerTick: (data: { tick: number; cash: number; rank: number }) => void;
}

export const useGameStore = create<GameState>((set, get) => ({
  tick: 0,
  macro: null,
  leaderboard: [],
  activeEvents: [],
  cash: 0,
  playerRank: 0,
  netWorthHistory: [],
  // Called once on socket connect with the value from getState().
  // Pre-seeds the chart with the starting net worth so it isn't
  // blank for the first ~10 ticks (50 seconds) of a session.
  seedHistory: (initialNetWorth, atTick) => {
    const seed = Array.from({ length: 5 }, (_, i) => ({
      tick: Math.max(0, atTick - (4 - i)),
      value: initialNetWorth,
    }));
    set({ netWorthHistory: seed });
  },
  applyTickBroadcast: (data) =>
    set({ tick: data.tick, macro: data.macro, leaderboard: data.leaderboard, activeEvents: data.activeEvents }),
  applyPlayerTick: (data) => {
    const prev = get().netWorthHistory;
    const player = get().leaderboard.find((p) => p.rank === data.rank);
    const entry = player ? { tick: data.tick, value: player.netWorth } : null;
    set({
      cash: data.cash,
      playerRank: data.rank,
      netWorthHistory: entry ? [...prev.slice(-99), entry] : prev,
    });
  },
}));
```

- [ ] **Step 3: Create marketStore**

Create `frontend/src/stores/marketStore.ts`:
```typescript
import { create } from 'zustand';
import type { Position } from '../types';
import { placeOrder as apiPlaceOrder } from '../api/market';

interface MarketState {
  prices: Record<string, number>;
  priceHistory: Record<string, number[]>; // last 20 ticks per asset
  positions: Position[];
  setPrices: (prices: Record<string, number>) => void;
  setPositions: (positions: Position[]) => void;
  placeOrder: (assetId: string, quantity: number, side: 'buy' | 'sell') => Promise<void>;
}

export const useMarketStore = create<MarketState>((set, get) => ({
  prices: {},
  priceHistory: {},
  positions: [],
  setPrices: (prices) => {
    const prev = get().priceHistory;
    const updated: Record<string, number[]> = {};
    for (const [id, price] of Object.entries(prices)) {
      updated[id] = [...(prev[id] ?? []).slice(-19), price];
    }
    set({ prices, priceHistory: updated });
  },
  setPositions: (positions) => set({ positions }),
  placeOrder: async (assetId, quantity, side) => {
    await apiPlaceOrder(assetId, quantity, side);
  },
}));
```

- [ ] **Step 4: Create businessStore**

Create `frontend/src/stores/businessStore.ts`:
```typescript
import { create } from 'zustand';
import type { BusinessRow } from '../types';
import { createBusiness as apiCreate, updateSettings as apiUpdate } from '../api/business';

interface BusinessState {
  businesses: BusinessRow[];
  setBusinesses: (businesses: BusinessRow[]) => void;
  createBusiness: (type: string) => Promise<void>;
  updateSettings: (id: string, settings: { pricePoint?: number; staffCount?: number; marketingSpend?: number }) => Promise<void>;
}

export const useBusinessStore = create<BusinessState>((set, get) => ({
  businesses: [],
  setBusinesses: (businesses) => set({ businesses }),
  createBusiness: async (type) => {
    const res = await apiCreate(type);
    set({ businesses: [...get().businesses, res.data] });
  },
  updateSettings: async (id, settings) => {
    const res = await apiUpdate(id, settings);
    set({
      businesses: get().businesses.map((b) => (b.id === id ? res.data : b)),
    });
  },
}));
```

- [ ] **Step 5: Commit**

```bash
git add . && git commit -m "feat: add Zustand stores for auth, game state, market, and businesses"
```

---

### Task 4: Socket.io Client

**Files:**
- Create: `frontend/src/socket.ts`

- [ ] **Step 1: Create the Socket.io singleton**

Create `frontend/src/socket.ts`:
```typescript
import { io, Socket } from 'socket.io-client';
import { useAuthStore } from './stores/authStore';
import { useGameStore } from './stores/gameStore';
import { useMarketStore } from './stores/marketStore';
import { useBusinessStore } from './stores/businessStore';
import { getState } from './api/game';
import type { TickBroadcast, PlayerTickUpdate } from './types';

let socket: Socket | null = null;

export function connectSocket() {
  const token = useAuthStore.getState().token;
  if (!token || socket?.connected) return;

  socket = io('http://localhost:3000', {
    auth: { token },
    transports: ['websocket'],
  });

  socket.on('connected', async () => {
    // Load full state snapshot on connect/reconnect
    const res = await getState();
    const state = res.data;
    useMarketStore.getState().setPrices(state.prices);
    useMarketStore.getState().setPositions(state.player.positions);
    useBusinessStore.getState().setBusinesses(state.player.businesses);
    useGameStore.getState().applyTickBroadcast({
      tick: state.tick,
      macro: state.macro,
      leaderboard: state.leaderboard,
      activeEvents: state.activeEvents,
    });
    // Seed chart history with the starting net worth so the line chart
    // is visible immediately rather than blank for the first ~10 ticks.
    const initialNetWorth = state.player.cash; // positions + businesses start at 0 on new account
    useGameStore.getState().seedHistory(initialNetWorth, state.tick);
    useGameStore.getState().applyPlayerTick({
      tick: state.tick,
      cash: state.player.cash,
      rank: 0,
    });
  });

  socket.on('tick', (data: TickBroadcast) => {
    useMarketStore.getState().setPrices(data.prices);
    useGameStore.getState().applyTickBroadcast({
      tick: data.tick,
      macro: data.macro,
      leaderboard: data.leaderboard,
      activeEvents: data.activeEvents,
    });
  });

  socket.on('player_tick', (data: PlayerTickUpdate) => {
    useMarketStore.getState().setPositions(data.positions);
    useBusinessStore.getState().setBusinesses(data.businesses);
    useGameStore.getState().applyPlayerTick({
      tick: data.tick,
      cash: data.cash,
      rank: data.rank,
    });
  });

  socket.on('disconnect', () => {
    console.warn('Socket disconnected');
  });
}

export function disconnectSocket() {
  socket?.disconnect();
  socket = null;
}
```

- [ ] **Step 2: Commit**

```bash
git add . && git commit -m "feat: add Socket.io client singleton with full state sync on connect"
```

---

### Task 5: Routing & Layout

**Files:**
- Create: `frontend/src/components/Layout.tsx`
- Create: `frontend/src/components/ProtectedRoute.tsx`
- Create: `frontend/src/components/AdminRoute.tsx`
- Modify: `frontend/src/App.tsx`
- Modify: `frontend/src/main.tsx`

- [ ] **Step 1: Create ProtectedRoute and AdminRoute**

Create `frontend/src/components/ProtectedRoute.tsx`:
```typescript
import { Navigate } from 'react-router-dom';
import { useAuthStore } from '../stores/authStore';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const token = useAuthStore((s) => s.token);
  return token ? <>{children}</> : <Navigate to="/login" replace />;
}
```

Create `frontend/src/components/AdminRoute.tsx`:
```typescript
import { Navigate } from 'react-router-dom';
import { useAuthStore } from '../stores/authStore';

export function AdminRoute({ children }: { children: React.ReactNode }) {
  const role = useAuthStore((s) => s.role);
  return role === 'admin' ? <>{children}</> : <Navigate to="/" replace />;
}
```

- [ ] **Step 2: Create Layout**

Create `frontend/src/components/Layout.tsx`:
```typescript
import { NavLink, Outlet } from 'react-router-dom';
import { useAuthStore } from '../stores/authStore';
import { disconnectSocket } from '../socket';

const navLinks = [
  { to: '/', label: 'Dashboard' },
  { to: '/market', label: 'Market' },
  { to: '/business', label: 'Business' },
  { to: '/leaderboard', label: 'Leaderboard' },
];

export function Layout() {
  const { role, clearAuth } = useAuthStore();

  function handleLogout() {
    disconnectSocket();
    clearAuth();
  }

  return (
    <div className="layout">
      <nav className="sidebar">
        <span className="sidebar-logo">FinanceGame</span>
        {navLinks.map((l) => (
          <NavLink
            key={l.to}
            to={l.to}
            end={l.to === '/'}
            className={({ isActive }) => `nav-link${isActive ? ' active' : ''}`}
          >
            {l.label}
          </NavLink>
        ))}
        {role === 'admin' && (
          <NavLink
            to="/admin"
            className={({ isActive }) => `nav-link${isActive ? ' active active--admin' : ''}`}
          >
            Admin
          </NavLink>
        )}
        <div className="sidebar-spacer" />
        <button className="btn btn--ghost btn--full" onClick={handleLogout}>
          Logout
        </button>
      </nav>
      <main className="main-content">
        <Outlet />
      </main>
    </div>
  );
}
```

- [ ] **Step 3: Wire up App.tsx with routing**

Replace `frontend/src/App.tsx`:
```typescript
import { useEffect } from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import { connectSocket } from './socket';
import { useAuthStore } from './stores/authStore';
import { Layout } from './components/Layout';
import { ProtectedRoute } from './components/ProtectedRoute';
import { AdminRoute } from './components/AdminRoute';
import { Login } from './pages/Login';
import { Register } from './pages/Register';
import { Dashboard } from './pages/Dashboard';
import { Market } from './pages/Market';
import { Business } from './pages/Business';
import { Leaderboard } from './pages/Leaderboard';
import { Admin } from './pages/Admin';

export default function App() {
  const token = useAuthStore((s) => s.token);

  useEffect(() => {
    if (token) connectSocket();
  }, [token]);

  return (
    <Routes>
      <Route path="/login" element={<Login />} />
      <Route path="/register" element={<Register />} />
      <Route
        path="/"
        element={<ProtectedRoute><Layout /></ProtectedRoute>}
      >
        <Route index element={<Dashboard />} />
        <Route path="market" element={<Market />} />
        <Route path="business" element={<Business />} />
        <Route path="leaderboard" element={<Leaderboard />} />
        <Route path="admin" element={<AdminRoute><Admin /></AdminRoute>} />
      </Route>
      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}
```

Update `frontend/src/main.tsx`:
```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>,
);
```

- [ ] **Step 4: Create stub page files (so routing doesn't error)**

Create empty stubs for each page — they'll be filled in the next tasks.

Create `frontend/src/pages/Login.tsx`:
```typescript
export function Login() { return <div>Login</div>; }
```

Create `frontend/src/pages/Register.tsx`:
```typescript
export function Register() { return <div>Register</div>; }
```

Create `frontend/src/pages/Dashboard.tsx`:
```typescript
export function Dashboard() { return <div>Dashboard</div>; }
```

Create `frontend/src/pages/Market.tsx`:
```typescript
export function Market() { return <div>Market</div>; }
```

Create `frontend/src/pages/Business.tsx`:
```typescript
export function Business() { return <div>Business</div>; }
```

Create `frontend/src/pages/Leaderboard.tsx`:
```typescript
export function Leaderboard() { return <div>Leaderboard</div>; }
```

Create `frontend/src/pages/Admin.tsx`:
```typescript
export function Admin() { return <div>Admin</div>; }
```

- [ ] **Step 5: Verify the app compiles and routes work**

```bash
npm run dev
```
Expected: App runs at `http://localhost:5173`. Navigating to `/login` shows "Login". No TypeScript errors.

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add routing, layout, and page stubs"
```

---

### Task 6: Auth Pages

**Files:**
- Modify: `frontend/src/pages/Login.tsx`
- Modify: `frontend/src/pages/Register.tsx`

- [ ] **Step 1: Implement Login page**

Replace `frontend/src/pages/Login.tsx`:
```typescript
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { login } from '../api/auth';
import { useAuthStore } from '../stores/authStore';

export function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const setAuth = useAuthStore((s) => s.setAuth);
  const navigate = useNavigate();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError('');
    try {
      const res = await login(email, password);
      setAuth(res.data.accessToken);
      navigate('/');
    } catch {
      setError('Invalid email or password');
    }
  }

  return (
    <div className="auth-wrapper card card--glass fade-in">
      <h1 className="auth-title">Sign In</h1>
      <form onSubmit={handleSubmit} className="form-group">
        <input className="input" type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <input className="input" type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} />
        {error && <span className="status-error">{error}</span>}
        <button type="submit" className="btn btn--primary btn--full">Sign In</button>
      </form>
      <p className="auth-footer">No account? <Link to="/register">Register</Link></p>
    </div>
  );
}
```

- [ ] **Step 2: Implement Register page**

Replace `frontend/src/pages/Register.tsx`:
```typescript
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { register } from '../api/auth';
import { useAuthStore } from '../stores/authStore';

export function Register() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const setAuth = useAuthStore((s) => s.setAuth);
  const navigate = useNavigate();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError('');
    try {
      const res = await register(email, password);
      setAuth(res.data.accessToken);
      navigate('/');
    } catch (err: any) {
      setError(err?.response?.data?.message ?? 'Registration failed');
    }
  }

  return (
    <div className="auth-wrapper card card--glass fade-in">
      <h1 className="auth-title">Create Account</h1>
      <form onSubmit={handleSubmit} className="form-group">
        <input className="input" type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <input className="input" type="password" placeholder="Password (8+ chars)" value={password} onChange={(e) => setPassword(e.target.value)} />
        {error && <span className="status-error">{error}</span>}
        <button type="submit" className="btn btn--success btn--full">Create Account</button>
      </form>
      <p className="auth-footer">Have an account? <Link to="/login">Sign in</Link></p>
    </div>
  );
}
```

- [ ] **Step 3: Test auth flow manually**

```bash
npm run dev
```
1. Navigate to `http://localhost:5173/register`
2. Register with a new email — should redirect to Dashboard
3. Log out — should redirect to Login
4. Log in with the same credentials — should redirect to Dashboard
5. Refresh page — should stay on Dashboard (token persisted)

- [ ] **Step 4: Commit**

```bash
git add . && git commit -m "feat: add login and register pages with JWT auth"
```

---

### Task 7: Dashboard Page

**Files:**
- Create: `frontend/src/components/NetWorthChart.tsx`
- Create: `frontend/src/components/MacroPanel.tsx`
- Create: `frontend/src/components/EventBanner.tsx`
- Modify: `frontend/src/pages/Dashboard.tsx`

- [ ] **Step 1: Register Chart.js components**

Add to `frontend/src/main.tsx` (before ReactDOM.createRoot):
```typescript
import {
  Chart as ChartJS, CategoryScale, LinearScale,
  PointElement, LineElement, Title, Tooltip, Legend,
} from 'chart.js';

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend);
```

- [ ] **Step 2: Create NetWorthChart**

Create `frontend/src/components/NetWorthChart.tsx`:
```typescript
import { Line } from 'react-chartjs-2';
import { useGameStore } from '../stores/gameStore';

export function NetWorthChart() {
  const history = useGameStore((s) => s.netWorthHistory);

  const data = {
    labels: history.map((h) => `T${h.tick}`),
    datasets: [{
      label: 'Net Worth',
      data: history.map((h) => h.value),
      borderColor: '#63b3ed',
      backgroundColor: 'rgba(99,179,237,0.1)',
      tension: 0.3,
      pointRadius: 2,
    }],
  };

  const options = {
    responsive: true,
    plugins: { legend: { display: false } },
    scales: {
      x: { ticks: { color: '#718096' }, grid: { color: '#2d3748' } },
      y: { ticks: { color: '#718096' }, grid: { color: '#2d3748' } },
    },
  };

  if (history.length < 2) return <div className="chart-placeholder">Waiting for data...</div>;
  return <Line data={data} options={options} />;
}
```

- [ ] **Step 3: Create MacroPanel**

Create `frontend/src/components/MacroPanel.tsx`:
```typescript
import { useGameStore } from '../stores/gameStore';

function pct(v: number) { return `${(v * 100).toFixed(2)}%`; }

export function MacroPanel() {
  const macro = useGameStore((s) => s.macro);
  if (!macro) return null;

  const tiles = [
    { label: 'GDP Growth',          value: pct(macro.gdpGrowth),          cls: macro.gdpGrowth >= 0 ? 'price--up' : 'price--down' },
    { label: 'Inflation',           value: pct(macro.inflation),           cls: macro.inflation > 0.05 ? 'price--down' : '' },
    { label: 'Interest Rate',       value: pct(macro.interestRate),        cls: '' },
    { label: 'Consumer Confidence', value: `${(macro.consumerConfidence * 100).toFixed(0)}`, cls: macro.consumerConfidence > 0.6 ? 'price--up' : 'price--down' },
  ];

  return (
    <div className="grid grid--4 mb-2">
      {tiles.map((t) => (
        <div key={t.label} className="stat-tile">
          <div className="card-label">{t.label}</div>
          <div className={`card-value ${t.cls}`}>{t.value}</div>
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Create EventBanner**

Create `frontend/src/components/EventBanner.tsx`:
```typescript
import { useGameStore } from '../stores/gameStore';

export function EventBanner() {
  const events = useGameStore((s) => s.activeEvents);
  if (events.length === 0) return null;

  return (
    <div className="event-banner">
      {events.map((e) => (
        <div key={e.id} className="event-banner__row">
          <span className="event-banner__name">{e.name}</span>
          <span className="event-banner__desc">{e.description} · {e.remainingTicks} ticks remaining</span>
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 5: Implement Dashboard page**

Replace `frontend/src/pages/Dashboard.tsx`:
```typescript
import { useGameStore } from '../stores/gameStore';
import { MacroPanel } from '../components/MacroPanel';
import { EventBanner } from '../components/EventBanner';
import { NetWorthChart } from '../components/NetWorthChart';

export function Dashboard() {
  const { tick, cash, playerRank } = useGameStore();

  return (
    <div className="flex-col">
      <div className="page-header">
        <h1 className="page-title">Dashboard</h1>
        <span className="page-subtitle">Tick #{tick} · Rank #{playerRank}</span>
      </div>

      <div className="grid grid--2">
        <div className="stat-tile">
          <div className="card-label">Cash Balance</div>
          <div className="card-value price--up">
            ${cash.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}
          </div>
        </div>
        <div className="stat-tile">
          <div className="card-label">Leaderboard Rank</div>
          <div className="card-value">#{playerRank}</div>
        </div>
      </div>

      <EventBanner />
      <MacroPanel />

      <div className="card card--sm">
        <h2 className="section-label">Net Worth</h2>
        <NetWorthChart />
      </div>
    </div>
  );
}
```

- [ ] **Step 6: Test Dashboard manually**

```bash
npm run dev
```
Expected: Dashboard shows macro tiles, cash balance, rank, and (after a few ticks) the net worth chart begins plotting. Events banner appears when an event fires.

- [ ] **Step 7: Commit**

```bash
git add . && git commit -m "feat: add dashboard with macro panel, event banner, and net worth chart"
```

---

### Task 8: Market Page

**Files:**
- Create: `frontend/src/components/PriceTable.tsx`
- Create: `frontend/src/components/BuySellPanel.tsx`
- Create: `frontend/src/components/PositionsList.tsx`
- Modify: `frontend/src/pages/Market.tsx`

- [ ] **Step 1: Create PriceTable**

Create `frontend/src/components/PriceTable.tsx`:
```typescript
import { useMarketStore } from '../stores/marketStore';

interface Props {
  onSelectAsset: (assetId: string) => void;
  selectedAsset: string | null;
}

export function PriceTable({ onSelectAsset, selectedAsset }: Props) {
  const { prices, priceHistory } = useMarketStore();

  return (
    <table className="table">
      <thead>
        <tr>
          <th>Asset</th>
          <th className="col-right">Price</th>
          <th className="col-right">Change</th>
        </tr>
      </thead>
      <tbody>
        {Object.entries(prices).map(([id, price]) => {
          const history = priceHistory[id] ?? [];
          const prev = history.length > 1 ? history[history.length - 2] : price;
          const changePct = ((price - prev) / prev) * 100;
          return (
            <tr
              key={id}
              onClick={() => onSelectAsset(id)}
              className={selectedAsset === id ? 'row--active' : ''}
            >
              <td><strong>{id.toUpperCase()}</strong></td>
              <td className="col-right">${price.toFixed(2)}</td>
              <td className={`col-right ${changePct >= 0 ? 'price--up' : 'price--down'}`}>
                {changePct >= 0 ? '+' : ''}{changePct.toFixed(2)}%
              </td>
            </tr>
          );
        })}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 2: Create BuySellPanel**

Create `frontend/src/components/BuySellPanel.tsx`:
```typescript
import { useState } from 'react';
import { useMarketStore } from '../stores/marketStore';

interface Props { assetId: string; }

export function BuySellPanel({ assetId }: Props) {
  const [quantity, setQuantity] = useState('1');
  const [side, setSide] = useState<'buy' | 'sell'>('buy');
  const [status, setStatus] = useState<'idle' | 'queued' | 'error'>('idle');
  const [errorMsg, setErrorMsg] = useState('');
  const { prices, placeOrder } = useMarketStore();

  const price = prices[assetId] ?? 0;
  const total = price * Number(quantity);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setStatus('idle');
    try {
      await placeOrder(assetId, Number(quantity), side);
      setStatus('queued');
      setTimeout(() => setStatus('idle'), 2000);
    } catch (err: any) {
      setErrorMsg(err?.response?.data?.message ?? 'Order failed');
      setStatus('error');
    }
  }

  return (
    <div className="card card--sm">
      <h3 className="mb-2">{assetId.toUpperCase()} @ ${price.toFixed(2)}</h3>
      <form onSubmit={handleSubmit} className="flex-col">
        <div className="flex-wrap">
          {(['buy', 'sell'] as const).map((s) => (
            <button
              key={s} type="button" onClick={() => setSide(s)}
              className={`btn btn--tab${side === s ? ` active--${s}` : ''}`}
            >
              {s.toUpperCase()}
            </button>
          ))}
        </div>
        <input
          className="input"
          type="number" min="0.0001" step="0.0001" value={quantity}
          onChange={(e) => setQuantity(e.target.value)}
        />
        <span className="status-muted">Total: ${isNaN(total) ? '—' : total.toFixed(2)}</span>
        {status === 'error' && <span className="status-error">{errorMsg}</span>}
        {status === 'queued' && <span className="status-success">Order queued for next tick</span>}
        <button type="submit" className={`btn btn--full ${side === 'buy' ? 'btn--success' : 'btn--danger'}`}>
          {side === 'buy' ? 'Buy' : 'Sell'} {assetId.toUpperCase()}
        </button>
      </form>
    </div>
  );
}
```

- [ ] **Step 3: Create PositionsList**

Create `frontend/src/components/PositionsList.tsx`:
```typescript
import { useMarketStore } from '../stores/marketStore';

export function PositionsList() {
  const { positions, prices } = useMarketStore();
  const nonZero = positions.filter((p) => p.quantity > 0);

  if (nonZero.length === 0) return <p className="status-muted">No positions yet.</p>;

  return (
    <table className="table">
      <thead>
        <tr>
          <th>Asset</th>
          <th className="col-right">Qty</th>
          <th className="col-right">Value</th>
        </tr>
      </thead>
      <tbody>
        {nonZero.map((p) => (
          <tr key={p.assetId}>
            <td><strong>{p.assetId.toUpperCase()}</strong></td>
            <td className="col-right">{p.quantity}</td>
            <td className="col-right rank--self">
              ${((prices[p.assetId] ?? 0) * p.quantity).toFixed(2)}
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 4: Implement Market page**

Replace `frontend/src/pages/Market.tsx`:
```typescript
import { useState } from 'react';
import { PriceTable } from '../components/PriceTable';
import { BuySellPanel } from '../components/BuySellPanel';
import { PositionsList } from '../components/PositionsList';

export function Market() {
  const [selectedAsset, setSelectedAsset] = useState<string | null>(null);

  return (
    <div className="flex-col">
      <h1 className="page-title">Market</h1>
      <div className="grid grid--market">
        <div className="flex-col">
          <div className="card card--sm">
            <h2 className="section-label">Live Prices</h2>
            <PriceTable onSelectAsset={setSelectedAsset} selectedAsset={selectedAsset} />
          </div>
          <div className="card card--sm">
            <h2 className="section-label">Your Positions</h2>
            <PositionsList />
          </div>
        </div>
        <div>
          {selectedAsset
            ? <BuySellPanel assetId={selectedAsset} />
            : <div className="card status-muted">Select an asset from the table to trade.</div>
          }
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Test Market page manually**

1. Open Market page — live price table ticks every 5 seconds
2. Click an asset — Buy/Sell panel appears
3. Enter quantity and submit — "Order queued for next tick" appears
4. After one tick — position appears in Your Positions with updated value
5. Try to buy more than cash allows — error message appears

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add market page with live prices, buy/sell panel, and positions"
```

---

### Task 9: Business Page

**Files:**
- Create: `frontend/src/components/BusinessCard.tsx`
- Modify: `frontend/src/pages/Business.tsx`

- [ ] **Step 1: Create BusinessCard**

Create `frontend/src/components/BusinessCard.tsx`:
```typescript
import { useState } from 'react';
import { useBusinessStore } from '../stores/businessStore';
import type { BusinessRow } from '../types';

const TYPE_LABELS: Record<string, string> = {
  retail: 'Retail Store',
  tech_startup: 'Tech Startup',
  real_estate: 'Real Estate',
};

interface Props { business: BusinessRow; }

export function BusinessCard({ business }: Props) {
  const updateSettings = useBusinessStore((s) => s.updateSettings);
  const [pricePoint, setPricePoint] = useState(business.pricePoint);
  const [staffCount, setStaffCount] = useState(business.staffCount);
  const [marketingSpend, setMarketingSpend] = useState(business.marketingSpend);
  const [saving, setSaving] = useState(false);

  async function handleSave() {
    setSaving(true);
    try {
      await updateSettings(business.id, { pricePoint, staffCount, marketingSpend });
    } finally {
      setSaving(false);
    }
  }

  const valuation = Math.max(0, business.profitEma) * 52;

  return (
    <div className="card">
      <div className="flex-between mb-2">
        <h3>{TYPE_LABELS[business.type]}</h3>
        <span className="rank--self">Valuation: ${valuation.toFixed(0)}</span>
      </div>

      <div className="flex-col">
        <label className="form-label">
          Price Point: ${pricePoint}
          <input className="input" type="range" min={10} max={500} value={pricePoint}
            onChange={(e) => setPricePoint(Number(e.target.value))} />
        </label>

        <label className="form-label">
          Staff Count: {staffCount}
          <input className="input" type="range" min={1} max={50} value={staffCount}
            onChange={(e) => setStaffCount(Number(e.target.value))} />
        </label>

        <label className="form-label">
          Marketing Spend: ${marketingSpend}/tick
          <input className="input" type="range" min={0} max={5000} step={50} value={marketingSpend}
            onChange={(e) => setMarketingSpend(Number(e.target.value))} />
        </label>

        <span className="status-muted">EMA profit/tick: ${business.profitEma.toFixed(2)}</span>

        <button onClick={handleSave} disabled={saving}
          className={`btn btn--full ${saving ? 'btn--ghost' : 'btn--primary'}`}>
          {saving ? 'Saving...' : 'Apply Settings'}
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Implement Business page**

Replace `frontend/src/pages/Business.tsx`:
```typescript
import { useState } from 'react';
import { useBusinessStore } from '../stores/businessStore';
import { BusinessCard } from '../components/BusinessCard';

const BUSINESS_TYPES = [
  { value: 'retail',       label: 'Retail Store',   desc: 'High traffic, price-sensitive customers.' },
  { value: 'tech_startup', label: 'Tech Startup',   desc: 'High costs, low elasticity, strong upside.' },
  { value: 'real_estate',  label: 'Real Estate',    desc: 'Stable income, sensitive to interest rates.' },
];

export function Business() {
  const { businesses, createBusiness } = useBusinessStore();
  const [creating, setCreating] = useState(false);
  const [error, setError] = useState('');

  async function handleCreate(type: string) {
    setCreating(true);
    setError('');
    try {
      await createBusiness(type);
    } catch (err: any) {
      setError(err?.response?.data?.message ?? 'Failed to create business');
    } finally {
      setCreating(false);
    }
  }

  return (
    <div className="flex-col">
      <h1 className="page-title">Business</h1>

      {businesses.length > 0 && (
        <div className="grid grid--auto">
          {businesses.map((b) => <BusinessCard key={b.id} business={b} />)}
        </div>
      )}

      <div className="card">
        <h2 className="section-label">Start a New Business</h2>
        {error && <p className="status-error mb-2">{error}</p>}
        <div className="grid grid--3">
          {BUSINESS_TYPES.map((bt) => (
            <button
              key={bt.value}
              onClick={() => handleCreate(bt.value)}
              disabled={creating}
              className="business-picker__card"
            >
              <div className="business-picker__card-title">{bt.label}</div>
              <div className="business-picker__card-desc">{bt.desc}</div>
            </button>
          ))}
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Test Business page manually**

1. Create a Retail business — card appears with sliders
2. Adjust Price Point slider — change is local until "Apply Settings"
3. Click Apply Settings — settings saved to backend
4. After one tick — EMA profit/tick value updates in the card

- [ ] **Step 4: Commit**

```bash
git add . && git commit -m "feat: add business page with BusinessCard settings sliders"
```

---

### Task 10: Leaderboard Page

**Files:**
- Create: `frontend/src/components/LeaderboardTable.tsx`
- Modify: `frontend/src/pages/Leaderboard.tsx`

- [ ] **Step 1: Create LeaderboardTable**

Create `frontend/src/components/LeaderboardTable.tsx`:
```typescript
import { useGameStore } from '../stores/gameStore';
import { useAuthStore } from '../stores/authStore';

export function LeaderboardTable() {
  const leaderboard = useGameStore((s) => s.leaderboard);
  const userId = useAuthStore((s) => s.userId);
  const playerRank = useGameStore((s) => s.playerRank);

  return (
    <table className="table">
      <thead>
        <tr>
          <th>Rank</th>
          <th>Player</th>
          <th className="col-right">Net Worth</th>
        </tr>
      </thead>
      <tbody>
        {leaderboard.map((player) => (
          <tr
            key={player.userId}
            className={player.userId === userId ? 'row--self' : ''}
          >
            <td className={player.rank <= 3 ? 'rank--top3' : ''}>#{player.rank}</td>
            <td className={player.userId === userId ? 'rank--self' : ''}>
              {player.userId === userId ? 'You' : player.userId.slice(0, 8) + '…'}
            </td>
            <td className="col-right">
              <strong>${player.netWorth.toLocaleString('en-US', { maximumFractionDigits: 0 })}</strong>
            </td>
          </tr>
        ))}
        {playerRank > 100 && (
          <tr className="row--self">
            <td className="rank--self">#{playerRank}</td>
            <td className="rank--self">You</td>
            <td className="col-right status-muted">Outside top 100</td>
          </tr>
        )}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 2: Implement Leaderboard page**

Replace `frontend/src/pages/Leaderboard.tsx`:
```typescript
import { useGameStore } from '../stores/gameStore';
import { LeaderboardTable } from '../components/LeaderboardTable';

export function Leaderboard() {
  const tick = useGameStore((s) => s.tick);

  return (
    <div className="flex-col">
      <div className="page-header">
        <h1 className="page-title">Leaderboard</h1>
        <span className="page-subtitle">Updated every tick · Tick #{tick}</span>
      </div>
      <div className="card card--sm">
        <LeaderboardTable />
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Test manually**

Open Leaderboard — table shows all players ranked by net worth. Your row is highlighted in blue. If you're outside top 100, your rank appears below the table.

- [ ] **Step 4: Commit**

```bash
git add . && git commit -m "feat: add leaderboard page with personal rank highlight"
```

---

### Task 11: Admin Page

**Files:**
- Create: `frontend/src/components/admin/SessionsTable.tsx`
- Create: `frontend/src/components/admin/EventControls.tsx`
- Create: `frontend/src/components/admin/MacroControls.tsx`
- Modify: `frontend/src/pages/Admin.tsx`

- [ ] **Step 1: Create SessionsTable**

Create `frontend/src/components/admin/SessionsTable.tsx`:
```typescript
import { useEffect, useState } from 'react';
import { getSessions, banPlayer, resetPlayer } from '../../api/admin';

interface Session {
  user_id: string; email: string; cash: string; current_tick: number;
}

export function SessionsTable() {
  const [sessions, setSessions] = useState<Session[]>([]);

  useEffect(() => { getSessions().then((r) => setSessions(r.data)); }, []);

  async function handleBan(userId: string) {
    if (!confirm('Ban this player?')) return;
    await banPlayer(userId);
    setSessions((s) => s.filter((p) => p.user_id !== userId));
  }

  async function handleReset(userId: string) {
    if (!confirm('Reset this player to starting state?')) return;
    await resetPlayer(userId);
  }

  return (
    <table className="table">
      <thead>
        <tr>
          <th>Email</th>
          <th className="col-right">Cash</th>
          <th className="col-right">Tick</th>
          <th className="col-right">Actions</th>
        </tr>
      </thead>
      <tbody>
        {sessions.map((s) => (
          <tr key={s.user_id}>
            <td>{s.email}</td>
            <td className="col-right">${Number(s.cash).toFixed(0)}</td>
            <td className="col-right">{s.current_tick}</td>
            <td className="col-right">
              <button onClick={() => handleReset(s.user_id)} className="btn btn--ghost btn--sm">Reset</button>
              {' '}
              <button onClick={() => handleBan(s.user_id)} className="btn btn--danger btn--sm">Ban</button>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 2: Create EventControls**

Create `frontend/src/components/admin/EventControls.tsx`:
```typescript
import { useState } from 'react';
import { triggerEvent } from '../../api/admin';

const EVENTS = [
  { id: 'tech_boom',      label: 'Tech Boom' },
  { id: 'recession',      label: 'Recession' },
  { id: 'supply_shock',   label: 'Supply Shock' },
  { id: 'rate_hike',      label: 'Rate Hike' },
  { id: 'consumer_boom',  label: 'Consumer Boom' },
];

export function EventControls() {
  const [status, setStatus] = useState('');

  async function handleTrigger(eventId: string) {
    const res = await triggerEvent(eventId);
    setStatus(`Triggered: ${res.data.name}`);
    setTimeout(() => setStatus(''), 3000);
  }

  return (
    <div>
      <div className="flex-wrap mb-1">
        {EVENTS.map((e) => (
          <button key={e.id} onClick={() => handleTrigger(e.id)} className="btn btn--ghost">
            {e.label}
          </button>
        ))}
      </div>
      {status && <span className="status-success">{status}</span>}
    </div>
  );
}
```

- [ ] **Step 3: Create MacroControls**

Create `frontend/src/components/admin/MacroControls.tsx`:
```typescript
import { useState } from 'react';
import { nudgeMacro } from '../../api/admin';
import { useGameStore } from '../../stores/gameStore';

export function MacroControls() {
  const macro = useGameStore((s) => s.macro);
  const [status, setStatus] = useState('');

  async function nudge(key: string, delta: number) {
    await nudgeMacro({ [key]: delta });
    setStatus(`Nudged ${key} by ${delta > 0 ? '+' : ''}${delta}`);
    setTimeout(() => setStatus(''), 2000);
  }

  return (
    <div className="flex-col">
      {[
        { key: 'inflationDelta',          label: 'Inflation' },
        { key: 'interestRateDelta',       label: 'Interest Rate' },
        { key: 'consumerConfidenceDelta', label: 'Consumer Confidence' },
        { key: 'gdpGrowthDelta',          label: 'GDP Growth' },
      ].map(({ key, label }) => (
        <div key={key} className="macro-row">
          <span className="macro-row__label">{label}</span>
          <button onClick={() => nudge(key, -0.01)} className="btn btn--danger btn--sm">−1%</button>
          <button onClick={() => nudge(key, 0.01)} className="btn btn--success btn--sm">+1%</button>
        </div>
      ))}
      {status && <span className="status-success">{status}</span>}
      {macro && (
        <p className="status-muted">
          Current: GDP {(macro.gdpGrowth * 100).toFixed(2)}% · Inflation {(macro.inflation * 100).toFixed(2)}% · Rate {(macro.interestRate * 100).toFixed(2)}% · Confidence {(macro.consumerConfidence * 100).toFixed(0)}
        </p>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Implement Admin page**

Replace `frontend/src/pages/Admin.tsx`:
```typescript
import { SessionsTable } from '../components/admin/SessionsTable';
import { EventControls } from '../components/admin/EventControls';
import { MacroControls } from '../components/admin/MacroControls';

export function Admin() {
  return (
    <div className="flex-col">
      <h1 className="page-title admin-title">Admin Panel</h1>

      <div className="card">
        <h2 className="section-label">Trigger Events</h2>
        <EventControls />
      </div>

      <div className="card">
        <h2 className="section-label">Macro Controls</h2>
        <MacroControls />
      </div>

      <div className="card">
        <h2 className="section-label">Active Sessions</h2>
        <SessionsTable />
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Test Admin page manually**

Log in as an admin user (created in the backend smoke test). Navigate to `/admin`:
1. Macro Controls — click "+1% Inflation" — event banner on other browser tabs should reflect macro change within 1 tick
2. Trigger Events — click "Tech Boom" — active event appears in all connected clients' event banners within 1 tick
3. Sessions Table — all registered users appear with their cash balance and current tick

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add admin page with event controls, macro nudge, and session management"
```

---

### Task 12: Build Verification

- [ ] **Step 1: Run TypeScript type check**

```bash
npm run build
```
Expected: No TypeScript errors. `dist/` folder created.

- [ ] **Step 2: Check for console errors in browser**

```bash
npm run dev
```

Open browser devtools (F12). Navigate through all pages: Dashboard → Market → Business → Leaderboard.

Expected: No red errors in the Console tab. Network tab shows WebSocket connection as `101 Switching Protocols`. Tick events visible in Network → WS frame log every 5 seconds.

- [ ] **Step 3: Test full user flow**

1. Register a new account
2. Check Dashboard — macro tiles show values, cash = $10,000
3. Go to Market — prices tick every 5 seconds
4. Buy 1 share of AAPL — "queued" message appears, position shows after next tick
5. Go to Business — create a Retail business
6. Adjust price sliders, click Apply Settings
7. After a few ticks — EMA profit/tick updates on the card, cash changes on Dashboard
8. Go to Leaderboard — your name appears with rank

- [ ] **Step 4: Final commit**

```bash
git add . && git commit -m "feat: frontend implementation complete — all pages functional"
```

---

## Self-Review

- **Spec coverage:** Login/Register ✓ · Dashboard (macro, events, net worth chart) ✓ · Market (live prices, buy/sell, positions) ✓ · Business (sliders, EMA valuation) ✓ · Leaderboard (top 100 + personal rank) ✓ · Admin panel (events, macro, sessions) ✓ · Socket.io reconnect + state snapshot ✓ · Three Zustand stores ✓ · Zustand persist for auth token ✓
- **No placeholders:** All components include complete implementation code
- **Type consistency:** `MacroSnapshot`, `Position`, `BusinessRow`, `RankedPlayer`, `ActiveEvent`, `TickBroadcast`, `PlayerTickUpdate` defined in `types/index.ts` and used consistently across all stores, pages, and components
