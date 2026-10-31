# Foundation milestone: design

Date: 2026-10-31. Status: approved section by section in conversation; awaiting owner review of this written form. Product context: `2026-10-31-quincejam-product-scope.md`.

## 1. Goal and scope

Foundation delivers everything the canvas milestone will stand on, with no canvas: a monorepo that builds TypeScript, Rust and Go; five Cloudflare Workers (four product Workers plus the internal remote cache) with three Durable Object classes; Postgres metadata with a billing-ready schema; authentication; entitlements and rate limiting; a BoardRoom Durable Object that relays, validates and persists opaque CRDT bytes; room modes and a signaling skeleton; a self-host image that runs the production bundles; OpenAPI-first code generation for SDKs and MCP; CI/CD; observability; a fast dev loop.

Done means: a user can sign up on staging and in the self-host stack, create a workspace and a board, open a WebSocket to the board's room with a ticket, exchange Loro updates between two clients, see a snapshot land in object storage, hit an entitlement limit and get the standard error, and the generated SDKs and CLI can list boards.

Out of scope: the canvas, rendering, history UX, comments, thumbnails, exports, upload UI, billing provider integration, multi-region, self-host clustering, the bare-workerd variant, the WebRTC implementation for peer rooms.

## 2. Principles that decide ties

1. One codebase, one set of bundles, two runtimes: Cloudflare in production, Miniflare-on-workerd in Docker for self-host. Adapters only where Cloudflare has no local equivalent.
2. Postgres is the only system of record for metadata. Durable Objects hold per-board document state and per-workspace strict counters, nothing else.
3. D1 and KV are not used anywhere.
4. The Durable Objects are TypeScript in this milestone. Rust lives in `crates/qj-doc` (board schema, board-as-text, native tests), compiled to wasm for the browser and linked natively by the CLI. The BoardRoom treats document bytes as opaque.
5. Everything billing will need (plans, features, subscriptions, overrides, usage ledger, audit) is modeled now with no provider code.
6. The realtime Worker never opens a database connection. Everything it needs rides in the ticket.

## 3. Repo layout and tooling

Package manager pnpm 10 (pinned via `packageManager`). Turborepo 2.11 pinned exactly with `futureFlags: { experimentalCargoWorkspaces, experimentalGoWorkspaces, experimentalTaskCommand }`. Root virtual Cargo workspace (`members = ["crates/*"]`, resolver 3, `[workspace.metadata] name = "qj-cargo"`), `rust-toolchain.toml` (stable, wasm32-unknown-unknown target), root `go.work` (Go 1.25). Biome for TypeScript lint and format, lefthook for pre-commit. Remote cache self-hosted on Cloudflare (Workers plus R2, the `turborepo-remote-cache-cloudflare` project) as a fifth internal Worker.

```
quincejam/
  package.json  pnpm-workspace.yaml  turbo.json  Cargo.toml  Cargo.lock  rust-toolchain.toml  go.work  biome.json  lefthook.yml
  .github/workflows/      ci.yml, neon-preview.yml, deploy-staging.yml, deploy-prod.yml, selfhost-image.yml
  docs/adr/               architecture decision records (0001 sync model, 0002 Postgres over D1, 0003 Miniflare as self-host runtime)
  apps/
    web/                  Vite + React 19 SPA served as Worker static assets; hosts the console later
    api/                  Hono 4 + hono-openapi (zod 4) + Better Auth + Drizzle; /auth, /v1, /openapi.json, /mcp
    rt/                   Realtime Worker: BoardRoom, WorkspaceMeter, Signaling Durable Objects; sync engine seam
    jobs/                 Queue consumers and crons
    selfhost/             Dockerfile, compose.yaml, Miniflare bootstrap, migrate script, AI shim, Caddyfile
    cache/                Turborepo remote cache Worker (internal)
  packages/
    db/                   Drizzle schema, SQL migrations (including hand-written partition DDL), client, seed
    auth/                 Better Auth factory, ticket mint/verify
    entitlements/         generated feature types, pure resolver, assert helpers
    adapters/             ports: messenger, ai, blobs; drivers: cloudflare/, selfhost/, test/
    protocol/             zod schemas for WS frames, DTOs, queue messages, webhook payloads; msgpack codecs
    do-schema/            SQLite DDL and migrate() for each DO class
    doc-engine-wasm/      wasm-pack output of crates/qj-doc (types committed, artefact built in CI)
    log/                  structured logger, request-id middleware, Sentry wrapper
    config/               tsconfig base, biome preset, vitest presets
  crates/
    qj-doc/               pure Rust: board schema over loro 1.17, validation, board-as-text codec, history helpers
    qj-cli/               qj CLI (clap, reqwest, tokio-tungstenite); links qj-doc and qj-sdk
    qj-sdk/               progenitor-generated client
  go/
    sdk/                  oapi-codegen client
    loadgen/              WebSocket load generator (N clients x M boards, latency histograms)
    mailrelay/            HTTP JSON to SMTP sidecar for self-host
  sdks/
    ts/                   @quincejam/sdk via @hey-api/openapi-ts plus hand-written realtime client
    python/               openapi-python-client output; plain gen task, no Python workspace flag
  spec/
    openapi.json          emitted from api routes; committed; CI fails on drift
    features.yaml         entitlement keys, kinds, plan defaults
    ws-protocol.md        frame catalogue and versioning
    board-text.md         board-as-text format
  tools/
    gen/                  gen-openapi, gen-sdks, gen-features, gen-protocol, check-drift
    scripts/              check-do-lifecycle, size-report, do-export, seed-plans
```

