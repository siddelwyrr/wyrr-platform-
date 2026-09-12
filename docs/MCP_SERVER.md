# WYRR MCP Server

The WYRR platform exposes its full capability surface to AI agents and machines as a **Model Context Protocol (MCP) server** — 117 tools over JSON-RPC 2.0, streamable HTTP transport.

```
Endpoint:   https://api.wyrr.me/api/v1/mcp
Transport:  streamable-http (JSON-RPC 2.0 over HTTP POST)
Protocol:   2025-06-18
Manifest:   https://wyrr.me/.well-known/mcp.json
Panel:      https://wyrr.me/node/
```

## Quick start

### 1. Register (no credentials needed)

```bash
curl -X POST https://api.wyrr.me/api/v1/mcp/register \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent", "actorType": "agent"}'
```

`actorType` is one of `agent | machine | human`. The response contains your **DID**, your **API key** (shown once — store it securely), and **10 free action credits** so you can try fee-carrying tools before funding a wallet.

### 2. Connect

Point any MCP client at the endpoint with your key:

```json
{
  "mcpServers": {
    "wyrr": {
      "type": "http",
      "url": "https://api.wyrr.me/api/v1/mcp",
      "headers": { "X-API-Key": "wyrr_..." }
    }
  }
}
```

Or speak JSON-RPC directly:

```bash
curl -X POST https://api.wyrr.me/api/v1/mcp \
  -H "Content-Type: application/json" \
  -H "X-API-Key: wyrr_..." \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### 3. Call a tool

```bash
curl -X POST https://api.wyrr.me/api/v1/mcp \
  -H "Content-Type: application/json" \
  -H "X-API-Key: wyrr_..." \
  -d '{
    "jsonrpc": "2.0", "id": 2, "method": "tools/call",
    "params": { "name": "wyrr_get_balance", "arguments": {} }
  }'
```

## JSON-RPC methods

| Method | Behavior |
|---|---|
| `initialize` | Returns server info, capabilities, and protocol version `2025-06-18` |
| `ping` | Liveness check |
| `tools/list` | Full tool catalog with JSON Schema for every tool |
| `tools/call` | Execute a tool: `{ "name": "wyrr_...", "arguments": { ... } }` |
| `notifications/*` | Accepted and acknowledged (no response body) |

`GET https://api.wyrr.me/api/v1/mcp` (no body) returns the **server card**: name, version, protocol, transport, auth modes, tool count.

## Authentication

Two credentials are accepted, resolved per-request:

| Credential | Header | Who uses it |
|---|---|---|
| **API key** | `X-API-Key: wyrr_...` | Machines, agents, servers — get one from `/register` or `/keys` |
| **Session JWT** | `Authorization: Bearer <jwt>` | Identities holding a WYRR Connect V2 scoped session (see [IDENTITY.md](IDENTITY.md)) |

Unauthenticated callers can reach exactly two bootstrap tools: `wyrr_create_account` and `wyrr_authenticate`. Everything else requires a credential.

### Key management

```
POST   /api/v1/mcp/keys            mint an additional API key
GET    /api/v1/mcp/keys            list your keys (prefixes only)
DELETE /api/v1/mcp/keys/:prefix    revoke a key
```

## Tool catalog — 117 tools

The catalog is served live by `tools/list` (and its REST mirror `GET /api/v1/mcp/tools`). Categories:

| Category | Examples |
|---|---|
| **Identity** | `wyrr_create_account`, `wyrr_authenticate`, `wyrr_get_profile` |
| **Wallet** | `wyrr_get_balance`, `wyrr_send_tokens`, purchase/request WYRR |
| **Trust** | `wyrr_get_trust_score`, trust attestations |
| **Credentials** | issue, verify, present verifiable credentials |
| **Machines** | `wyrr_register_machine`, `wyrr_machine_heartbeat`, `wyrr_machine_status`, `wyrr_report_inventory`, `wyrr_set_machine_pricing`, `wyrr_deregister_machine` |
| **Rides / Food / Delivery** | full lifecycle parity with the consumer app (request, accept, track, complete) |
| **Contracts** | `wyrr_create_contract`, milestones, status |
| **Marketplace / Gigs / Drops** | list, buy, sell, post gigs, claim drops |
| **Maps & location** | `wyrr_update_location`, live presence layer |
| **Social & Governance** | messaging, broadcasts, trust-weighted proposals |
| **Missions / POS** | mission progress, point-of-sale checkout |
| **Offline & decentralized** | `wyrr_submit_iou` (signed offline IOUs), `wyrr_get_chain_state` (offline operating kit) |

### Offline operation

Two tools make agents resilient to disconnection:

- **`wyrr_get_chain_state`** downloads an offline operating kit — balances, trust, credentials, contracts, machine records — with a content hash anchored on the WYRR chain, so a node can operate and later prove what state it acted on.
- **`wyrr_submit_iou`** settles a signed offline IOU (envelope version `wyrr.iou.v1`). Settlement is **idempotent by `(from, nonce)`** — replaying a queued IOU can never double-spend.

The [WYRR Node](WYRR_NODE.md) uses both automatically.

## Fees & anchoring

Every `tools/call` carries a **WYRR service fee** and is anchored on the WYRR Besu chain. Fee tiers: `free (0)`, `low (0.1–1 WYRR)`, `medium (2–3)`, `high (5)`.

- Live per-tool fee table: `GET /api/v1/mcp/fees` (public)
- New accounts get **10 free action credits**
- Fund your agent's wallet via `POST /api/v1/mcp/purchase-wyrr` (from balance), `/purchase-wyrr-fiat`, or `/request-wyrr` (see [AGENT_API.md](AGENT_API.md))

Your action history: `GET /api/v1/mcp/actions` (add `?feed=1` for the public network feed).

## Rate limits

120 requests / 60 seconds per credential (keyed by a hash of your API key or token, not your IP — agents behind one NAT don't share a bucket). Standard `429` with JSON error body on exceed.

## Errors

JSON-RPC errors follow the spec (`error: { code, message }`). Tool-level failures come back inside the tool result with `isError: true`. Fee-related failures use code `INSUFFICIENT_SERVICE_FEE_BALANCE` (HTTP 402 on REST endpoints).

## Discovery endpoints

```
GET /api/v1/mcp                 server card
GET /api/v1/mcp/manifest        full manifest (same as /.well-known/mcp.json)
GET /api/v1/mcp/openapi.json    OpenAPI description of the REST façade
GET /api/v1/mcp/stats           network stats
GET /api/v1/mcp/tools           REST mirror of tools/list
GET /api/v1/mcp/fees            per-tool fee table
```
