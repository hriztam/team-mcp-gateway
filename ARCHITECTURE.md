# ARCHITECTURE.md — WooCommerce MCP Gateway Platform

> Audience: a coding agent (Claude Code / Cursor) implementing this repo phase by phase.
> Rule of thumb: when this document and a guess conflict, follow this document. When this document is silent, pick the simplest option and record the decision in `docs/decisions.md`.

---

## 1. Goal

A Kubernetes-native platform where a team talks to a WooCommerce store (and any other registered MCP servers) through **one MCP gateway endpoint** and an **in-dashboard chat agent**.

Hard constraints:
- Runs on a local K8s cluster (k3d recommended) and on a VPS (k3s) with **identical Helm charts**. Only `values-local.yaml` / `values-prod.yaml` differ.
- **No Kustomize.** Helm only.
- **No hardcoded secrets.** Everything sensitive comes from K8s Secrets.
- Every workload has readiness/liveness probes and resource requests/limits.
- The WooCommerce MCP server is **ClusterIP only**, reachable only from the gateway.

---

## 2. System Diagram

```
                     ┌─────────────────────────────────────────────────────┐
                     │                 Kubernetes cluster                  │
                     │                                                     │
 Browser ──────────► │  Ingress ──► shop.<domain>  ──► WordPress ──► MySQL │
 (storefront)        │     │                              ▲   (PVCs)       │
                     │     │                              │ WC REST API    │
 Browser ──────────► │     └────► app.<domain>  ──► Gateway (API + SPA)    │
 (dashboard/chat)    │                              │   ▲  │ SQLite (PVC)  │
                     │                              │   │  │               │
 Claude Code/Cursor ─┼──► app.<domain>/mcp ─────────┘   │  ├─► woo-mcp  (ClusterIP, NetworkPolicy: gateway only)
 (Bearer API key)    │                                  │  ├─► weather-mcp (upstream #2)
                     │            LLM provider ◄────────┘  └─► calculator-mcp (upstream #3)
                     │            (external API, key in Secret)            │
                     └─────────────────────────────────────────────────────┘
```

Four layers: **Store**, **WooCommerce MCP server**, **Gateway + Dashboard**, **Chat agent** (lives inside the gateway backend).

---

## 3. Tech Stack (decisions made — do not relitigate)