Turborepo rules (verified against 2.11 docs): library crate builds must set `cache: true` explicitly; `cargo run` and `go run` tasks are never cached; declare `inputs` for cross-crate `include_str!` and `env` for build-script variables; Go package names derive from the last path segment and must be unique across the whole graph (hence `@quincejam/sdk` for the npm package); `cargo` and `go` must be on PATH for every turbo invocation including `turbo prune` in CI images.

Languages in this milestone: TypeScript for Workers, web, TS SDK; Rust for qj-doc, CLI, Rust SDK; Go for SDK, load generator, mail relay; Python for the generated SDK only.

## 4. Cloudflare topology

| Worker | Production / staging host | Role | Key bindings |
|---|---|---|---|
| `qj-web` | quincejam.com / staging.quincejam.com | Static-asset SPA (`not_found_handling: single-page-application`, `run_worker_first` for `/healthz` and `/.well-known/*`); no data access | `ASSETS` |
| `qj-api` | api.quincejam.com / api.staging.quincejam.com | Better Auth at `/auth/*`, REST `/v1/*`, `/openapi.json`, MCP at `/mcp` (Streamable HTTP), ticket minting, TURN credentials, webhook management, usage ledger writes | `HYPERDRIVE`, `HYPERDRIVE_NOCACHE`, `BLOBS` (R2), `EVENTS`, `WEBHOOKS` (Queue producers), `BOARD_ROOM`, `WORKSPACE_METER`, `SIGNALING` (DO bindings with `script_name: qj-rt`), `AI`, `RL_AUTH`, `RL_API_FREE`, `RL_API_PRO`, `METRICS`, `VERSION` |
| `qj-rt` | rt.quincejam.com / rt.staging.quincejam.com | Exports `BoardRoom`, `WorkspaceMeter`, `Signaling` (all `storage: "sqlite"`); WS upgrade `/v1/boards/:id/ws?ticket=`; `/v1/boards/:id/snapshot`; `POST /v1/boards/:id/updates`; `/v1/rooms/:id/signal` | DO bindings to self, `BLOBS`, `EVENTS`, `HYPERDRIVE` (WorkspaceMeter flush only), `TICKET_SECRET`, `TICKET_SECRET_PREV`, `METRICS` |
| `qj-jobs` | no route | Queue consumers `qj-events` (batch 100, timeout 5 s, 5 retries, DLQ `qj-dlq`) and `qj-webhooks` (batch 10, 8 retries, 30 s delay); crons every 5 minutes (usage rollup, meter flush sweep) and nightly (snapshot GC, audit export, partition maintenance) | `HYPERDRIVE`, `BLOBS`, `AI`, DO bindings |
| `qj-cache` | cache.quincejam.com (internal) | Turborepo remote cache | R2 bucket `qj-turbo-cache` |

All Durable Object classes live in `qj-rt` because a Worker whose config contains `exports` entries cannot use `wrangler versions upload` (verified). `qj-api` and `qj-web` therefore keep PR preview versions and gradual rollouts; `qj-rt` always deploys with plain `wrangler deploy`, guarded by the CI lifecycle rule.

Declarative DO lifecycle (`apps/rt/wrangler.jsonc`):

```jsonc
"exports": {
  "BoardRoom":      { "type": "durable-object", "storage": "sqlite" },
  "WorkspaceMeter": { "type": "durable-object", "storage": "sqlite" },
  "Signaling":      { "type": "durable-object", "storage": "sqlite" }
}
```

`migrations` arrays are never used (mutually exclusive with DO exports). Renames use the three-deploy `state: "renamed"` flow; `deleted` tombstones destroy data and are gated by CI plus production approval.

Environments: `env.staging` and `env.production` blocks per Worker; non-inheritable bindings (R2, Queues, DO, Hyperdrive) redefined per env with `-staging` suffixes; `custom_domain: true` routes. Hyperdrive configs: `qj-pg-production` and `qj-pg-production-nocache` (Neon `main`), `qj-pg-staging` and `qj-pg-staging-nocache` (Neon `staging`). R2 buckets `qj-blobs`, `qj-blobs-staging`, `qj-turbo-cache`. Queues `qj-events`, `qj-webhooks`, `qj-dlq` (plus `-staging`). Observability `head_sampling_rate` 1 on staging, 0.1 on production. Compatibility date pinned per Worker (`2026-10-01`) and bumped deliberately.

Not used: D1, KV, Pages, Workflows, Vectorize, Browser Rendering, Email Service.

## 5. Metadata database

Postgres 18 on Neon, reached through Hyperdrive with `postgres.js` over `cloudflare:sockets`, Drizzle ORM, SQL migrations committed under `packages/db/migrations`. Two Hyperdrive configs per environment: the default (cached) for plans and the boards index, and a caching-disabled one for sessions, ACLs and share links, because Hyperdrive has no write invalidation. Only `qj-api` and `qj-jobs` open connections, plus `WorkspaceMeter` for its flush. Self-host: a `postgres:18` container through the same `HYPERDRIVE` binding (Miniflare points it at an external TCP service).

Why Neon: free through dogfood, Postgres 18, copy-on-write branches with expiry (a database per pull request), millisecond point-in-time restore, documented Hyperdrive guide, pg_partman 5.1 and pgvector 0.8. Runner-up PlanetScale Postgres when a 99.99% SLA, three-zone HA, Insights or Neki sharding is wanted; the migration is logical replication. Why not D1: no self-host path outside the simulator, Better Auth's D1 dialect runs multi-step auth writes non-atomically (verified), and cross-tenant billing and admin queries want one relational store.

