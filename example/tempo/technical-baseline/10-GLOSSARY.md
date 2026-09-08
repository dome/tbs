# 10 — Glossary & TIP Index

## Terminology

### Core Concepts

| Term | Definition |
|------|------------|
| **Tempo** | Layer 1 blockchain purpose-built for stablecoin payments |
| **Zone** | Private L2 blockchain anchored to Tempo L1 with confidential balances |
| **L1** | Layer 1 — the main Tempo chain providing consensus and settlement |
| **L2** | Layer 2 — Zones running on top of L1 |
| **Simplex BFT** | Threshold BFT consensus protocol via Commonware |
| **Commonware** | Consensus library providing Simplex BFT, DKG, P2P networking |
| **Reth** | Rust Ethereum execution client (Paradigm) — Tempo's foundation |
| **DKG** | Distributed Key Generation — Feldman-Desmedt scheme for threshold signatures |
| **Epoch** | Fixed number of blocks (default 302,400) after which validator set can rotate |
| **Subblock** | Partial block built by validator within an epoch (currently disabled) |

### Fee & Gas

| Term | Definition |
|------|------------|
| **Attodollar** | 10^-18 USD — smallest fee unit |
| **Base fee** | Minimum gas price (20B attodollars at T1+) |
| **Fee AMM** | Automated market maker converting user-paid stablecoins to validator-preferred tokens |
| **General gas limit** | Gas limit for non-payment transactions (30M at T1+) |
| **Shared gas limit** | Shared gas pool (disabled at T4+) |
| **Storage credits** | Credits earned by clearing storage slots (TIP-1060, T7+) |
| **pathUSD** | USD stablecoin used as gas token at launch |

### Precompiles & Contracts

| Term | Definition |
|------|------------|
| **Precompile** | Protocol-level contract implemented in Rust (not EVM bytecode) |
| **TIP-20** | Enshrined ERC-20 token standard with extensions |
| **TIP-403** | Permissioned token compliance registry |
| **TIP20 Factory** | Precompile for creating TIP-20 tokens |
| **ZoneFactory** | Precompile for creating Zones (T10+) |
| **ZonePortal** | L1 contract for lock-and-mint bridge to Zones |
| **ZoneInbox** | Zone precompile for processing deposits |
| **ZoneOutbox** | Zone precompile for processing withdrawals |
| **StablecoinDEX** | Fee AMM precompile |
| **NonceManager** | 2D nonce system precompile |
| **AccountKeychain** | Passkey/WebAuthn key management precompile |

### Zone-Specific

| Term | Definition |
|------|------------|
| **Sequencer** | Zone node that produces blocks (leader-follower model) |
| **Leader** | Sequencer producing blocks for current anchor |
| **Follower** | Sequencer replicating/validating leader's blocks |
| **RPC-only follower** | Non-sequencer node serving public RPC |
| **SPF** | Stateless Proof Function — stateless state transition function |
| **Batch witness** | MPT trie proofs for Zone and L1 state |
| **Batch output** | Commitments produced by SPF execution |
| **Redacted RPC** | Authenticated RPC returning 0 for non-self queries |
| **Confidential balance** | Encrypted token balance (visible only to owner) |
| **Anchor block** | Finalized L1 block to which Zone genesis is anchored |

### Infrastructure

| Term | Definition |
|------|------------|
| **Nitro Enclave** | AWS isolated compute environment for prover |
| **EIF** | Enclave Image File — Nitro Enclave binary |
| **VSOCK** | Virtual socket for host ↔ enclave communication |
| **Localnet** | Docker-based local development network (chain ID 1337) |
| **tempoup** | Binary installer with provenance verification |

---

## TIP Standards Index

TIP = Tempo Improvement Proposal

| TIP | Title | Hardfork | Description |
|-----|-------|----------|-------------|
| **TIP-20** | Token Standard | Genesis | Enshrined ERC-20 with extensions (fee sponsorship, batch transfers) |
| **TIP-403** | Compliance Registry | Genesis | Permissioned token compliance (whitelisting) |
| **TIP-1000** | Gas Costs | T1 | Custom gas parameters for state-creating operations |
| **TIP-1016** | Gas Split | T4 (disabled) | Split gas into regular (compute) and state (storage) components |
| **TIP-1060** | Storage Credits | T7 | Storage credit system for state cleanup incentives |
| **TIP-1067** | Dynamic Base Fee | T7 | EIP-1559-style dynamic base fee adjustment |

---

## Address Map

### L1 System Addresses

