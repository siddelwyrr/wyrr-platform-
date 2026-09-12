# Vehicle Agent

A vehicle running the WYRR Vehicle Agent is a **mobile network node**: it proves where it's been, relays mesh traffic for nearby devices, contributes road data, and earns WYRR for all of it — settling to a dedicated `vehicle` sub-account of its owner's wallet.

```
Base URL:  https://api.wyrr.me/api/v1/vehicle
Auth:      Authorization: Bearer <session JWT>   (or API key for server-side fleet integrations)
```

## Identity — deterministic vehicle DIDs

A vehicle's DID is derived, not registered — no provisioning round-trip, and the same (owner, device, vehicle) always resolves to the same identity:

```
material = "<ownerDID>|vehicle|<vehicleID>"
did      = "did:wyrr:vehicle:" + hex(SHA256(material))[0..16]
```

The backend rejects any vehicle DID that doesn't carry the `did:wyrr:vehicle:` prefix.

## Protocol

All responses are **flat** top-level objects (`{ "success": true, ... }`) — there is no `data` envelope.

### Register / deregister

```
POST /nodes/register      { "vehicleDID", "ownerDID", "kind", "capabilities": [...] }
POST /nodes/deregister    { "vehicleDID" }
```

`capabilities` is capped at 12 entries. Registration is idempotent — re-registering refreshes the record.

### Heartbeat

```
POST /nodes/heartbeat     { "vehicleDID", "sessionKm", "relays" }
```

Send every **60 seconds** while driving. The heartbeat carries session distance and relay counts; it keeps the node visible on the network and feeds earnings accrual.

### Proof-of-location attestations

Vehicles emit a **tamper-evident hash chain** of location attestations:

```
POST /attestations        { "id", "vehicleDID", "lat", "lon", "speedKmh", "at", "chainHash" }
```

Each attestation's `chainHash` commits to the previous one:

```
canonical = "<lastChainHash>|<vehicleDID>|<lat>|<lon>|<unixSeconds>"
chainHash = SHA256(canonical)
```

Any gap or edit breaks the chain. The server accepts and stores attestations into a rolling window (last 500 per vehicle); verification is performed out-of-band — submission never blocks driving.

### Earnings settlement

```
POST /earnings/settle     { "vehicleDID", "entries": [ { "id", "kind", "amount", "detail", "at" }, ... ] }
```

- **Idempotent by entry `id`** — the client re-sends until it gets a 2xx; replayed ids are deduplicated and can never double-credit. Retry freely.
- Limits per call: **200 entries**, **25 WYRR total**.
- Per-entry ceilings by kind: `relay 0.002`, `distance 2.0`, `dataContribution 2.0` (drop claims are cross-checked against the drop's own claim rail).
- **Client telemetry never mints WYRR directly.** Settled entries land as *pending*; a platform treasury pass verifies and credits real balance to the owner's `vehicle` sub-account.

Earning kinds and rates:

| Kind | Rate |
|---|---|
| `relay` | 0.001 WYRR per mesh relay |
| `distance` | 0.01 WYRR per km driven |
| `dataContribution` | per-contribution, capped 2.0 |
| `dropClaim` | face value of the claimed drop |

### Node status

```
GET /nodes/:did           node record + pending-earnings summary
```

## Trust

Driving builds the vehicle owner's [trust score](TOKEN_ECONOMY.md#trust-score): **+0.1 per 10 miles**, **+0.05 per relay**, on the canonical trust rail.

## Mesh relay

The vehicle participates in the WYRR device mesh over local radio (peer-to-peer Wi-Fi + Bluetooth discovery, service type `wyrr-vmesh`):

- Relay packets: `{ id, originDID, sealedPayload, hopCount, createdAt }`
- **Max 8 hops**, 24-hour TTL — packets drop after either.
- Payloads are **sender-sealed** (a relay can never read what it carries) with an additional per-link AES-GCM transport wrap so hops can't be correlated.
- Packets queue on-device and forward opportunistically — a vehicle with no internet still moves data toward one that has it.

Relay settlement for the general device mesh runs on a separate rail (`/api/v1/mesh`): idempotent settlement by envelope id, relay receipts, per-user earnings, and network coverage density (`GET /api/v1/mesh/coverage`). Relayers keep 80% of relay fees.

## CarPlay

The Vehicle Agent ships with a CarPlay experience — Map (drops on route), Agent (live earnings + mesh stats), Wallet (read-only while driving; no financial actions from the car), and Drops (one-tap claim, nearest first). Connecting to CarPlay is the agent's power switch: attach registers the node and starts earning; detach settles and signs off.

> **Status:** CarPlay distribution requires an Apple entitlement that is pending approval — the CarPlay surface is currently testable in development environments only. The Vehicle Agent protocol above (registration, heartbeats, attestations, earnings, mesh) is independent of CarPlay and fully live.

## Integration checklist

1. Derive the vehicle DID from the owner DID + a stable vehicle identifier.
2. `POST /nodes/register` on trip start; `POST /nodes/heartbeat` every 60 s.
3. Emit chained attestations as you move (10 m accuracy, ~25 m distance filter is the reference tuning).
4. Accumulate earnings locally; `POST /earnings/settle` in batches with stable entry ids; re-send until 2xx.
5. `POST /nodes/deregister` (or just stop heartbeating) on trip end.