Conventions: ids are UUIDv7 generated in the Worker and stored as native `uuid` (the ULID extension is Neon-only); timestamps are `timestamptz`; emails get a unique index on `lower(email)` (no citext); JSONB columns get GIN `jsonb_path_ops` indexes where queried; generated columns are `STORED`; quota checks use atomic `UPDATE ... RETURNING`, never advisory locks, and nothing uses session-level `SET`, `LISTEN/NOTIFY` or prepared-statement session state, because Hyperdrive rejects them. Partitioned tables are hand-written SQL migrations (Drizzle cannot model `PARTITION BY`); partition maintenance is called from the nightly Workers cron via `partman.run_maintenance_proc()`.

### 5.1 Tables

Better Auth core (adapter defaults, `generateId` set to UUIDv7):
- `user(id, name, email, email_verified, image, is_anonymous, handle UNIQUE NULL, created_at, updated_at)`
- `session(id, user_id, token UNIQUE, expires_at, ip_address, user_agent, active_organization_id NULL, created_at, updated_at)`
- `account(id, user_id, provider_id, account_id, access_token, refresh_token, id_token, token expiries, scope, password NULL, created_at, updated_at)`
- `verification(id, identifier, value, expires_at, created_at, updated_at)`
- `passkey(id, user_id, name, public_key, credential_id UNIQUE, counter, device_type, backed_up, transports, aaguid, created_at)`

Workspaces (Better Auth organization plugin, `organization` renamed to `workspace`):
- `workspace(id, name, slug UNIQUE, logo NULL, metadata jsonb, created_at)`
- `member(id, workspace_id, user_id, role CHECK IN ('owner','admin','member'), created_at, UNIQUE(workspace_id, user_id))`
- `invitation(id, workspace_id, email, role, status, expires_at, inviter_id, created_at)`

Boards and access:
- `boards(id, workspace_id, title, slug, created_by, visibility enum('private','workspace','link'), room_mode enum('hosted','e2e') DEFAULT 'hosted', doc_codec text DEFAULT 'loro-v1', doc_schema_version int, snapshot_key NULL, snapshot_seq bigint DEFAULT 0, doc_frontiers jsonb NULL, size_bytes bigint DEFAULT 0, thumbnail_key NULL, last_edited_at, archived_at NULL, deleted_at NULL, created_at, updated_at; INDEX(workspace_id, updated_at DESC))`. Ephemeral and peer rooms have no board row.
- `board_acl(board_id, principal_type enum('user','workspace'), principal_id, role enum('owner','editor','commenter','viewer'), granted_by, created_at; PK(board_id, principal_type, principal_id))`
- `share_links(id, board_id, token_hash UNIQUE, role enum('editor','commenter','viewer'), password_hash NULL, allow_anonymous bool, expires_at NULL, max_uses NULL, use_count DEFAULT 0, created_by, revoked_at NULL, created_at)`
- `board_versions(id, board_id, label, message NULL, frontiers jsonb, snapshot_key, author_id, parent_version_id NULL, created_at; INDEX(board_id, created_at DESC))` (schema only in this milestone)

Programmatic surfaces:
- `api_keys(id, workspace_id, user_id, name, prefix char(8) UNIQUE, key_hash, scopes text[], last_used_at NULL, expires_at NULL, revoked_at NULL, created_at)`
- `webhooks(id, workspace_id, url, secret_hash, events text[], active, failure_count DEFAULT 0, created_at)`
- `webhook_deliveries(id, webhook_id, event_type, payload jsonb, status enum('pending','ok','failed','dead'), attempts, next_attempt_at NULL, response_code NULL, created_at; INDEX(webhook_id, created_at DESC))`

Plans and entitlements:
- `plans(id text PK ('free','pro','team','selfhost'), name, is_default, is_public, rank, created_at)`
- `plan_features(plan_id, feature_key, value jsonb {kind:'limit'|'flag'|'enum', value}; PK(plan_id, feature_key))`
- `subscriptions(id, workspace_id, plan_id, status enum('active','trialing','past_due','canceled','paused','comped'), provider enum('none','stripe','polar') DEFAULT 'none', provider_customer_id NULL, provider_subscription_id NULL, provider_price_id NULL, current_period_start NULL, current_period_end NULL, cancel_at NULL, trial_end NULL, metadata jsonb, created_at, updated_at; UNIQUE partial index on workspace_id WHERE status IN ('active','trialing','past_due','comped'))`
- `entitlement_overrides(id, scope_type enum('workspace','user'), scope_id, feature_key, value jsonb, reason, expires_at NULL, created_by, created_at; INDEX(scope_type, scope_id))`
- `billing_customers(scope_type, scope_id, provider, provider_customer_id, created_at; PK(scope_type, scope_id, provider))` (empty until billing)

Usage and audit:
- `usage_events(id bigint GENERATED, workspace_id, user_id NULL, board_id NULL, feature_key, qty bigint, occurred_at, idempotency_key UNIQUE, meta jsonb) PARTITION BY RANGE (occurred_at)`, monthly partitions managed by pg_partman
- `usage_counters(workspace_id, feature_key, period enum('day','month','lifetime'), period_start date, qty bigint, updated_at; PK(workspace_id, feature_key, period, period_start))`
- `audit_events(id, workspace_id, actor_type enum('user','api_key','system','share_link'), actor_id, action, target_type, target_id, ip inet NULL, user_agent NULL, meta jsonb, created_at; INDEX(workspace_id, created_at DESC))`

