# 06 — Zones (L2)

## Overview

**Tempo Zones** are private L2 blockchains anchored to Tempo L1 with native support for:

- **Confidential balances** — Encrypted token balances
- **Confidential transactions** — Hidden transfer amounts
- **Redacted RPC** — Per-caller privacy (balances/nonce return 0 for non-self)
- **Compliance inheritance** — TIP-403 policies from L1

Zones are **not generic appchains** — they are privacy-preserving rollups with a lock-and-mint bridge to L1.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              TEMPO L1 CHAIN                      │
│                                                  │
│  ┌────────────────┐  ┌────────────────┐         │
│  │  ZoneFactory   │  │  ZonePortal    │         │
│  │  (0x5AF200...) │  │  (bridge)      │         │
│  └────────────────┘  └────────────────┘         │
└──────────────────────┬──────────────────────────┘
                       │
          ┌────────────┴────────────┐
          │   TEMPO ZONE (L2)       │
          │                         │
          │  ┌───────────────────┐  │
          │  │   Sequencer       │  │
          │  │   (1-8 nodes)     │  │
          │  │   Leader-follower │  │
          │  └───────────────────┘  │
          │                         │
          │  ┌───────────────────┐  │
          │  │   Confidential    │  │
          │  │   EVM State       │  │
          │  │   (encrypted)     │  │
          │  └───────────────────┘  │
          │                         │
          │  ┌───────────────────┐  │
          │  │   Prover          │  │
          │  │   (AWS Nitro)     │  │
          │  │   SPF verifier    │  │
          │  └───────────────────┘  │
          │                         │
          │  ┌───────────────────┐  │
          │  │   Checker         │  │
          │  │   (solvency ExEx) │  │
          │  └───────────────────┘  │
          └─────────────────────────┘
```

---

## L1 Relationship

### Lock-and-Mint Bridge

Zones use a **lock-and-mint** bridge via the `ZonePortal` contract on L1:

**Deposit (L1 → Zone):**
1. User calls `ZonePortal.deposit()` on L1
2. Tokens are locked in Portal escrow
3. Deposit is ECIES-encrypted (recipient/memo hidden)
4. Hash-chain deposit queue on Portal
5. Zone Sequencer detects deposit via L1 finality
6. ZoneInbox precompile processes deposit
7. Mint confidential tokens on Zone

**Withdrawal (Zone → L1):**
1. User calls `ZoneOutbox.withdraw()` on Zone
2. Confidential tokens are burned
3. Batch proof generated (SPF in Nitro Enclave)
4. Threshold certificate from sequencer quorum
5. Batch submitted to L1 ZonePortal
6. Portal releases escrowed tokens (FIFO order)

### Anchor Block

Zone genesis anchors to a **finalized Tempo block** before portal creation. This ensures:
- Deterministic zone initialization
- No race conditions with L1 state
- Domain-separated chain ID (prevents replay attacks)

---

## Sequencer Model

### Leader-Follower Topology

Zones use a **leader-follower** sequencer model with 1-8 sequencers:

| Role | Description |
|------|-------------|
| **Leader** | Produces blocks per Tempo anchor block |
| **Follower** | Replicates/validates leader's blocks |
| **RPC-only follower** | Serves public RPC without settlement keys |

### Settlement

- **Threshold signatures** required for batch submission to L1
- Followers sign settlement attestations
- Quorum of signatures forms threshold certificate

### Leader Rotation

- Leader rotates per Tempo anchor block
- Forced recovery mechanism for crashed leaders
- P2P networking via Commonware (authenticated static topology)

---

## Prover Infrastructure

### AWS Nitro Enclaves

Zone proofs are generated in **AWS Nitro Enclaves** — isolated compute environments with no external network access.

**Components:**

| Binary | Description |
|--------|-------------|
| `tempo-zone-prover-enclave` | Runs inside Nitro Enclave |
| `tempo-zone-prover-host` | Amazon Linux 2023 host for Nitro |
| `tempo-zone-prover-vsock-proxy` | Bridges TCP ↔ VSOCK for enclave communication |
| `tempo-zone-prover-eif-builder` | Builds Enclave Image File (EIF) |

### Prover Protocol

External service communicating via **JSON over TCP/VSOCK**:

```
Client → TCP:5000 → VSOCK Proxy → Enclave
                                    ↓
                              SPF Verifier
                              (prove_zone_batch)
                                    ↓
                              BatchOutput
                              (commitments)
