# System Technical Baseline Package — Tempo Blockchain

> **Document Type:** System Context Pack / Technical Baseline for Procurement
> **Version:** As-Is (Snapshot: 2026-09-08)
> **Classification:** Internal — For Vendor Assessment & Development Planning

---

## Purpose

This package constitutes the complete technical baseline of the Tempo blockchain system as of the snapshot date. It is intended for:

- **Procurement & RFP** — As a Technical Annex for vendors to scope and price new feature development
- **AI Coding Agents** — As a Context Pack / Repository Knowledge Base for code-aware assistants
- **Engineering Onboarding** — As a System Architecture Document (SAD) for new team members

---

## Document Index

| # | Document | Focus | Audience |
|---|----------|-------|----------|
| 01 | [Executive Summary](01-EXECUTIVE-SUMMARY.md) | What is Tempo, value proposition, system overview | All stakeholders |
| 02 | [System Architecture](02-ARCHITECTURE.md) | L1/L2 topology, crate map, dependency graph, data flow | Architects, Tech Leads |
| 03 | [Consensus & Networking](03-CONSENSUS.md) | Simplex BFT, Commonware, P2P, epochs, DKG | Consensus engineers |
| 04 | [EVM & Precompiles](04-EVM-PRECOMPILES.md) | EVM customization, 16+ precompiles, address scheme | Smart contract devs |
| 05 | [Fee & Gas System](05-FEE-GAS-SYSTEM.md) | Stablecoin fees, Fee AMM, gas limits, TIP-1000/1016/1060 | Economists, core devs |
| 06 | [Zones (L2)](06-ZONES.md) | Privacy rollups, sequencer model, bridge, prover | L2 engineers |
| 07 | [Deployment & Infrastructure](07-DEPLOYMENT.md) | Docker, topology, ports, production setup | DevOps, SRE |
| 08 | [Genesis & Configuration](08-GENESIS.md) | Genesis generation, chain params, hardfork schedule | Chain operators |
| 09 | [Developer Reference](09-DEVELOPER-REFERENCE.md) | Crate API, CLI flags, Justfile recipes, CI/CD | Developers |
| 10 | [Glossary & TIP Index](10-GLOSSARY.md) | Terminology, TIP standards, address map | All |

---

## Repository Layout

```
/Users/dome/project/tempo/
├── tempo/              ← L1 Core (ThaiFi fork of tempoxyz/tempo, v1.13.2)
│   ├── bin/tempo/      ← L1 node binary
│   ├── bin/tempo-sidecar/  ← Sidecar process
│   ├── crates/         ← 22 Rust crates (consensus, evm, precompiles, etc.)
│   ├── xtask/          ← Genesis generation & build tooling
│   ├── scripts/        ← Shell scripts (transfers, token creation, etc.)
│   ├── config/         ← Node configuration files
│   ├── examples/localnet/  ← Docker-based localnet
│   └── tempoup/        ← Binary installer with provenance verification
├── zones/              ← L2 Zones (tempoxyz/zones, v0.2.2)
│   ├── bin/tempo-zone/ ← Zone node binary
│   ├── bin/prover/     ← AWS Nitro Enclave prover (enclave, host, vsock-proxy)
│   ├── crates/         ← 15 Rust crates (sequencer, prover, l1, rpc, etc.)
│   ├── xtask/          ← Zone provisioning tooling
│   ├── specs/          ← Formal specification
│   ├── docker/         ← Container builds (zone, prover, EIF builder)
│   └── zone-genesis/   ← Pre-built zone genesis templates
├── node/               ← Deployment wrapper (ThaiFi/node)
│   ├── tempo/          ← Git submodule → ThaiFi/tempo (feat/private-chain-genesis)
│   └── zones/          ← Git submodule → tempoxyz/zones (pinned @ 48e63e37)
└── tbs/
    ├── SKILL.md              ← Skill definition (generate this package for any codebase)
    └── example/tempo/
        └── technical-baseline/  ← This document package
```

---

## Key Technology Stack

| Layer | Technology |
|-------|-----------|
| Language | Rust (edition 2024, rust-version 1.95.0) |
| Execution Client SDK | Reth (Paradigm, rev `5f02aa3`) |
| EVM Target | Ethereum Osaka hardfork |
| Consensus | Simplex BFT via Commonware (v2026.7.1) |
| Cryptography | Ed25519 (P2P), BLS12-381 (threshold signing), ECIES (encryption) |
| Prover TEE | AWS Nitro Enclaves |
| Container | Docker (multi-stage, cargo-chef, reproducible builds) |
| CI/CD | GitHub Actions (35+ workflows) |
| Build Tooling | `just` (Justfile), `cargo xtask` |
| Token Standard | TIP-20 (enshrined ERC-20 extensions) |
| Fee Token | USD stablecoins (pathUSD at launch) |
| Account Abstraction | Native P256/WebAuthn, 2D nonces, batch calls |

---

## How to Use This Package

### For Procurement / RFP
1. Read **01 (Executive Summary)** for business context
2. Read **02 (Architecture)** for system boundaries
3. Reference **09 (Developer Reference)** for effort estimation
4. Use **10 (Glossary)** to align terminology in RFP documents

### For AI Coding Agents
1. Ingest all 10 documents as context
2. Use the crate map in **02** and address map in **10** for navigation
3. Reference **04** and **05** for protocol-level constraints
4. Check **08** for chain configuration parameters

### For New Engineers
1. Start with **01** → **02** → **03** for fundamentals
2. Deep-dive into your domain: **04** (contracts), **05** (fees), **06** (L2)
3. Use **07** and **09** for day-to-day development
