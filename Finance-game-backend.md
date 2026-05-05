# Finance Game — Backend Implementation Plan

**Goal:** Build a server-authoritative NestJS backend with a real-time WebSocket tick engine, shared player market, business simulation, admin controls, and crash recovery.

**Architecture:** NestJS modular monolith. A 5-second @Cron tick loop runs MacroEngine → EventManager → MarketEngine → BusinessEngine → LeaderboardService → Socket.io broadcast. Trades are validated immediately and queued in Redis; all financial state is persisted to PostgreSQL with ACID guarantees. On startup, RecoveryService replays the ledger from the last checkpoint to restore Redis state.

**Tech Stack:** NestJS + TypeScript, Socket.io, PostgreSQL (pg), Redis (ioredis), JWT (passport-jwt), bcrypt, class-validator, Jest, Supertest

**Spec:** `docs/superpowers/specs/2026-04-25-finance-game-design.md`

---

## File Map

```
backend/
  src/
    main.ts                          # Bootstrap: ValidationPipe, CORS, Socket.io adapter
    app.module.ts                    # Root module
    config/
      configuration.ts              # Env config schema
    database/
      database.module.ts            # Global module
      database.service.ts           # pg Pool, query(), withTransaction()
    redis/
      redis.module.ts               # Global module
      redis.service.ts              # get/set/getJson/setJson/lpush/popAll
    auth/
      auth.module.ts
      auth.controller.ts            # POST /auth/register, POST /auth/login
      auth.service.ts               # register(), login()
      jwt.strategy.ts               # passport-jwt strategy
      jwt-auth.guard.ts             # JwtAuthGuard
      roles.decorator.ts            # @Roles('admin')
      roles.guard.ts                # RolesGuard
      dto/register.dto.ts
      dto/login.dto.ts
    macro/
      macro.module.ts
      macro.service.ts              # MacroSnapshot state, tick()
      macro.types.ts                # MacroSnapshot interface
    market/
      market.module.ts
      market.service.ts             # Price series, processTick(actions)
      market.types.ts               # AssetPrice, TradeAction interfaces
    business/
      business.module.ts
      business.controller.ts        # POST /business, PATCH /business/:id/settings
      business.service.ts           # create(), updateSettings(), computeRevenueTick()
      business.types.ts             # Business, BusinessType, BusinessSettings
      dto/create-business.dto.ts
      dto/update-settings.dto.ts
    events/
      events.module.ts
      events.service.ts             # evaluateTick(), triggerEvent(), cancelEvent()
      events.types.ts               # GameEvent, EventDefinition interfaces
      event-library.ts              # 5 named EventDefinitions with weights
    leaderboard/
      leaderboard.module.ts
      leaderboard.service.ts        # computeNetWorth(), getRankedList()
    orders/
      orders.module.ts
      orders.controller.ts          # POST /market/order
      orders.service.ts             # validateAndEnqueue()
      dto/create-order.dto.ts
    game/
      game.module.ts
      game.gateway.ts               # Socket.io server, JWT handshake, rooms, broadcast
      game.service.ts               # tick orchestration, getStateSnapshot()
      game.controller.ts            # GET /game/state, GET /leaderboard
    admin/
      admin.module.ts
      admin.controller.ts           # /admin/* endpoints
      admin.service.ts              # triggerEvent, nudgeMacro, banPlayer, resetPlayer
    recovery/
      recovery.module.ts
      recovery.service.ts           # onModuleInit: checkpoint + ledger replay
  migrations/
    001_initial_schema.sql
  scripts/
    migrate.ts
  test/
    auth.e2e-spec.ts
    orders.e2e-spec.ts
    admin.e2e-spec.ts
    tick.integration-spec.ts
```

---

### Task 1: Project Scaffold & Configuration

**Files:**
- Create: `backend/` (NestJS project)
- Create: `backend/.env.example`
- Create: `backend/src/config/configuration.ts`
- Modify: `backend/src/app.module.ts`
- Modify: `backend/src/main.ts`

- [ ] **Step 1: Install NestJS CLI and scaffold project**

```bash
npm i -g @nestjs/cli
nest new backend --package-manager npm --skip-git
cd backend
```

Expected: Project created with `src/main.ts`, `src/app.module.ts`, `src/app.controller.ts`.

- [ ] **Step 2: Install all backend dependencies**

```bash
npm install \
  @nestjs/websockets @nestjs/platform-socket.io socket.io \
  @nestjs/jwt @nestjs/passport passport passport-jwt \
  @nestjs/schedule @nestjs/throttler @nestjs/config \
  pg ioredis bcrypt class-validator class-transformer

npm install --save-dev \
  @types/passport-jwt @types/bcrypt @types/pg \
  socket.io-client @types/socket.io-client \
  supertest @types/supertest ts-node dotenv
```

Expected: No peer dependency errors.

- [ ] **Step 3: Create `.env.example`**

Create `backend/.env.example`:
```
DATABASE_URL=postgres://postgres:postgres@localhost:5432/financegame
REDIS_URL=redis://localhost:6379
JWT_SECRET=changeme-use-a-long-random-string
JWT_EXPIRES_IN=7d
TICK_INTERVAL_MS=5000
CHECKPOINT_INTERVAL_MS=120000
```

Copy to `backend/.env` and fill in real local values.

- [ ] **Step 4: Create configuration schema**

Create `backend/src/config/configuration.ts`:
```typescript
export default () => ({
  database: { url: process.env.DATABASE_URL },
  redis: { url: process.env.REDIS_URL },
  jwt: {
    secret: process.env.JWT_SECRET ?? 'dev-secret',
    expiresIn: process.env.JWT_EXPIRES_IN ?? '7d',
  },
  // Cast to number here so ConfigService preserves the type at the source.
  // Consumers must use config.get<number>('tick.intervalMs') — omitting the
  // type parameter makes ConfigService return unknown, defeating the cast.
  tick: { intervalMs: Number(process.env.TICK_INTERVAL_MS ?? 5000) },
  checkpoint: { intervalMs: Number(process.env.CHECKPOINT_INTERVAL_MS ?? 120000) },
});
```

- [ ] **Step 5: Delete boilerplate files**

```bash
rm src/app.controller.ts src/app.controller.spec.ts src/app.service.ts
```

- [ ] **Step 6: Replace `app.module.ts`**

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [configuration] }),
  ],
})
export class AppModule {}
```

- [ ] **Step 7: Update `main.ts`**

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  app.enableCors();
  await app.listen(3000);
}
bootstrap();
```

- [ ] **Step 8: Verify the app boots**

```bash
npm run start:dev
```
Expected: `NestApplication successfully started` — no errors. Ctrl+C to stop.

- [ ] **Step 9: Commit**

```bash
git init && git add . && git commit -m "chore: scaffold NestJS backend with global config"
```

---

### Task 2: Database Module & Schema

**Files:**
- Create: `backend/migrations/001_initial_schema.sql`
- Create: `backend/scripts/migrate.ts`
- Create: `backend/src/database/database.service.ts`
- Create: `backend/src/database/database.module.ts`
- Create: `backend/src/database/database.service.spec.ts`

- [ ] **Step 1: Write the SQL migration**

Create `backend/migrations/001_initial_schema.sql`:
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TYPE user_role AS ENUM ('player', 'admin');

CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role user_role NOT NULL DEFAULT 'player',
  is_banned BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS game_sessions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  cash NUMERIC(20,4) NOT NULL DEFAULT 10000.00,
  current_tick INTEGER NOT NULL DEFAULT 0,
  last_checkpoint_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id)
);

CREATE TABLE IF NOT EXISTS portfolio_positions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  asset_id TEXT NOT NULL,
  quantity NUMERIC(20,4) NOT NULL DEFAULT 0,
  UNIQUE(user_id, asset_id)
);

CREATE TABLE IF NOT EXISTS businesses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type TEXT NOT NULL CHECK (type IN ('retail','tech_startup','real_estate')),
  price_point NUMERIC(10,2) NOT NULL DEFAULT 100,
  staff_count INTEGER NOT NULL DEFAULT 5,
  marketing_spend NUMERIC(10,2) NOT NULL DEFAULT 500,
  profit_ema NUMERIC(20,4) NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type TEXT NOT NULL,
  asset_id TEXT,
  quantity NUMERIC(20,4),
  price NUMERIC(20,4),
  amount NUMERIC(20,4) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_transactions_user_created ON transactions(user_id, created_at);