Row-level security is enabled as defense in depth: each request transaction sets `app.workspace_id` via `set_config(..., true)` and `SET LOCAL ROLE app_user`; policies are declared with Drizzle `pgPolicy`. The primary ACL gate stays in the Worker. pg_stat_statements is enabled from day one. pgvector is installed but no vector columns exist yet.

### 5.2 Durable Object SQLite

DDL lives in `packages/do-schema` with an idempotent `migrate()` keyed on a `schema_version` table.

- `BoardRoom`: `meta(k TEXT PK, v BLOB)` (board_id, workspace_id, codec, schema_version, snapshot_seq, snapshot_vv, snapshot_key, size_bytes, flushed_vv, snapshot_at); `updates(seq INTEGER PK AUTOINCREMENT, bytes BLOB, author TEXT, ts INTEGER)` (one row per flush); `snapshot(part INTEGER PK, bytes BLOB)` (900 KB chunks under the 2 MB row limit).
- `WorkspaceMeter`: `counters(feature_key, period_start, qty, dirty, PK(feature_key, period_start))`; `ent_cache(k TEXT PK, json TEXT, fetched_at INTEGER)`.
- `Signaling`: no persisted tables; in-memory peer list only.

### 5.3 Blob keys

`boards/{board_id}/snap/{seq}.loro`, `boards/{board_id}/versions/{version_id}.loro`, `boards/{board_id}/thumb.png`, `uploads/{workspace_id}/{uuid}.{ext}`, `usage/{workspace_id}/{yyyy-mm}.ndjson`, `audit/{workspace_id}/{yyyy-mm-dd}.ndjson`, `exports/{job_id}.zip`.

## 6. Auth

`packages/auth/createAuth(env)` builds Better Auth 1.7.x with the Drizzle adapter on Postgres, `basePath: "/auth"`, `trustedOrigins` from env, cross-subdomain cookies on `.quincejam.com` (disabled when `QJ_SINGLE_HOST=true`), session cookie cache (5 minutes) so Hyperdrive stays off the hot path, UUIDv7 ids.

Sign-in methods:
- Email plus password (core), email verification required before a session is usable.
- Email OTP (`emailOTP` plugin) for sign-in, verification and password reset.
- Magic links (`magicLink` plugin).
- Passkeys (`@better-auth/passkey`, `registration.requireSession: false` so a passkey can be the first credential; `rpID` from `PUBLIC_URL`).
- Google and GitHub social sign-in, enabled only when client ids are configured.
- Anonymous guests (`anonymous` plugin): a real user row flagged anonymous; `onLinkAccount` re-points `boards.created_by`, `board_acl` rows and `usage_events` to the linked user. A purge job removes stale anonymous users.
- Phone OTP is a later config flip (the messenger supports SMS).

Workspaces: `organization` plugin with `modelName: "workspace"`, `creatorRole: "owner"`, anonymous users may not create workspaces, invitations expire after 48 hours, a personal workspace is created on first sign-in. `bearer()` lets SDKs and the CLI present session tokens; `openAPI()` puts the auth routes into `spec/openapi.json`.

Messenger port (`packages/adapters/messenger`): `send({ template, channel: 'email'|'sms', to, tags, idempotencyKey })`. Production driver: `@tilli.dev/nudge.js` (`createNudgeClient` + `sendByTemplate`; the library is fetch-only with no Node-specific imports, so it runs in Workers). Templates by external ref: `magic-link`, `email-otp`, `verify-email`, `reset-password`, `workspace-invite`, `mention`. Idempotency keys derive from the auth token or invitation id. Test driver: Nudge's fake transport. Self-host drivers: `nudge` (if a key is configured), `smtp-relay` (Go sidecar), `console` (prints links and codes to the log; the default on first boot).

API keys: `qj_live_<prefix>_<secret>` / `qj_test_...`, SHA-256 `key_hash`, scopes `boards:read`, `boards:write`, `workspace:admin`, `mcp`. One Hono middleware resolves cookie session, bearer session, API key or share-link token into `ctx.principal = { type, userId, workspaceId, scopes, anonymous }`; the MCP endpoint reuses it.

Board tickets (`packages/auth/ticket.ts`): `POST /v1/boards/:id/ticket` (or `/v1/rooms/:id/ticket` for ephemeral and peer rooms) checks `board_acl`, `share_links` and `visibility`, asks `WorkspaceMeter` for resolved entitlements, records a `board.session` usage event, and returns an HS256 JWT `{ sub, bid, wid, role, cid, anon, mode, lim: { editors, bytes }, jti, exp: now + 60 s }` signed with `TICKET_SECRET`. `qj-rt` verifies with `TICKET_SECRET` then `TICKET_SECRET_PREV` (rotation without downtime) and never touches Postgres. Revocation on ACL change: `api` calls `BOARD_ROOM.get(boardId).revoke(userId)`, which closes sockets tagged `u:{userId}` with code 4401. Share-link tokens are 32 random bytes stored as SHA-256; link passwords use WebCrypto PBKDF2.

Self-host bootstrap: `PUBLIC_URL` drives `baseURL`, `rpID` and cookie domain; `BETTER_AUTH_SECRET` and `TICKET_SECRET` are generated into `/data/secrets.json` on first boot; the first magic link is printed to the container log.

