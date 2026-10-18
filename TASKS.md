# TASKS.md — Step-by-Step Build Plan

Work top to bottom. Each phase ends with a **Done when** checklist. Don't move on until it passes. Tick boxes as you go. Companion doc: `ARCHITECTURE.md`.

Estimated effort (focused work): Phase 0 ½ day · P1 1–1.5 days · P2 1 day · P3 1.5–2 days · P4 2 days · P5 1.5–2 days. Roughly 8–10 days total.

---

## Phase 0 — Prep (½ day)

Tools to install: Docker, `kubectl`, `helm`, `k3d`, Node 20+, `pnpm` or `npm`, `jq`, `git`.

- [ ] Create the Git repo, add the folder layout from ARCHITECTURE §4 (empty dirs with `.gitkeep`)
- [ ] Add `.gitignore` (include `node_modules`, `.env*`, `values-secrets.yaml`, `*.db`)
- [ ] Copy `ARCHITECTURE.md` and `TASKS.md` into the repo root
- [ ] Pick your LLM provider and grab a free API key (Groq, OpenRouter, or Gemini). Test that it supports **tool calling** with a quick curl
- [ ] Create `docs/decisions.md` (a running log of choices)

---

## Phase 1 — Local Cluster & Store (1–1.5 days)

**Goal:** a working WooCommerce store on k3d with persistent data, reachable by URL.

### Steps

1. **Create the cluster** with a local registry and host port 80 mapped:
   ```bash
   k3d cluster create mcp --agents 1 \
     -p "80:80@loadbalancer" \
     --registry-create mcp-registry:0.0.0.0:5001
   kubectl get nodes
   ```
   Save this as `deploy/k3d-config.yaml` (or a `make cluster` target) so it's reproducible.
2. **Verify ingress works:** k3s ships Traefik. Deploy a throwaway nginx + Ingress for `test.127.0.0.1.nip.io` and open it in your browser. Delete it afterwards.
3. **Scaffold the Helm chart:**
   ```bash
   helm create deploy/helm/mcp-platform
   ```
   Delete the sample templates, create `values.yaml`, `values-local.yaml`, `values-prod.yaml`, and `_helpers.tpl`.
4. **Secrets template:** one template that renders Secrets from values (dummy passwords in `values-local.yaml`). Never real secrets in git.
5. **MySQL:** StatefulSet + headless/ClusterIP Service + PVC (`volumeClaimTemplates`). Add readiness probe (`mysqladmin ping`), resource requests/limits.
6. **WordPress:** Deployment + Service + PVC for `wp-content`. Env from Secrets. Probes on `/wp-login.php` (or `/`). Set `WORDPRESS_CONFIG_EXTRA` with `WP_ENVIRONMENT_TYPE=local` (local values only).
7. **Ingress:** host `shop.127.0.0.1.nip.io` → WordPress service.
8. **Install:**
   ```bash
   helm upgrade --install mcp deploy/helm/mcp-platform \
     -n mcp-platform --create-namespace \
     -f deploy/helm/mcp-platform/values-local.yaml
   kubectl -n mcp-platform get pods -w
   ```
9. **WordPress + WooCommerce setup.** Do it **manually first** to learn what's needed: open the shop URL, finish install, install the WooCommerce plugin, run basic setup, enable **Cash on delivery**, set permalinks to "Post name".
10. **Place a test order** from the storefront, confirm it appears in WC Admin → Orders.
11. **Prove persistence:** `kubectl -n mcp-platform delete pod <wordpress-pod> <mysql-pod>`; after restart, the store and order must still be there.
12. **Automate (do this after it all works manually):** convert your manual steps into the `wordpress-setup` Job using `wp-cli` (idempotent: check `wp core is-installed` first). Run `helm uninstall` + delete PVCs + reinstall to prove a clean install gets a working store.
13. **Create the `mcp-bot` user** (role `shop_manager`) and an Application Password:
    ```bash
    kubectl -n mcp-platform exec deploy/mcp-wordpress -- \
      wp user create mcp-bot bot@example.com --role=shop_manager --allow-root
    kubectl -n mcp-platform exec deploy/mcp-wordpress -- \
      wp user application-password create mcp-bot woo-mcp --porcelain --allow-root
    ```
    Store the result in the `woo-mcp-credentials` Secret.
14. **Test the REST API from inside the cluster:**
    ```bash
    kubectl -n mcp-platform run curl --rm -it --image=curlimages/curl -- \
      curl -u mcp-bot:<app-password> http://wordpress/wp-json/wc/v3/products
    ```
    If you get 401, check `WP_ENVIRONMENT_TYPE=local` and permalinks.

