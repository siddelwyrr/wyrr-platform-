# WYRR Node

`wyrr-node` is a single, dependency-free Go binary that turns any machine — a Raspberry Pi, a server, a laptop, a kiosk — into a participant in the WYRR network. It mints its own identity, registers itself with the platform, exposes a local tool API, keeps working offline, and discovers peer nodes on the local network.

```
Binary:      wyrr-node (Go, CGO_ENABLED=0, static)
Version:     0.1.0
Local port:  7777
Cloud API:   https://wyrr.me/api/v1/mcp  (JSON-RPC 2.0)
Data dir:    ~/.wyrr-node
```

**Hardware floor:** any 64-bit CPU or ARMv7 · ~16 MB RAM · ~15 MB disk · no GPU, no libc required.

## Install & run

Build from source (Go 1.22+):

```bash
git clone <this repo> && cd wyrr-node
make build          # → ./wyrr-node
./wyrr-node
```

`make release` cross-compiles all six supported targets into `dist/`: `darwin-arm64`, `darwin-amd64`, `linux-amd64`, `linux-arm64`, `linux-armv7`, `windows-amd64.exe`.

Docker:

```bash
docker build -t wyrr-node .
docker run -d -p 7777:7777 -v wyrr-data:/home/wyrr/.wyrr-node wyrr-node
```

The image runs as a non-root `wyrr` user and persists identity + queue in the volume.

## First run: identity & self-registration

On first start the node:

1. **Mints a DID** — a keypair generated locally and stored in `~/.wyrr-node/identity.json`. The private key never leaves the machine.
2. **Self-registers** with the platform by calling the MCP tool `wyrr_create_account` with `{"name": "<node name>", "actorType": "machine"}` — no pre-provisioned credentials needed. The returned API key is stored locally.
3. Starts the local HTTP API on port 7777 and begins mDNS announcements.

To attach the node to an existing account instead, pass an API key you minted from the [MCP server](MCP_SERVER.md):

```bash
./wyrr-node --api-key wyrr_...
```

## Configuration

Flags override environment variables, which override defaults:

| Flag | Env | Default |
|---|---|---|
| `--api` | `WYRR_API` | `https://wyrr.me/api/v1/mcp` |
| `--api-key` | `WYRR_API_KEY` | empty → self-registers |
| `--name` | `WYRR_NODE_NAME` | `wyrr-node-<hostname>` |
| `--port` | — | `7777` |
| `--data-dir` | — | `~/.wyrr-node` |

## Local HTTP API

The node listens on loopback port 7777 (unauthenticated by design — bind it only to interfaces you trust):

| Route | Behavior |
|---|---|
| `GET /` | Node info |
| `GET /health` | Liveness + queue depth |
| `GET /identity` | The node's DID and public key |
| `POST /tool` | Execute a platform tool: `{ "tool": "wyrr_...", "args": { ... } }` |

`POST /tool` responses:

- **200** — tool executed against the cloud, MCP result passed through
- **202** — offline; call was queued: `{ "queued": true, "queueDepth": N }`
- **400** — malformed request
- **502** — cloud reachable but returned a transport error

## Offline resilience

When the cloud is unreachable, tool calls that can be safely deferred are appended to `~/.wyrr-node/queue.jsonl` — one JSON object per line: `{"ts": ..., "tool": "...", "args": {...}}`.

- The queue is **replayed every 30 seconds** in order.
- Replay stops at the first transport failure (the queue is preserved).
- Entries the platform *rejects* (as opposed to transport failures) are dropped — a poison entry can't wedge the queue.

Value transfer while offline is handled by **signed IOUs**: the node signs an IOU envelope (`wyrr.iou.v1`) that any peer or the platform can settle later via the `wyrr_submit_iou` tool. Settlement is idempotent by `(from, nonce)` — a replayed IOU can never double-spend.

## Mesh participation (LAN discovery)

Nodes announce themselves on the local network over **mDNS** (service type `_wyrr-node._tcp`, announced every 60 s) with TXT records carrying `did=<did>` and `v=1`. Any WYRR node can therefore discover peers on the same LAN with zero configuration and:

- call a peer's local tools (JSON-RPC `tools/call` against `http://<peer>:7777/`), and
- hand a signed IOU to a peer for later settlement when only one of them has internet.

Peer calls have a 10 s timeout and a 4 MiB response cap.

> **Status note:** the repository also contains richer node packages (`mesh/`, `mcp/`, `store/`, `identity/`, `tools/`) implementing a node-local MCP server, P-256 DIDs with BIP39 mnemonic backup, and an mDNS library integration. These are **in-development and not wired into the shipping binary yet** — the daemon that `make build` produces is the self-contained `main.go`. Treat the behavior documented on this page (which describes the shipping binary) as authoritative.

## Node roles on the network

A registered node is a machine actor like any other: it holds a wallet, has a trust score, can accept tasks, and earns WYRR for verifiable work (relays, data contributions, deliveries — see [TOKEN_ECONOMY.md](TOKEN_ECONOMY.md)). Advanced infrastructure roles for nodes (content relay, WebRTC signaling, proof-of-display witnessing, radius broadcast) are in **network preview** and not yet enabled.
