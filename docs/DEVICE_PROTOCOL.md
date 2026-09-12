# Device Protocol

Machines join WYRR through two complementary rails:

1. **Machine ID** — a DID-native identity for autonomous machines (drones, robots, vehicles, IoT, sensors) that participate in the economy: accept tasks, hold sessions, get paid.
2. **IoT device bridge** — a lightweight shadow/telemetry/command API for conventional connected devices.

Both use the standard [auth](MCP_SERVER.md#authentication): an API key (`X-API-Key: wyrr_...`) or a WYRR Connect session JWT.

---

## 1. Machine ID (`/api/v1/connect/machine`)

A **Machine** is a first-class DID identity owned by a person or organization. It carries capabilities, a geofence, an operating policy, and its own audit trail.

### Machine record

```jsonc
{
  "machineId": "…",
  "did": "did:wyrr:machine:…",          // or a did:key the machine controls
  "type": "drone | robot | vehicle | iot | sensor",
  "name": "courier-04",
  "ownerUid": "…", "ownerDID": "did:…",
  "fleetId": "…",                        // optional fleet grouping
  "capabilities": ["delivery", "payload:5kg", "range:20km"],
  "modules": ["delivery", "wallet"],     // platform modules it may use
  "walletAddress": "0x…",                // public payout address — never a key
  "geoFence": { "centerLat": 0, "centerLng": 0, "radiusMeters": 5000 },
  "batteryThreshold": 20,                // min battery % to accept a job
  "status": "active | idle | busy | disabled | maintenance"
}
```

### Endpoints

```
POST /api/v1/connect/machine/register           register a machine
GET  /api/v1/connect/machine/fleet              list your machines
GET  /api/v1/connect/machine/:id                machine record
POST /api/v1/connect/machine/:id/credential     issue the machine a verifiable credential
POST /api/v1/connect/machine/:id/session        mint the machine a scoped session
POST /api/v1/connect/machine/:id/geofence       set geofence
POST /api/v1/connect/machine/:id/geofence/check test a coordinate against the fence
POST /api/v1/connect/machine/:id/status         update status
POST /api/v1/connect/machine/:id/sign           machine-signed payloads
POST /api/v1/connect/machine/:id/revoke         kill switch — revokes the machine and
                                                cascades revocation to its active sessions
GET  /api/v1/connect/machine/:id/audit          audit trail
GET  /api/v1/connect/machine/:id/tasks          tasks assigned to the machine
POST /api/v1/connect/machine/:id/dispatch       dispatch a task to the machine
```

### Task lifecycle

```
POST /api/v1/connect/machine/tasks/:taskId/accept    machine evaluates & accepts
POST /api/v1/connect/machine/tasks/:taskId/deliver   machine reports completion → payout
GET  /api/v1/connect/machine/tasks/:taskId           task status
```

Acceptance returns an evaluation the machine (or its operator) can inspect: `{ batteryOk, geoOk, capacityOk, reasons[] }` — a task outside the geofence, beyond capacity, or below the battery threshold is refused with reasons. Delivery returns the payout: `{ to, asset, amount }`.

### Fleet operations (`/api/v1/connect/fleet`)

```
POST /enroll                       enroll a machine into a fleet
GET  /                             list fleet machines
GET  /:machineId                   fleet view of one machine
POST /:machineId/rotate            rotate the machine's keys
POST /:machineId/decommission      permanently retire
POST /:machineId/transfer          transfer ownership
```

### MCP equivalents

Machines that speak MCP directly can use: `wyrr_register_machine`, `wyrr_machine_status`, `wyrr_machine_heartbeat`, `wyrr_machine_transaction`, `wyrr_report_inventory`, `wyrr_set_machine_pricing`, `wyrr_get_machine_stats`, `wyrr_deregister_machine`.

Machine scopes for session-based auth: `machine:read`, `machine:register`, `machine:control`, `machine:revoke`, `machine:sign`.

---

## 2. IoT device bridge (`/api/v1/iot`)

For conventional devices that need state sync and remote commands rather than economic agency. Also mounted at the developer alias `/api/v1/devices/*` (`POST /api/v1/devices/register`, `GET /api/v1/devices/:deviceId/shadow`, …).

### Register

```bash
curl -X POST https://api.wyrr.me/api/v1/iot/devices/register \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "greenhouse-sensor-1", "type": "sensor", "protocol": "http"}'
```

Returns **201** with the device record, including a per-device `authToken` (32-byte random, shown once):

```jsonc
{
  "deviceId": "dev_…",
  "name": "greenhouse-sensor-1",
  "type": "sensor",
  "protocol": "mqtt | http | websocket",
  "authToken": "…",
  "status": "provisioning"        // → online | offline
}
```

**Device authentication:** flash the `authToken` into the device. The device then calls its own endpoints headlessly with `X-Device-Token: <authToken>` — no user session required. A device may read its shadow, update `reported` state, and post telemetry; issuing commands and setting `desired` state remain owner-only.

### Device shadow (desired/reported state)

The shadow follows the classic desired/reported/delta model with optimistic versioning:

```
GET /api/v1/iot/devices/:deviceId/shadow
PUT /api/v1/iot/devices/:deviceId/shadow
```

```jsonc
{ "deviceId": "dev_…", "desired": { … }, "reported": { … }, "delta": { … }, "version": 7 }
```

Set `desired` from your application; the device reports back `reported`; `delta` is the computed difference the device should reconcile.

### Commands

```
POST /api/v1/iot/devices/:deviceId/command      { "command": "…", "payload": { … } }
```

Command lifecycle: `pending → delivered → acknowledged | failed | expired`. Every command records the issuer and timestamps for audit.

### Telemetry

```
POST /api/v1/iot/devices/:deviceId/telemetry    ingest a reading: { "data": { … } }
GET  /api/v1/iot/devices/:deviceId/telemetry    query recent readings
```

---

## Choosing a rail

| Need | Use |
|---|---|
| The machine earns, pays, accepts tasks, holds credentials | **Machine ID** |
| The machine is a vehicle | [Vehicle Agent](VEHICLE_AGENT.md) (a specialization of Machine ID) |
| State sync, remote commands, telemetry for a conventional device | **IoT bridge** |
| Both | Register a Machine ID and an IoT device record; link them by putting the `deviceId` in the machine's metadata |
