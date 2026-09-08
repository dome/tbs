# 02 — System Architecture

## High-Level Architecture

Tempo is a two-layer blockchain system:

- **Layer 1 (Tempo Core)** — A full L1 blockchain with its own consensus (not dependent on Ethereum for security), built on the Reth SDK
- **Layer 2 (Tempo Zones)** — Privacy-preserving rollups anchored to L1 via a lock-and-mint bridge

---

## L1 Crate Map

The L1 workspace (`tempo/`) contains 22 crates organized as follows:

### Binaries

| Crate | Path | Description |
|-------|------|-------------|
| `tempo` | `bin/tempo/` | Main L1 node binary |
| `tempo-sidecar` | `bin/tempo-sidecar/` | Sidecar process (auxiliary services) |
| `tempo-xtask` | `xtask/` | Genesis generation & build tooling |

### Core Crates

| Crate | Path | Description |
|-------|------|-------------|
| `tempo-primitives` | `crates/primitives/` | Core types: `Block`, `TempoHeader`, `TempoTxEnvelope`, `TempoSignature`, `SubBlock` |
| `tempo-chainspec` | `crates/chainspec/` | Chain specification, hardfork activation timestamps |
| `tempo-hardfork` | `crates/hardfork/` | `TempoHardfork` enum (Genesis → T12), 16 variants |
| `tempo-consensus` | `crates/consensus/` | Simplex BFT engine via Commonware, actor-based architecture |
| `tempo-consensus-config` | `crates/consensus-config/` | Consensus configuration types, `SigningShare` (BLS12-381) |
| `tempo-validator-config` | `crates/validator-config/` | Validator set management, registration, rotation |

### EVM & Execution

| Crate | Path | Description |
|-------|------|-------------|
| `tempo-evm` | `crates/evm/` | `TempoEvmConfig`, `TempoBlockAssembler`, `TempoBlockExecutor`, pool validation |
| `tempo-revm` | `crates/revm/` | `TempoEvm`, `TempoFeeManager`, `FeeTokenResolver`, gas parameters, storage credits |
| `tempo-precompiles` | `crates/precompiles/` | 16+ protocol-level precompiles at fixed addresses |
| `tempo-precompiles-macros` | `crates/precompiles-macros/` | Proc macros for precompile definitions (`tempo_precompile!`) |
| `tempo-contracts` | `crates/contracts/` | Predeployed Ethereum-compatible contracts (Multicall3, CreateX, Permit2) |

### Transaction & Payload

| Crate | Path | Description |
|-------|------|-------------|
| `tempo-transaction-pool` | `crates/transaction-pool/` | `TempoTransactionPool`, `AA2dPool` (2D nonce support), address filtering |
| `tempo-payload-builder` | `crates/payload/builder/` | Block/payload construction |
| `tempo-payload-types` | `crates/payload/types/` | Payload type definitions |

### Node & Infrastructure

| Crate | Path | Description |
|-------|------|-------------|
| `tempo-node` | `crates/node/` | `TempoFullNode`, network builder, RPC, telemetry, engine, gossip |
| `tempo-faucet` | `crates/faucet/` | Faucet service for development networks |
| `tempo-telemetry-util` | `crates/telemetry-util/` | Telemetry/observability utilities |
| `tempo-ext` | `crates/ext/` | Extension traits and utilities |
| `tempo-eyre` | `crates/eyre/` | Error handling (eyre wrapper) |
| `tempo-alloy` | `crates/alloy/` | Alloy integration adapters |
| `tempo-dkg-onchain-artifacts` | `crates/dkg-onchain-artifacts/` | DKG (Distributed Key Generation) on-chain artifacts |
| `tempo-e2e` | `crates/e2e/` | End-to-end test infrastructure |

---

## L2 Crate Map

The L2 workspace (`zones/`) contains 15 crates + 3 prover binaries:

### Binaries

| Crate | Path | Description |
|-------|------|-------------|
| `tempo-zone` | `bin/tempo-zone/` | Zone node binary (L2 sequencer/follower) |
| `tempo-zone-prover-enclave` | `bin/prover/enclave/` | AWS Nitro Enclave prover binary |
| `tempo-zone-prover-utils` | `bin/prover/utils/` | Prover utility tools |
| `tempo-zone-prover-vsock-proxy` | `bin/prover/vsock-proxy/` | VSOCK ↔ TCP bridge for enclave communication |
| `tempo-zone-xtask` | `xtask/` | Zone provisioning & genesis tooling |

### Core Crates

| Crate | Path | Description |
|-------|------|-------------|
| `zone-primitives` | `crates/primitives/` | Zone-specific core types |
| `zone-chainspec` | `crates/chainspec/` | Zone chain specification, domain-separated from parent |
| `zone-hardfork` | `crates/hardfork/` | Zone hardfork definitions |
| `zone-evm` | `crates/evm/` | Zone EVM with privacy extensions |
| `zone-precompiles` | `crates/precompiles/` | Zone-specific precompiles (Inbox, Outbox, FeeManager, TempoState, etc.) |
| `zone-contracts` | `crates/contracts/` | Zone predeployed contracts |

