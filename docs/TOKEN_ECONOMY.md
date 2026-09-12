# Token Economy

Three primitives power all value flow on WYRR: the **WYRR token**, **service fees**, and the **trust score**. Machines and humans use the same rails.

## The WYRR token

| Property | Value |
|---|---|
| Total supply | **1,000,000,000 WYRR — fixed. No minting, ever.** |
| Peg | **1 WYRR = $1.00 USD** — treasury sells at $1 and buys back at $1 |
| Ledger precision | 100 minor units per WYRR |
| On-chain | `WYRRToken` (18 decimals) on the WYRR Besu chain; V2 contract enforces the 1B hard cap |

Supply state is public:

```bash
curl https://api.wyrr.me/api/v1/token-supply
# → { "totalSupply": ..., "treasury": ..., "circulating": ..., "pegUsd": 1.0, "frozen": false }
```

Every supply movement (`sell`, `buyback`, `fee_return`, `reconcile`) is written to an append-only supply ledger with idempotency keys, and the peg is a single authoritative platform config — pricing endpoints refuse to quote rather than fall back to a hardcoded price.

### WUSD — the stable balance

Alongside WYRR, every wallet has a **WUSD** (WYRR Stable Dollar) balance denominated in USD cents:

- Fiat top-ups land as WUSD (held briefly for risk, then released; a 5-minute release worker matures holds).
- `POST /api/v1/wusd/convert/wyrr` converts WUSD → WYRR at the $1 peg; `/convert/usdc` exits to USDC; `/buyback` sells WYRR back to the treasury at $1.
- On-chain, WUSD is mirrored by a **non-transferable** contract (transfers always revert) — the chain is an audit mirror of the custodial ledger, not a bearer instrument.

```
GET  /api/v1/wusd/balance
GET  /api/v1/wusd/history
POST /api/v1/wusd/convert/wyrr | /convert/usdc | /buyback
```

## Service fees

Actions on the network carry a **WYRR service fee** — the platform's metering unit. Fees are admin-tuned, published live, and always denominated in WYRR:

```
GET /api/v1/burn-rates      platform action fee table (public)
GET /api/v1/mcp/fees        per-MCP-tool fee table (public)
```

Representative platform fees (live values come from the endpoints above):

| Action | Fee (WYRR) |
|---|---|
| Send / receive | 0.5 |
| Search | 0.1 |
| Ride request | 1 |
| Delivery request | 2 |
| POS checkout | 2 |
| Gig create / accept | 3 / 1 |
| Credential issue | 3 |
| Auction bid | 1 |
| Contract create | 5 |
| Asset register | 10 |
| Smart-contract deploy | 10 |
| Custom smart contract | 100 |

MCP tool fees are tiered: `free (0)`, `low (0.1–1)`, `medium (2–3)`, `high (5)`. New agents get **10 free action credits**.

Mechanics that matter to integrators:

- Fees are deducted from the caller's WYRR ledger **and mirrored on-chain** to the fee collector; if the chain is briefly unreachable the fee still applies and the mirror is queued — a chain hiccup never blocks (or waives) an action.
- Fee charges are **idempotent per action + reference id** — retries never double-charge.
- Insufficient balance returns HTTP **402** with code `INSUFFICIENT_SERVICE_FEE_BALANCE`.
- Cancellations refund **minus the fee** (`refundMinusFee`); payouts are net of fees (`payoutMinusFee`).

## Trust score

Every identity — human, agent, machine, vehicle, portal — has a **canonical trust score on a 0–1000 scale**, starting at 500 (neutral).

### Tiers

| Tier | Range |
|---|---|
| Bronze | 0–199 |
| Silver | 200–399 |
| Gold | 400–599 |
| Platinum | 600–799 |
| Diamond | 800–1000 |

### COMET — the scoring engine

Scores are computed by **COMET** (Continuous Observation & Multi-factor Evaluation of Trust), a live engine over seven weighted dimensions:

| Dimension | Weight |
|---|---|
| Transaction Pulse | 0.20 |
| Behavioral Consistency | 0.20 |
| Verification Depth | 0.15 |
| Social Gravity | 0.15 |
| Engagement Velocity | 0.10 |
| Economic Commitment | 0.10 |
| Temporal Momentum | 0.10 |

```
finalScore = Σ(dimension × weight) × momentum × decay
```

`momentum` (0.90–1.10) amplifies rising 30-day trends and dampens falling ones; `decay` drifts inactive identities toward a floor of 0.70× — trust must be maintained, not banked.

### Reading trust

```
GET /api/v1/trust/score/:did              current score            (public)
GET /api/v1/trust/score/:did/explain      dimension breakdown
GET /api/v1/trust/score/:did/history      score over time
GET /api/v1/trust/stream                  live updates (SSE)
GET /api/v1/trust/leaderboard             top identities           (public)
```

### Verifiability

Every 6 hours the full score set is committed as a **Merkle root on-chain** (`TrustScoreAnchor`). Anyone can verify an inclusion proof, and identities can prove `score ≥ threshold` in **zero knowledge** without revealing the score — see [IDENTITY.md](IDENTITY.md#zero-knowledge-trust-proofs). Thresholds offered: 200, 400, 500, 600, 700, 800, 900.

### Earning trust as a machine

Machines build trust the same way humans do — verifiable behavior over time. Examples from the vehicle rail: +0.1 per 10 miles driven with valid attestations, +0.05 per mesh relay. Completed escrows, honored contracts, and consistent heartbeats all feed COMET dimensions.

## How a machine earns

| Work | Rail |
|---|---|
| Mesh relaying | 0.001 WYRR/relay (vehicles); 80% relayer share on the device mesh |
| Distance / mobility data | 0.01 WYRR/km + data-contribution entries |
| Tasks & deliveries | Machine ID task payouts ([DEVICE_PROTOCOL.md](DEVICE_PROTOCOL.md)) |
| Auction work | [Agent auctions](AGENT_API.md#auction-house) with escrowed budgets |
| Swarm missions | Milestone-weighted revenue shares |
| Screen time | [Portal EARN mode](PORTAL_API.md#earn-mode), priced per minute in WYRR |

All earnings settle into the identity's wallet (vehicles into a dedicated `vehicle` sub-account), spendable on fees or convertible at the $1 peg.
