# 03 — Consensus & Networking

## Overview

Tempo uses **Simplex Consensus via Commonware** — a threshold BFT consensus protocol. This is not a PoS/PoA system built on top of Ethereum; Tempo has its own independent consensus layer.

---

## Consensus Mechanism: Simplex BFT

### Architecture

The consensus engine (`tempo-consensus` crate) is modeled after Commonware's `alto` toy blockchain and uses an **actor-based architecture** with the following components:

| Component | Role |
|-----------|------|
| **Application** | State machine (Tempo blockchain) |
| **DKG Manager** | Distributed Key Generation lifecycle |
| **Epoch Manager** | Epoch transitions, validator rotation |
| **Execution** | Block execution and validation |
| **Peer Manager** | P2P connection management |
| **Storage** | Persistent consensus state |
| **Subblocks** | Partial blocks built by validators within an epoch |

### Cryptography

| Algorithm | Purpose |
|-----------|---------|
| **Ed25519** | P2P signing keys (validator identity) |
| **BLS12-381** | Threshold signing shares (collective block certification) |
| **Feldman-Desmedt DKG** | Distributed key generation for threshold signatures |

### Operational Modes

Tempo nodes operate in one of two modes:

#### 1. Consensus Stack (`run_consensus_stack`)
Full validator participating in consensus:
- Holds Ed25519 signing key for P2P identity
- Holds BLS12-381 signing share for threshold signatures
- Participates in voting, certificate formation, and block proposal
- Connects to commonware's authenticated lookup network

#### 2. Follow Stack (`run_follow_stack`)
Non-validating node:
- Syncs via RPC from an upstream validator
- Does not hold signing keys
- Serves as a public RPC endpoint
- Does not participate in consensus

### Epochs & Validator Rotation

- **Epoch length**: Configurable, default 302,400 blocks
- **Epoch mechanism**: `FixedEpocher` from commonware
- **DKG rotation**: Via `SchemeProvider` — each epoch can have a new threshold key set
- **Validator set changes**: Applied at epoch boundaries

### P2P Channels

The consensus layer uses multiple P2P channels, each with configurable rate limits:

| Channel | Purpose |
|---------|---------|
| `votes` | Validator vote messages |
| `certificates` | Threshold certificate messages |
| `resolver` | State/data resolution requests |
| `broadcaster` | Block/transaction broadcasting |
| `marshal` | Message marshaling/unmarshaling |
| `dkg` | DKG protocol messages |
| `subblocks` | Sub-block propagation |

### Subblocks

The codebase includes infrastructure for **sub-blocks** — partial blocks built by validators within an epoch. However, `with_subblocks: false` is currently the default, meaning subblocks are not actively used in production.

---

## Validator Set Management

### Validator Registration

Validators are configured at genesis via the `ValidatorConfigV2` precompile (address `0xCCCCCCCC...01`, activates at T2).

Each validator has:
- **Ed25519 signing key** — P2P consensus identity
- **BLS12-381 signing share** — Threshold signature participation
- **On-chain Ethereum address** — Fee recipient, governance
- **Network endpoints** — Ingress/egress addresses

### Registration Requirements

Validator registration requires an **Ed25519 signature** over a message containing:

```
chainId || contractAddr || validatorAddr || ingress || egress || feeRecipient
```

This proves ownership of the Ed25519 key and binds it to the on-chain identity.

### Active Committee

The `CurrentCommittee` precompile (activates at T8) exposes the active validator set. It can be queried by any smart contract to determine the current consensus participants.

---

## Network Topology (Production)

### ThaiFi Private Chain (Chain ID 17)

Deployed on `192.168.1.199`:

```
┌─────────────────────────────────────────────────────┐
│  192.168.1.199                                       │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │Validator │  │Validator │  │Validator │          │
│  │    #1    │  │    #2    │  │    #3    │          │
│  │ :8545    │  │ :8547    │  │ :8549    │          │
│  │ :30303   │  │ :30304   │  │ :30305   │          │
│  │ :3000    │  │ :3001    │  │ :3002    │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                       │
│  ┌──────────┐                                        │
│  │ RPC Node │  (non-validating follower)             │
│  │ :28545   │  follows ws://127.0.0.1:8546          │
│  └──────────┘                                        │
└─────────────────────────────────────────────────────┘
```

### Port Allocation

| Node | HTTP RPC | WS RPC | P2P | Consensus |
|------|----------|--------|-----|-----------|
| Validator #1 | 8545 | 8546 | 30303 | 3000 |
| Validator #2 | 8547 | 8548 | 30304 | 3001 |
| Validator #3 | 8549 | 8550 | 30305 | 3002 |
| RPC Follower | 28545 | — | — | — |

All services run via Docker Compose with `network_mode: host`.

---

## Commonware Integration

Tempo's consensus is deeply integrated with Commonware (v2026.7.1):

| Commonware Crate | Usage |
|-----------------|-------|
| `commonware-consensus` | Simplex consensus protocol |
| `commonware-cryptography` | Ed25519, BLS12-381, DKG (Feldman-Desmedt) |
| `commonware-runtime` | Actor runtime, async execution |
| `commonware-p2p` | Authenticated peer-to-peer networking |
| `commonware-storage` | Persistent consensus state |

The dependency is specified in `Cargo.toml`:
```toml
commonware-consensus = { version = "2026.7.1", ... }
```

---

## DKG (Distributed Key Generation)

### Scheme

Tempo uses the **Feldman-Desmedt** DKG scheme from Commonware:
- Each validator holds a **signing share** (BLS12-381)
- The collective threshold signature requires a quorum of shares
- DKG is performed at epoch boundaries for key rotation

### On-Chain Artifacts

The `tempo-dkg-onchain-artifacts` crate provides:
- DKG output serialization
- On-chain verification of DKG participation
- Genesis header `extra_data` encoding of DKG outcome

### Genesis DKG

At genesis, the DKG outcome is written into the genesis header's `extra_data` field. This allows all nodes to verify the initial validator set's threshold keys.

---

## Consensus Configuration

Key parameters (from `tempo-consensus-config`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `epoch_length` | 302,400 blocks | Blocks per epoch |
| `signing_share` | BLS12-381 | Validator's threshold signing share |
| `ed25519_key` | Ed25519 | Validator's P2P identity key |
| `p2p_channels` | 7 | Number of P2P channels (votes, certs, etc.) |
| `rate_limits` | per-channel | Configurable message rate limits |

---

## Key Constraints

1. **T10 Portal ABI** — Zone sequencer requires T10 runtime. Activating T11/T12 breaks compatibility.
2. **Epoch-boundary validator changes** — Validator set changes only take effect at epoch boundaries.
3. **DKG participation required** — Validators must participate in DKG to receive signing shares.
4. **Subblocks disabled** — `with_subblocks: false` in production; infrastructure exists but is not active.
