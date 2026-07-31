# tavily-proxy

A Cloudflare Worker that acts as an MCP (Model Context Protocol) proxy for the [Tavily API](https://tavily.com). It provides the same tools as the official Tavily MCP server, but with an **API key pool** — automatically rotating through multiple Tavily keys and selecting the one with the most remaining credit.

## Features

- **MCP Server** — Streamable HTTP transport at `POST /mcp`, compatible with any MCP client
- **4 Tavily Tools** — `tavily-search`, `tavily-extract`, `tavily-crawl`, `tavily-map`
- **API Key Pool** — Multiple Tavily API keys stored in Cloudflare KV; each request picks the least-recently-used healthy key
- **Health-aware routing** — Tavily 432 (quota exhausted) keys are skipped until the next UTC month; 429 rate limits apply a transient cooldown; 401/403 keys are invalidated
- **Key Management API + Admin Panel** — HTTP endpoints plus an embedded web panel (`GET /admin`) to add/delete keys and monitor usage
- **Auth Protected** — All endpoints except `GET /` and `GET /admin` require an `x-api-key` header

## Endpoints

| Method   | Path            | Description                                               |
|----------|-----------------|-----------------------------------------------------------|
| `POST`   | `/mcp`          | MCP Streamable HTTP endpoint (tool calls)                 |
| `POST`   | `/api/keys`     | Add a Tavily API key to the pool                          |
| `DELETE` | `/api/keys`     | Remove a Tavily API key from the pool                     |
| `GET`    | `/api/keys`     | List all keys and their status (auto-triggers lazy sync)  |
| `POST`   | `/api/keys/sync`| Force a `/usage` sync for all keys                        |
| `GET`    | `/admin`        | Admin panel (opens without auth; prompts for AUTH_KEY)    |
| `GET`    | `/`             | Health check (no auth required)                           |

All endpoints except `GET /` and `GET /admin` require the `x-api-key` header matching your configured `AUTH_KEY`.

## Setup

### Prerequisites

- [Node.js](https://nodejs.org/) v18+
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) (`npm install -g wrangler`)
- A Cloudflare account

### Local Development

```bash
# Install dependencies
npm install

# Set your auth key for local dev (already in .dev.vars)
# AUTH_KEY=test-secret-key

# Start local server
npm run dev
```

Wrangler simulates KV locally — no Cloudflare account needed for development.

### Deploy to Production

1. **Create a KV namespace:**
   ```bash
   npx wrangler kv namespace create KV
   ```

2. **Update `wrangler.toml`** — replace `YOUR_KV_NAMESPACE_ID` with the real ID from step 1.

3. **Set the auth secret:**
   ```bash
   npx wrangler secret put AUTH_KEY
   ```

4. **Deploy:**
   ```bash
   npm run deploy
   ```

5. **Add Tavily API keys to the pool:**
   ```bash
   curl -X POST https://your-worker.workers.dev/api/keys \
     -H "Content-Type: application/json" \
     -H "x-api-key: your-auth-key" \
     -d '{"apiKey": "tvly-xxx"}'
   ```

## Usage

### Connect MCP Clients

With `mcp-remote` (for clients like Cursor, Claude Desktop, etc.):

```json
{
  "mcpServers": {
    "tavily-proxy": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://your-worker.workers.dev/mcp",
        "--header", "x-api-key:${AUTH_KEY}"
      ],
      "env": {
        "AUTH_KEY": "your-auth-key"
      }
    }
  }
}
```

### Admin Panel

Open `https://your-worker.workers.dev/admin` in a browser. The panel asks for your `AUTH_KEY` (kept in the browser's `sessionStorage`), then lets you:

- List all keys — masked key, status (`active` / `exhausted` / cooling), remaining/limit credit, last-used and last-synced times
- Add and delete keys
- Force a usage sync via the **Sync usage now** button

### Key Management

```bash
# Add a key (auto-queries remaining credit from Tavily)
curl -X POST https://your-worker.workers.dev/api/keys \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-auth-key" \
  -d '{"apiKey": "tvly-xxx"}'

# List all keys and credits
curl https://your-worker.workers.dev/api/keys \
  -H "x-api-key: your-auth-key"

# Force a usage sync for all keys
curl -X POST https://your-worker.workers.dev/api/keys/sync \
  -H "x-api-key: your-auth-key"

# Delete a key
curl -X DELETE https://your-worker.workers.dev/api/keys \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-auth-key" \
  -d '{"apiKey": "tvly-xxx"}'
```

## How the Key Pool Works

The design follows [tavily-hikari](https://github.com/IvanLi-CN/tavily-hikari): the request hot path does **not** estimate or deduct credits locally. Key health is driven by real upstream signals:

1. Each request picks the **least-recently-used healthy key** (LRU).
2. **HTTP 432** (Tavily quota exhausted) → the key is marked `exhausted` and skipped for the rest of the current UTC month; it is restored automatically at the next monthly reset.
3. **HTTP 429** (rate limited) → the key is cooled down for the `Retry-After` window (fallback 60s) and the next key is tried.
4. **HTTP 401/403** → the key is marked `exhausted` and retried against the next key.
5. Remaining credit is only a **display field**, synced from Tavily's `/usage` endpoint in the background:
   - `POST /api/keys` queries usage when adding a key.
   - `GET /api/keys` returns cached values and kicks off a background sync when they are stale (>30 min).
   - `POST /api/keys/sync` forces a full sync.

### Workers Free Plan Note

Cron triggers do **not** fire on the Workers Free plan. The `scheduled` handler is still included, but on Free you should rely on the lazy sync built into `GET /api/keys` and the admin panel's **Sync usage now** button. Uncomment the `[triggers]` block in `wrangler.toml` to enable hourly cron syncs after upgrading to a paid plan.

## License

ISC