| Concern | Choice | Why |
|---|---|---|
| Language | TypeScript (Node 20+) everywhere | One language, official MCP SDK |
| MCP SDK | `@modelcontextprotocol/sdk` | Streamable HTTP server + client built in |
| HTTP framework | Fastify | Small, fast, good schema validation |
| Validation | Zod | Shared with MCP SDK tool schemas |
| Gateway DB | SQLite via `better-sqlite3`, file on a PVC | Minimal ops; single replica is fine |
| Password hashing | `argon2` (fallback `bcryptjs`) | |
| API key storage | SHA-256 hash of key, never plaintext | Keys are high-entropy, so fast hash is OK |
| Frontend | React + Vite + TypeScript + Tailwind + TanStack Query + React Router | |
| LLM | Any **OpenAI-compatible** chat-completions API with tool calling (Groq, OpenRouter free tier, Gemini OpenAI-compat, or local Ollama) | Provider swappable via env |
| Containers | Multi-stage Dockerfiles, non-root user | |
| Local cluster | **k3d** (k3s in Docker) | Enforces NetworkPolicy out of the box (kind's default CNI does not) and matches prod k3s |
| Ingress | Traefik (k3d/k3s default) | |
| Deployment | One umbrella Helm chart | |

---

## 4. Repository Layout

```
.
├── ARCHITECTURE.md
├── TASKS.md
├── README.md
├── docs/
│   ├── system-design.md
│   ├── threat-model.md
│   └── decisions.md
├── apps/
│   ├── woo-mcp/                 # WooCommerce MCP server
│   │   ├── src/{index.ts,tools/,woo-client.ts,config.ts}
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── gateway/                 # Gateway backend (Fastify)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── config.ts
│   │   │   ├── db/{schema.sql,client.ts,migrate.ts}
│   │   │   ├── auth/{session.ts,apikey.ts,password.ts,guards.ts}
│   │   │   ├── mcp/{endpoint.ts,router.ts,upstream.ts,prefix.ts}
│   │   │   ├── audit/interceptor.ts
│   │   │   ├── api/{auth.ts,servers.ts,users.ts,keys.ts,audit.ts,chat.ts}
│   │   │   └── chat/{agent.ts,llm.ts}
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── dashboard/               # React SPA (built, then served by gateway as static files)
│   │   └── src/{pages/,components/,api/}
│   └── mcp-extras/
│       ├── weather-mcp/
│       └── calculator-mcp/
└── deploy/
    ├── k3d-config.yaml
    └── helm/
        └── mcp-platform/
            ├── Chart.yaml
            ├── values.yaml            # shared defaults
            ├── values-local.yaml
            ├── values-prod.yaml
            └── templates/
                ├── _helpers.tpl
                ├── namespace-and-secrets.yaml
                ├── mysql-statefulset.yaml / mysql-service.yaml
                ├── wordpress-deployment.yaml / wordpress-service.yaml
                ├── wordpress-setup-job.yaml
                ├── woo-mcp-deployment.yaml / woo-mcp-service.yaml
                ├── woo-mcp-networkpolicy.yaml
                ├── gateway-deployment.yaml / gateway-service.yaml / gateway-pvc.yaml
                ├── extras-deployments.yaml
                ├── ingress.yaml
                └── NOTES.txt
```

Decision: the **gateway serves the built dashboard as static files** (one image, one Ingress route, no CORS). The Vite dev server proxies `/api` and `/mcp` to the gateway during development.

---

## 5. Component Specs

### 5.1 Store (WordPress + MySQL)

- MySQL 8 as a **StatefulSet** with a PVC at `/var/lib/mysql`.
- WordPress (official `wordpress:php8.x-apache` image) as a Deployment with a PVC at `/var/www/html/wp-content`.
- Env: `WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD`, `WORDPRESS_DB_NAME` from Secrets.
- Set `WORDPRESS_CONFIG_EXTRA` to include `define('WP_ENVIRONMENT_TYPE','local');` in local values **only**, so Application Passwords work over plain HTTP. In prod, terminate TLS at Ingress and use `https` site URL (then this define is unnecessary).
- Setup Job (`wp-cli`, Helm post-install hook, idempotent):
  1. `wp core install` (url from values, admin creds from Secret)
  2. Set permalinks to `/%postname%/` (required, plain permalinks break `/wp-json`)
  3. `wp plugin install woocommerce --activate`
  4. Enable COD payment gateway; disable others
  5. Basic WC setup (currency, country) to allow checkout without wizard
  6. Create an Application Password for the admin (or a dedicated `mcp-bot` user with `shop_manager` role). **Preferred: dedicated `mcp-bot` user** (least privilege).
- Application Password handoff: first version is **manual** (`wp user application-password create mcp-bot woo-mcp --porcelain`, then `kubectl create secret`). Automating it requires the Job's ServiceAccount to have RBAC to create a Secret; do that only as a stretch goal.

### 5.2 WooCommerce MCP Server (`apps/woo-mcp`)

- Implements MCP **Streamable HTTP**: single endpoint `POST /mcp` (plus `GET /mcp` returning 405 or an SSE stream if the SDK needs it). Run in **stateless** mode (new transport per request) for simplicity.
- Health endpoints (not MCP): `GET /healthz` (liveness), `GET /readyz` (readiness; may check WC reachability).
- Config (env): `WOO_BASE_URL` (in-cluster, e.g. `http://wordpress`), `WOO_USER`, `WOO_APP_PASSWORD`, `PORT`.
- Auth to WooCommerce: HTTP Basic using `WOO_USER:WOO_APP_PASSWORD` against `/wp-json/wc/v3/...`.
- Optional hardening: require a shared secret header from the gateway (`X-Upstream-Token`) in addition to NetworkPolicy.

**Tools (exactly these five, Zod input schemas, JSON text results):**

| Tool | Input | WC endpoint |
|---|---|---|
| `list_products` | `search?`, `status?`, `per_page? (default 20, max 100)`, `page?` | `GET /products` |
| `create_product` | `name`, `regular_price` (string), `description?`, `short_description?`, `status? (default "publish")`, `stock_quantity?` | `POST /products` |
| `list_orders` | `status?`, `after?` (ISO8601), `before?`, `per_page?`, `page?` | `GET /orders` |
| `get_order` | `id` (number) | `GET /orders/{id}` |
| `update_order_status` | `id`, `status` (enum: pending, processing, on-hold, completed, cancelled, refunded, failed) | `PUT /orders/{id}` |

Rules:
- Return trimmed, LLM-friendly objects (id, name, price, status, totals, line items), not the raw giant WC payload.
- On WC errors return an MCP tool result with `isError: true` and a short message. Never throw out of the handler.
- Tool descriptions must be precise; the chat agent's quality depends on them. Mention that "today's orders" requires the `after` parameter.

### 5.3 Gateway Backend (`apps/gateway`)

Single Fastify process exposing three surfaces:

1. `/mcp` — MCP server for downstream clients (Bearer API key auth).
2. `/api/*` — REST for the dashboard (session cookie auth).
3. `/*` — static dashboard SPA.

#### 5.3.1 Database (SQLite, WAL mode, file at `/data/gateway.db`)

```sql
CREATE TABLE users (
  id            TEXT PRIMARY KEY,           -- uuid
  email         TEXT NOT NULL UNIQUE,
  name          TEXT,
  password_hash TEXT,                       -- null until invite accepted
  role          TEXT NOT NULL CHECK (role IN ('admin','member')),
  status        TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','disabled','invited')),
  created_at    TEXT NOT NULL
);

CREATE TABLE invites (
  id          TEXT PRIMARY KEY,
  email       TEXT NOT NULL,
  role        TEXT NOT NULL,
  token_hash  TEXT NOT NULL UNIQUE,
  expires_at  TEXT NOT NULL,
  used_at     TEXT,
  created_by  TEXT NOT NULL REFERENCES users(id)
);

CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,             -- random, stored hashed or opaque
  user_id     TEXT NOT NULL REFERENCES users(id),
  expires_at  TEXT NOT NULL
);

CREATE TABLE api_keys (
  id           TEXT PRIMARY KEY,
  user_id      TEXT NOT NULL REFERENCES users(id),
  name         TEXT NOT NULL,
  key_prefix   TEXT NOT NULL,               -- first 8 chars, for display
  key_hash     TEXT NOT NULL UNIQUE,        -- sha256 hex
  created_at   TEXT NOT NULL,
  last_used_at TEXT,
  revoked_at   TEXT                         -- non-null = revoked
);

CREATE TABLE mcp_servers (
  id          TEXT PRIMARY KEY,
  slug        TEXT NOT NULL UNIQUE,         -- tool prefix, [a-z0-9]+ e.g. "woo"
  name        TEXT NOT NULL,
  url         TEXT NOT NULL,                -- upstream /mcp URL
  auth_type   TEXT NOT NULL DEFAULT 'none' CHECK (auth_type IN ('none','bearer','header')),
  auth_secret TEXT,                         -- encrypted at rest (AES-GCM, key from K8s Secret)
  enabled     INTEGER NOT NULL DEFAULT 1,
  status      TEXT NOT NULL DEFAULT 'unknown', -- connected | unreachable | disabled | unknown
  last_checked_at TEXT,
  last_error  TEXT,
  created_at  TEXT NOT NULL
);

CREATE TABLE audit_logs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  ts          TEXT NOT NULL,                -- ISO8601 UTC
  user_id     TEXT,                         -- null only for failed auth attempts
  source      TEXT NOT NULL,                -- 'mcp' | 'chat' | 'admin'
  server_slug TEXT,
  tool        TEXT,                         -- unprefixed upstream tool name
  action      TEXT NOT NULL,                -- 'tool_call' | 'key_created' | 'key_revoked' | 'server_added' | ...
  duration_ms INTEGER,
  outcome     TEXT NOT NULL,                -- 'success' | 'error' | 'timeout' | 'denied'
  detail      TEXT                          -- short JSON: error message, args summary (redacted)
);
CREATE INDEX idx_audit_ts ON audit_logs(ts DESC);
CREATE INDEX idx_audit_user ON audit_logs(user_id);
```

Bootstrap: on first start, if no users exist, create an admin from `ADMIN_EMAIL` / `ADMIN_PASSWORD` env (from Secret), and **upsert the pre-registered WooCommerce server** (`slug=woo`, `url=http://woo-mcp:PORT/mcp`).

#### 5.3.2 Auth

- **Dashboard:** `POST /api/auth/login` → sets `HttpOnly; SameSite=Lax; Secure(in prod)` session cookie. Guards: `requireUser`, `requireAdmin`.
- **Invites:** admin creates an invite → gateway returns a one-time URL `/accept-invite?token=...` (no email infra; admin copies the link). Accepting sets the password.
- **API keys:** format `mcpgw_<32 random bytes base64url>`. Shown **once** at creation. Lookup by SHA-256 hash on **every** request (no caching) so revocation is **immediate**. Reject if key revoked or user not `active`. Update `last_used_at`.
- Rate-limit login and `/mcp` (e.g. `@fastify/rate-limit`).

#### 5.3.3 MCP Endpoint (`/mcp`)

- Streamable HTTP, **stateless**, using the SDK's server + transport per request.
- Auth: `Authorization: Bearer <api key>` → resolves `user`. 401 on failure (and audit a `denied` row).
- Handles `initialize`, `tools/list`, `tools/call`. (Resources/prompts out of scope.)
- The MCP layer is a thin adapter over the **Router** (below). The chat agent calls the same Router in-process.

#### 5.3.4 Router (`mcp/router.ts`) — the core

```ts
interface CallerContext { userId: string; source: 'mcp' | 'chat' }

listTools(ctx): Promise<Tool[]>
callTool(ctx, prefixedName: string, args: unknown): Promise<CallToolResult>
```

**listTools:**
1. Load enabled servers from DB.
2. Fan out `tools/list` to all in parallel with `Promise.allSettled`, **per-server timeout 5s**.
3. Prefix each tool: `${slug}_${tool.name}` (e.g. `woo_list_orders`). Prefix the description too: `[woo] ...`.
4. Failed/timed-out servers contribute zero tools and have their `status` set to `unreachable` (with `last_error`). Successful ones set `connected`. **One bad server must never fail the whole call.**
5. Short-lived cache (e.g. 10s) per server to avoid hammering upstreams; invalidate on admin changes.

**callTool:**
1. Resolve server by matching the **longest registered slug** such that `name.startsWith(slug + '_')`. (Slugs are `[a-z0-9]+` and must not contain `_`, so parsing is unambiguous.)
2. Reject if server disabled/unknown → `isError` result "Unknown tool".
3. Call upstream `tools/call` with unprefixed name, **timeout 30s** (configurable).
4. Wrap with the **audit interceptor** (measures duration, records outcome).
5. On upstream failure: return `isError: true` with a clear message (`"Server 'woo' is unreachable"`), mark server `unreachable`. Never crash.

**Upstream client (`upstream.ts`):** SDK `Client` + `StreamableHTTPClientTransport`, created per call (stateless) or pooled with reconnect. Applies stored auth headers. Always uses `AbortController` for timeouts.

**Health checker:** background loop every 15–30s calls `tools/list` (or `ping`) on every enabled server to refresh `status` so the dashboard shows "Unreachable" even without traffic. Disabled servers show `disabled`.

#### 5.3.5 Audit Interceptor

One function wrapping `callTool`:
```
start = now
try { result = await inner(); outcome = result.isError ? 'error' : 'success' }
catch (e) { outcome = isTimeout(e) ? 'timeout' : 'error' }
finally { insert audit row (user, source, slug, tool, ts, duration, outcome, detail) }
```
Also write audit rows for admin actions (server add/edit/disable/remove, key create/revoke, invite, role change). **Never log API keys, secrets, or full tool arguments containing PII** — store a redacted/truncated summary.

#### 5.3.6 REST API (all JSON, prefix `/api`)

| Method & Path | Role | Purpose |
|---|---|---|
| `POST /auth/login`, `POST /auth/logout`, `GET /auth/me` | any | Session |
| `POST /auth/accept-invite` | public (token) | Set password |
| `GET /servers` | admin | List with status |
| `POST /servers` | admin | Add `{slug,name,url,auth_type,auth_secret}` (validate by calling `tools/list` first; show result) |
| `PATCH /servers/:id` | admin | Edit / enable / disable |
| `DELETE /servers/:id` | admin | Remove |
| `GET /servers/:id/tools` | admin | Preview upstream tools |
| `GET /users`, `POST /users/invite`, `PATCH /users/:id` (role/status) | admin | Team mgmt |
| `GET /keys` (own), `POST /keys`, `DELETE /keys/:id` | member | Own keys |
| `GET /admin/keys`, `DELETE /admin/keys/:id` | admin | See/revoke anyone's keys |
| `GET /audit?page&pageSize&user&server&outcome` | admin | Paginated audit log |
| `GET /config` | any | Returns public gateway MCP URL for the "copy URL" box |
| `POST /chat` (SSE stream) | any | Chat agent |

### 5.4 Chat Agent (`apps/gateway/src/chat`)

- `POST /api/chat` body: `{ messages: [{role, content}] }`. Auth = session user.
- Loop (max ~8 iterations):
  1. `tools = router.listTools(ctx)` → convert to OpenAI function-tool format (names already prefixed; ensure names match `^[a-zA-Z0-9_-]{1,64}$`).
  2. Call LLM with messages + tools.
  3. If response has `tool_calls`: for each, `router.callTool({userId, source:'chat'}, name, args)`; append `tool` messages; loop.
  4. Else stream final assistant text to the browser.
- Stream events to the UI over **SSE**: `token`, `tool_call` (name + status), `tool_result` (short), `done`, `error`. The UI shows tool calls inline so users can see what happened.
- System prompt: short, states the current date/time and timezone (needed for "today's orders"), says to prefer tools over guessing, and to confirm before destructive actions (e.g. status change to cancelled/refunded).
- Env: `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY` (Secret).
- Because the agent calls the Router in-process with the logged-in user's ID, audit rows are attributed to that user with `source='chat'`. No API key is needed for chat.

### 5.5 Dashboard (`apps/dashboard`)

Routes:
- `/login`, `/accept-invite`
- Member: `/keys` (create, list with prefix/last used/revoke; one-time reveal modal; **Gateway URL** copy box with a ready-to-paste Claude Code snippet: `claude mcp add --transport http woo-gateway <url>/mcp --header "Authorization: Bearer <key>"`), `/chat`
- Admin: `/admin/servers` (table with status badge Connected / Unreachable / Disabled, add/edit/disable/remove, tool preview), `/admin/team` (invite, role, disable user, revoke keys), `/admin/audit` (paginated, filterable)

Nav shows admin items only if `role === 'admin'` (server enforces regardless).

### 5.6 Extra Upstream MCP Servers (`apps/mcp-extras`)

Two tiny Streamable-HTTP servers deployed as ClusterIP services:
- `weather-mcp` → tool `get_weather(city)` (use Open-Meteo, no key needed)
- `calculator-mcp` → tools `add`, `multiply`, `evaluate(expression)` (safe parser, **never `eval`**)

Registered via the admin UI with slugs `weather`, `calc`. A cross-server prompt for testing: *"Get the weather in Mumbai and tell me what 18% GST on my latest order total would be."*

---

## 6. Kubernetes & Helm

### 6.1 Naming and Domains

- Namespace: `mcp-platform`.
- Local: `shop.127.0.0.1.nip.io` and `app.127.0.0.1.nip.io` (resolve to 127.0.0.1 automatically; k3d maps host port 80 → loadbalancer).
- Prod: real domains, or `shop.<vps-ip>.nip.io` / `app.<vps-ip>.nip.io` if no domain yet.

### 6.2 Workloads

| Workload | Kind | Storage | Exposure |
|---|---|---|---|
| mysql | StatefulSet (1) | PVC | ClusterIP |
| wordpress | Deployment (1) | PVC (`wp-content`) | ClusterIP + Ingress `shop.*` |
| woo-mcp | Deployment | none | **ClusterIP only**, NetworkPolicy allows ingress only from gateway pods |
| gateway | Deployment (**replicas: 1**, SQLite) | PVC (`/data`) | ClusterIP + Ingress `app.*` |
| weather-mcp / calculator-mcp | Deployments | none | ClusterIP only |
| wordpress-setup | Job (hook) | — | — |

Every container: `readinessProbe`, `livenessProbe`, `resources.requests/limits`, `securityContext` (`runAsNonRoot` where the image allows; MySQL/WordPress official images may need exceptions).

### 6.3 Secrets (all created by Helm from values supplied at install time or pre-created by hand; **never committed with real values**)

- `mysql-credentials` (root + wp user passwords)
- `wordpress-admin` (admin user/password)
- `woo-mcp-credentials` (`WOO_USER`, `WOO_APP_PASSWORD`)
- `gateway-secrets` (`SESSION_SECRET`, `ENCRYPTION_KEY` for upstream secrets, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `LLM_API_KEY`)

Repo contains `values-*.yaml` with **placeholder/dummy** secret values for local only; prod secrets are passed via `--set` or a git-ignored `values-secrets.yaml` / pre-created Secrets referenced by `existingSecret`.

### 6.4 values files — what differs

| Key | local | prod |
|---|---|---|
| `ingress.shopHost` / `ingress.appHost` | `*.127.0.0.1.nip.io` | real domains |
| `ingress.tls` | off | on (cert-manager or Traefik ACME) |
| `image.pullPolicy`, `image.repository` | local registry (`k3d-registry`) / imported images | GHCR/Docker Hub |
| `resources` | small | moderate |
| `wordpress.siteUrl` | `http://shop.127.0.0.1.nip.io` | `https://...` |
| `wordpress.environmentType` | `local` | `production` |
| `persistence.storageClass` | default | `local-path` (k3s default) |
| `gateway.cookieSecure` | false | true |

Chart templates must be **identical**; any prod/local branching happens only through values (avoid `if .Values.env == "prod"` style logic).

---

## 7. Security Requirements (feeds the Threat Model doc)

1. API keys stored as hashes; displayed once; revocation checked on every request (no cache) → immediate cutoff.
2. Passwords hashed with argon2; sessions are HttpOnly cookies; CSRF protection for state-changing `/api` routes (SameSite=Lax + `Origin` check).
3. Upstream credentials encrypted at rest (AES-256-GCM) with a key from a K8s Secret.
4. `woo-mcp` unreachable from outside the cluster: ClusterIP + NetworkPolicy (+ optional shared-secret header).
5. Least privilege for WooCommerce: dedicated `mcp-bot` user instead of the main admin.
6. Admin-only routes enforced server-side by role guard.
7. Audit log is append-only from the app's perspective (no update/delete endpoints).
8. SSRF consideration: admin-supplied upstream URLs can point anywhere in the cluster. Mitigate by restricting to admins, logging, and (stretch) blocking link-local/metadata IPs.
9. Prompt injection: tool results (e.g. product descriptions) are untrusted input to the LLM. Mitigate with confirmation for destructive tools and by always attributing calls to the human user.
10. Rate limiting on login and `/mcp`.
11. Secrets never appear in logs, audit detail, Helm values committed to git, or container images.

---

## 8. Behavioral Requirements (acceptance tests)

1. Admin logs in, sees `woo` pre-registered as **Connected**.
2. Member creates a key, copies URL; external client lists tools and sees `woo_list_products`, `woo_create_product`, `woo_list_orders`, `woo_get_order`, `woo_update_order_status` plus extras (`weather_*`, `calc_*`).
3. Chat (User A): "Create a product called X priced 499" → product visible on storefront.
4. Place an order manually → chat "show today's orders" returns it.
5. External client (User B) answers an orders question; and a prompt using tools from two different servers.
6. Revoke User B's key → next call returns 401 immediately.
7. Disable an upstream → its tools disappear from chat and external clients on next `tools/list`.
8. `kubectl delete pod` / scale an upstream to 0 → dashboard shows **Unreachable** within ~30s; other servers keep working; gateway stays up.
9. Every action above appears in the audit log with correct user, server, tool, duration, outcome.
10. Same chart deploys to k3s VPS with only `values-prod.yaml` changes.

---

## 9. Conventions for the Coding Agent

- Implement **one phase at a time** (see `TASKS.md`). Do not start phase N+1 until phase N's "Done when" checks pass.
- After each unit of work: run lint, typecheck, and tests; show the commands and results.
- Add a minimal test for each non-trivial module: prefix parsing, router timeout behavior, API key auth, audit interceptor.
- Keep modules small; no premature abstractions; no extra dependencies without a one-line justification in `docs/decisions.md`.
- Every new env var is documented in the app's `README` section and wired through Helm values and Secrets.
- Never put secrets, API keys, or real credentials in code, charts, or commit history.
- Configuration comes from env vars validated at startup with Zod; fail fast with a clear message.
