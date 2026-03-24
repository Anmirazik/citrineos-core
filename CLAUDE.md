# CitrineOS Core — Claude Instructions

## What This Project Is

CitrineOS Core is an open-source **OCPP 2.0.1 / 1.6 Charge Point Management System (CSMS)** server built with TypeScript. It manages EV charging stations via WebSocket connections and exposes REST APIs via Fastify.

---

## Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Language | TypeScript | ^5.8.2 |
| Runtime | Node.js | >=24.4.1 (pin: v24.4.1 via .nvmrc) |
| HTTP Framework | Fastify + @fastify/cors | - |
| WebSocket | ws | 8.17.1 |
| Message Broker | RabbitMQ (amqplib) | AMQP 0-9-1 |
| Database | PostgreSQL 16 (PostGIS) | via Sequelize ORM |
| Migrations | sequelize-cli | ^6.6.2 |
| Testing | Vitest | - |
| Linting | ESLint + Prettier | - |
| Containerization | Docker / Docker Compose | - |

---

## Monorepo Structure

This is an **npm workspaces monorepo**. Each package has its own `src/`, `test/`, `dist/`, and `tsconfig.json`.

```
00_Base/        → Core interfaces, OCPP types, validators, money utils
01_Data/        → Sequelize models, repositories, mappers (OCPP 1.6 + 2.0.1)
02_Util/        → Network connections, WebSocket auth, certificate utilities
03_Modules/     → Feature modules (each is an independent npm workspace)
  Certificates/ → Certificate management
  Configuration/→ Boot, heartbeat, firmware handling
  EVDriver/     → Authorization, local list, reservations
  Monitoring/   → Variable monitoring, event notifications
  OcppRouter/   → Message routing, webhook dispatch
  Reporting/    → Logs, reports, diagnostics
  SmartCharging/→ Charging profiles, schedules
  Tenant/       → Multi-tenant support
  Transactions/ → Meter values, status notifications, transaction lifecycle
Server/         → Entry point, Fastify server, Docker configs, config.json
migrations/     → Sequelize migration files
```

---

## Key Entry Points

- **Server entry**: `Server/src/index.ts` → `Server/src/citrineOSServer.ts`
- **Config file**: `Server/data/config.json` — runtime config (ports, modules, broker URL, etc.)
- **Database models**: `01_Data/src/layers/sequelize/model/`
- **OCPP message routing**: `03_Modules/OcppRouter/`

---

## Port Assignments (config.json)

| Port | Purpose |
|---|---|
| 8080 | Central System REST API (Fastify) — **host mapped to 8085 in docker-compose** |
| 8081 | WebSocket — OCPP 2.0.1, security profile 0 (no auth) — **host mapped to 8086 in docker-compose** |
| 8082 | WebSocket — OCPP 2.0.1, security profile 1 (basic auth) |
| 8443 | WebSocket — OCPP 2.0.1, security profile 2 (TLS) |
| 8444 | WebSocket — OCPP 2.0.1, security profile 3 (mTLS) |
| 8092 | WebSocket — OCPP 1.6 |
| 8085 | OCPI server |
| 8090 | Hasura GraphQL (host port → container 8080) |
| 9000/9001 | MinIO object storage |
| 5672/15672 | RabbitMQ / management UI |

> **Note:** Port 8080 (host) is remapped to 8085 in `Server/docker-compose.yml` to avoid conflicts with other local services.

---

## Common Commands

```bash
# Install all workspace dependencies
npm run install-all

# Build entire monorepo (TypeScript project references)
npm run build

# Start locally (with nodemon hot reload)
npm run start

# Start with Docker (full stack)
cd Server && docker-compose -f docker-compose.yml up -d

# Run unit tests
npm test

# Run tests with coverage
npm run coverage

# Lint
npm run lint

# Lint + auto-fix
npm run lint-fix

# Format with Prettier
npm run prettier

# Run DB migrations
npm run migrate

# Clean build artifacts
npm run clean

# Full clean (dist + node_modules + lock files)
npm run fresh && npm run install-all
# or shorthand:
npm run fi
```

---

## Code Style

- **Prettier**: single quotes, trailing commas, 100-char line width, 2-space indent, semicolons
- **File naming**: PascalCase for classes/models (`BootNotificationService.ts`), camelCase for utilities
- **Test files**: `*.test.ts` inside a `test/` directory mirroring `src/`
- **Imports**: ESM (`"type": "module"` in all packages)
- **Error handling**: Errors propagate up; modules use `try/catch` selectively — avoid wrapping everything
- **Async**: `async/await` throughout; no callbacks

---

## Testing

- **Runner**: Vitest (`npm test` from root runs all workspaces)
- **Test location**: `<package>/test/**/*.test.ts`
- **Integration tests**: suffixed `.integration.test.ts` (skipped in normal runs)
- **Coverage**: `npm run coverage`

---

## Docker Setup

```bash
# From the Server/ directory:
docker-compose -f docker-compose.yml up -d

# Services started:
# - ocpp-db (PostgreSQL/PostGIS)
# - amqp-broker (RabbitMQ)
# - minio + minio-init
# - citrine (the CSMS server)
# - graphql-engine (Hasura)
```

The `citrine` service waits for `ocpp-db` and `amqp-broker` to be healthy before starting.

---

## Environment Variables (Docker)

| Variable | Description |
|---|---|
| `APP_NAME` | Module to run (`all` = everything) |
| `APP_ENV` | Environment (`local`, `docker`) |
| `DB_STRATEGY` | `migrate` or `sync` |
| `BOOTSTRAP_CITRINEOS_DATABASE_HOST` | PostgreSQL host |
| `BOOTSTRAP_CITRINEOS_CONFIG_FILENAME` | Config file name (`config.json`) |
| `BOOTSTRAP_CITRINEOS_FILE_ACCESS_TYPE` | `local` or `s3` |

---

## Multi-Tenant

CitrineOS supports multi-tenancy. WebSocket servers in `config.json` have a `tenantId` and can use `dynamicTenantResolution: true` to resolve tenants from the connection URL.

---

## Known Local Port Conflicts

- Port **8080** on the host is already in use. It is remapped to **8085** in `Server/docker-compose.yml`.
- Port **8081** on the host is already in use. It is remapped to **8086** in `Server/docker-compose.yml`.
- Access the REST API at: `http://localhost:8085`
- Swagger docs: `http://localhost:8085/docs`
- Connect OCPP 2.0.1 chargers (no auth) to: `ws://localhost:8086`

---

## Git Conventions

- Commit style: short imperative messages (e.g., `add timeout to prevent restart`)
- PRs merged via merge commits into `main`
- Branch naming: `feature/<name>`, `bugfix/<name>`
- Pre-commit hook via Husky runs `prettier --write` on staged files
