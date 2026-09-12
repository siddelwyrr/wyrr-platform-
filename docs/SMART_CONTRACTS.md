# Smart Contracts

WYRR's on-chain layer runs on a **permissioned Hyperledger Besu network** (chain ID `1337`, free gas, BFT consensus). The chain anchors what must be tamper-evident — identities, credentials, trust epochs, escrows, agent-economy state — while high-frequency state lives off-chain with content hashes committed on-chain.

> **Access model:** the chain's RPC is not publicly exposed. External integrators interact through the platform APIs (`https://api.wyrr.me`), which sign and submit on the platform's behalf; every fee-carrying action is anchored automatically. Contract addresses below are published for verification and event-level audit by permissioned node operators.

Contract sources live in `backend/contracts/` (V1), `backend/contracts/v2/` (proxy-compatible V2), and `backend/contracts/proxy/`. All contracts are dependency-free Solidity `^0.8.19`. Compiled ABIs are in `backend/contracts/artifacts/<Name>.json`.

## Deployed contracts (chain ID 1337)

### Platform core

| Contract | Address | Purpose |
|---|---|---|
| **WYRRToken** | `0xbBCeB59101A8399DB5e1dB03323BE7b00fEEF004` | Platform-governed ERC-20 (`WYRR`, 18 decimals). Platform-only mint with an audit `reason`; self-burn; optional per-address KYC flags. |
| **WYRRBesuEscrow** | `0x7eF84473a4E772fB6aDfA1B0C6728A3dbf268Dd7` | Custodial escrow with disputes. States: `Created → Funded → Released \| Refunded \| Disputed → Resolved`. Auto-release by anyone after `releaseAfter`. |
| **LendingPool** | `0x619A83c9368aDa9fFb98c3F14b662724dD19E943` | Collateralized lending pool. |
| **WYRRGovernance** | `0x6aA8b700cD034Ab4B897B59447f268b33B8cF699` | Token-weighted proposals & voting. Power = balance + delegated power (snapshot at delegation). States: `Pending, Active, Passed, Rejected, Executed, Expired`. |

### Identity & trust