CREATE TABLE IF NOT EXISTS event_log (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  event_name TEXT NOT NULL,
  modifiers JSONB NOT NULL,
  duration_ticks INTEGER NOT NULL,
  fired_at_tick INTEGER NOT NULL,
  fired_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- [ ] **Step 2: Write migration runner**

Create `backend/scripts/migrate.ts`:
```typescript
import { Pool } from 'pg';
import { readFileSync } from 'fs';
import { join } from 'path';
import * as dotenv from 'dotenv';

dotenv.config();

async function migrate() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const sql = readFileSync(join(__dirname, '../migrations/001_initial_schema.sql'), 'utf8');
  await pool.query(sql);
  console.log('Migration complete');
  await pool.end();
}

migrate().catch((err) => { console.error(err); process.exit(1); });
```

Add to `backend/package.json` scripts section:
```json
"migrate": "ts-node scripts/migrate.ts"
```

- [ ] **Step 3: Run the migration**

```bash
npm run migrate
```
Expected: `Migration complete`. Verify in psql with `\dt` — all 6 tables appear.

- [ ] **Step 4: Write the test first**

Create `backend/src/database/database.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import configuration from '../config/configuration';
import { DatabaseService } from './database.service';

describe('DatabaseService', () => {
  let db: DatabaseService;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ load: [configuration] })],
      providers: [DatabaseService],
    }).compile();
    db = module.get(DatabaseService);
    await db.onModuleInit();
  });

  afterAll(() => db.onModuleDestroy());

  it('executes a simple query', async () => {
    const { rows } = await db.query<{ one: string }>('SELECT 1::text AS one');
    expect(rows[0].one).toBe('1');
  });

  it('rolls back the transaction on error', async () => {
    await expect(
      db.withTransaction(async (client) => {
        await client.query('SELECT 1');
        throw new Error('force rollback');
      }),
    ).rejects.toThrow('force rollback');
  });
});
```

- [ ] **Step 5: Run test and verify it fails**

```bash
npm test -- --testPathPattern database.service.spec
```
Expected: FAIL — `DatabaseService` not found.

- [ ] **Step 6: Implement DatabaseService and DatabaseModule**

Create `backend/src/database/database.service.ts`:
```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Pool, PoolClient } from 'pg';

@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
  private pool: Pool;

  constructor(private config: ConfigService) {}

  onModuleInit() {
    this.pool = new Pool({ connectionString: this.config.get<string>('database.url') });
  }

  async onModuleDestroy() {
    await this.pool.end();
  }

  query<T = any>(text: string, params?: any[]): Promise<{ rows: T[]; rowCount: number }> {
    return this.pool.query(text, params);
  }

  async withTransaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}
```

Create `backend/src/database/database.module.ts`:
```typescript
import { Global, Module } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Global()
@Module({ providers: [DatabaseService], exports: [DatabaseService] })
export class DatabaseModule {}
```

Add `DatabaseModule` to the imports array in `app.module.ts`.

- [ ] **Step 7: Run test and verify it passes**

```bash
npm test -- --testPathPattern database.service.spec
```
Expected: 2 tests pass.

- [ ] **Step 8: Commit**

```bash
git add . && git commit -m "feat: add database module with pg pool and initial schema migration"
```

---

### Task 3: Redis Module

**Files:**
- Create: `backend/src/redis/redis.service.ts`
- Create: `backend/src/redis/redis.module.ts`
- Create: `backend/src/redis/redis.service.spec.ts`

- [ ] **Step 1: Write the test first**

Create `backend/src/redis/redis.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import configuration from '../config/configuration';
import { RedisService } from './redis.service';

describe('RedisService', () => {
  let redis: RedisService;

  // Unique prefix per test run avoids dirty state if a previous run crashed
  // mid-test without cleanup. Avoids flushdb(), which would wipe unrelated
  // keys from other services sharing the same Redis instance.
  const P = `test:${Date.now()}`;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [ConfigModule.forRoot({ load: [configuration] })],
      providers: [RedisService],
    }).compile();
    redis = module.get(RedisService);
    await redis.onModuleInit();
  });

  afterAll(() => redis.onModuleDestroy());

  it('sets and gets a string', async () => {
    await redis.set(`${P}:key`, 'hello');
    expect(await redis.get(`${P}:key`)).toBe('hello');
  });

  it('pushes and pops all from a list atomically', async () => {
    await redis.lpush(`${P}:list`, 'a', 'b', 'c');
    const items = await redis.popAll(`${P}:list`);
    expect(items).toHaveLength(3);
    expect(await redis.popAll(`${P}:list`)).toHaveLength(0);
  });

  it('sets and gets JSON', async () => {
    await redis.setJson(`${P}:json`, { val: 99 });
    expect(await redis.getJson<{ val: number }>(`${P}:json`)).toEqual({ val: 99 });
  });
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm test -- --testPathPattern redis.service.spec
```
Expected: FAIL — `RedisService` not found.

- [ ] **Step 3: Implement RedisService**

Create `backend/src/redis/redis.service.ts`:
```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis;

  constructor(private config: ConfigService) {}

  onModuleInit() {
    this.client = new Redis(this.config.get<string>('redis.url')!);
  }

  async onModuleDestroy() {
    await this.client.quit();
  }

  get(key: string): Promise<string | null> {
    return this.client.get(key);
  }

  set(key: string, value: string, ttlSeconds?: number): Promise<'OK'> {
    if (ttlSeconds) return this.client.set(key, value, 'EX', ttlSeconds) as Promise<'OK'>;
    return this.client.set(key, value) as Promise<'OK'>;
  }

  del(key: string): Promise<number> {
    return this.client.del(key);
  }

  async getJson<T>(key: string): Promise<T | null> {
    const raw = await this.client.get(key);
    return raw ? (JSON.parse(raw) as T) : null;
  }

  setJson(key: string, value: unknown, ttlSeconds?: number): Promise<'OK'> {
    return this.set(key, JSON.stringify(value), ttlSeconds);
  }

  lpush(key: string, ...values: string[]): Promise<number> {
    return this.client.lpush(key, ...values);
  }

  async popAll(key: string): Promise<string[]> {
    const pipeline = this.client.pipeline();
    pipeline.lrange(key, 0, -1);
    pipeline.del(key);
    const results = await pipeline.exec();
    return (results?.[0]?.[1] as string[]) ?? [];
  }
}
```

Create `backend/src/redis/redis.module.ts`:
```typescript
import { Global, Module } from '@nestjs/common';
import { RedisService } from './redis.service';

@Global()
@Module({ providers: [RedisService], exports: [RedisService] })
export class RedisModule {}
```

Add `RedisModule` to `app.module.ts` imports.

- [ ] **Step 4: Run test and verify it passes**

```bash
npm test -- --testPathPattern redis.service.spec
```
Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add . && git commit -m "feat: add redis module with typed get/set/json/list operations"
```

---

### Task 4: Auth (Register, Login, JWT Guards)

**Files:**
- Create: `backend/src/auth/dto/register.dto.ts`
- Create: `backend/src/auth/dto/login.dto.ts`
- Create: `backend/src/auth/auth.service.ts`
- Create: `backend/src/auth/auth.controller.ts`
- Create: `backend/src/auth/jwt.strategy.ts`
- Create: `backend/src/auth/jwt-auth.guard.ts`
- Create: `backend/src/auth/roles.decorator.ts`
- Create: `backend/src/auth/roles.guard.ts`
- Create: `backend/src/auth/auth.module.ts`
- Create: `backend/test/auth.e2e-spec.ts`

- [ ] **Step 1: Write failing e2e tests**

Create `backend/test/auth.e2e-spec.ts`:
```typescript
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Auth (e2e)', () => {
  let app: INestApplication;
  const email = `auth-test-${Date.now()}@example.com`;
  const password = 'Password123!';

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();
  });

  afterAll(() => app.close());

  it('POST /auth/register creates account and returns token', async () => {
    const res = await request(app.getHttpServer())
      .post('/auth/register')
      .send({ email, password })
      .expect(201);
    expect(res.body.accessToken).toBeDefined();
  });

  it('POST /auth/register rejects duplicate email with 409', async () => {
    await request(app.getHttpServer())
      .post('/auth/register')
      .send({ email, password })
      .expect(409);
  });

  it('POST /auth/login returns token', async () => {
    const res = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email, password })
      .expect(200);
    expect(res.body.accessToken).toBeDefined();
  });

  it('POST /auth/login rejects wrong password with 401', async () => {
    await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email, password: 'wrongpassword' })
      .expect(401);
  });
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm run test:e2e -- --testPathPattern auth.e2e
```
Expected: FAIL — routes return 404.

- [ ] **Step 3: Create DTOs**

Create `backend/src/auth/dto/register.dto.ts`:
```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

Create `backend/src/auth/dto/login.dto.ts`:
```typescript
import { IsEmail, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  email: string;

  @IsString()
  password: string;
}
```

- [ ] **Step 4: Implement AuthService**

Create `backend/src/auth/auth.service.ts`:
```typescript
import { ConflictException, Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';
import { DatabaseService } from '../database/database.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  constructor(private db: DatabaseService, private jwt: JwtService) {}

  async register(dto: RegisterDto): Promise<{ accessToken: string }> {
    const { rows } = await this.db.query('SELECT id FROM users WHERE email = $1', [dto.email]);
    if (rows.length > 0) throw new ConflictException('Email already registered');

    const hash = await bcrypt.hash(dto.password, 10);
    const { rows: created } = await this.db.query<{ id: string; role: string }>(
      'INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, role',
      [dto.email, hash],
    );
    const user = created[0];
    await this.db.query(
      'INSERT INTO game_sessions (user_id) VALUES ($1) ON CONFLICT DO NOTHING',
      [user.id],
    );
    return { accessToken: this.jwt.sign({ sub: user.id, role: user.role }) };
  }

  async login(dto: LoginDto): Promise<{ accessToken: string }> {
    const { rows } = await this.db.query<{
      id: string; password_hash: string; role: string; is_banned: boolean;
    }>('SELECT id, password_hash, role, is_banned FROM users WHERE email = $1', [dto.email]);

    const user = rows[0];
    if (!user || !(await bcrypt.compare(dto.password, user.password_hash))) {
      throw new UnauthorizedException();
    }
    if (user.is_banned) throw new UnauthorizedException('Account suspended');
    return { accessToken: this.jwt.sign({ sub: user.id, role: user.role }) };
  }
}
```

- [ ] **Step 5: Implement AuthController**

Create `backend/src/auth/auth.controller.ts`:
```typescript
import { Body, Controller, HttpCode, Post } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private auth: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.auth.register(dto);
  }

  @Post('login')
  @HttpCode(200)
  login(@Body() dto: LoginDto) {
    return this.auth.login(dto);
  }
}
```

- [ ] **Step 6: Implement JWT strategy and guards**

Create `backend/src/auth/jwt.strategy.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: config.get<string>('jwt.secret'),
    });
  }

  validate(payload: { sub: string; role: string }) {
    return { userId: payload.sub, role: payload.role };
  }
}
```

Create `backend/src/auth/jwt-auth.guard.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

Create `backend/src/auth/roles.decorator.ts`:
```typescript
import { SetMetadata } from '@nestjs/common';
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

Create `backend/src/auth/roles.guard.ts`:
```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', ctx.getHandler());
    if (!roles) return true;
    const { user } = ctx.switchToHttp().getRequest();
    return roles.includes(user?.role);
  }
}
```

- [ ] **Step 7: Create AuthModule**

Create `backend/src/auth/auth.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './jwt.strategy';
import { RolesGuard } from './roles.guard';

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (cfg: ConfigService) => ({
        secret: cfg.get<string>('jwt.secret'),
        signOptions: { expiresIn: cfg.get<string>('jwt.expiresIn') },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy, RolesGuard],
  controllers: [AuthController],
  exports: [JwtModule, RolesGuard],
})
export class AuthModule {}
```

Add `AuthModule` to `app.module.ts` imports.

- [ ] **Step 8: Run e2e tests and verify they pass**

```bash
npm run test:e2e -- --testPathPattern auth.e2e
```
Expected: 4 tests pass.

- [ ] **Step 9: Commit**

```bash
git add . && git commit -m "feat: add auth module with register, login, and JWT/role guards"
```

---

### Task 5: MacroEngine

**Files:**
- Create: `backend/src/macro/macro.types.ts`
- Create: `backend/src/macro/macro.service.ts`
- Create: `backend/src/macro/macro.module.ts`
- Create: `backend/src/macro/macro.service.spec.ts`

- [ ] **Step 1: Define types**

Create `backend/src/macro/macro.types.ts`:
```typescript
export interface MacroSnapshot {
  gdpGrowth: number;        // e.g. 0.02 = 2% growth
  inflation: number;        // e.g. 0.03 = 3%
  interestRate: number;     // e.g. 0.05 = 5%
  consumerConfidence: number; // 0–1 scale
}

export interface MacroModifier {
  gdpGrowthDelta?: number;
  inflationDelta?: number;
  interestRateDelta?: number;
  consumerConfidenceDelta?: number;
}
```