### Sequencer & Bridge

| Crate | Path | Description |
|-------|------|-------------|
| `zone-sequencer` | `crates/sequencer/` | Leader-follower sequencer model (1-8 sequencers) |
| `zone-l1` | `crates/l1/` | L1 interaction: portal bridge, state reads, finality tracking |
| `zone-p2p` | `crates/p2p/` | Commonware-based authenticated static P2P networking |
| `zone-payload` | `crates/payload/` | Zone payload/block building |

### Prover & Verification

| Crate | Path | Description |
|-------|------|-------------|
| `zone-prover` | `crates/prover/` | Prover client (JSON over TCP/VSOCK, protocol v1) |
| `zone-spf` | `crates/spf/` | Stateless Proof Function — `prove_zone_batch()` with MPT trie proofs |
| `zone-checker` | `crates/checker/` | Observe-only solvency verification ExEx (ghost accounting) |

### RPC & Node

| Crate | Path | Description |
|-------|------|-------------|
| `zone-rpc` | `crates/rpc/` | Redacted JSON-RPC with per-caller privacy, signed auth tokens |
| `zone-node` | `crates/node/` | Zone node configuration and lifecycle |

---

## Dependency Graph

```
┌──────────────────────────────────────────────────────┐
│                   TEMPO L1 WORKSPACE                  │
│                                                       │
│  tempo-node ──┬── tempo-evm ──┬── tempo-revm         │
│               │               └── tempo-precompiles   │
│               ├── tempo-consensus ── commonware-*     │
│               ├── tempo-transaction-pool               │
│               ├── tempo-payload-{builder,types}       │
│               └── tempo-chainspec ── tempo-hardfork   │
│                                                       │
│  tempo-primitives (core types, used by all)           │
│  tempo-contracts (predeployed bytecodes)              │
└───────────────────────┬──────────────────────────────┘
                        │ git dependency (pinned rev)
┌───────────────────────┴──────────────────────────────┐
│                   ZONES L2 WORKSPACE                  │
│                                                       │
│  zone-node ──┬── zone-sequencer                       │
│              ├── zone-l1 ── (L1 portal interaction)   │
│              ├── zone-evm ── zone-precompiles         │
│              ├── zone-prover ── zone-spf              │
│              ├── zone-rpc (redacted, authenticated)   │
│              └── zone-checker (solvency ExEx)         │
│                                                       │
│  zone-primitives (zone core types)                    │
│  zone-chainspec (domain-separated from L1)            │
└──────────────────────────────────────────────────────┘
```

---

## Data Flow: Transaction Lifecycle

### L1 Transaction

```
User → JSON-RPC → Tx Pool (2D nonce validation)
     → Payload Builder → Block Assembly
     → EVM Execution (TempoBlockExecutor)
       → Fee Resolution (TempoFeeManager → pathUSD)
       → Precompile Dispatch (if applicable)
       → State Transition
     → Consensus (Simplex BFT via Commonware)
       → Validator Voting → Threshold Signature
     → Block Finalization → State Commitment
```

### L2 Deposit (L1 → Zone)

```
User → ZonePortal (L1) → ECIES-encrypted deposit
     → Hash-chain deposit queue on Portal
     → Zone Sequencer detects deposit via L1 finality
     → ZoneInbox precompile processes deposit
     → Mint confidential tokens on Zone
```

### L2 Withdrawal (Zone → L1)

```
User → ZoneOutbox (Zone) → Burn confidential tokens
     → Batch proof generated (SPF in Nitro Enclave)
     → Threshold certificate from sequencer quorum
     → Batch submitted to L1 ZonePortal
     → Portal releases escrowed tokens (FIFO order)
```

---

## Key Design Decisions

1. **Reth SDK Foundation** — Rather than forking geth or building from scratch, Tempo extends Reth (Paradigm's Rust execution client), inheriting EVM compatibility while customizing consensus, fees, and precompiles.

2. **Commonware Consensus** — Simplex BFT provides sub-second finality without relying on Ethereum's consensus. Validators use BLS12-381 threshold signatures for collective block certification.

3. **Stablecoin-First Economics** — Gas is priced and paid in USD stablecoins. The Fee AMM (StablecoinDEX precompile) handles conversion to validator-preferred tokens.

4. **Progressive Hardfork Schedule** — 16 hardforks (Genesis → T12) activate features incrementally via timestamps, allowing smooth upgrades.

5. **Privacy via TEE** — Zone confidentiality relies on AWS Nitro Enclaves for proof generation, with the Stateless Proof Function (SPF) designed for future migration to SP1/RISC-V.

6. **Composability via Precompiles** — Protocol-level functionality is exposed as precompiles at deterministic addresses, making it accessible to any EVM smart contract.