| Address | Contract | Activates |
|---------|----------|-----------|
| `0x20c0000000000000000000000000000000000000` | pathUSD (gas token) | Genesis |
| `0x20fc000000000000000000000000000000000000` | TIP20 Factory | Genesis |
| `0x403c000000000000000000000000000000000000` | TIP403 Registry | Genesis |
| `0xfeec000000000000000000000000000000000000` | Tip Fee Manager | Genesis |
| `0xdec000000000000000000000000000000000000` | Stablecoin DEX (Fee AMM) | Genesis |
| `0x4e4f4e4345000000000000000000000000000000` | Nonce Manager | Genesis |
| `0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` | Validator Config (v1) | Genesis |
| `0xb10c000000000000000000000000000000000000` | Account Keychain | Genesis |
| `0xcccccccccccccccccccccccccccccccccccccc01` | Validator Config V2 | T2 |
| `0x5165300000000000000000000000000000000000` | Address Registry | T3 |
| `0x515e510000000000000000000000000000000000` | Signature Verifier | T3 |
| `0x1060000000000000000000000000000000000000` | Storage Credits | T7 |
| `0x5af2000000000000000000000000000000000000` | Zone Factory | T10 |
| `0x5ad1000000000000000000000000000000000000` | Zone Portal | T10 |
| `0xcA11bde05977b3631167028862bE2a173976CA11` | Multicall3 | Genesis |
| `0xba5Ed099633D3B313e4D5F7bdc1305d3c28ba5Ed` | CreateX | Genesis |
| `0x914d7Fec6aaC8cd542e72Bca78B30650d45643d7` | Safe Deployer | Genesis |
| `0x000000000022d473030f116ddee9f6b43ac78ba3` | Permit2 | Genesis |
| `0x4e59b44847b379578588920cA78FbF26c0B4956C` | Arachnid CREATE2 Factory | Genesis |

### L2 Zone System Addresses

| Address | Contract | Notes |
|---------|----------|-------|
| `0x20c0000000000000000000000000000000000000` | pathUSD | Gas token |
| `0x1c00000000000000000000000000000000000000` | ZoneInbox | Deposits |
| `0x1c00000000000000000000000000000000000002` | ZoneOutbox | Withdrawals |
| `0xcA11bde05977b3631167028862bE2a173976CA11` | Multicall3 | Standard |

---

## Chain ID Reference

| Network | Chain ID | Status |
|---------|----------|--------|
| Tempo Mainnet | 4217 | Production |
| Tempo Testnet (Moderato) | 42431 | Testing |
| ThaiFi Private Chain | 17 | Production |
| Localnet (dev) | 1337 | Development |
| Zone (dev) | 1337 (domain-separated) | Development |

---

## Hardfork Reference

| Hardfork | Key Features | Status |
|----------|--------------|--------|
| **Genesis** | Initial state, basic precompiles | Active |
| **T0** | Base fee = 10B attodollars | Active |
| **T1** | TIP-1000 gas costs, base fee = 20B attodollars | Active |
| **T1A** | Per-tx gas cap = 30M | Active |
| **T1B** | Minor adjustments | Active |
| **T1C** | Minor adjustments | Active |
| **T2** | Validator Config V2 precompile | Active |
| **T3** | Address Registry, Signature Verifier | Active |
| **T4** | TIP-1016 gas split (disabled), shared gas limit = 0 | Active |
| **T5** | TIP20 Channel Reserve | Active |
| **T6** | Receive Policy Guard | Active |
| **T7** | TIP-1060 storage credits, TIP-1067 dynamic base fee | Active |
| **T8** | Current Committee precompile | Active |
| **T9** | Minor adjustments | Active |
| **T10** | Zone Factory, zone system | Active |
| **T11** | Expiring nonce capacity | Future (disabled in production) |
| **T12** | Latest hardfork | Future (disabled in production) |

**Note:** T11 and T12 are disabled in production (set to far-future timestamps) to maintain T10 Portal ABI compatibility with Zones.

---

## Cryptography Reference

| Algorithm | Purpose | Library |
|-----------|---------|---------|
| **Ed25519** | P2P signing keys (validator identity) | `commonware-cryptography` |
| **BLS12-381** | Threshold signing shares | `commonware-cryptography` |
| **Feldman-Desmedt** | DKG scheme | `commonware-cryptography` |
| **ECIES** | Encrypted deposits (Zone) | — |
| **Chaum-Pedersen** | ECDH proof verification (Zone) | Zone precompile |
| **AES-GCM** | Encrypted deposit decryption | Zone precompile |
| **P256 (secp256r1)** | Passkey/WebAuthn signatures | Account Keychain |
| **secp256k1** | Standard Ethereum signatures | — |

---

## Port Reference

### L1 Production (192.168.1.199)

| Node | HTTP RPC | WS RPC | P2P | Consensus |
|------|----------|--------|-----|-----------|
| Validator #1 | 8545 | 8546 | 30303 | 3000 |
| Validator #2 | 8547 | 8548 | 30304 | 3001 |
| Validator #3 | 8549 | 8550 | 30305 | 3002 |
| RPC Follower | 28545 | — | — | — |

### L2 Zone (default)

| Service | Port |
|---------|------|
| Zone HTTP RPC | 9545 |
| Prover (TCP) | 5000 |

### Localnet

| Service | Port |
|---------|------|
| HTTP RPC | 8545 |