- [ ] **Step 2: Write the tests**

Create `backend/src/macro/macro.service.spec.ts`:
```typescript
import { MacroService } from './macro.service';

describe('MacroService', () => {
  let service: MacroService;

  beforeEach(() => { service = new MacroService(); });

  it('returns a valid initial snapshot', () => {
    const snap = service.getSnapshot();
    expect(snap.consumerConfidence).toBeGreaterThanOrEqual(0);
    expect(snap.consumerConfidence).toBeLessThanOrEqual(1);
    expect(snap.inflation).toBeGreaterThanOrEqual(0);
  });

  it('tick changes snapshot values', () => {
    const before = { ...service.getSnapshot() };
    service.tick([]);
    const after = service.getSnapshot();
    const changed = Object.keys(before).some(
      (k) => before[k as keyof typeof before] !== after[k as keyof typeof after],
    );
    expect(changed).toBe(true);
  });

  it('applies a positive inflation modifier', () => {
    const before = service.getSnapshot().inflation;
    service.tick([{ inflationDelta: 0.05 }]);
    expect(service.getSnapshot().inflation).toBeGreaterThan(before);
  });

  it('clamps consumerConfidence to [0, 1]', () => {
    service.tick([{ consumerConfidenceDelta: -99 }]);
    expect(service.getSnapshot().consumerConfidence).toBeGreaterThanOrEqual(0);
    service.tick([{ consumerConfidenceDelta: 99 }]);
    expect(service.getSnapshot().consumerConfidence).toBeLessThanOrEqual(1);
  });
});
```

- [ ] **Step 3: Run and verify it fails**

```bash
npm test -- --testPathPattern macro.service.spec
```
Expected: FAIL — `MacroService` not found.

- [ ] **Step 4: Implement MacroService**

Create `backend/src/macro/macro.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { MacroModifier, MacroSnapshot } from './macro.types';

const BOUNDS = {
  gdpGrowth:          [-0.08, 0.08] as const,
  inflation:          [ 0.00, 0.15] as const,
  interestRate:       [ 0.00, 0.20] as const,
  consumerConfidence: [ 0.00, 1.00] as const,
};

function clamp(v: number, [min, max]: readonly [number, number]) {
  return Math.max(min, Math.min(max, v));
}

@Injectable()
export class MacroService {
  private tickCount = 0;
  private snapshot: MacroSnapshot = {
    gdpGrowth: 0.02,
    inflation: 0.03,
    interestRate: 0.05,
    consumerConfidence: 0.65,
  };

  getSnapshot(): MacroSnapshot { return { ...this.snapshot }; }

  tick(modifiers: MacroModifier[]): MacroSnapshot {
    this.tickCount++;
    const phase = (this.tickCount / 500) * 2 * Math.PI;
    const sine = Math.sin(phase) * 0.002;
    const noise = () => (Math.random() - 0.5) * 0.002;

    this.snapshot = {
      gdpGrowth:          clamp(this.snapshot.gdpGrowth + sine + noise(), BOUNDS.gdpGrowth),
      inflation:          clamp(this.snapshot.inflation + noise(),         BOUNDS.inflation),
      interestRate:       clamp(this.snapshot.interestRate + noise() * 0.5, BOUNDS.interestRate),
      consumerConfidence: clamp(this.snapshot.consumerConfidence + sine * 2 + noise(), BOUNDS.consumerConfidence),
    };

    for (const mod of modifiers) {
      if (mod.gdpGrowthDelta)          this.snapshot.gdpGrowth          = clamp(this.snapshot.gdpGrowth          + mod.gdpGrowthDelta,          BOUNDS.gdpGrowth);
      if (mod.inflationDelta)          this.snapshot.inflation          = clamp(this.snapshot.inflation          + mod.inflationDelta,          BOUNDS.inflation);
      if (mod.interestRateDelta)       this.snapshot.interestRate       = clamp(this.snapshot.interestRate       + mod.interestRateDelta,       BOUNDS.interestRate);
      if (mod.consumerConfidenceDelta) this.snapshot.consumerConfidence = clamp(this.snapshot.consumerConfidence + mod.consumerConfidenceDelta, BOUNDS.consumerConfidence);
    }

    return this.getSnapshot();
  }

  setState(snap: MacroSnapshot) { this.snapshot = { ...snap }; }
}
```

Create `backend/src/macro/macro.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { MacroService } from './macro.service';

@Module({ providers: [MacroService], exports: [MacroService] })
export class MacroModule {}
```

- [ ] **Step 5: Run and verify tests pass**

```bash
npm test -- --testPathPattern macro.service.spec
```
Expected: 4 tests pass.

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add macro engine with bounded random walk and modifier support"
```

---

### Task 6: MarketEngine

**Files:**
- Create: `backend/src/market/market.types.ts`
- Create: `backend/src/market/market.service.ts`
- Create: `backend/src/market/market.module.ts`
- Create: `backend/src/market/market.service.spec.ts`

- [ ] **Step 1: Define types**

Create `backend/src/market/market.types.ts`:
```typescript
export type AssetClass = 'stock' | 'bond' | 'real_estate';

export interface Asset {
  id: string;
  name: string;
  class: AssetClass;
  price: number;
  marketCap: number;
}

export interface TradeAction {
  userId: string;
  assetId: string;
  quantity: number;    // positive = buy, negative = sell
  priceAtOrder: number;
}

export type AssetPrices = Record<string, number>;
```

- [ ] **Step 2: Write the tests**

Create `backend/src/market/market.service.spec.ts`:
```typescript
import { MacroService } from '../macro/macro.service';
import { MarketService } from './market.service';
import { TradeAction } from './market.types';