| Contract | Address | Purpose |
|---|---|---|
| **WYRRDIDRegistry** | `0x51D4903ef5F871273e5B4172898B18809CFd7881` | ERC-1056-style registry for the `did:wyrr` method: ownership, expiring delegates, attribute events with a changed-block pointer chain. |
| **CredentialRegistry** | `0xF216B6b2D9E76F94f97bE597e2Cec81730520585` | Anchors keccak256 hashes of verifiable credentials + one-way revocation. **No PII on-chain — hashes only.** |
| **TrustScoreAnchor** | `0xa2b80D63b1f72a4D26dfc33D62EbE80148Ddd326` | Per-epoch Merkle-root commitments of the trust-score set (anchored every 6 h). `verifyScore(epochId, leaf, proof)` is a public view — the basis of [ZK trust proofs](IDENTITY.md#zero-knowledge-trust-proofs). |
| **SoulboundTrustBadge** | `0x0F095aeA9540468B19829d02cC811Ebe5173D615` | Non-transferable (ERC-5192) trust badges. No transfer functions exist; issuer-only one-way revocation. |
| **CredentialMarketplace** | `0xe52155361a36C7d445F2c6784B14Bf7A3C306e15` | Credential request market: requester posts `schemaId` + spec hash; an authorized issuer fulfills with the credential hash. |

### Agent economy

| Contract | Address | Purpose |
|---|---|---|
| **AgentRegistry** | `0x47b33c2D3e928FDf2c0A82FcD7042Ae0cFd5862A` | On-chain identity anchor for every agent/machine registered via the [MCP server](MCP_SERVER.md). Keyed by `keccak256(did)`. Trust/earnings figures are anchors; the platform ledger is source of truth. |
| **AgentPartnership** | `0xed78Cb21Ce10A086a7973fB44e96d34F31D45cF1` | Agent↔agent revenue-sharing. `Proposed → Active → (Settled) → Ended \| Expired`. With a token address set it routes real WYRRToken to each partner's payout address; with token = 0 it anchors the split math as events. |
| **AuctionEscrow** | `0x218d5fe2E168656eBDE49e7a4A3C97E699D0be78` | Escrow for the agent auction house. Budget locks at posting. `Open → Awarded → Completed \| Disputed → Resolved`, or `Open → Cancelled`. |
| **SwarmCoordination** | `0x3F0BE59Bc74c7368Aa049a0B064ce9Dc32890669` | Swarm missions: leader escrows budget, members join with role + fixed share, milestone-weighted settlement. `Recruiting → Active → Settled \| Aborted`. |

### World & assets

| Contract | Address | Purpose |
|---|---|---|
| **WYRRLDRegistry** | `0xEEE98917D56774d2F1FfAfbEA2e9b04Ce8ef7a11` | WYRRLD land/property registry backing the Global Ownership Network. |

### Compiled, pending deployment

These are in the repo with final ABIs but not yet live — treat as roadmap:

| Contract | Purpose |
|---|---|
| **WYRRStableDollar (WUSD)** | Non-transferable on-chain mirror of the platform's USD ledger. `transfer`/`approve` always revert; platform-only mint/burn, each tagged to a deposit or conversion id. |
| **WYRRFractionalOwnership** | Dependency-free ERC-1155 for fractional ownership slots (1 unit = 1 slot; holders transfer freely; platform is sole minter). |
| **ProvenanceRegistry** | Append-only asset-provenance ledger. All references are keccak256 hashes (asset, actor DIDs, metadata). |
| **ValidatorPermissioning** | On-chain QBFT validator & node permissioning — implements `getValidators()` for Besu's contract validator-selection mode. |
| **V2 suite** (`WYRRTokenV2`, `CredentialRegistryV2`, `TrustScoreAnchorV2` behind `WYRRTransparentProxy` + `WYRRProxyAdmin`) | Gas-optimized, upgradeable (EIP-1967 transparent proxy) versions for fresh deployments. `WYRRTokenV2` enforces a hard cap: `MAX_SUPPLY = 1,000,000,000 WYRR`. |

### Public EVM (Polygon)

A separate, minimal footprint exists on Polygon:

| Contract | Address |
|---|---|
| WYRREscrow | `0x05CdF1d3D4214b1Fc1a0749bF65DDf45D3c2D0b8` |
| WYRRToken | `0x674B6c8a6fBa3cbA8c652FC7098A3fCa27444b8c` |
| WYRRTokenV2 | `0x8a9085FebCe46992A90f1cB8e296A5c36667e536` |

## How to interact

### Through the platform (recommended)

Every MCP tool call and agent-economy action anchors on-chain automatically. Specific surfaces:

- `GET /api/v1/chain/health` and `GET /api/v1/chain/anchors/stats` — public chain observability
- `GET /api/v1/explorer` — public block explorer API
- `POST /api/v1/trust/zk/verify` — verify a trust proof against the on-chain `TrustScoreAnchor`
- Escrow, credential, and governance operations via their respective REST/MCP surfaces

### ABIs & events

Grab the ABI for any contract from `backend/contracts/artifacts/<Name>.json`. Compile from source with `node contracts/compile.js` (V1) or `compile-v2.js` (V2/proxy). Event streams are the intended audit interface: `CredentialRegistry`, `TrustScoreAnchor`, `AgentRegistry`, and `ProvenanceRegistry` are all designed so that replaying their events reconstructs the registry state.

### Anchoring semantics

Three patterns repeat across the platform:

1. **Content-hash anchoring** — off-chain records (MCP actions, agent-economy records, credentials) commit `keccak256`/`sha256` hashes on-chain. The chain proves existence and integrity; the platform serves the data.
2. **Merkle-epoch anchoring** — the full trust-score set is committed as a sorted-pair Merkle root every 6 hours (leaf = `keccak256(abi.encode(subject, score, nonce))`), enabling per-user inclusion proofs without publishing scores.
3. **Escrowed settlement** — value between untrusted parties locks in a state-machine escrow contract and releases on completion, dispute resolution, or timeout.

## Consensus & topology

The network runs BFT consensus (IBFT 2.0, migrating to QBFT with on-chain validator permissioning) with an expansion roadmap from the current validator set toward a geographically distributed, partner-operated 20+ node consortium. Node-operator onboarding is permissioned; consensus and peer ports are never internet-exposed, and no third-party node ever holds platform keys.
