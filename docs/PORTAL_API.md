# Portal API — Screens as Network Nodes

Any display with a browser — a TV, a kiosk, a venue screen, a spare monitor — becomes an addressable WYRR endpoint by opening one URL:

```
https://wyrr.me/portal/screen/
```

The page mints an identity **locally in the browser** (a P-256 WebCrypto keypair and an 11-digit portal number), shows a QR code, and waits. A phone or authenticated web client scans it, and from that moment the screen is a **portal**: it can be commanded, broadcast to, sent drops, booked, and put into earning mode.

```
Base URL:  https://api.wyrr.me/api/v1/portal
```

## Lifecycle

```
   screen opens /portal/screen/          phone scans the QR
   ─ mints P-256 identity + number  ──▶  POST /accept  { portalId, portalDID, sessionPublicKey }
                                          └─ server mints a channelToken for the screen
   screen polls its command stream       phone (controller) sends commands
   GET /events?portalId&since&token ◀──  POST /command
                                          POST /claim  → portal becomes persistent
```

- **Temporary portals** (accepted but unclaimed) live **12 hours**, then expire.
- **Claimed portals** are persistent: they appear in the owner's `GET /mine`, keep their config, and survive restarts.
- Portal numbers are 6–16 digits (11 for new screens), never starting with 0.

## Authentication model

Portals use capability-token auth designed for screens that have no account:

| Party | Credential |
|---|---|
| **Screen** | `channelToken` (issued at accept) as `?token=` on `GET /events`; `x-portal-screen-secret` for the live relay |
| **Controller** (phone / web) | normal platform auth (session JWT); `x-portal-device-secret` for the live relay |

Only hash digests of secrets are stored server-side; comparisons are timing-safe. A `410 Gone` on any relay call means the session is over — stop polling.

## Endpoints

```
GET  /config                       public network config (modes, limits, polling interval)
POST /accept                       pair with a screen (mints channelToken)
POST /claim                        claim ownership → persistent
POST /command                      send a command to a portal
GET  /events?portalId&since&limit  poll the event stream (screens add &token=)
GET  /mine                         portals you own or control
GET  /nearby?lat&lng&radiusKm      discover public portals near a point (max 100 km)
GET  /:portalId                    portal record
POST /:portalId/config             update config (EARN mode, content rules)
POST /:portalId/broadcast          broadcast content to the portal
POST /:portalId/drop               send a drop to the portal
POST /bookings                     book portal screen time
POST /proofs/verify                verify a proof-of-display
```

### Live relay (low-latency session path)

For real-time control the relay path replaces polling storage with a direct command/event relay:

```
phone  ── POST /session/accept   ──▶  relay  ◀── GET  /session/commands/:portalId ── screen
phone  ── POST /session/command  ──▶  relay  ◀── POST /session/emit               ── screen
phone  ◀─ GET  /session/events/:portalId ─┘         (screen registers via POST /session/register)
```

Command payloads may be **sealed end-to-end** against the portal's ephemeral public key — the relay never sees plaintext.

## Portal record

```jsonc
{
  "portalId": "64774847484",
  "portalDID": "did:key:z…",
  "ownerDID": "did:…",
  "controllerDIDs": ["did:…"],
  "name": "Lobby screen",
  "status": "READY | ACTIVE | EXPIRED",
  "persistent": true,
  "capabilities": { "show": true, "cast": true, "broadcast": true,
                    "static": true, "reveal": true, "drops": true, "earn": false },
  "location": { "lat": 0, "lng": 0, "visibility": "EXACT | APPROXIMATE | HIDDEN" },
  "screen": { "w": 3840, "h": 2160, "dpr": 2 },
  "trust": { "score": 640 }
}
```

## The seven portal modes

| Mode | What it does |
|---|---|
| `show` | Display content pushed by a controller |
| `cast` | Live casting from a controller device |
| `broadcast` | Receive network-wide broadcasts |
| `static` | Persistent static content (signage) |
| `reveal` | Proximity-triggered reveals |
| `drops` | Receive and display token drops |
| `earn` | **Sell screen time for WYRR** (off by default) |

### EARN mode

A portal owner can put a screen on the market:

```
POST /api/v1/portal/:portalId/config
{
  "earnEnabled": true,
  "pricePerMinuteWYRR": 2,
  "contentRules": { "images": true, "video": true, "web": false }
}
```

Buyers book time via `POST /bookings`; display is verifiable via `POST /proofs/verify`. Config changes are controller-gated and persist across screen restarts.

## Network limits (from `GET /config`)

| Limit | Value |
|---|---|
| Event poll page | 100 events |
| Command payload | 8,192 bytes |
| Nearby radius | 100 km max |
| Recommended poll interval | 2,500 ms |
| Temporary portal TTL | 12 h |
| Trust scale | 0–1000 |

## Roadmap (network preview)

Deeper node roles for portals — content relay, P2P/WebRTC signaling, local mDNS/BLE discovery, proof-of-display witnessing, radius broadcast — are in **network preview** and surfaced (off by default) on [wyrr.me/node/](https://wyrr.me/node/). They are not part of the stable API yet.