### Done when
- [ ] Cluster up, ingress working, stable URLs
- [ ] Helm chart + both values files exist; no Kustomize
- [ ] WordPress + MySQL running with PVCs; data survives pod deletion
- [ ] COD checkout works; test order visible in WC Admin
- [ ] Secrets are K8s Secrets; probes and resources set on all workloads
- [ ] REST API call with the app password returns products

---

## Phase 2 — WooCommerce MCP Server (1 day)

**Goal:** a containerized MCP server with 5 tools, reachable only inside the cluster.

### Steps

1. `cd apps/woo-mcp && npm init`; install `@modelcontextprotocol/sdk`, `zod`, `express` or `fastify`, `typescript`, `tsx`, `vitest`.
2. **`config.ts`:** validate env `WOO_BASE_URL`, `WOO_USER`, `WOO_APP_PASSWORD`, `PORT` with Zod.
3. **`woo-client.ts`:** small `fetch` wrapper that adds Basic auth, a timeout, and turns non-2xx responses into typed errors.
4. **Tools (one file each):** `list_products`, `create_product`, `list_orders`, `get_order`, `update_order_status` with Zod input schemas (exact params in ARCHITECTURE §5.2). Return trimmed JSON.
5. **HTTP + MCP wiring:** `POST /mcp` using the SDK's `StreamableHTTPServerTransport` in stateless mode (new server + transport per request). Add `/healthz` and `/readyz`.
6. **Test locally first** with the MCP Inspector against a port-forwarded WordPress:
   ```bash
   kubectl -n mcp-platform port-forward svc/wordpress 8081:80
   WOO_BASE_URL=http://localhost:8081 ... npm run dev
   npx @modelcontextprotocol/inspector   # connect to http://localhost:3001/mcp (Streamable HTTP)
   ```
   Call all five tools. Create a product, confirm it appears on the storefront.
7. **Dockerfile** (multi-stage, non-root), then build + push to the k3d registry:
   ```bash
   docker build -t localhost:5001/woo-mcp:dev apps/woo-mcp
   docker push localhost:5001/woo-mcp:dev
   ```
