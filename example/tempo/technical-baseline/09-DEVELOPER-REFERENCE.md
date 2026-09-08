# 09 — Developer Reference

## Crate API Reference

### L1 Core Crates

#### `tempo-primitives`

Core types used throughout the system:

```rust
// Block types
type Block = alloy_consensus::Block<TempoTxEnvelope, TempoHeader>;

struct TempoHeader {
    // Standard Ethereum header fields
    parent_beacon_block_root: Option<B256>,
    withdrawals: Option<Withdrawals>,
    
    // Tempo-specific fields
    general_gas_limit: u64,
    shared_gas_limit: u64,
    timestamp_millis_part: u64,
    consensus_context: ConsensusContext,
}

struct ConsensusContext {
    epoch: u64,
    view: u64,
    parent_view: u64,
    proposer_public_key: Ed25519PublicKey,
}

// Transaction types
enum TempoTxEnvelope {
    Legacy(Signed<TxLegacy>),
    Eip2930(Signed<TxAccessList>),
    Eip1559(Signed<TxEip1559>),
    Eip7702(Signed<TxEip7702>),
    P256(Signed<TxP256>),        // Passkey/WebAuthn
    AABatch(Signed<TxBatchCall>), // Account abstraction
}

// Signature types
enum TempoSignature {
    Secp256k1(Secp256k1Signature),
    P256(P256Signature),
    AA(AASignature),
}

// Subblock types
struct SubBlock {
    signature: Ed25519Signature,
    fee_recipient: Address,
    // ...
}
```

#### `tempo-chainspec`

Chain specification:

```rust
struct TempoChainSpec {
    chain_id: u64,
    hardforks: TempoHardforks,
    genesis: Genesis,
    // ...
}

struct TempoHardforks {
    genesis: u64,
    t0: Option<u64>,
    t1: Option<u64>,
    // ...
    t12: Option<u64>,
}
```

#### `tempo-hardfork`

Hardfork definitions:

```rust
enum TempoHardfork {
    Genesis,
    T0,
    T1,
    T1A,
    T1B,
    T1C,
    T2,
    T3,
    T4,
    T5,
    T6,
    T7,
    T8,
    T9,
    T10,
    T11,
    T12,
}

impl TempoHardfork {
    fn evm_spec_id(&self) -> SpecId {
        SpecId::OSAKA // All hardforks map to Osaka
    }
}
```

#### `tempo-consensus`

Consensus engine:

```rust
// Main entry points
fn run_consensus_stack(config: ConsensusConfig) -> Result<()>;
fn run_follow_stack(config: FollowConfig) -> Result<()>;

// Actor components
struct Application;
struct DkgManager;
struct EpochManager;
struct Execution;
struct PeerManager;
struct Storage;
struct Subblocks;
```

#### `tempo-evm`

EVM configuration:

```rust
struct TempoEvmConfig {
    chain_spec: TempoChainSpec,
    evm_factory: TempoEvmFactory,
}

struct TempoBlockAssembler;
struct TempoBlockExecutor;
struct TempoPoolValidationEvm;
struct ExpiringNonceReplay;
```

#### `tempo-revm`

REVM customization:

```rust
struct TempoEvm;
struct TempoFeeManager;
struct FeeTokenResolver;
struct ProtocolFeeManager;
struct TempoTxEnv;
struct TempoBatchCallEnv;
struct ValidationContext;

fn calculate_aa_batch_intrinsic_gas(tx: &TxBatchCall) -> u64;
```

#### `tempo-precompiles`

Precompile registration:

```rust
// Address constants
const TIP20_TOKEN_PREFIX: [u8; 2] = [0x20, 0xC0];
const TIP20_FACTORY: Address = address!("0x20FC00...");
const TIP403_REGISTRY: Address = address!("0x403C00...");
const TIP_FEE_MANAGER: Address = address!("0xFEEC00...");
const STABLECOIN_DEX: Address = address!("0xDEC000...");
const NONCE_MANAGER: Address = address!("0x4E4F4E4345...");
const VALIDATOR_CONFIG_V1: Address = address!("0xAAAAAAAA...");
const ACCOUNT_KEYCHAIN: Address = address!("0xB10C00...");
const VALIDATOR_CONFIG_V2: Address = address!("0xCCCCCCCC...01");
const ADDRESS_REGISTRY: Address = address!("0x516530...");
const SIGNATURE_VERIFIER: Address = address!("0x515E51...");
const STORAGE_CREDITS: Address = address!("0x106000...");
const ZONE_FACTORY: Address = address!("0x5AF200...");

// Helper functions
fn is_tip20_prefix(addr: &Address) -> bool;
```