describe('MarketService', () => {
  let market: MarketService;
  let macro: MacroService;

  beforeEach(() => {
    macro = new MacroService();
    market = new MarketService();
  });

  it('returns positive initial prices for all assets', () => {
    const prices = market.getPrices();
    expect(Object.keys(prices).length).toBeGreaterThan(0);
    for (const price of Object.values(prices)) expect(price).toBeGreaterThan(0);
  });

  it('prices change each tick', () => {
    const before = { ...market.getPrices() };
    market.tick(macro.getSnapshot(), []);
    const after = market.getPrices();
    const changed = Object.keys(before).some((k) => before[k] !== after[k]);
    expect(changed).toBe(true);
  });

  it('large buy order nudges price up', () => {
    const assetId = 'aapl';
    const before = market.getPrices()[assetId];
    const buy: TradeAction = { userId: 'u1', assetId, quantity: 5000, priceAtOrder: before };
    market.tick(macro.getSnapshot(), [buy]);
    expect(market.getPrices()[assetId]).toBeGreaterThan(before * 0.995);
  });

  it('prices stay positive after 100 ticks', () => {
    for (let i = 0; i < 100; i++) market.tick(macro.getSnapshot(), []);
    for (const price of Object.values(market.getPrices())) expect(price).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 3: Run and verify it fails**

```bash
npm test -- --testPathPattern market.service.spec
```
Expected: FAIL — `MarketService` not found.

- [ ] **Step 4: Implement MarketService**

Create `backend/src/market/market.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { MacroSnapshot } from '../macro/macro.types';
import { Asset, AssetPrices, TradeAction } from './market.types';

const INITIAL_ASSETS: Asset[] = [
  { id: 'aapl',  name: 'Apple Inc.',         class: 'stock',       price: 180, marketCap: 2_800_000 },
  { id: 'googl', name: 'Alphabet Inc.',       class: 'stock',       price: 140, marketCap: 1_700_000 },
  { id: 'jnj',   name: 'Johnson & Johnson',   class: 'stock',       price: 160, marketCap: 400_000 },
  { id: 'us10y', name: 'US 10Y Treasury',     class: 'bond',        price: 100, marketCap: 5_000_000 },
  { id: 'reit1', name: 'Urban REIT Fund',     class: 'real_estate', price: 50,  marketCap: 200_000 },
];

const DRIFT_WEIGHTS: Record<string, Record<string, number>> = {
  stock:       { gdpGrowth: 2.0, consumerConfidence: 1.5, interestRate: -1.0 },
  bond:        { interestRate: 1.5, gdpGrowth: -0.5 },
  real_estate: { interestRate: -2.0, gdpGrowth: 1.0, consumerConfidence: 0.5 },
};

@Injectable()
export class MarketService {
  private assets: Map<string, Asset>;

  constructor() {
    this.assets = new Map(INITIAL_ASSETS.map((a) => [a.id, { ...a }]));
  }

  getPrices(): AssetPrices {
    const out: AssetPrices = {};
    for (const [id, asset] of this.assets) out[id] = asset.price;
    return out;
  }

  getAssets(): Asset[] {
    return Array.from(this.assets.values()).map((a) => ({ ...a }));
  }

  tick(macro: MacroSnapshot, actions: TradeAction[]): AssetPrices {
    for (const [id, asset] of this.assets) {
      const w = DRIFT_WEIGHTS[asset.class] ?? {};
      let drift =
        (w.gdpGrowth ?? 0)           * macro.gdpGrowth * 0.01 +
        (w.consumerConfidence ?? 0)  * (macro.consumerConfidence - 0.5) * 0.005 +
        (w.interestRate ?? 0)        * macro.interestRate * 0.005;

      for (const action of actions.filter((a) => a.assetId === id)) {
        const impact = (action.quantity * action.priceAtOrder) / asset.marketCap;
        drift += impact * 0.1;
      }

      const noise = (Math.random() - 0.5) * 0.012;
      asset.price = Math.max(0.01, asset.price * (1 + drift + noise));
    }
    return this.getPrices();
  }

  setState(prices: AssetPrices) {
    for (const [id, price] of Object.entries(prices)) {
      const asset = this.assets.get(id);
      if (asset) asset.price = price;
    }
  }
}
```

Create `backend/src/market/market.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { MarketService } from './market.service';

@Module({ providers: [MarketService], exports: [MarketService] })
export class MarketModule {}
```

- [ ] **Step 5: Run and verify tests pass**

```bash
npm test -- --testPathPattern market.service.spec
```
Expected: 4 tests pass.

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add market engine with macro-driven prices and player impact"
```

---

### Task 7: BusinessEngine

**Files:**
- Create: `backend/src/business/business.types.ts`
- Create: `backend/src/business/business.service.ts`
- Create: `backend/src/business/business.controller.ts`
- Create: `backend/src/business/dto/create-business.dto.ts`
- Create: `backend/src/business/dto/update-settings.dto.ts`
- Create: `backend/src/business/business.module.ts`
- Create: `backend/src/business/business.service.spec.ts`

- [ ] **Step 1: Define types**

Create `backend/src/business/business.types.ts`:
```typescript
export type BusinessType = 'retail' | 'tech_startup' | 'real_estate';

export interface BusinessConfig {
  baseRevenue: number;
  baseCost: number;
  priceElasticity: number;
  staffCostPerHead: number;
}

export const BUSINESS_CONFIGS: Record<BusinessType, BusinessConfig> = {
  retail:       { baseRevenue: 5000, baseCost: 2000, priceElasticity: 1.5, staffCostPerHead: 200 },
  tech_startup: { baseRevenue: 3000, baseCost: 4000, priceElasticity: 0.8, staffCostPerHead: 500 },
  real_estate:  { baseRevenue: 8000, baseCost: 1000, priceElasticity: 0.4, staffCostPerHead: 150 },
};

export interface BusinessSettings {
  pricePoint: number;
  staffCount: number;
  marketingSpend: number;
}

export interface BusinessRow {
  id: string;
  userId: string;
  type: BusinessType;
  pricePoint: number;
  staffCount: number;
  marketingSpend: number;
  profitEma: number;
}
```

- [ ] **Step 2: Write the tests**

Create `backend/src/business/business.service.spec.ts`:
```typescript
import { BusinessService } from './business.service';
import { MacroService } from '../macro/macro.service';

describe('BusinessService (static methods)', () => {
  const macro = new MacroService();

  it('returns higher revenue when consumer confidence is high', () => {
    const highConfMacro = new MacroService();
    highConfMacro.tick([{ consumerConfidenceDelta: 0.35 }]);
    const settings = { pricePoint: 100, staffCount: 10, marketingSpend: 1000 };
    const low  = BusinessService.computeTickRevenue('retail', settings, macro.getSnapshot());
    const high = BusinessService.computeTickRevenue('retail', settings, highConfMacro.getSnapshot());
    expect(high).toBeGreaterThan(low);
  });

  it('higher price point reduces retail revenue (elastic)', () => {
    const snap = macro.getSnapshot();
    const cheap     = BusinessService.computeTickRevenue('retail', { pricePoint: 50,  staffCount: 5, marketingSpend: 500 }, snap);
    const expensive = BusinessService.computeTickRevenue('retail', { pricePoint: 200, staffCount: 5, marketingSpend: 500 }, snap);
    expect(cheap).toBeGreaterThan(expensive);
  });

  it('price point barely affects real_estate revenue (inelastic)', () => {
    const snap = macro.getSnapshot();
    const cheap     = BusinessService.computeTickRevenue('real_estate', { pricePoint: 50,  staffCount: 5, marketingSpend: 500 }, snap);
    const expensive = BusinessService.computeTickRevenue('real_estate', { pricePoint: 500, staffCount: 5, marketingSpend: 500 }, snap);
    expect(Math.abs(expensive - cheap)).toBeLessThan(2000);
  });

  it('updateEma converges toward new value', () => {
    const ema = BusinessService.updateEma(0, 1000, 0.1);
    expect(ema).toBeCloseTo(100, 1);
  });
});
```

- [ ] **Step 3: Run and verify it fails**

```bash
npm test -- --testPathPattern business.service.spec
```
Expected: FAIL — `BusinessService` not found.

- [ ] **Step 4: Implement BusinessService**

Create `backend/src/business/business.service.ts`:
```typescript
import { BadRequestException, ForbiddenException, Injectable, NotFoundException } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { MacroSnapshot } from '../macro/macro.types';
import { BUSINESS_CONFIGS, BusinessRow, BusinessSettings, BusinessType } from './business.types';

@Injectable()
export class BusinessService {
  constructor(private db: DatabaseService) {}

  static computeTickRevenue(type: BusinessType, settings: BusinessSettings, macro: MacroSnapshot): number {
    const cfg = BUSINESS_CONFIGS[type];
    const demandMultiplier = 0.5 + macro.consumerConfidence; // 0.5–1.5
    const priceRatio  = settings.pricePoint / 100;
    const priceFactor = 1 / Math.pow(priceRatio, cfg.priceElasticity);
    const marketingBonus = Math.sqrt(settings.marketingSpend) * 5;
    const revenue = cfg.baseRevenue * demandMultiplier * priceFactor + marketingBonus;
    const staffCost = settings.staffCount * cfg.staffCostPerHead;
    return revenue - cfg.baseCost - staffCost - settings.marketingSpend;
  }

  static updateEma(currentEma: number, newValue: number, alpha = 0.1): number {
    return alpha * newValue + (1 - alpha) * currentEma;
  }

  async create(userId: string, type: BusinessType): Promise<BusinessRow> {
    if (!['retail', 'tech_startup', 'real_estate'].includes(type)) {
      throw new BadRequestException('Invalid business type');
    }
    const { rows } = await this.db.query<BusinessRow>(
      `INSERT INTO businesses (user_id, type) VALUES ($1, $2)
       RETURNING id, user_id AS "userId", type,
                 price_point AS "pricePoint", staff_count AS "staffCount",
                 marketing_spend AS "marketingSpend", profit_ema AS "profitEma"`,
      [userId, type],
    );
    return rows[0];
  }

  async updateSettings(userId: string, businessId: string, settings: Partial<BusinessSettings>): Promise<BusinessRow> {
    const { rows } = await this.db.query<{ user_id: string }>(
      'SELECT user_id FROM businesses WHERE id = $1', [businessId],
    );
    if (!rows[0]) throw new NotFoundException('Business not found');
    if (rows[0].user_id !== userId) throw new ForbiddenException();

    const updates: string[] = [];
    const values: any[] = [];
    let idx = 1;
    if (settings.pricePoint !== undefined)    { updates.push(`price_point = $${idx++}`);     values.push(settings.pricePoint); }
    if (settings.staffCount !== undefined)    { updates.push(`staff_count = $${idx++}`);     values.push(settings.staffCount); }
    if (settings.marketingSpend !== undefined){ updates.push(`marketing_spend = $${idx++}`); values.push(settings.marketingSpend); }
    values.push(businessId);

    const { rows: updated } = await this.db.query<BusinessRow>(
      `UPDATE businesses SET ${updates.join(', ')} WHERE id = $${idx}
       RETURNING id, user_id AS "userId", type,
                 price_point AS "pricePoint", staff_count AS "staffCount",
                 marketing_spend AS "marketingSpend", profit_ema AS "profitEma"`,
      values,
    );
    return updated[0];
  }

  async getForUser(userId: string): Promise<BusinessRow[]> {
    const { rows } = await this.db.query<BusinessRow>(
      `SELECT id, user_id AS "userId", type,
              price_point AS "pricePoint", staff_count AS "staffCount",
              marketing_spend AS "marketingSpend", profit_ema AS "profitEma"
       FROM businesses WHERE user_id = $1`,
      [userId],
    );
    return rows;
  }

  async applyTick(userId: string, macro: MacroSnapshot): Promise<number> {
    const businesses = await this.getForUser(userId);
    let totalNetProfit = 0;
    for (const biz of businesses) {
      const netProfit = BusinessService.computeTickRevenue(
        biz.type,
        { pricePoint: biz.pricePoint, staffCount: biz.staffCount, marketingSpend: biz.marketingSpend },
        macro,
      );
      const newEma = BusinessService.updateEma(biz.profitEma, netProfit, 0.1);
      await this.db.query('UPDATE businesses SET profit_ema = $1 WHERE id = $2', [newEma, biz.id]);
      totalNetProfit += netProfit;
    }
    return totalNetProfit;
  }
}
```

- [ ] **Step 5: Create DTOs and Controller**

Create `backend/src/business/dto/create-business.dto.ts`:
```typescript
import { IsIn } from 'class-validator';
import { BusinessType } from '../business.types';

export class CreateBusinessDto {
  @IsIn(['retail', 'tech_startup', 'real_estate'])
  type: BusinessType;
}
```

Create `backend/src/business/dto/update-settings.dto.ts`:
```typescript
import { IsNumber, IsOptional, Min } from 'class-validator';

export class UpdateSettingsDto {
  @IsOptional() @IsNumber() @Min(1) pricePoint?: number;
  @IsOptional() @IsNumber() @Min(1) staffCount?: number;
  @IsOptional() @IsNumber() @Min(0) marketingSpend?: number;
}
```

Create `backend/src/business/business.controller.ts`:
```typescript
import { Body, Controller, Get, Param, Patch, Post, Request, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { BusinessService } from './business.service';
import { CreateBusinessDto } from './dto/create-business.dto';
import { UpdateSettingsDto } from './dto/update-settings.dto';

@Controller('business')
@UseGuards(JwtAuthGuard)
export class BusinessController {
  constructor(private business: BusinessService) {}

  @Post()
  create(@Request() req: any, @Body() dto: CreateBusinessDto) {
    return this.business.create(req.user.userId, dto.type);
  }

  @Patch(':id/settings')
  updateSettings(@Request() req: any, @Param('id') id: string, @Body() dto: UpdateSettingsDto) {
    return this.business.updateSettings(req.user.userId, id, dto);
  }

  @Get()
  list(@Request() req: any) {
    return this.business.getForUser(req.user.userId);
  }
}
```

Create `backend/src/business/business.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { BusinessService } from './business.service';
import { BusinessController } from './business.controller';

@Module({
  providers: [BusinessService],
  controllers: [BusinessController],
  exports: [BusinessService],
})
export class BusinessModule {}
```

Add `BusinessModule` to `app.module.ts` imports.

- [ ] **Step 6: Run tests and verify they pass**

```bash
npm test -- --testPathPattern business.service.spec
```
Expected: 4 tests pass.

- [ ] **Step 7: Commit**

```bash
git add . && git commit -m "feat: add business engine with demand function, EMA valuation, and REST endpoints"
```

---

### Task 8: EventManager

**Files:**
- Create: `backend/src/events/events.types.ts`
- Create: `backend/src/events/event-library.ts`
- Create: `backend/src/events/events.service.ts`
- Create: `backend/src/events/events.module.ts`
- Create: `backend/src/events/events.service.spec.ts`

- [ ] **Step 1: Define types and event library**

Create `backend/src/events/events.types.ts`:
```typescript
import { MacroModifier } from '../macro/macro.types';

export interface EventDefinition {
  id: string;
  name: string;
  description: string;
  weight: number;
  durationTicks: number;
  modifiers: MacroModifier;
}

export interface ActiveEvent {
  definition: EventDefinition;
  firedAtTick: number;
  remainingTicks: number;
}
```

Create `backend/src/events/event-library.ts`:
```typescript
import { EventDefinition } from './events.types';

export const EVENT_LIBRARY: EventDefinition[] = [
  {
    id: 'tech_boom', name: 'Tech Boom',
    description: 'A wave of innovation drives tech investment.',
    weight: 3, durationTicks: 10,
    modifiers: { gdpGrowthDelta: 0.01, consumerConfidenceDelta: 0.1 },
  },
  {
    id: 'recession', name: 'Recession',
    description: 'Economic contraction reduces consumer spending.',
    weight: 2, durationTicks: 20,
    modifiers: { gdpGrowthDelta: -0.03, consumerConfidenceDelta: -0.25, interestRateDelta: -0.02 },
  },
  {
    id: 'supply_shock', name: 'Supply Chain Crisis',
    description: 'Global supply disruptions spike inflation.',
    weight: 3, durationTicks: 8,
    modifiers: { inflationDelta: 0.04, consumerConfidenceDelta: -0.1 },
  },
  {
    id: 'rate_hike', name: 'Central Bank Rate Hike',
    description: 'Interest rates rise sharply to combat inflation.',
    weight: 4, durationTicks: 15,
    modifiers: { interestRateDelta: 0.03, consumerConfidenceDelta: -0.05 },
  },
  {
    id: 'consumer_boom', name: 'Consumer Spending Boom',
    description: 'Pent-up demand drives a spending surge.',
    weight: 3, durationTicks: 12,
    modifiers: { consumerConfidenceDelta: 0.2, gdpGrowthDelta: 0.015 },
  },
];

export const TOTAL_WEIGHT = EVENT_LIBRARY.reduce((s, e) => s + e.weight, 0);
export const EVENT_FIRE_PROBABILITY = 0.02;
```

- [ ] **Step 2: Write the tests**

Create `backend/src/events/events.service.spec.ts`:
```typescript
import { EventsService } from './events.service';

describe('EventsService', () => {
  let service: EventsService;

  beforeEach(() => { service = new EventsService(); });

  it('starts with no active events', () => {
    expect(service.getActiveEvents()).toHaveLength(0);
  });

  it('triggered event becomes active', () => {
    service.triggerEvent('tech_boom', 1);
    expect(service.getActiveEvents()[0].definition.id).toBe('tech_boom');
  });

  it('getActiveModifiers returns one modifier per active event', () => {
    service.triggerEvent('tech_boom', 1);
    service.triggerEvent('supply_shock', 1);
    expect(service.getActiveModifiers()).toHaveLength(2);
  });

  it('event expires after durationTicks decrements', () => {
    service.triggerEvent('tech_boom', 1); // durationTicks = 10
    for (let i = 0; i < 10; i++) service.decrementTicks();
    expect(service.getActiveEvents()).toHaveLength(0);
  });

  it('cancelEvent removes the event immediately', () => {
    service.triggerEvent('recession', 1);
    service.cancelEvent('recession');
    expect(service.getActiveEvents()).toHaveLength(0);
  });
});
```

- [ ] **Step 3: Run and verify it fails**

```bash
npm test -- --testPathPattern events.service.spec
```
Expected: FAIL — `EventsService` not found.

- [ ] **Step 4: Implement EventsService**

Create `backend/src/events/events.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { MacroModifier } from '../macro/macro.types';
import { EVENT_LIBRARY, EVENT_FIRE_PROBABILITY, TOTAL_WEIGHT } from './event-library';
import { ActiveEvent, EventDefinition } from './events.types';

@Injectable()
export class EventsService {
  private activeEvents: ActiveEvent[] = [];

  getActiveEvents(): ActiveEvent[] { return [...this.activeEvents]; }

  getActiveModifiers(): MacroModifier[] {
    return this.activeEvents.map((e) => e.definition.modifiers);
  }

  evaluateTick(currentTick: number): EventDefinition | null {
    if (Math.random() > EVENT_FIRE_PROBABILITY) return null;
    let roll = Math.random() * TOTAL_WEIGHT;
    for (const def of EVENT_LIBRARY) {
      roll -= def.weight;
      if (roll <= 0) { this.triggerEvent(def.id, currentTick); return def; }
    }
    return null;
  }

  triggerEvent(eventId: string, currentTick: number): ActiveEvent | null {
    const def = EVENT_LIBRARY.find((e) => e.id === eventId);
    if (!def) return null;
    const event: ActiveEvent = { definition: def, firedAtTick: currentTick, remainingTicks: def.durationTicks };
    this.activeEvents.push(event);
    return event;
  }

  cancelEvent(eventId: string): void {
    this.activeEvents = this.activeEvents.filter((e) => e.definition.id !== eventId);
  }

  decrementTicks(): void {
    for (const e of this.activeEvents) e.remainingTicks--;
    this.activeEvents = this.activeEvents.filter((e) => e.remainingTicks > 0);
  }

  setState(events: ActiveEvent[]) { this.activeEvents = [...events]; }
}
```

Create `backend/src/events/events.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { EventsService } from './events.service';

@Module({ providers: [EventsService], exports: [EventsService] })
export class EventsModule {}
```

- [ ] **Step 5: Run and verify tests pass**

```bash
npm test -- --testPathPattern events.service.spec
```
Expected: 5 tests pass.

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add event manager with weighted random events and tick expiry"
```

---

### Task 9: LeaderboardService

**Files:**
- Create: `backend/src/leaderboard/leaderboard.service.ts`
- Create: `backend/src/leaderboard/leaderboard.module.ts`
- Create: `backend/src/leaderboard/leaderboard.service.spec.ts`

- [ ] **Step 1: Write the tests**

Create `backend/src/leaderboard/leaderboard.service.spec.ts`:
```typescript
import { LeaderboardService } from './leaderboard.service';

describe('LeaderboardService (static methods)', () => {
  it('computeNetWorth sums cash + positions + EMA business valuation', () => {
    const nw = LeaderboardService.computeNetWorth({
      cash: 5000,
      positions: [
        { assetId: 'aapl', quantity: 10, currentPrice: 180 },
        { assetId: 'us10y', quantity: 5, currentPrice: 100 },
      ],
      businessProfitEmas: [1000, 500],
    });
    // 5000 + (10*180 + 5*100) + ((1000+500)*52)
    expect(nw).toBeCloseTo(5000 + 2300 + 78000, 0);
  });

  it('rankPlayers sorts descending and assigns 1-based rank', () => {
    const ranked = LeaderboardService.rankPlayers([
      { userId: 'a', netWorth: 50000 },
      { userId: 'b', netWorth: 120000 },
      { userId: 'c', netWorth: 80000 },
    ]);
    expect(ranked[0].userId).toBe('b');
    expect(ranked[0].rank).toBe(1);
    expect(ranked[2].rank).toBe(3);
  });

  it('getTop100 caps the list at 100', () => {
    const players = Array.from({ length: 150 }, (_, i) => ({ userId: `u${i}`, netWorth: i * 100 }));
    const top = LeaderboardService.getTop100(LeaderboardService.rankPlayers(players));
    expect(top).toHaveLength(100);
  });
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm test -- --testPathPattern leaderboard.service.spec
```
Expected: FAIL — `LeaderboardService` not found.

- [ ] **Step 3: Implement LeaderboardService**

Create `backend/src/leaderboard/leaderboard.service.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { AssetPrices } from '../market/market.types';

export interface NetWorthInput {
  cash: number;
  positions: { assetId: string; quantity: number; currentPrice: number }[];
  businessProfitEmas: number[];
}

export interface RankedPlayer {
  userId: string;
  netWorth: number;
  rank: number;
}

const EARNINGS_MULTIPLE = 52;

@Injectable()
export class LeaderboardService {
  constructor(private db: DatabaseService) {}

  static computeNetWorth(input: NetWorthInput): number {
    const posValue = input.positions.reduce((s, p) => s + p.quantity * p.currentPrice, 0);
    const bizValue = input.businessProfitEmas.reduce((s, ema) => s + Math.max(0, ema) * EARNINGS_MULTIPLE, 0);
    return input.cash + posValue + bizValue;
  }

  static rankPlayers(players: { userId: string; netWorth: number }[]): RankedPlayer[] {
    return [...players]
      .sort((a, b) => b.netWorth - a.netWorth)
      .map((p, i) => ({ ...p, rank: i + 1 }));
  }

  static getTop100(ranked: RankedPlayer[]): RankedPlayer[] {
    return ranked.slice(0, 100);
  }

  async computeAllNetWorths(prices: AssetPrices): Promise<{ userId: string; netWorth: number }[]> {
    const { rows: sessions } = await this.db.query<{ user_id: string; cash: string }>(
      'SELECT user_id, cash FROM game_sessions',
    );
    const results: { userId: string; netWorth: number }[] = [];

    for (const session of sessions) {
      const { rows: positions } = await this.db.query<{ asset_id: string; quantity: string }>(
        'SELECT asset_id, quantity FROM portfolio_positions WHERE user_id = $1', [session.user_id],
      );
      const { rows: businesses } = await this.db.query<{ profit_ema: string }>(
        'SELECT profit_ema FROM businesses WHERE user_id = $1', [session.user_id],
      );

      results.push({
        userId: session.user_id,
        netWorth: LeaderboardService.computeNetWorth({
          cash: Number(session.cash),
          positions: positions.map((p) => ({
            assetId: p.asset_id,
            quantity: Number(p.quantity),
            currentPrice: prices[p.asset_id] ?? 0,
          })),
          businessProfitEmas: businesses.map((b) => Number(b.profit_ema)),
        }),
      });
    }
    return results;
  }
}
```

Create `backend/src/leaderboard/leaderboard.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { LeaderboardService } from './leaderboard.service';

@Module({ providers: [LeaderboardService], exports: [LeaderboardService] })
export class LeaderboardModule {}
```

- [ ] **Step 4: Run and verify tests pass**

```bash
npm test -- --testPathPattern leaderboard.service.spec
```
Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add . && git commit -m "feat: add leaderboard service with EMA net worth and top-100 ranking"
```

---

### Task 10: Orders Endpoint

**Files:**
- Create: `backend/src/orders/dto/create-order.dto.ts`
- Create: `backend/src/orders/orders.service.ts`
- Create: `backend/src/orders/orders.controller.ts`
- Create: `backend/src/orders/orders.module.ts`
- Create: `backend/test/orders.e2e-spec.ts`

- [ ] **Step 1: Write failing e2e tests**

Create `backend/test/orders.e2e-spec.ts`:
```typescript
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Orders (e2e)', () => {
  let app: INestApplication;
  let token: string;

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    const email = `orders-${Date.now()}@test.com`;
    const res = await request(app.getHttpServer())
      .post('/auth/register').send({ email, password: 'Password123!' });
    token = res.body.accessToken;
  });

  afterAll(() => app.close());

  it('POST /market/order requires auth', () =>
    request(app.getHttpServer()).post('/market/order').send({}).expect(401));

  it('POST /market/order rejects unknown asset', () =>
    request(app.getHttpServer())
      .post('/market/order')
      .set('Authorization', `Bearer ${token}`)
      .send({ assetId: 'not_real', quantity: 10, side: 'buy' })
      .expect(400));

  it('POST /market/order rejects buy with insufficient funds', () =>
    request(app.getHttpServer())
      .post('/market/order')
      .set('Authorization', `Bearer ${token}`)
      .send({ assetId: 'aapl', quantity: 10000, side: 'buy' })
      .expect(400));

  it('POST /market/order accepts a valid small buy', async () => {
    const res = await request(app.getHttpServer())
      .post('/market/order')
      .set('Authorization', `Bearer ${token}`)
      .send({ assetId: 'aapl', quantity: 1, side: 'buy' })
      .expect(201);
    expect(res.body.queued).toBe(true);
  });
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm run test:e2e -- --testPathPattern orders.e2e
```
Expected: FAIL — route returns 404.

- [ ] **Step 3: Create DTO**

Create `backend/src/orders/dto/create-order.dto.ts`:
```typescript
import { IsIn, IsNumber, IsString, Min } from 'class-validator';

export class CreateOrderDto {
  @IsString()
  assetId: string;

  @IsNumber()
  @Min(0.0001)
  quantity: number;

  @IsIn(['buy', 'sell'])
  side: 'buy' | 'sell';
}
```

- [ ] **Step 4: Implement OrdersService**

Create `backend/src/orders/orders.service.ts`:
```typescript
import { BadRequestException, Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { MarketService } from '../market/market.service';
import { RedisService } from '../redis/redis.service';
import { TradeAction } from '../market/market.types';
import { CreateOrderDto } from './dto/create-order.dto';

const QUEUE_KEY = 'tick_actions_queue';

@Injectable()
export class OrdersService {
  constructor(
    private db: DatabaseService,
    private market: MarketService,
    private redis: RedisService,
  ) {}

  async validateAndEnqueue(userId: string, dto: CreateOrderDto): Promise<{ queued: boolean }> {
    const prices = this.market.getPrices();
    const currentPrice = prices[dto.assetId];
    if (currentPrice === undefined) throw new BadRequestException('Unknown asset');

    const { rows: sessions } = await this.db.query<{ cash: string }>(
      'SELECT cash FROM game_sessions WHERE user_id = $1', [userId],
    );
    if (!sessions[0]) throw new BadRequestException('No active game session');

    if (dto.side === 'buy') {
      const cost = currentPrice * dto.quantity;
      if (cost > Number(sessions[0].cash)) throw new BadRequestException('Insufficient funds');
    } else {
      const { rows: pos } = await this.db.query<{ quantity: string }>(
        'SELECT quantity FROM portfolio_positions WHERE user_id = $1 AND asset_id = $2',
        [userId, dto.assetId],
      );
      if (dto.quantity > Number(pos[0]?.quantity ?? 0)) throw new BadRequestException('Insufficient position');
    }

    const action: TradeAction = {
      userId,
      assetId: dto.assetId,
      quantity: dto.side === 'buy' ? dto.quantity : -dto.quantity,
      priceAtOrder: currentPrice,
    };
    await this.redis.lpush(QUEUE_KEY, JSON.stringify(action));
    return { queued: true };
  }
}
```

- [ ] **Step 5: Create OrdersController and OrdersModule**

Create `backend/src/orders/orders.controller.ts`:
```typescript
import { Body, Controller, Post, Request, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { OrdersService } from './orders.service';
import { CreateOrderDto } from './dto/create-order.dto';

@Controller('market')
@UseGuards(JwtAuthGuard)
export class OrdersController {
  constructor(private orders: OrdersService) {}

  @Post('order')
  placeOrder(@Request() req: any, @Body() dto: CreateOrderDto) {
    return this.orders.validateAndEnqueue(req.user.userId, dto);
  }
}
```

Create `backend/src/orders/orders.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { OrdersService } from './orders.service';
import { OrdersController } from './orders.controller';
import { MarketModule } from '../market/market.module';

@Module({
  imports: [MarketModule],
  providers: [OrdersService],
  controllers: [OrdersController],
})
export class OrdersModule {}
```

Add `OrdersModule` to `app.module.ts` imports.

- [ ] **Step 6: Run e2e tests and verify they pass**

```bash
npm run test:e2e -- --testPathPattern orders.e2e
```
Expected: 4 tests pass.

- [ ] **Step 7: Commit**

```bash
git add . && git commit -m "feat: add orders endpoint with validation and Redis action queue"
```

---

### Task 11: Game Gateway (Socket.io)

**Files:**
- Create: `backend/src/game/game.gateway.ts`
- Create: `backend/src/game/game.module.ts`
- Create: `backend/test/tick.integration-spec.ts`

- [ ] **Step 1: Update main.ts to use IoAdapter**

Replace `backend/src/main.ts`:
```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { IoAdapter } from '@nestjs/platform-socket.io';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  app.enableCors();
  app.useWebSocketAdapter(new IoAdapter(app));
  await app.listen(3000);
}
bootstrap();
```

Also add `ScheduleModule.forRoot()` to `app.module.ts` imports:
```typescript
import { ScheduleModule } from '@nestjs/schedule';
// add to imports: ScheduleModule.forRoot()
```

- [ ] **Step 2: Write the connection test**

Create `backend/test/tick.integration-spec.ts`:
```typescript
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { IoAdapter } from '@nestjs/platform-socket.io';
import { io, Socket } from 'socket.io-client';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('GameGateway + Tick (integration)', () => {
  let app: INestApplication;
  let token: string;
  let client: Socket;
  const PORT = 3099;

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    app.useWebSocketAdapter(new IoAdapter(app));
    await app.listen(PORT);

    const email = `ws-${Date.now()}@test.com`;
    const res = await request(app.getHttpServer())
      .post('/auth/register').send({ email, password: 'Password123!' });
    token = res.body.accessToken;
  });

  afterAll(async () => { client?.disconnect(); await app.close(); });

  it('connects with valid JWT and receives connected event', (done) => {
    client = io(`http://localhost:${PORT}`, { auth: { token }, transports: ['websocket'] });
    client.on('connected', (data: any) => {
      expect(data.userId).toBeDefined();
      done();
    });
    client.on('connect_error', done);
  });

  it('rejects connection without token', (done) => {
    const bad = io(`http://localhost:${PORT}`, { transports: ['websocket'] });
    bad.on('connect_error', (err) => {
      expect(err.message).toContain('unauthorized');
      bad.disconnect();
      done();
    });
  });

  it('receives a tick payload within 10 seconds', (done) => {
    client.on('tick', (data: any) => {
      expect(data.tick).toBeDefined();
      expect(data.prices).toBeDefined();
      expect(data.macro).toBeDefined();
      expect(data.leaderboard).toBeDefined();
      done();
    });
  }, 10000);

  it('GET /game/state returns full snapshot', async () => {
    const res = await request(`http://localhost:${PORT}`)
      .get('/game/state')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(res.body.prices).toBeDefined();
    expect(res.body.player).toBeDefined();
  });

  it('GET /leaderboard returns array', async () => {
    const res = await request(`http://localhost:${PORT}`)
      .get('/leaderboard')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(Array.isArray(res.body)).toBe(true);
  });
});
```

- [ ] **Step 3: Run and verify it fails**

```bash
npm run test:e2e -- --testPathPattern tick.integration
```
Expected: FAIL — cannot connect.

- [ ] **Step 4: Implement GameGateway**

Create `backend/src/game/game.gateway.ts`:
```typescript
import {
  OnGatewayConnection, OnGatewayDisconnect,
  WebSocketGateway, WebSocketServer,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@WebSocketGateway({ cors: true })
export class GameGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;

  constructor(private jwt: JwtService, private config: ConfigService) {}

  async handleConnection(client: Socket) {
    const token = client.handshake.auth?.token as string;
    if (!token) {
      client.emit('error', { message: 'unauthorized' });
      client.disconnect();
      return;
    }
    try {
      const payload = this.jwt.verify(token, {
        secret: this.config.get<string>('jwt.secret'),
      }) as { sub: string; role: string };

      client.data.userId = payload.sub;
      client.data.role = payload.role;
      await client.join('market');
      if (payload.role === 'admin') await client.join('admin');
      client.emit('connected', { userId: payload.sub });
    } catch {
      client.emit('error', { message: 'unauthorized' });
      client.disconnect();
    }
  }

  handleDisconnect(_client: Socket) {}

  broadcastTick(payload: object) {
    this.server.to('market').emit('tick', payload);
  }

  sendToPlayer(userId: string, event: string, data: object) {
    for (const [, socket] of this.server.sockets.sockets) {
      if (socket.data.userId === userId) socket.emit(event, data);
    }
  }
}
```

Create a stub `backend/src/game/game.module.ts` (full version in Task 12):
```typescript
import { Module } from '@nestjs/common';
import { GameGateway } from './game.gateway';

@Module({ providers: [GameGateway], exports: [GameGateway] })
export class GameModule {}
```

Add `GameModule` to `app.module.ts` imports.

- [ ] **Step 5: Run connection tests and verify first 2 pass**

```bash
npm run test:e2e -- --testPathPattern tick.integration
```
Expected: First 2 tests pass. Test 3 (tick) still times out — no GameService yet.

- [ ] **Step 6: Commit**

```bash
git add . && git commit -m "feat: add game gateway with JWT auth handshake and market/admin rooms"
```

---

### Task 12: Tick Loop & Game State Endpoint

**Files:**
- Create: `backend/src/game/game.service.ts`
- Create: `backend/src/game/game.controller.ts`
- Modify: `backend/src/game/game.module.ts`

- [ ] **Step 1: Implement GameService (tick orchestration)**

Create `backend/src/game/game.service.ts`:
```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { DatabaseService } from '../database/database.service';
import { RedisService } from '../redis/redis.service';
import { MacroService } from '../macro/macro.service';
import { MarketService } from '../market/market.service';
import { BusinessService } from '../business/business.service';
import { EventsService } from '../events/events.service';
import { LeaderboardService } from '../leaderboard/leaderboard.service';
import { GameGateway } from './game.gateway';
import { TradeAction } from '../market/market.types';

const QUEUE_KEY    = 'tick_actions_queue';
const MACRO_KEY    = 'game:macro';
const PRICES_KEY   = 'game:prices';
const BOARD_KEY    = 'game:leaderboard';

@Injectable()
export class GameService {
  private readonly logger = new Logger(GameService.name);
  private currentTick = 0;

  constructor(
    private db: DatabaseService,
    private redis: RedisService,
    private macro: MacroService,
    private market: MarketService,
    private business: BusinessService,
    private events: EventsService,
    private leaderboard: LeaderboardService,
    private gateway: GameGateway,
  ) {}

  @Cron('*/5 * * * * *')
  async runTick() {
    try {
      this.currentTick++;

      // Step 1 — MacroEngine
      const modifiers = this.events.getActiveModifiers();
      const macro = this.macro.tick(modifiers);
      await this.redis.setJson(MACRO_KEY, macro);

      // Step 2 — EventManager
      const firedEvent = this.events.evaluateTick(this.currentTick);
      this.events.decrementTicks();

      // Step 3 — MarketEngine: pop queued actions, apply impact, compute prices
      const rawActions = await this.redis.popAll(QUEUE_KEY);
      const actions: TradeAction[] = rawActions.map((r) => JSON.parse(r));
      const prices = this.market.tick(macro, actions);
      await this.redis.setJson(PRICES_KEY, prices);

      for (const action of actions) {
        await this.applyTradeToDb(action, prices[action.assetId]);
      }

      // Step 4 — BusinessEngine
      const { rows: sessions } = await this.db.query<{ user_id: string }>(
        'SELECT user_id FROM game_sessions',
      );
      for (const s of sessions) {
        const profit = await this.business.applyTick(s.user_id, macro);
        if (profit !== 0) {
          await this.db.query(
            'UPDATE game_sessions SET cash = cash + $1, current_tick = $2 WHERE user_id = $3',
            [profit, this.currentTick, s.user_id],
          );
          await this.db.query(
            `INSERT INTO transactions (user_id, type, amount) VALUES ($1, 'business_revenue', $2)`,
            [s.user_id, profit],
          );
        }
      }

      // Step 5 — LeaderboardService
      const netWorths = await this.leaderboard.computeAllNetWorths(prices);
      const ranked    = LeaderboardService.rankPlayers(netWorths);
      const top100    = LeaderboardService.getTop100(ranked);
      await this.redis.setJson(BOARD_KEY, top100);

      // Step 6 — Broadcast
      this.gateway.broadcastTick({
        tick: this.currentTick,
        macro,
        prices,
        leaderboard: top100,
        activeEvents: this.events.getActiveEvents().map((e) => ({
          id: e.definition.id,
          name: e.definition.name,
          description: e.definition.description,
          remainingTicks: e.remainingTicks,
        })),
        ...(firedEvent ? { newEvent: { id: firedEvent.id, name: firedEvent.name } } : {}),
      });

      // Personal tick updates
      for (const player of ranked) {
        const { rows: ps } = await this.db.query<{ cash: string }>(
          'SELECT cash FROM game_sessions WHERE user_id = $1', [player.userId],
        );
        const { rows: pos } = await this.db.query<{ asset_id: string; quantity: string }>(
          'SELECT asset_id, quantity FROM portfolio_positions WHERE user_id = $1', [player.userId],
        );
        const businesses = await this.business.getForUser(player.userId);
        this.gateway.sendToPlayer(player.userId, 'player_tick', {
          tick: this.currentTick,
          cash: Number(ps[0]?.cash ?? 0),
          positions: pos.map((p) => ({ assetId: p.asset_id, quantity: Number(p.quantity) })),
          businesses,
          rank: player.rank,
        });
      }
    } catch (err) {
      this.logger.error('Tick error', err);
    }
  }

  private async applyTradeToDb(action: TradeAction, executedPrice: number) {
    const cost  = executedPrice * Math.abs(action.quantity);
    const isBuy = action.quantity > 0;

    await this.db.withTransaction(async (client) => {
      if (isBuy) {
        await client.query(
          'UPDATE game_sessions SET cash = cash - $1 WHERE user_id = $2',
          [cost, action.userId],
        );
        await client.query(
          `INSERT INTO portfolio_positions (user_id, asset_id, quantity) VALUES ($1, $2, $3)
           ON CONFLICT (user_id, asset_id)
           DO UPDATE SET quantity = portfolio_positions.quantity + EXCLUDED.quantity`,
          [action.userId, action.assetId, action.quantity],
        );
      } else {
        await client.query(
          'UPDATE game_sessions SET cash = cash + $1 WHERE user_id = $2',
          [cost, action.userId],
        );
        await client.query(
          'UPDATE portfolio_positions SET quantity = quantity + $1 WHERE user_id = $2 AND asset_id = $3',
          [action.quantity, action.userId, action.assetId],
        );
      }
      await client.query(
        `INSERT INTO transactions (user_id, type, asset_id, quantity, price, amount)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [action.userId, isBuy ? 'buy' : 'sell', action.assetId, Math.abs(action.quantity), executedPrice, cost],
      );
    });
  }

  async getStateSnapshot(userId: string) {
    const macro      = await this.redis.getJson(MACRO_KEY) ?? this.macro.getSnapshot();
    const prices     = await this.redis.getJson(PRICES_KEY) ?? this.market.getPrices();
    const leaderboard = await this.redis.getJson(BOARD_KEY) ?? [];

    const { rows: sessions } = await this.db.query<{ cash: string }>(
      'SELECT cash FROM game_sessions WHERE user_id = $1', [userId],
    );
    const { rows: positions } = await this.db.query<{ asset_id: string; quantity: string }>(
      'SELECT asset_id, quantity FROM portfolio_positions WHERE user_id = $1', [userId],
    );
    const businesses = await this.business.getForUser(userId);

    return {
      tick: this.currentTick,
      macro,
      prices,
      leaderboard,
      activeEvents: this.events.getActiveEvents(),
      player: {
        cash: Number(sessions[0]?.cash ?? 0),
        positions: positions.map((p) => ({ assetId: p.asset_id, quantity: Number(p.quantity) })),
        businesses,
      },
    };
  }
}
```

- [ ] **Step 2: Create GameController**

Create `backend/src/game/game.controller.ts`:
```typescript
import { Controller, Get, Request, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { GameService } from './game.service';
import { RedisService } from '../redis/redis.service';

const BOARD_KEY = 'game:leaderboard';

@Controller()
@UseGuards(JwtAuthGuard)
export class GameController {
  constructor(private game: GameService, private redis: RedisService) {}

  @Get('game/state')
  getState(@Request() req: any) {
    return this.game.getStateSnapshot(req.user.userId);
  }

  @Get('leaderboard')
  async getLeaderboard() {
    return (await this.redis.getJson(BOARD_KEY)) ?? [];
  }
}
```

- [ ] **Step 3: Update GameModule to wire all dependencies**

Replace `backend/src/game/game.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { GameGateway } from './game.gateway';
import { GameService } from './game.service';
import { GameController } from './game.controller';
import { MacroModule } from '../macro/macro.module';
import { MarketModule } from '../market/market.module';
import { BusinessModule } from '../business/business.module';
import { EventsModule } from '../events/events.module';
import { LeaderboardModule } from '../leaderboard/leaderboard.module';

@Module({
  imports: [MacroModule, MarketModule, BusinessModule, EventsModule, LeaderboardModule],
  providers: [GameGateway, GameService],
  controllers: [GameController],
  exports: [GameGateway, GameService],
})
export class GameModule {}
```

- [ ] **Step 4: Run all integration tests and verify they pass**

```bash
npm run test:e2e -- --testPathPattern tick.integration
```
Expected: All 5 tests pass (connect, reject bad token, receive tick, game/state, leaderboard).

- [ ] **Step 5: Commit**

```bash
git add . && git commit -m "feat: add tick loop, game state endpoint, and leaderboard endpoint"
```

---

### Task 13: Admin Module

**Files:**
- Create: `backend/src/admin/admin.service.ts`
- Create: `backend/src/admin/admin.controller.ts`
- Create: `backend/src/admin/admin.module.ts`
- Create: `backend/test/admin.e2e-spec.ts`

- [ ] **Step 1: Write failing e2e tests**

Create `backend/test/admin.e2e-spec.ts`:
```typescript
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import * as bcrypt from 'bcrypt';
import { AppModule } from '../src/app.module';
import { DatabaseService } from '../src/database/database.service';

describe('Admin (e2e)', () => {
  let app: INestApplication;
  let playerToken: string;
  let adminToken: string;

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    const pRes = await request(app.getHttpServer())
      .post('/auth/register')
      .send({ email: `player-${Date.now()}@test.com`, password: 'Password123!' });
    playerToken = pRes.body.accessToken;

    const db = app.get(DatabaseService);
    const adminEmail = `admin-${Date.now()}@test.com`;
    const hash = await bcrypt.hash('AdminPass123!', 10);
    const { rows } = await db.query<{ id: string }>(
      `INSERT INTO users (email, password_hash, role) VALUES ($1, $2, 'admin') RETURNING id`,
      [adminEmail, hash],
    );
    await db.query('INSERT INTO game_sessions (user_id) VALUES ($1) ON CONFLICT DO NOTHING', [rows[0].id]);
    const aRes = await request(app.getHttpServer())
      .post('/auth/login').send({ email: adminEmail, password: 'AdminPass123!' });
    adminToken = aRes.body.accessToken;
  });

  afterAll(() => app.close());

  it('GET /admin/sessions blocked for player role', () =>
    request(app.getHttpServer())
      .get('/admin/sessions')
      .set('Authorization', `Bearer ${playerToken}`)
      .expect(403));

  it('GET /admin/sessions returns array for admin role', async () => {
    const res = await request(app.getHttpServer())
      .get('/admin/sessions')
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(200);
    expect(Array.isArray(res.body)).toBe(true);
  });

  it('POST /admin/events/trigger fires a named event', async () => {
    const res = await request(app.getHttpServer())
      .post('/admin/events/trigger')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ eventId: 'tech_boom' })
      .expect(201);
    expect(res.body.name).toBe('Tech Boom');
  });

  it('PATCH /admin/macro adjusts a macro indicator', () =>
    request(app.getHttpServer())
      .patch('/admin/macro')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ inflationDelta: 0.01 })
      .expect(200));
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm run test:e2e -- --testPathPattern admin.e2e
```
Expected: FAIL — routes return 404.

- [ ] **Step 3: Implement AdminService**

Create `backend/src/admin/admin.service.ts`:
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { EventsService } from '../events/events.service';
import { MacroService } from '../macro/macro.service';
import { MacroModifier } from '../macro/macro.types';
import { ActiveEvent } from '../events/events.types';

@Injectable()
export class AdminService {
  constructor(
    private db: DatabaseService,
    private events: EventsService,
    private macro: MacroService,
  ) {}

  async getSessions() {
    const { rows } = await this.db.query(
      `SELECT gs.user_id, u.email, gs.cash, gs.current_tick
       FROM game_sessions gs JOIN users u ON u.id = gs.user_id`,
    );
    return rows;
  }

  triggerEvent(eventId: string): ActiveEvent {
    const event = this.events.triggerEvent(eventId, 0);
    if (!event) throw new NotFoundException(`Event '${eventId}' not found in library`);
    return event;
  }

  cancelEvent(eventId: string): void {
    this.events.cancelEvent(eventId);
  }

  nudgeMacro(modifier: MacroModifier) {
    this.macro.tick([modifier]);
    return this.macro.getSnapshot();
  }

  async banPlayer(userId: string): Promise<void> {
    const { rowCount } = await this.db.query(
      'UPDATE users SET is_banned = true WHERE id = $1', [userId],
    );
    if (!rowCount) throw new NotFoundException('User not found');
  }

  async resetPlayer(userId: string): Promise<void> {
    await this.db.withTransaction(async (client) => {
      await client.query('UPDATE game_sessions SET cash = 10000, current_tick = 0 WHERE user_id = $1', [userId]);
      await client.query('DELETE FROM portfolio_positions WHERE user_id = $1', [userId]);
      await client.query('DELETE FROM businesses WHERE user_id = $1', [userId]);
    });
  }
}
```

- [ ] **Step 4: Implement AdminController**

Create `backend/src/admin/admin.controller.ts`:
```typescript
import { Body, Controller, Delete, Get, HttpCode, IsNumber, IsOptional, Param, Patch, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';
import { RolesGuard } from '../auth/roles.guard';
import { Roles } from '../auth/roles.decorator';
import { AdminService } from './admin.service';
import { MacroModifier } from '../macro/macro.types';

class TriggerEventDto { eventId: string; }

class NudgeMacroDto implements MacroModifier {
  @IsOptional() @IsNumber() gdpGrowthDelta?: number;
  @IsOptional() @IsNumber() inflationDelta?: number;
  @IsOptional() @IsNumber() interestRateDelta?: number;
  @IsOptional() @IsNumber() consumerConfidenceDelta?: number;
}

@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
export class AdminController {
  constructor(private admin: AdminService) {}

  @Get('sessions')
  getSessions() { return this.admin.getSessions(); }

  @Post('events/trigger')
  triggerEvent(@Body() dto: TriggerEventDto) {
    const event = this.admin.triggerEvent(dto.eventId);
    return { id: event.definition.id, name: event.definition.name };
  }

  @Delete('events/:id')
  @HttpCode(204)
  cancelEvent(@Param('id') id: string) { this.admin.cancelEvent(id); }

  @Patch('macro')
  nudgeMacro(@Body() dto: NudgeMacroDto) { return this.admin.nudgeMacro(dto); }

  @Post('players/:id/ban')
  @HttpCode(204)
  banPlayer(@Param('id') id: string) { return this.admin.banPlayer(id); }

  @Post('players/:id/reset')
  @HttpCode(204)
  resetPlayer(@Param('id') id: string) { return this.admin.resetPlayer(id); }
}
```

- [ ] **Step 5: Create AdminModule**

Create `backend/src/admin/admin.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { AdminService } from './admin.service';
import { AdminController } from './admin.controller';
import { EventsModule } from '../events/events.module';
import { MacroModule } from '../macro/macro.module';

@Module({
  imports: [EventsModule, MacroModule],
  providers: [AdminService],
  controllers: [AdminController],
})
export class AdminModule {}
```

Add `AdminModule` to `app.module.ts` imports.

- [ ] **Step 6: Run e2e tests and verify they pass**

```bash
npm run test:e2e -- --testPathPattern admin.e2e
```
Expected: 4 tests pass.

- [ ] **Step 7: Commit**

```bash
git add . && git commit -m "feat: add admin module with event control, macro nudge, ban, and player reset"
```

---

### Task 14: Recovery Service

**Files:**
- Create: `backend/src/recovery/recovery.service.ts`
- Create: `backend/src/recovery/recovery.module.ts`
- Create: `backend/src/recovery/recovery.service.spec.ts`

- [ ] **Step 1: Write the test**

Create `backend/src/recovery/recovery.service.spec.ts`:
```typescript
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import configuration from '../config/configuration';
import { DatabaseModule } from '../database/database.module';
import { RedisModule } from '../redis/redis.module';
import { MacroModule } from '../macro/macro.module';
import { MarketModule } from '../market/market.module';
import { RecoveryService } from './recovery.service';

describe('RecoveryService', () => {
  let recovery: RecoveryService;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [
        ConfigModule.forRoot({ load: [configuration] }),
        DatabaseModule, RedisModule, MacroModule, MarketModule,
      ],
      providers: [RecoveryService],
    }).compile();
    recovery = module.get(RecoveryService);
  });

  it('runs onModuleInit without throwing', async () => {
    await expect(recovery.onModuleInit()).resolves.not.toThrow();
  });
});
```

- [ ] **Step 2: Run and verify it fails**

```bash
npm test -- --testPathPattern recovery.service.spec
```
Expected: FAIL — `RecoveryService` not found.

- [ ] **Step 3: Implement RecoveryService**

Create `backend/src/recovery/recovery.service.ts`:
```typescript
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';
import { RedisService } from '../redis/redis.service';
import { MacroService } from '../macro/macro.service';
import { MarketService } from '../market/market.service';
import { MacroSnapshot } from '../macro/macro.types';
import { AssetPrices } from '../market/market.types';

const MACRO_KEY  = 'game:macro';
const PRICES_KEY = 'game:prices';

@Injectable()
export class RecoveryService implements OnModuleInit {
  private readonly logger = new Logger(RecoveryService.name);

  constructor(
    private db: DatabaseService,
    private redis: RedisService,
    private macro: MacroService,
    private market: MarketService,
  ) {}

  async onModuleInit() {
    this.logger.log('Running crash recovery check...');

    const cachedMacro  = await this.redis.getJson<MacroSnapshot>(MACRO_KEY);
    const cachedPrices = await this.redis.getJson<AssetPrices>(PRICES_KEY);

    if (cachedMacro && cachedPrices) {
      this.macro.setState(cachedMacro);
      this.market.setState(cachedPrices);
      this.logger.log('Redis intact — engines restored from cache');
      return;
    }

    // Redis was cleared: replay ledger since last checkpoint
    this.logger.warn('Redis missing — replaying ledger to restore cash balances');

    const { rows: sessions } = await this.db.query<{
      user_id: string; cash: string; last_checkpoint_at: Date;
    }>('SELECT user_id, cash, last_checkpoint_at FROM game_sessions');

    for (const session of sessions) {
      const { rows: txns } = await this.db.query<{ type: string; amount: string }>(
        `SELECT type, amount FROM transactions
         WHERE user_id = $1 AND created_at > $2
         ORDER BY created_at ASC`,
        [session.user_id, session.last_checkpoint_at],
      );

      let delta = 0;
      for (const txn of txns) {
        if (['sell', 'business_revenue'].includes(txn.type)) delta += Number(txn.amount);
        if (txn.type === 'buy') delta -= Number(txn.amount);
      }

      if (delta !== 0) {
        await this.db.query(
          'UPDATE game_sessions SET cash = cash + $1 WHERE user_id = $2',
          [delta, session.user_id],
        );
        this.logger.log(`Replayed ${txns.length} txns for user ${session.user_id}: delta ${delta}`);
      }
    }

    await this.db.query('UPDATE game_sessions SET last_checkpoint_at = NOW()');
    this.logger.log('Recovery complete');
  }
}
```

Create `backend/src/recovery/recovery.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { RecoveryService } from './recovery.service';
import { MacroModule } from '../macro/macro.module';
import { MarketModule } from '../market/market.module';

@Module({
  imports: [MacroModule, MarketModule],
  providers: [RecoveryService],
})
export class RecoveryModule {}
```

Add `RecoveryModule` to `app.module.ts` imports.

- [ ] **Step 4: Run test and verify it passes**

```bash
npm test -- --testPathPattern recovery.service.spec
```
Expected: 1 test passes.

- [ ] **Step 5: Commit**

```bash
git add . && git commit -m "feat: add recovery service to replay ledger after crash"
```

---

### Task 15: Rate Limiting & Final Smoke Test

**Files:**
- Modify: `backend/src/app.module.ts`
- Modify: `backend/src/orders/orders.controller.ts`

- [ ] **Step 1: Add global ThrottlerModule**

In `backend/src/app.module.ts`, add to imports:
```typescript
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

// In imports array:
ThrottlerModule.forRoot([{ ttl: 60000, limit: 100 }]),

// In providers array:
{ provide: APP_GUARD, useClass: ThrottlerGuard },
```

- [ ] **Step 2: Tighten rate limit on orders endpoint**

In `backend/src/orders/orders.controller.ts`, add throttle override on the handler:
```typescript
import { Throttle } from '@nestjs/throttler';

// On placeOrder method, add before @Post:
@Throttle({ default: { ttl: 60000, limit: 20 } })
```

- [ ] **Step 3: Run all tests to confirm nothing broke**

```bash
npm test && npm run test:e2e
```
Expected: All unit and e2e tests pass.

- [ ] **Step 4: Manual smoke test**

Start the server:
```bash
npm run build && npm run start:prod
```
Expected: Server starts on port 3000. RecoveryService log appears.

Register a user, then check game state:
```bash
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@smoke.com","password":"Password123!"}' | jq .accessToken
```

Save the token, then:
```bash
# Replace TOKEN with the value from above
curl -s http://localhost:3000/game/state -H "Authorization: Bearer TOKEN" | jq '{tick, macro, player}'
```
Expected: JSON with `tick: 0`, `macro` object, `player.cash: 10000`.

Wait 12 seconds, then check game state again — `tick` should now be 2 or more and prices will have changed.

- [ ] **Step 5: Final commit**

```bash
git add . && git commit -m "feat: add rate limiting; backend implementation complete"
```

---

## Self-Review

- **Spec coverage:** Auth ✓ · MacroEngine ✓ · MarketEngine + action queue ✓ · BusinessEngine ✓ · EventManager ✓ · LeaderboardService (EMA + top-100) ✓ · GameGateway (rooms + JWT) ✓ · Tick loop (correct engine order) ✓ · Orders endpoint ✓ · Admin module (all endpoints) ✓ · RecoveryService (ledger replay) ✓ · Rate limiting ✓ · Append-only ledger ✓ · ACID trades ✓ · Personal tick updates ✓
- **No placeholders:** All steps include runnable code
- **Type consistency:** `TradeAction`, `MacroSnapshot`, `MacroModifier`, `BusinessRow`, `AssetPrices`, `ActiveEvent`, `RankedPlayer` defined once and used consistently across all tasks
