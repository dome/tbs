# 07 — Deployment & Infrastructure

## Repository Structure

The Tempo system is organized across three repositories:

| Repository | Purpose | Branch/Ref |
|------------|---------|------------|
| `ThaiFi/tempo` | L1 core (fork) | `feat/private-chain-genesis` |
| `tempoxyz/zones` | L2 zones | Pinned @ `48e63e37` |
| `ThaiFi/node` | Deployment wrapper | `main` |

The `node/` repository contains the other two as **git submodules**:

```bash
node/
├── tempo/   ← submodule → ThaiFi/tempo
├── zones/   ← submodule → tempoxyz/zones
└── .gitmodules
```

---

## Build System

### Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Rust | 1.95.0+ (edition 2024) | Compilation |
| Docker | Latest | Container builds |
| `just` | Latest | Task runner (Justfile recipes) |
| `cargo-chef` | Latest | Dependency pre-compilation |
| `mold` | Latest | Fast linker (Linux) |

### L1 Build

```bash
cd tempo/
cargo build --release
```

**Binaries:**
- `tempo` — L1 node
- `tempo-sidecar` — Sidecar process
- `tempo-xtask` — Genesis tooling

**Features:**
- `asm-keccak` — Assembly-optimized Keccak
- `jemalloc` — jemalloc memory allocator
- `otlp` — OpenTelemetry support

### L2 Build

```bash
cd zones/
cargo build --release
```

**Binaries:**
- `tempo-zone` — Zone node
- `tempo-zone-xtask` — Zone provisioning
- `tempo-zone-prover-enclave` — Nitro prover
- `tempo-zone-prover-utils` — Prover utilities

---

## Docker Builds

### L1 Dockerfiles

| Dockerfile | Purpose | Base Image |
|------------|---------|------------|
| `Dockerfile` | Multi-target production build | `rust:1.96-bookworm` |
| `Dockerfile.chef` | cargo-chef dependency pre-compilation | `rust:1.96-bookworm` |
| `Dockerfile.reproducible` | Byte-deterministic build | Pinned Debian snapshot |

**Build targets:**
- `tempo` — L1 node binary
- `tempo-localnet` — Localnet with health check
- `tempo-sidecar` — Sidecar process
- `tempo-xtask` — Genesis tooling

**Docker Bake (`docker-bake.hcl`):**
- Platforms: `linux/amd64`, `linux/arm64`
- Registry: `ghcr.io/tempoxyz/`
- Groups: `default`, `nightly`

### L2 Dockerfiles

| Dockerfile | Purpose | Base Image |
|------------|---------|------------|
| `Dockerfile` | Zone node + xtask | `rust:1.96-bookworm` |
| `Dockerfile.prover-enclave` | Nitro enclave binary | `rust:1.96-bookworm` |
| `Dockerfile.prover-host` | Nitro host (Amazon Linux 2023) | `amazonlinux:2023` |
| `Dockerfile.prover-eif-builder` | EIF image builder | Pinned AWS Nitro kernel |
| `Dockerfile.prover-utils` | Prover utilities | `rust:1.96-bookworm` |

**Docker Bake (`docker-bake.hcl`):**
- Platform: `linux/amd64` only (zones are x86-only due to Nitro)
- Targets: `tempo-zone`, `tempo-zone-xtask`, `tempo-zone-prover-*`

### Reproducible Builds

Tempo supports **byte-deterministic builds** for verification:

```bash
# L1
docker build -f Dockerfile.reproducible -t tempo-reproducible .

# L2
cd zones/
./scripts/reproducible-build.sh
```

**Features:**
- Pinned Debian snapshot (`20260501T000000Z`)
- Pinned Rust 1.96.0
- `--profile reproducible` with `panic = "abort"`
- `SOURCE_DATE_EPOCH` for deterministic timestamps
- `--remap-path-prefix` for reproducible paths
- jemalloc override for deterministic allocation

Any third party can reproduce the binary and verify the SHA256 matches the GitHub release.

---

## Production Deployment: ThaiFi Private Chain

### Topology

Deployed on `192.168.1.199`:

```
┌─────────────────────────────────────────────────┐
│  192.168.1.199                                   │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │Validator │  │Validator │  │Validator │      │
│  │    #1    │  │    #2    │  │    #3    │      │
│  │ :8545    │  │ :8547    │  │ :8549    │      │
│  │ :30303   │  │ :30304   │  │ :30305   │      │
│  │ :3000    │  │ :3001    │  │ :3002    │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                   │
│  ┌──────────┐                                    │
│  │ RPC Node │  (non-validating follower)         │
│  │ :28545   │  follows ws://127.0.0.1:8546      │
│  └──────────┘                                    │
└─────────────────────────────────────────────────┘
```

### Port Allocation

| Node | HTTP RPC | WS RPC | P2P | Consensus |
|------|----------|--------|-----|-----------|
| Validator #1 | 8545 | 8546 | 30303 | 3000 |
| Validator #2 | 8547 | 8548 | 30304 | 3001 |
| Validator #3 | 8549 | 8550 | 30305 | 3002 |
| RPC Follower | 28545 | — | — | — |