#### `tempo-transaction-pool`

Transaction pool:

```rust
struct TempoTransactionPool;
struct AA2dPool; // 2D nonce support
struct AddressFilter;
struct StateAwareBestTransactions;
```

#### `tempo-node`

Node configuration:

```rust
type TempoFullNode = FullNode<TempoNodeAdapter, TempoAddOns<TempoFullNodeTypes>>;

struct TempoNetworkBuilder;
struct TempoNode;
struct TempoNodeArgs;
struct TempoPayloadBuilderBuilder;
struct TempoPoolBuilder;
```

### L2 Zone Crates

#### `zone-primitives`

Zone-specific types:

```rust
struct ZoneBlock;
struct ZoneTxEnvelope;
struct BatchWitness;
struct BatchOutput;
```

#### `zone-sequencer`

Sequencer logic:

```rust
struct Sequencer;
enum SequencerRole {
    Leader,
    Follower,
    RpcOnly,
}

fn produce_block(transactions: Vec<ZoneTx>) -> ZoneBlock;
fn validate_block(block: &ZoneBlock) -> Result<()>;
```

#### `zone-l1`

L1 interaction:

```rust
struct ZonePortal;
struct L1Client;

fn detect_deposit(block: &L1Block) -> Option<Deposit>;
fn submit_batch(batch: &BatchProof) -> Result<()>;
```

#### `zone-prover`

Prover client:

```rust
struct ProverClient;

enum ProverRequest {
    Verify(BatchWitness),
}

enum ProverResponse {
    VerifyResult(BatchOutput),
}

fn connect(addr: &str) -> Result<ProverClient>;
```

#### `zone-spf`

Stateless Proof Function:

```rust
fn prove_zone_batch(witness: BatchWitness) -> BatchOutput;
```

#### `zone-rpc`

Redacted RPC:

```rust
struct RedactedRpc;
struct AuthToken;

fn authenticate(token: &AuthToken) -> Result<Address>;
fn redact_balance(balance: U256, caller: Address, queried: Address) -> U256;
```

---

## CLI Reference

### L1 Node (`tempo`)

```bash
tempo [OPTIONS]

# Chain selection
--chain <PATH>              # Path to genesis.json

# Data directory
--datadir <PATH>            # Data directory (default: ~/.tempo)

# RPC
--http                      # Enable HTTP RPC
--http.addr <ADDR>          # HTTP RPC address (default: 127.0.0.1)
--http.port <PORT>          # HTTP RPC port (default: 8545)
--ws                        # Enable WebSocket RPC
--ws.addr <ADDR>            # WS RPC address
--ws.port <PORT>            # WS RPC port (default: 8546)

# P2P
--port <PORT>               # P2P port (default: 30303)
--bootnodes <ENODE>         # Bootnode enodes

# Consensus (validator only)
--consensus.port <PORT>     # Consensus port
--signing-key <KEY>         # Ed25519 signing key
--signing-share <SHARE>     # BLS12-381 signing share

# Follow mode (RPC node)
--follow <WS_URL>           # Follow upstream validator

# Tempo-specific
--tempo.fee-token <TOKEN>   # Fee token (pathUSD)
```

### L2 Zone Node (`tempo-zone`)

```bash
tempo-zone [OPTIONS]

# Chain selection
--chain <PATH>              # Path to zone-genesis.json

# Mode
--sequencer                 # Run as sequencer
--follower                  # Run as follower

# L1 connection
--l1-rpc <WS_URL>           # L1 WebSocket RPC

# Prover
--prover <ADDR>             # Prover address (tcp:127.0.0.1:5000)

# RPC
--http                      # Enable HTTP RPC
--http.addr <ADDR>          # HTTP RPC address
--http.port <PORT>          # HTTP RPC port
```

### Genesis Tooling (`tempo-xtask`)