## 7. Entitlements and rate limiting

`spec/features.yaml` is the single source: feature key, kind (`limit`, `flag`, `enum`), default per plan. `tools/gen/gen-features.ts` emits `packages/entitlements/types.ts` and `packages/db/seed.ts`. Initial keys: `boards.max_per_workspace`, `board.max_bytes`, `board.max_concurrent_editors`, `workspace.max_members`, `history.retention_days`, `storage.max_bytes`, `api.rpm_tier`, `api.requests_per_day`, `ai.units_per_day`, `webhooks.max`, `sharelinks.enabled`, `sharelinks.password`, `guests.allowed`, `rooms.peer_enabled`, `export.formats`.

`resolveEntitlements({ plan, overrides, usage })` is a pure function returning `{ [key]: { limit, used, remaining, source: 'plan'|'override' } }`. Precedence: `SELFHOST=1` selects plan `selfhost` (unlimited unless `SELFHOST_PLAN` is set); otherwise an active, trialing, past-due or comped subscription selects the plan; otherwise the default plan; then plan features; then non-expired overrides (workspace scope, then user scope; later-created wins).

`WorkspaceMeter` (one per workspace, `idFromName(workspaceId)`): `check({ feature, delta })` compares `counters + delta` against the cached limit in one SQLite transaction and returns `{ ok, used, limit }`; `getEntitlements()` returns the resolved object cached 5 minutes in `ent_cache`, refreshed from Postgres on miss; `invalidate()` is called by `api` on subscription or override change; a 60-second alarm flushes dirty counters to `usage_counters` and clears itself when clean. `api` middleware `requireEntitlement(key)` calls the meter before the Postgres write. `BoardRoom` enforces `board.max_concurrent_editors` on upgrade and `board.max_bytes` on ingest from the ticket's embedded limits.

Denials: `ENTITLEMENT_EXCEEDED` with `feature_key`, `current`, `limit`, HTTP 429 with a 402-style body.

Usage ledger: `recordUsage({ workspaceId, featureKey, qty, idempotencyKey })` inserts `usage_events`; hard daily quotas go through `WorkspaceMeter.check`. The five-minute cron rolls events into month and lifetime counters and appends NDJSON to `usage/{ws}/{yyyy-mm}.ndjson`.

Rate limiting:
1. Native `ratelimits` bindings (approximate, per location, one namespace per tier): `RL_AUTH` 20/60 s by IP on `/auth/*` and ticket minting; `RL_API_FREE` 300/60 s and `RL_API_PRO` 3000/60 s keyed by `u:{userId}` | `k:{prefix}` | `ip:{ip}` on `/v1/*` and `/mcp`, selected by `api.rpm_tier`. Miniflare's simulator resolves the same bindings in self-host.
2. Strict quotas in `WorkspaceMeter`.
3. Per-socket token bucket in `BoardRoom` (120 frames/s, 1 MiB max frame) stored in the WebSocket attachment. Better Auth's own limiter stays on for auth routes.

## 8. BoardRoom, sync seam, room modes

### 8.1 Sync-engine seam

`apps/rt/src/sync/engine.ts`:

```ts
interface SyncEngine {
  ingest(bytes: Uint8Array, author: string): { accepted: boolean; broadcast: boolean };
  since(vv: Uint8Array): Uint8Array | null;   // null => caller sends a full snapshot
  snapshot(): Uint8Array;
  versionVector(): Uint8Array;
  frontiers(): unknown;
  static loadFrom(snapshot: Uint8Array, updates: Uint8Array[]): SyncEngine;
}
```

`LoroEngine` wraps `loro-crdt` 1.17 and uses only the four JavaScript export modes (`snapshot`, `update` from a version vector, `shallow-snapshot`, `updates-in-range`). `loro` (Rust) and `loro-crdt` (JS) are pinned to the same minor so encodings match. `boards.doc_codec` records which engine wrote a board; mixed-mode boards are unsupported. A Figma-style server-ordered last-writer-wins engine or a rebase engine can be added later behind the same frames, persistence and auth; `go/loadgen` is the measuring stick; ADR-0001 records the trade-off.

### 8.2 BoardRoom behavior