### Docker Images

**L1:**
- Image: `tempo:legacy` (must NOT use newer images)
- Reason: Legacy has no T11/T12, keeping portal at T10 for zone compatibility

**L2:**
- Built locally from `zones/docker/`

### Configuration

**Chain parameters:**
- Chain ID: 17
- Gas token: pathUSD (`0x20c0000000000000000000000000000000000000`)
- ZoneFactory: `0x5af2000000000000000000000000000000000000`
- ZonePortal: `0x5ad1000000000000000000000000000000000000`
- Admin/ZoneFactory owner: `0x5266Dfa5ae013674f8FdC8322b7c601B838D94eE6` (Ledger)

**Docker Compose:**
- `network_mode: host` for all services
- Compose files stored on remote server at `/data/thaifi/` and `/data/thaifi-rpc/`

---

## Genesis Generation

### L1 Genesis

```bash
cd tempo/
cargo xtask genesis \
  --chain-id 17 \
  --validators 3 \
  --gas-token-name pathUSD \
  --gas-token-symbol USD \
  --gas-token-currency USD \
  --zone-factory-owner 0x5266Dfa5ae013674f8FdC8322b7c601B838D94eE6 \
  --genesis-timestamp 1234567890 \
  --output genesis.json
```

**Custom flags (ThaiFi fork):**
- `--zone-factory-owner` — Custom ZoneFactory owner (not hardcoded)
- `--genesis-timestamp` — Chain start time
- `--gas-token-name/symbol/currency` — Customize gas token

### L2 Zone Genesis

```bash
cd zones/
cargo xtask generate-zone-genesis \
  --zone 0xZoneAddress \
  --l1-rpc ws://127.0.0.1:8546 \
  --output zone-genesis.json
```

---

## Running Nodes

### L1 Validator

```bash
tempo \
  --chain genesis.json \
  --datadir /data/validator1 \
  --http --http.addr 0.0.0.0 --http.port 8545 \
  --ws --ws.addr 0.0.0.0 --ws.port 8546 \
  --port 30303 \
  --consensus.port 3000 \
  --signing-key ed25519:... \
  --signing-share bls12-381:... \
  --tempo.fee-token pathUSD
```

### L1 RPC Follower

```bash
tempo \
  --chain genesis.json \
  --datadir /data/rpc \
  --http --http.addr 0.0.0.0 --http.port 28545 \
  --follow ws://127.0.0.1:8546
```

### L2 Zone Sequencer

```bash
tempo-zone \
  --chain zone-genesis.json \
  --sequencer \
  --l1-rpc ws://127.0.0.1:8546 \
  --prover tcp:127.0.0.1:5000 \
  --http --http.addr 0.0.0.0 --http.port 9545
```

---

## CI/CD

### GitHub Actions (35+ workflows)

| Category | Workflows |
|----------|-----------|
| Build & Test | `build.yml`, `test.yml`, `coverage.yml` |
| Lint | `lint.yml`, `clippy.yml`, `fmt.yml` |
| Docker | `docker.yml`, `docker-profiling.yml`, `docker-reproducible.yml` |
| Release | `release.yml`, `publish-crates.yml` |
| E2E | `e2e-single-region.yml`, `e2e-multi-region.yml` |
| Docs | `docs.yml` |
| Security | `audit.yml`, `semver.yml` |
| Hardfork | `add-hardfork.yml` |

### Reproducible Build Verification

The `docker-reproducible.yml` workflow:
1. Builds binary using `Dockerfile.reproducible`
2. Computes SHA256
3. Compares against GitHub release artifact
4. Fails if mismatch

---

## Localnet

### Docker-Based Localnet

```bash
cd tempo/examples/localnet/
docker compose up
```

**Configuration:**
- Image: `ghcr.io/tempoxyz/tempo-localnet:latest`
- Chain ID: 1337
- Block time: 200ms (default)
- RPC: `127.0.0.1:8545`

**Pre-funded:**
- 50,000 dev accounts (from mnemonic)
- Faucet service
- Fee AMM and DEX liquidity

---

## Installer: tempoup

The `tempoup/` directory contains an installer script for pre-built binaries:

**Features:**
- SHA256 checksum verification
- GPG signature verification
- Sigstore/SLSA provenance verification

**Usage:**
```bash
curl -fsSL https://tempo.xyz/tempoup | bash
```

---

## Key Constraints

1. **Docker image pinning** — Production uses `tempo:legacy` (no T11/T12)
2. **x86_64 only for zones** — AWS Nitro Enclaves require x86_64
3. **network_mode: host** — All production services use host networking
4. **P2P key regeneration** — `rm data/*` regenerates P2P keys → stale enodes → tx propagation failure
5. **Manual deployment** — No Kubernetes/Terraform; deployment is manual (build, SCP, docker compose)