8. **Helm:** `woo-mcp-deployment.yaml`, `woo-mcp-service.yaml` (**ClusterIP**), env from `woo-mcp-credentials`, probes, resources.
9. **NetworkPolicy:** allow ingress to woo-mcp only from pods labeled as the gateway. (Gateway doesn't exist yet; label it now.)
10. **Verify isolation:** a random curl pod should **fail** to reach `woo-mcp:PORT`; a pod with the gateway label should succeed. Also confirm there's no Ingress rule for it.

### Done when
- [ ] All 5 tools work end to end via Inspector
- [ ] Deployed as ClusterIP with probes, limits, Secret-based credentials
- [ ] NetworkPolicy blocks non-gateway pods
- [ ] Unit tests for tool input validation pass

---

## Phase 3 — Gateway Backend (1.5–2 days)

**Goal:** auth, DB, MCP aggregation with prefixing, timeouts, audit logging. Test with curl/Inspector before any UI.

### Steps

1. **Scaffold** `apps/gateway` (Fastify + TS). Config via Zod. `/healthz`.
2. **DB:** implement `schema.sql` from ARCHITECTURE §5.3.1 with a tiny migration runner. Enable WAL. DB file path from env (`/data/gateway.db`).
3. **Bootstrap:** on first start create the admin from `ADMIN_EMAIL`/`ADMIN_PASSWORD`; upsert the `woo` server row.
4. **Auth:**
   - password hashing (argon2) + login/logout/me with session cookie
   - API key generation (`mcpgw_...`), SHA-256 storage, `authenticateApiKey()` hitting the DB on every call
   - `requireUser` / `requireAdmin` guards
5. **REST API:** servers (CRUD + tools preview), users/invites, keys (own + admin), audit list (paginated), config. Write an audit row for each admin action.
6. **Upstream client (`upstream.ts`):** SDK client + `StreamableHTTPClientTransport`, AbortController timeouts, optional auth headers (decrypt `auth_secret`).
7. **Prefix module (`prefix.ts`):** `prefixName(slug, tool)` and `resolve(name, slugs)` using the longest-slug rule. **Write unit tests** (collision: two servers both exposing `list_items`; slug containing no underscore; unknown prefix).
8. **Router (`router.ts`):**
   - `listTools` with `Promise.allSettled`, 5s per-server timeout, status updates, short cache
   - `callTool` with 30s timeout, `isError` results instead of throws
   - **Test:** make a fake upstream that sleeps 10s; confirm `listTools` still returns the other servers' tools within ~5s
9. **Audit interceptor** wrapping `callTool` (user, server, tool, ts, duration, outcome).
10. **`/mcp` endpoint:** Bearer auth → stateless MCP server that maps `tools/list` and `tools/call` to the Router.
11. **Health checker loop** (every ~20s) that refreshes server statuses.
12. **Local test (no K8s needed yet):** run gateway on your machine with `WOO` URL pointing at a port-forward; use the Inspector with `Authorization: Bearer <key>`; verify tools appear prefixed, calls work, audit rows are written.
13. **Dockerfile + Helm:** `gateway-deployment.yaml` (**replicas: 1**), PVC at `/data`, Service, Secrets (`SESSION_SECRET`, `ENCRYPTION_KEY`, admin creds), Ingress `app.127.0.0.1.nip.io`. Add the gateway pod label used by the NetworkPolicy.
14. **In-cluster smoke test:**
    ```bash
    curl -s -X POST http://app.127.0.0.1.nip.io/mcp \
      -H "Authorization: Bearer $KEY" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json, text/event-stream" \
      -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq
    ```
15. **Revocation test:** revoke the key via API, repeat the curl → must be 401 immediately.

### Done when
- [ ] Admin bootstrap works; `woo` pre-registered and Connected
- [ ] Tools from upstream returned as `woo_*`
- [ ] Slow/dead upstream doesn't break other servers
- [ ] Revoked key rejected on the very next call
- [ ] Every tool call produces an audit row with all required fields
- [ ] Gateway reachable through Ingress; woo-mcp still not reachable except via gateway

---

## Phase 4 — Dashboard & Chat Agent (2 days)

**Goal:** usable UI for admins and members, plus a working chat agent.

### Steps

1. **Scaffold** `apps/dashboard` (Vite + React + TS + Tailwind + TanStack Query + React Router). Dev proxy `/api`, `/mcp` → gateway.
2. **Auth pages:** login, accept-invite, auth context (`/api/auth/me`), route guards.
3. **Member views:**
   - `/keys`: list keys (name, prefix, created, last used, revoked), create dialog with **one-time reveal** + copy, revoke button
   - Gateway URL box with copyable Claude Code / Cursor config snippets
4. **Admin views:**
   - `/admin/servers`: table with status badges (Connected / Unreachable / Disabled), add (validate before save), edit, disable toggle, delete, tools preview
   - `/admin/team`: invite (shows copyable link), change role, disable user, view/revoke a user's keys
   - `/admin/audit`: paginated table with filters (user, server, outcome)
   - Poll server status every ~10s
5. **Serve the SPA from the gateway:** build step copies `dashboard/dist` into the gateway image; Fastify static plugin with SPA fallback (don't shadow `/api` and `/mcp`).
6. **Chat backend (`chat/agent.ts`):**
   - `llm.ts`: OpenAI-compatible client (`LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`)
   - loop: list tools → LLM → execute `tool_calls` through `router.callTool` with `{userId, source:'chat'}` → feed results back → repeat (max ~8 iterations)
   - `POST /api/chat` streaming SSE events (`token`, `tool_call`, `tool_result`, `done`, `error`)
   - system prompt with current date/time + timezone + "confirm before destructive actions"
7. **Chat UI:** message list, input, streaming text, collapsible "tool call" chips showing which tool ran and its status.
8. **LLM Secret:** add `LLM_API_KEY` to `gateway-secrets`, wire through Helm.
9. **Test the user stories** (browser):
   - User A: "Create a product 'Test Mug' at 499" → check storefront
   - Place an order on the storefront → "What orders came in today?"
10. **Check audit:** chat tool calls appear with `source=chat` and User A's ID.

### Done when
- [ ] Admin can add/edit/disable/remove servers and sees live status
- [ ] Admin can invite a user (copy link), revoke keys, read paginated audit log
- [ ] Member can create/revoke keys and copy the gateway URL
- [ ] Chat creates a product and lists today's orders
- [ ] LLM key comes from a K8s Secret

---

## Phase 5 — Hardening, Extras, Production (1.5–2 days)

### 5A. Extra upstream servers
- [ ] Build `weather-mcp` (Open-Meteo `get_weather`) and `calculator-mcp` (`add`, `multiply`, `evaluate` with a safe parser, **no `eval`**)
- [ ] Dockerfiles, Helm Deployments + ClusterIP Services
- [ ] Register both via the admin UI (slugs `weather`, `calc`)

### 5B. External client
- [ ] Create a key as **User B**; add the gateway to Claude Code:
  ```bash
  claude mcp add --transport http woo-gateway http://app.127.0.0.1.nip.io/mcp \
    --header "Authorization: Bearer <USER_B_KEY>"
  ```
  (Or Cursor `mcp.json` with the same URL + header.)
- [ ] Ask an orders question
- [ ] Run a two-server prompt, e.g. *"What's the weather in Mumbai and what's 18% of my latest order total?"*

### 5C. Failure scenarios (record these for the demo)
- [ ] Revoke User B's key → next call rejected immediately
- [ ] Disable `weather` → its tools vanish from chat and Claude Code (after reconnect / re-list)
- [ ] `kubectl -n mcp-platform scale deploy/mcp-weather-mcp --replicas=0` → dashboard shows **Unreachable** in ≤30s; Woo tools still work; gateway pod not restarted
- [ ] Scale it back up → returns to **Connected**
- [ ] Audit log shows every action above with correct attribution

### 5D. Security pass
- [ ] Rate limiting on login and `/mcp`
- [ ] CSRF/Origin check on state-changing `/api` routes
- [ ] No secrets in logs, audit detail, repo, or images (`git grep -i password`, check `helm template` output)
- [ ] NetworkPolicy verified for woo-mcp (and extras if desired)

### 5E. VPS deployment (k3s)
1. Get a small VPS (2 vCPU / 4 GB is enough). Point DNS (or use `nip.io` with its IP).
2. Install k3s: `curl -sfL https://get.k3s.io | sh -`; copy kubeconfig locally.
3. Push images to a real registry (GHCR/Docker Hub); set `image.repository` in `values-prod.yaml`.
4. Enable TLS (Traefik ACME resolver or cert-manager) and set `ingress.tls`, `wordpress.siteUrl=https://...`, `cookieSecure=true`, `wordpress.environmentType=production`.
5. Create prod secrets (`--set` or git-ignored `values-secrets.yaml`):
   ```bash
   helm upgrade --install mcp deploy/helm/mcp-platform -n mcp-platform --create-namespace \
     -f values-prod.yaml -f values-secrets.yaml
   ```
6. Re-run the WooCommerce setup Job / manual app-password step; create the `woo-mcp-credentials` Secret; restart woo-mcp.
7. Re-run the end-to-end checks against the prod URLs. The **chart must be unchanged**; only values differ.

### 5F. Documentation & delivery
- [ ] `README.md`: prerequisites, local setup (cluster → images → helm install → first login), VPS setup, env/secret reference, troubleshooting
- [ ] `docs/system-design.md`: architecture diagram, request flow (external client and chat), routing/prefix strategy, timeouts and failure handling, SQLite tradeoff (single replica; migrate to Postgres to scale), stateless MCP tradeoff, what you'd do next
- [ ] `docs/threat-model.md`: assets, actors, trust boundaries, threats and mitigations (use ARCHITECTURE §7 as the checklist), residual risks
- [ ] Record the demo video (suggested script below)

**Demo script (≈6–8 min):**
1. Show cluster pods, storefront, dashboard (admin) with 3 servers Connected
2. Chat: create a product → show it on storefront
3. Place an order → chat: today's orders
4. Claude Code as User B: orders question, then the two-server prompt
5. Revoke B's key → fails
6. Disable a server → tools vanish
7. Kill an upstream pod → Unreachable, others fine
8. Audit log showing all of the above
9. Quick look at the VPS deployment running the same chart

### Done when
- [ ] Everything in the original Definition of Done checklist (sections 1–9) is ticked
- [ ] Prod deployment passes the same user stories
- [ ] README, System Design, Threat Model written; demo recorded

---

## How to use a coding agent with this plan

Per phase, start a fresh session in the repo and prompt like this:

> Read `ARCHITECTURE.md` and `TASKS.md`. We are implementing **Phase N**. Work through the steps in order, one at a time. After each step, run tests/lint/typecheck and show me the output. Stop and ask before adding dependencies not listed in ARCHITECTURE §3. Do not touch later phases. When finished, walk through the "Done when" checklist and report what's verified vs. not.

Tips:
- Do Phase 1 mostly by hand (or with heavy supervision); cluster/Helm mistakes are the costliest to debug.
- Commit after every step so you can roll back.
- Keep `docs/decisions.md` updated, since it becomes raw material for the System Design note.

## Common gotchas checklist
- Application Passwords not working over HTTP → set `WP_ENVIRONMENT_TYPE=local` (local only)
- `/wp-json` returns 404 → permalinks still "Plain"
- NetworkPolicy has no effect → CNI doesn't enforce it (use k3d/k3s, not default kind)
- SQLite + more than one gateway replica → corruption/locking. Keep `replicas: 1`
- MCP clients failing on content negotiation → send `Accept: application/json, text/event-stream`
- Tool names rejected by LLM provider → keep `^[a-zA-Z0-9_-]{1,64}$`
- "Today's orders" returns nothing → system prompt must include current date/timezone; pass `after` in ISO8601
- Images not found in cluster → push to `localhost:5001/...` and reference as `mcp-registry:5000/...` from inside k3d (or use `k3d image import`)
