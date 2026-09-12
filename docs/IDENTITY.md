# Identity — DIDs, Credentials & WYRR Connect V2

WYRR identity is **self-sovereign and key-based**: every participant — person, agent, machine, vehicle, screen — is a DID, and every privileged action is a signature. There are no passwords.

> *WYRR requests · the user signs · the server verifies · the ledger records · the data stays with the user.*

Private keys never touch WYRR's servers.

## DID methods

| Method | Resolution | Used by |
|---|---|---|
| **`did:key`** | Self-certifying, resolved locally from the key material. Curves: **P-256, Ed25519, secp256k1** (base58btc multibase). | iOS devices (P-256 Secure Enclave), browsers (P-256 WebCrypto), nodes |
| **`did:wyrr`** | Resolved from the platform DID registry (anchored on-chain in `WYRRDIDRegistry`). | Issuer DIDs, pairwise DIDs, rotated keys |

`did:wyrr` sub-namespaces in production: `did:wyrr:machine:…` (Machine ID), `did:wyrr:business:{bizId}` (business credential issuers), `did:wyrr:vehicle:…` (vehicle agents, deterministically derived).

Signature formats verified: ECDSA-P256-SHA256 in DER (iOS), raw P1363 (WebCrypto), and Ed25519.

## WYRR Connect V2 — challenge → sign → verify → scoped session

```
Base URL: https://api.wyrr.me/api/v1/connect
```

### The flow

```
1. POST /challenge      → { nonce, … }            single-use, short-lived
2. sign the challenge with your DID's key
3. POST /verify         → { token, sid, scopes, modules, exp }
```

The result is a **scoped session JWT**:

| Claim | Meaning |
|---|---|
| `sub`, `did` | The authenticated identity |
| `sid` | Session id — the server-side revocation handle |
| `scopes[]`, `modules[]` | Exactly what this session may do |
| `aud: "wyrr-backend"` | Audience-bound; cannot be replayed against other services |
| `cnf: { kid }` | Bound to the key that authenticated |
| `authLevel` | `did-device` (full), `quick-connect` (read-mostly), `legacy-firebase` |
| `exp` | Default TTL **1 hour** |

Send it as `Authorization: Bearer <token>`. Revocation is immediate and platform-wide: `POST /disconnect` kills the `sid` everywhere, including in-flight tokens.

### Session management

```
GET  /sessions/me          introspect the current session
POST /sessions/derive      derive a narrower sub-session (fewer scopes; hand to a component)
POST /disconnect           revoke immediately
```

### Scopes & modules

Sessions are scoped to modules: `wallet, payments, swap, rides, delivery, food, gigs, land, missions, credentials, capital, escrow, maps, contracts, market, auction, assets, events, machine, identity`, and more. Value-bearing modules additionally carry escrow capabilities (`escrow:create`, `escrow:release`). A `did-device` session gets full module scopes; `quick-connect` sessions are read-mostly.

### Signed intents

Value-moving operations are **signed intents**, not bare API calls:

```
POST /intents/create        → server canonicalizes the intent (JCS) and returns bytes to sign
POST /intents/:id/submit    → signature verified against your DID → executed
POST /intents/:id/reject
GET  /intents/:id
```

Intent types span the platform: `payment.transfer`, `swap.execute`, `ride.book`, `escrow.create/release`, `asset.register/transfer`, `market.buy/sell`, `auction.bid`, `contract.sign`, `credential.revoke`, `did.rotate`, `session.grant`, and more. Every intent is replay-protected, expiry-bound, audited, and anchored.

### DID operations

```
GET  /did/resolve/:did     resolve any supported DID (public)
POST /did/register         register or link a DID document
POST /did/rotate           rotate keys (requires scope did:rotate)
```

## Verifiable credentials (W3C VC)

WYRR issues and verifies W3C Verifiable Credentials signed with Ed25519 by the platform issuer DID (`GET /credentials/issuer`), or by registered **business issuers** (`did:wyrr:business:…`) who can define templates and issue domain credentials.

```
POST /credentials/issue         issue a VC
POST /credentials/verify        verify a VC              (public)
POST /credentials/:id/revoke    revoke (scope credential:revoke)
POST /presentations/challenge   fresh challenge for a holder-bound presentation  (public)
POST /presentations/verify      verify a Verifiable Presentation                 (public)
```

- **Holder binding** — a presentation is valid only if the presenter signs a fresh challenge with the subject DID's key.
- **Selective disclosure** — SD-JWT-style: each claim is individually salted and hashed at issuance; the holder reveals only chosen `(salt, key, value)` triples. Verify at `POST /credential-status/sd-jwt/verify`.
- **Revocation** — StatusList2021-style status lists, served publicly at `GET /credential-status/status/:listId`. On-chain, only the keccak256 hash of a credential is anchored (`CredentialRegistry`) — **no PII ever goes on-chain**.

## Zero-knowledge trust proofs

Prove `trustScore ≥ T` without revealing the score:

```
Base: https://api.wyrr.me/api/v1/trust/zk

GET  /capabilities      offered thresholds: 200, 400, 500, 600, 700, 800, 900
POST /request           request a proof challenge
POST /prove             holder constructs the proof
POST /verify            anyone verifies — locally recomputed AND checked against
                        the on-chain TrustScoreAnchor Merkle epoch
```

The scheme is hash-chain based: the platform commits `C = SHA-256^(score+margin)(seed)` per identity per epoch into a Merkle root anchored on-chain; a proof of `score ≥ T` is the opening `W = SHA-256^(score+margin−T)(seed)`, which the verifier hashes `T` more times to reconstruct the commitment and Merkle path. A verified proof can also mint a `TrustThreshold` VC.

Additional public ZK verifiers: `POST /api/v1/zk/attribute/verify`, `/zk/predicate/verify`, `/zk/membership/verify`.

## Machine identity

Machines get the same identity machinery plus operational controls (geofence, battery policy, kill switch, fleets) — see [DEVICE_PROTOCOL.md](DEVICE_PROTOCOL.md). Vehicle DIDs are deterministically derived — see [VEHICLE_AGENT.md](VEHICLE_AGENT.md).

## Web SDK

For web apps integrating "Connect with WYRR":

```html
<script src="https://wyrr.me/sdk/wyrr-connect-sdk.js"></script>
```

```js
const app = await WYRRConnect.init({ manifest: '/wyrr-app.json' });
await app.connect({ mode: 'relay' });     // 'relay': phone signs via QR/deeplink
                                          // 'software': in-browser P-256 guest key
app.authHeader();                         // → { Authorization: 'Bearer …' }
app.wallet.holdings(); app.escrow.create(...); app.trust.score();
app.intent.create(...); app.intent.submit(...);
app.on('connected', s => …);
```

The manifest (`wyrr-app.json`) declares `{ id, name, modules[], scopes[], connectMode, guest }`. Recommended CSP: `default-src 'self' https://api.wyrr.me; frame-ancestors 'none'`.