- Construction: idempotent DDL migration inside `blockConcurrencyWhile`; `setWebSocketAutoResponse("ping", "pong")` so keepalives never wake the object.
- Upgrade: verify ticket (401 on failure); if `role === 'editor'` and the count of sockets tagged `r:editor` has reached `lim.editors`, respond `ENTITLEMENT_EXCEEDED` for `board.max_concurrent_editors`; `acceptWebSocket` with tags `u:{sub}`, `r:{role}`; attachment `{ uid, cid, role, anon, lim, bucket }` under 16 KiB; send `HELLO { seq, vv, codec, peers }`; broadcast `JOIN`.
- `UPDATE`: editors only; reject frames over 1 MiB (close 1009); reject when `size_bytes + pending + frame > lim.bytes` with `ENTITLEMENT_EXCEEDED` for `board.max_bytes`; `engine.ingest`; on accept push to pending, broadcast to all other sockets, schedule a flush alarm in 2 s.
- `SYNC`: reply with `engine.since(vv)`, or a full `SNAPSHOT` if the vector predates the compaction point.
- `PRESENCE`: broadcast only; never persisted.
- Close: broadcast `LEAVE`; if no sockets remain, schedule an immediate flush.
- Persistence: alarm-driven only (timers block hibernation). `flush()` writes one coalesced update row since `flushed_vv` inside `transactionSync`, updates `flushed_vv` and `size_bytes`. `snapshot()` runs when updates reach 200 rows, 1 MB, or 60 s of activity: write chunks, delete updates up to `seq`, update meta, put the snapshot to blobs at `boards/{id}/snap/{seq}.loro`, send `board.snapshotted` on `EVENTS` so `jobs` updates `boards.snapshot_key`, `snapshot_seq`, `size_bytes`, `doc_frontiers`, `last_edited_at`. After the alarm, if nothing is pending, no alarm is armed and the object may hibernate; if no sockets remain, the engine is dropped from memory.
- Load: snapshot chunks plus update rows from SQLite; if SQLite is empty, fetch the last `snapshot_key` from blobs. In-memory state is assumed lost at any time (local wrangler dev also evicts). Documented loss window: the unflushed 2-second tail on a crash; local-first clients hold and resend their ops on `SYNC`.
- Shallow snapshots are used only by retention GC in `jobs`, and only after all known peers have synced past the trim point; a stale `SYNC` gets a full snapshot.
- Typed RPC for `api` and `jobs`: `revoke(uid)`, `stats()`, `exportAt(frontiers)`, `refreshEntitlements()`.

Platform limits honored: 16 KiB attachments, 10 tags per socket, 2 MB SQLite rows (900 KB chunks), 10 GB per object on Paid, roughly 1,000 requests/s per object, incoming messages billed 20:1 (clients batch local ops every 50 to 100 ms).

### 8.3 Protocol

`packages/protocol` defines binary frames, byte 0 = type: `HELLO 0x01, UPDATE 0x02, SYNC 0x03, SYNC_REPLY 0x04, SNAPSHOT_REQ 0x05, SNAPSHOT 0x06, PRESENCE 0x10, JOIN 0x11, LEAVE 0x12, SIGNAL 0x20, ERROR 0x7f`; remainder is engine bytes or msgpack. Documented in `spec/ws-protocol.md` with a version byte reserved in `HELLO`.

### 8.4 Room modes and transport seam

Room mode is carried on the ticket (`mode`) and, for persistent boards, on `boards.room_mode`:

| Mode | Authority | Persistence | Lifetime |
|---|---|---|---|
| `hosted` | BoardRoom DO | SQLite + R2 | Until deleted |
| `e2e` | BoardRoom DO relaying ciphertext | SQLite + R2 (opaque) | Until deleted; key in URL fragment |
| `ephemeral` | BoardRoom DO, in memory only | None | Cleared when the host socket closes or after a TTL |
| `peer` | Host's browser | None on our side | Signaling DO clears when the host socket closes |

Foundation ships: the enum on tickets and boards; `Signaling` DO (`/v1/rooms/:id/signal`): accepts host and guest sockets via ticket, forwards `SIGNAL` frames (SDP offers and answers, ICE candidates) between peers, holds no document, closes all guest sockets and clears itself when the host socket closes; `POST /v1/turn/credentials` in `api` returning short-lived Cloudflare TURN credentials; a client transport interface in `sdks/ts` realtime client (`Transport { send(bytes), onMessage, close }`) with the WebSocket-to-DO implementation and stubs for WebRTC data channel and BroadcastChannel. Ephemeral mode, the WebRTC transport, E2E encryption and the save-local-copy flow are the sync milestone's work.

### 8.5 Tests

`@cloudflare/vitest-pool-workers` with `runInDurableObject` for all three classes, including a forced-eviction test (`/internal/evict` on staging only) proving the reload path, a two-client update exchange, the editor-limit rejection, the byte-limit rejection, and signaling teardown on host close. `go/loadgen` runs against staging as a smoke test.

## 9. Self-host

Runtime: one Docker image, `node:22-slim` plus a pinned stable Miniflare 4.x (the `workerd` npm version pinned through the pnpm catalog to match wrangler). Miniflare is Cloudflare's local-development runtime: workerd plus simulator Workers for bindings bare workerd only provides as designators (R2, Queues, rate limits, assets, Hyperdrive TCP). It is not a hardened multi-tenant sandbox; acceptable because only first-party code runs.