```bash
tempo-xtask genesis [OPTIONS]

# Chain configuration
--chain-id <ID>             # Chain ID (default: 1337)
--epoch-length <BLOCKS>     # Epoch length (default: 302400)
--gas-limit <GAS>           # Block gas limit (default: 500000000)

# Validators
--validators <COUNT>        # Number of validators
--validator-socket-addr <ADDR> # Validator addresses

# Hardfork timestamps
--t0-time <UNIX>            # T0 activation
--t1-time <UNIX>            # T1 activation
# ...
--t12-time <UNIX>           # T12 activation

# Gas token (ThaiFi fork)
--gas-token-name <NAME>     # Token name (default: pathUSD)
--gas-token-symbol <SYMBOL> # Token symbol (default: USD)
--gas-token-currency <CUR>  # Currency (default: USD)

# Zone configuration (ThaiFi fork)
--zone-factory-owner <ADDR> # ZoneFactory owner
--genesis-timestamp <UNIX>  # Genesis timestamp

# Output
--output <PATH>             # Output genesis.json
```

### Zone Provisioning (`tempo-zone-xtask`)

```bash
tempo-zone-xtask generate-zone-genesis [OPTIONS]

# Zone identification
--zone <ADDRESS>            # Zone address on L1

# L1 connection
--l1-rpc <WS_URL>           # L1 WebSocket RPC

# Output
--output <PATH>             # Output zone-genesis.json
```

---

## Justfile Recipes

### L1 Justfile

```bash
# Development
just tempo-dev-up            # Start localnet
just tempo-dev-down          # Stop localnet

# Build
just build                   # Build all binaries
just build-release           # Build release binaries

# Testing
just test                    # Run tests
just test-e2e                # Run E2E tests

# Linting
just lint                    # Run linters
just fmt                     # Format code

# Genesis
just genesis                 # Generate genesis.json

# ABI checking
just check-abi               # Check ABI compatibility
```

### L2 Justfile (47KB, extensive)

```bash
# Build
just build                   # Build zone binaries
just build-prover            # Build prover binaries

# Zone deployment
just create-zone             # Create zone on L1
just deploy-zone             # Deploy zone
just set-encryption-key      # Set zone encryption key
just generate-zone-genesis   # Generate zone genesis

# Zone operation
just zone-up                 # Start zone node
just zone-down               # Stop zone node

# Bridge operations
just max-approve-portal      # Approve ZonePortal
just send-deposit            # Deposit to zone
just send-withdrawal         # Withdraw from zone
just check-balance           # Check zone balance

# DEX
just deploy-router           # Deploy UniversalRouter
just demo-swap-and-deposit   # Demo swap + deposit

# Compliance
just demo-blacklist          # Demo TIP-403 blacklist
just verify-closed-loop      # Verify closed-loop compliance

# TIP-403 policy management
just tip403-create-policy    # Create policy
just tip403-add-address      # Add address to policy
just tip403-remove-address   # Remove address from policy
```

---

## CI/CD Workflows

### Key Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `build.yml` | Push/PR | Build all crates |
| `test.yml` | Push/PR | Run unit tests |
| `coverage.yml` | Push/PR | Code coverage |
| `lint.yml` | Push/PR | Run clippy, fmt |
| `docker.yml` | Push/PR | Build Docker images |
| `docker-reproducible.yml` | Release | Reproducible build verification |
| `e2e-single-region.yml` | Push/PR | Single-region E2E tests |
| `e2e-multi-region.yml` | Push/PR | Multi-region E2E tests |
| `release.yml` | Tag | Create release |
| `publish-crates.yml` | Release | Publish to crates.io |
| `audit.yml` | Scheduled | Dependency audit |
| `semver.yml` | PR | Semver compatibility check |
| `add-hardfork.yml` | Manual | Add new hardfork scaffold |

---

## Development Setup

### Prerequisites

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup install 1.95.0
rustup default 1.95.0

# Install just
cargo install just

# Install Docker
# (platform-specific)

# Clone repositories
git clone --recurse-submodules https://github.com/ThaiFi/node.git
cd node/
```

### Local Development

```bash
# Start localnet
cd tempo/examples/localnet/
docker compose up

# Run tests
cd tempo/
cargo test

# Build binaries
cargo build --release
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `tempo/Cargo.toml` | L1 workspace definition |
| `tempo/xtask/src/genesis_args.rs` | Genesis generation logic |
| `tempo/crates/hardfork/src/lib.rs` | Hardfork definitions |
| `tempo/crates/precompiles/src/lib.rs` | Precompile registration |
| `tempo/crates/consensus/src/lib.rs` | Consensus engine |
| `zones/Cargo.toml` | L2 workspace definition |
| `zones/specs/spec.md` | Zones formal specification |
| `zones/docs/ZONES.md` | Zones reference guide |
| `zones/Justfile` | Zone deployment recipes |
| `node/.gitmodules` | Submodule definitions |
| `node/README.md` | Deployment runbook |
