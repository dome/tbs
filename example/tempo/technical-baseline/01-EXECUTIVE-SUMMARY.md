# 01 — Executive Summary

## What is Tempo?

Tempo is a **Layer 1 blockchain purpose-built for stablecoin payments**. It is fully EVM-compatible (targeting the Ethereum Osaka hardfork) and built on the Reth SDK — Paradigm's Rust implementation of an Ethereum execution client.

The system consists of two layers:

- **Tempo L1** — The main chain providing consensus, settlement, and the economic layer
- **Tempo Zones (L2)** — Private blockchains anchored to L1 with native confidential balances and transactions

---

## Value Proposition

| Feature | Description |
|---------|-------------|
| **Sub-millidollar fees** | Standard TIP-20 transfer costs ~0.001 USD at T1 base fee |
| **Stablecoin-denominated gas** | Users pay fees in USD stablecoins (pathUSD), not volatile native tokens |
| **Sub-second finality** | Simplex BFT consensus via Commonware |
| **Native account abstraction** | P256/WebAuthn (passkeys), batched payments, fee sponsorship, scheduled payments |
| **Privacy-preserving L2** | Zones offer encrypted deposits, redacted RPC, confidential balances |
| **Compliance-ready** | TIP-403 permissioned token registry inherited across L1 and L2 |

---

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    TEMPO L1 CHAIN                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Validator │  │ Validator │  │ Validator │  (Simplex   │
│  │    #1     │  │    #2     │  │    #3     │   BFT)     │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘          │
│        │               │               │                 │
│  ┌─────┴───────────────┴───────────────┴─────┐          │
│  │          Reth Execution Client             │          │
│  │  ┌─────────┐ ┌──────────┐ ┌───────────┐  │          │
│  │  │  EVM    │ │Precompiles│ │  Tx Pool  │  │          │
│  │  │ (Osaka) │ │ (16+)    │ │ (2D nonce)│  │          │
│  │  └─────────┘ └──────────┘ └───────────┘  │          │
│  └───────────────────────────────────────────┘          │
│                                                          │
│  ┌────────────┐  ┌──────────┐  ┌──────────────┐        │
│  │ZoneFactory │  │ZonePortal│  │  Fee AMM /   │        │
│  │            │  │(bridge)  │  │  StableDEX   │        │
│  └─────┬──────┘  └────┬─────┘  └──────────────┘        │
└────────┼──────────────┼─────────────────────────────────┘
         │              │
         │    ┌─────────┴──────────┐
         │    │   TEMPO ZONE (L2)  │
         │    │                    │
         │    │ ┌───────────────┐  │
         │    │ │  Sequencer    │  │
         │    │ │  (1-8 nodes)  │  │
         │    │ └───────────────┘  │
         │    │ ┌───────────────┐  │
         │    │ │  Confidential │  │
         │    │ │  State (EVM)  │  │
         │    │ └───────────────┘  │
         │    │ ┌───────────────┐  │
         │    │ │  Nitro Prover │  │
         │    │ │  (TEE/SPF)    │  │
         │    │ └───────────────┘  │
         │    └────────────────────┘
         │
    ┌────┴─────┐
    │  RPC Node │  (non-validating follower)
    └──────────┘
```

---

## Chain Identifiers

| Network | Chain ID | Status |
|---------|----------|--------|
| Mainnet | 4217 | Production |
| Testnet (Moderato) | 42431 | Testing |
| Private Chain (ThaiFi) | 17 | Production (192.168.1.199) |
| Localnet (dev) | 1337 | Development |
| Zone (dev) | 1337 (domain-separated) | Development |

---

## Key Metrics (As-Is)

| Metric | Value |
|--------|-------|
| L1 Crates | 22 Rust crates |
| L2 Crates | 15 Rust crates + 3 prover binaries |
| Custom Precompiles (L1) | 16+ protocol-level contracts |
| Custom Precompiles (L2) | 6 zone-specific precompiles |
| Hardforks Defined | 16 (Genesis → T12) |
| CI/CD Workflows | 35+ GitHub Actions |
| Docker Build Targets | 10+ (L1 + L2 + prover) |
| Workspace Version (L1) | 1.13.2 |
| Workspace Version (L2) | 0.2.2 |

---

## Ownership & Forks

| Component | Repository | Branch/Ref | Notes |
|-----------|-----------|------------|-------|
| L1 Core (upstream) | `tempoxyz/tempo` | `main` | Public open-source |
| L1 Core (ThaiFi fork) | `ThaiFi/tempo` | `feat/private-chain-genesis` | Adds `--zone-factory-owner`, `--genesis-timestamp`, `--gas-token-*` flags |
| L2 Zones | `tempoxyz/zones` | pinned @ `48e63e37` | Pinned for T10 ABI compatibility |
| Deployment | `ThaiFi/node` | `main` | Wrapper with submodules |

---

## Critical Constraint: T10 Portal ABI

The Zone sequencer speaks the **T10 portal ABI**. If the L1 chain activates T11 or T12 hardforks, the portal runtime changes and becomes **incompatible** with the zones sequencer. The ThaiFi fork therefore sets `--t11-time` and `--t12-time` to far-future values, keeping the T10 runtime active.

This is the single most important constraint when upgrading the L1 chain in any environment that runs Zones.