`apps/selfhost/server.ts` declares the four Workers with `modules: true`, `defaultPersistRoot: "/data"`, per-binding `useSQLite: true` for every DO, `hyperdrives: { HYPERDRIVE: DATABASE_URL, HYPERDRIVE_NOCACHE: DATABASE_URL }`, queue producers and consumers, `ratelimits` with the same limits as production, `serviceBindings: { AI: "ai-shim" }`, and routes by hostname (`app.`, `api.`, `rt.` on the operator's domain) or path prefixes under `QJ_SINGLE_HOST=true`. Crons: a `croner` ticker calls Miniflare's `/cdn-cgi/handler/scheduled?cron=...` for the two schedules.

Native with no adapter: Worker bundles and compatibility flags; service bindings and RPC; Durable Objects with SQLite, alarms and hibernatable WebSockets (files under `/data/do`); static assets; Queues with batching, retries and DLQ (simulator); rate-limit bindings (simulator, exact because single-process); Hyperdrive to the Postgres container; `cloudflare:sockets`; crons via the ticker; Analytics Engine binding resolves and discards.

Ports with drivers (`packages/adapters`, driver chosen by env, Cloudflare binding auto-detected):

| Port | Production driver | Self-host drivers |
|---|---|---|
| `getMessenger(env)` | `nudge` | `nudge`, `smtp-relay` (Go sidecar, `SMTP_URL`), `console` (default on first boot) |
| `getAI(env)` | Workers AI binding | `ai-shim` Worker forwarding to an OpenAI-compatible endpoint (`QJ_AI_BASE_URL`, default Ollama); `none` disables AI and `/v1/capabilities` reports it |
| `getBlobs(env)` | R2 binding; aws4fetch SigV4 against R2's S3 endpoint for presigned browser uploads | `s3` via aws4fetch against the required S3-compatible store (MinIO in compose, or any S3). The R2 simulator is not used for blobs. |

Compose (`apps/selfhost/compose.yaml`): `quincejam` (image, port 8787, volume `qj-data:/data`), `postgres:18` (volume), `minio` (volume, bucket created on first boot), `migrate` one-shot (`drizzle-kit migrate`, `depends_on: service_completed_successfully`), `mailrelay`, `caddy` (ACME TLS), optional profile `ollama`. Backups: `pg_dump`, MinIO bucket sync, tar of `/data`. `qj selfhost doctor` checks Postgres through the binding, a blob put/get round trip, a writable DO directory, and the ticker.

Documented limits: single process, single node; no horizontal scaling of board rooms (workerd's experimental S3-coordinated cluster mode is a future path); upgrading Miniflare or workerd majors may change the on-disk DO layout, so the image pins both and `tools/scripts/do-export.ts` dumps any object to NDJSON.

Image build (`apps/selfhost/Dockerfile`, multi-stage): stage 1 (Node 22, Rust stable, wasm-pack, Go 1.25) runs `pnpm turbo run build` and `wrangler deploy --dry-run --outdir dist/<worker>` per Worker so self-host bundles are byte-identical to production; stage 2 is `node:22-slim` plus Miniflare, `dist/`, the `qj` CLI binary and `mailrelay`. Multi-arch to `ghcr.io/<owner>/quincejam:<tag>`.

## 10. Codegen and shared types

API routes are declared with `hono-openapi` and zod 4; `pnpm --filter api gen:openapi` emits `spec/openapi.json` (OpenAPI 3.1, conservative subset: no nested `oneOf`). `tools/gen/gen-sdks.sh` generates: TS (`@hey-api/openapi-ts` into `sdks/ts`), Go (`oapi-codegen` into `go/sdk`), Rust (`progenitor` into `crates/qj-sdk`), Python (`openapi-python-client` into `sdks/python`). MCP tools come from the same zod route definitions via `x-mcp` extensions and are served by `@modelcontextprotocol/sdk` at `/mcp`. WS frames, queue messages and webhook payloads are zod schemas exported as JSON Schema (`gen:protocol`) for the Rust CLI's wire types. TypeScript owns transport types; `crates/qj-doc` owns the document schema and board-as-text. All generated output is committed; `tools/gen/check-drift.sh` fails CI on drift. Turborepo caches `gen` by the hash of `spec/openapi.json` and `spec/features.yaml`.

## 11. CI/CD

- `ci.yml` (PR and main): every job installs pnpm 10, Node 22, Rust stable with wasm32 (`Swatinem/rust-cache`), Go 1.25; `pnpm install --frozen-lockfile`; `pnpm turbo run lint check test build gen --affected` with the self-hosted remote cache; drift check on `spec/` and generated SDKs; Worker tests under `@cloudflare/vitest-pool-workers`; `cargo test --workspace --locked`; `go test ./...`; `wrangler deploy --dry-run --outdir dist` per Worker; `size-report.ts` fails if `doc-engine-wasm` exceeds 1.3 MB gzipped (Loro's current build is about 1.17 MB) or the web bundle grows unexpectedly; `check-do-lifecycle.ts` fails a PR that changes `exports` in `apps/rt/wrangler.jsonc` together with unrelated code; PR preview versions via `wrangler versions upload --env staging` for `qj-api` and `qj-web` with URLs posted as a comment.
- `neon-preview.yml` (PR): create branch `pr-<n>`, run migrations, comment the URL; delete on close.
- `deploy-staging.yml` (push to main): migrate the Neon `staging` branch via its direct URL; deploy `rt` (plain `wrangler deploy --env staging`), then `jobs`, `api`, `web` with `cloudflare/wrangler-action`; smoke: `GET /healthz` on api and rt, `go/loadgen --boards 2 --clients 10 --seconds 20` against rt staging, Playwright magic-link login.
- `deploy-prod.yml` (tag `v*`, GitHub environment `production` with required reviewer): expand-then-contract migrations; `rt` via `wrangler deploy` (lifecycle changes are atomic, not gradual, not rollback-safe); `api` and `web` via `wrangler versions upload` then `wrangler versions deploy` 10% to 100%; `jobs` plain deploy; `gh release create` with `qj` CLI binaries (darwin and linux, arm64 and amd64), npm publish `@quincejam/sdk`, PyPI publish, Go module tag.
- `selfhost-image.yml` (tag `v*`): buildx multi-arch; `docker compose up -d` in CI; smoke (console magic-link login, create board, open WS, snapshot lands in MinIO, `qj selfhost doctor` green); push on success.
- Secrets per env via `wrangler secret put`: `BETTER_AUTH_SECRET`, `TICKET_SECRET`, `TICKET_SECRET_PREV`, `GOOGLE_CLIENT_SECRET`, `GITHUB_CLIENT_SECRET`, `NUDGE_API_KEY`, `TURN_API_TOKEN`, `SENTRY_DSN` (optional). `.dev.vars` locally; `.env` for compose. Wrangler pinned at or above 4.107 (first release with `exports`).

## 12. Observability

`observability.enabled = true` on every Worker. `packages/log` emits one JSON line per event `{ lvl, msg, reqId, wid, bid, uid, ms }` and propagates `X-QJ-Request-Id` web to api to rt to jobs. Analytics Engine dataset `qj_metrics` for `ws_connect`, `update_bytes`, `flush_ms`, `snapshot_ms`, `meter_check`, `hyperdrive_ms`. `@sentry/cloudflare` `withSentry()` is a no-op without `SENTRY_DSN`. `/healthz` on every Worker returns `{ version, runtime, checks: { postgres, blobs, queue } }`. Owner-only `GET /v1/admin/boards/:id/stats` and `/v1/admin/workspaces/:id/meter`. Self-host logs JSON to stdout with an optional OTLP exporter behind `OTEL_EXPORTER_OTLP_ENDPOINT`. `audit_events` is the product-level trail.

## 13. Dev loop

`pnpm i` once (Node 22 via corepack, pnpm 10, Rust stable plus wasm-pack, Go 1.25; or the devcontainer). `pnpm dev` runs in parallel: one `wrangler dev` process for api, rt and jobs with `--persist-to .wrangler/state` (so `script_name` DO bindings and service bindings resolve), Vite on 5173 proxying `/api` and `/rt`, and `wasm-pack build --dev --watch` for `crates/qj-doc`. Postgres: local container via `hyperdrive.localConnectionString`, or a Neon dev branch via `CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE`. Blobs: local MinIO or the R2 simulator (dev only). Messenger `console`; AI `none` or Ollama. `pnpm db:generate | db:migrate | db:seed | db:studio`. `pnpm gen` regenerates spec, SDKs, feature types and protocol types. `pnpm test` (vitest-pool-workers, cargo, go); `pnpm test:e2e` (Playwright against wrangler dev); `pnpm selfhost:dev` runs the Miniflare bootstrap locally; `pnpm selfhost:up` runs the compose stack. `cargo watch -x 'test -p qj-doc'` is sub-second. Pre-commit (lefthook): biome, `cargo fmt --check`, `go vet`, `turbo run check --affected`.

## 14. Error handling conventions

- REST errors are `{ code, message, details? }` with stable `code` strings; `ENTITLEMENT_EXCEEDED` adds `feature_key`, `current`, `limit`.
- WebSocket errors are `ERROR` frames with the same `code` strings; fatal errors close with application codes 4401 (revoked or unauthorized), 1008 (rate), 1009 (too large), 1011 (internal).
- Queue consumers are idempotent (keyed on event id); failures retry with backoff and land in `qj-dlq`, drained by the nightly cron within retention.
- Webhook deliveries follow `pending -> ok | failed -> dead` with exponential backoff; a webhook auto-disables after 8 consecutive failures.

## 15. Cost expectations

Workers Paid ($5/month) from the start: 10M requests, 30 s CPU, Durable Objects 10 GB per object and 50M included row writes, 1M Queue operations. Neon Free through dogfood, Launch (about $19/month with scale-to-zero disabled on production) before public launch. R2 within the free 10 GB for a long time. Expected total at public free-with-limits launch: $10 to $30 per month. First overage candidates: DO row writes and Queue operations, both bounded by free-plan entitlements.

## 16. Corrections from verification that shaped this design

1. Better Auth on D1 is non-atomic for multi-step writes, so D1 is not used.
2. Miniflare option names differ between 4.x stable and 5.x beta; the bootstrap pins 4.x and uses `defaultPersistRoot` plus per-binding `useSQLite`.
3. Miniflare's email simulator only captures mail; delivery always goes through the messenger port.
4. A Worker with DO `exports` cannot use `wrangler versions upload`; all DO classes live in `qj-rt`.
5. Local wrangler dev evicts Durable Objects; in-memory state is assumed lost everywhere and a forced-eviction test exists.
6. The JavaScript Loro build exposes four export modes; `StateOnly` and `SnapshotAt` are Rust-only and arrive with the history milestone through `qj-doc`.
7. Shallow snapshots strand peers holding older updates; GC trims only after all peers have synced past the point.
8. Loro's browser wasm is about 1.17 MB gzipped; the CI gate is 1.3 MB; brotli and streaming compile are required on the web Worker.
9. The workerd Hyperdrive designator must be an `external` TCP service (Miniflare handles this).
10. Rate-limit bindings are approximate and per location; strict limits live in `WorkspaceMeter`.
11. Hyperdrive rejects session-level state (advisory locks, LISTEN/NOTIFY, session SET); quota checks use atomic update-returning.
12. Drizzle cannot model partitioned tables; those migrations are hand-written SQL.
13. ULID and UUIDv7 Postgres extensions are provider-specific; ids are generated in the Worker.

## 17. Open items to verify in the first week

- Hyperdrive's warm pool against Neon scale-to-zero: cold-start latency and whether it keeps compute awake. Benchmark on a dev branch.
- `pg_advisory_xact_lock` through Hyperdrive (not needed by design, confirm it is rejected rather than silently wrong).
- Cloudflare TURN credential API shape and quota for `POST /v1/turn/credentials`.
- nudge.js under the Workers runtime end to end (no Node imports found in source; confirm with a live send from a Worker).
- Miniflare 4.x `hyperdrives` option with two configs pointing at the same database.