```

**Protocol v1:**
- `VerifyRequest` — Batch witness (MPT trie proofs)
- `VerifyResponse` — Batch output (commitments)

### Stateless Proof Function (SPF)

The SPF (`zone-spf` crate) is the **stateless state transition function**:

```rust
fn prove_zone_batch(witness: BatchWitness) -> BatchOutput
```

**BatchWitness contains:**
- MPT trie proofs for Zone state
- MPT trie proofs for Tempo L1 state
- Batch transactions

**SPF execution:**
1. Verify MPT proofs against state roots
2. Replay batch transactions
3. Compute new state root
4. Return BatchOutput with commitments

**Future:** Designed for migration to SP1/RISC-V (zero-knowledge proofs).

---

## Bridge Mechanics

### Deposit Queue

- **Hash-chain** deposit queue on Portal
- Each deposit references previous deposit hash
- Prevents reordering attacks

### Withdrawal Processing

- **LIFO** on ZoneOutbox (Zone processes last-in-first-out)
- **FIFO** on Portal (L1 processes first-in-first-out)
- Mismatch intentional: prevents griefing

### Bounce-Back Mechanism

Failed deposits/withdrawals trigger bounce-back:
- Deposit fails on Zone → tokens returned to user on L1
- Withdrawal fails on L1 → tokens returned to user on Zone

### Token Enablement

- **Append-only** — Once a token is enabled on a Zone, it cannot be disabled
- Prevents rug pulls via token deactivation

---

## Zone-Specific Precompiles

Zones adapt L1 precompiles for privacy and L2 context:

| Precompile | Address | Description |
|------------|---------|-------------|
| `TempoState` | — | Read L1 storage (TIP-403, etc.) |
| `ZoneInbox` | `0x1c00...0000` | Process deposits, mint confidential tokens |
| `ZoneOutbox` | `0x1c00...0002` | Process withdrawals, burn tokens |
| `ZoneFeeManager` | — | Replaces L1 fee manager (zone-specific fees) |
| `ChaumPedersenVerify` | — | ECDH proof verification (confidential transfers) |
| `AesGcmDecrypt` | — | Decrypt encrypted deposits |

### Adapted L1 Precompiles

| L1 Precompile | Zone Adaptation |
|---------------|-----------------|
| TIP-20 Token | Privacy extensions (encrypted balances) |
| TIP-403 Registry | Read-only against L1 state |
| NonceManager | Account-scoped reads (zone-local) |
| AccountKeychain | Account-scoped reads (zone-local) |

---

## Redacted RPC

Zone RPC is **authenticated and redacted** for privacy:

### Authentication

- Signed auth tokens (per-caller identity)
- Token specifies caller's address

### Redaction Rules

| Query | Self | Other |
|-------|------|-------|
| Balance | Actual balance | 0 |
| Nonce | Actual nonce | 0 |
| Block transactions | Full transactions | Empty array |
| Logs bloom | Actual bloom | Zeroed |
| Log queries | Scoped to caller | Empty |

### Non-Sequencer Nodes

Non-sequencer nodes (RPC-only followers):
- Serve redacted RPC
- Do not hold settlement keys
- Cannot produce blocks

---

## Checker (Solvency Verification)

The `zone-checker` crate is an **observe-only solvency verification ExEx**:

### Ghost Accounting

Maintains independent model of:
- Zone balances (decrypted via view keys)
- Token supply (minted vs. burned)
- Portal collateral (L1 escrow)

### Verification

- Compares on-chain state against independent model
- Detects insolvency or misbehavior
- Does not affect consensus (observe-only)

---

## Zone Genesis

Zone genesis is generated by `zone-xtask`:

1. Anchor to finalized Tempo block
2. Domain-separate chain ID from parent
3. Deploy system predeploys:
   - pathUSD (`0x20c0...`)
   - ZoneInbox (`0x1c00...0000`)
   - ZoneOutbox (`0x1c00...0002`)
   - TIP403Registry
   - NonceManager
   - ZoneFeeManager
   - Multicall3
   - UniversalRouter
   - Create2Factory
4. Configure fee token (pathUSD, 6 decimals)
5. Initialize storage for fee configuration

### Pre-Built Genesis

A dev zone genesis (chain ID 1337) is available at:
```
zones/zone-genesis/genesis.json
```

---

## Zone Deployment Workflow

### 1. Create Zone on L1

```bash
just create-zone \
  --zone-name "MyZone" \
  --owner 0xYourAddress \
  --sequencers 0xSeq1,0xSeq2,0xSeq3
```

This calls `ZoneFactory.createZone()` on L1.

### 2. Set Encryption Key

```bash
just set-encryption-key \
  --zone 0xZoneAddress \
  --key 0xEncryptionKey
```

Required for confidential transactions.

### 3. Generate Zone Genesis

```bash
just generate-zone-genesis \
  --zone 0xZoneAddress \
  --output genesis.json
```

Anchors to current finalized L1 block.

### 4. Run Zone Sequencer

```bash
tempo-zone \
  --chain genesis.json \
  --sequencer \
  --l1-rpc ws://127.0.0.1:8546 \
  --prover tcp:127.0.0.1:5000
```

### 5. Run Zone Followers

```bash
tempo-zone \
  --chain genesis.json \
  --follower \
  --l1-rpc ws://127.0.0.1:8546
```

---

## P2P Networking

Zone P2P uses **Commonware-based authenticated static networking**:

- **Manifest-defined topology** — Peers specified in configuration
- **Block backfill** — Sync missing blocks from peers
- **Transaction forwarding** — Propagate transactions to leader
- **Forced recovery** — Recover from crashed leaders

---

## Key Constraints

1. **T10 Portal ABI** — Zone sequencer requires L1 T10 runtime; T11/T12 breaks compatibility
2. **AWS Nitro only** — Prover requires Nitro Enclaves (x86_64 only)
3. **1-8 sequencers** — Maximum 8 sequencers per zone
4. **Append-only tokens** — Cannot disable tokens once enabled
5. **Redacted RPC** — Non-sequencer nodes serve redacted data
6. **Domain-separated chain ID** — Prevents replay attacks between zones
