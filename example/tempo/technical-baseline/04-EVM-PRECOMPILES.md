# 04 — EVM & Precompiles

## EVM Customization

Tempo targets the **Ethereum Osaka hardfork** via the Reth SDK. The EVM is customized at multiple levels:

### TempoEvmConfig (`tempo-evm` crate)

Wraps Reth's `EthEvmConfig` with Tempo-specific extensions:

| Component | Description |
|-----------|-------------|
| `TempoBlockAssembler` | Custom block assembly (no ommers/uncles) |
| `TempoBlockExecutor` | Custom block execution with subblock fee recipient handling |
| `TempoPoolValidationEvm` | Custom transaction pool validation |
| `ExpiringNonceReplay` | Replay protection for expiring nonces |

### TempoEvm (`tempo-revm` crate)

Custom EVM with Tempo-specific context:

| Component | Description |
|-----------|-------------|
| `TempoFeeManager` / `FeeTokenResolver` | Resolves fee tokens for transactions (pathUSD) |
| `ProtocolFeeManager` | Internal protocol fee hooks |
| `TempoTxEnv` / `TempoBatchCallEnv` | Extended transaction environment for batch calls |
| `ValidationContext` | Account abstraction validation |
| `calculate_aa_batch_intrinsic_gas` | Intrinsic gas calculation for batched AA transactions |

### Gas Parameters (Per Hardfork)

| Hardfork | Gas Parameters |
|----------|---------------|
| T1 | TIP-1000: Custom state-creation costs |
| T4 | TIP-1016: Regular vs. state gas split (currently disabled) |
| T7 | TIP-1060: Storage credit system |

---

## Custom Precompiles

Tempo registers **16+ protocol-level precompiles** at fixed addresses. These are not regular smart contracts — they are native EVM extensions implemented in Rust.

### Precompile Address Scheme

Tempo uses a deterministic address scheme for precompiles:

| Prefix | Category | Example |
|--------|----------|---------|
| `0x20C0...` | TIP-20 tokens (dynamic) | `0x20c0000000000000000000000000000000000000` (pathUSD) |
| `0x20FC00...` | TIP-20 Factory | Token creation |
| `0x403C00...` | TIP-403 Registry | Permissioned token registry |
| `0xFEEC00...` | Tip Fee Manager | Fee configuration |
| `0xDEC000...` | Stablecoin DEX | Fee AMM |
| `0x4E4F4E4345...` | Nonce Manager | 2D nonce system |
| `0xAAAAAAAA...` | Validator Config (v1) | Validator set (Genesis) |
| `0xB10C00...` | Account Keychain | Passkey/WebAuthn keys |
| `0xCCCCCCCC...01` | Validator Config V2 | Validator set (T2+) |
| `0x516530...` | Address Registry | Name/address mapping (T3+) |
| `0x515E51...` | Signature Verifier | Signature verification (T3+) |
| `0x106000...` | Storage Credits | Storage credit system (T7+) |
| `0x5AF200...` | Zone Factory | Zone creation (T10+) |

### TIP-20 Dynamic Dispatch

TIP-20 tokens use **address prefix matching** — any address matching the `0x20C0...` pattern is dynamically dispatched to the `TIP20Token` precompile. This is checked via `is_tip20_prefix()`.

This means:
- `0x20c0000000000000000000000000000000000000` → pathUSD
- `0x20c0000000000000000000000000000000000001` → Another TIP-20 token
- All share the same precompile logic, differentiated by address

### Precompile Activation Schedule

| Precompile | Activates | Purpose |
|------------|-----------|---------|
| TIP20 Token (dynamic) | Genesis | Enshrined ERC-20 with extensions |
| TIP20 Factory | Genesis | Token creation |
| TIP403 Registry | Genesis | Permissioned token compliance |
| Tip Fee Manager | Genesis | Fee configuration |
| Stablecoin DEX | Genesis | Fee AMM |
| Nonce Manager | Genesis | 2D nonce system |
| Validator Config (v1) | Genesis | Initial validator set |
| Account Keychain | Genesis | Passkey/WebAuthn management |
| Validator Config V2 | T2 | Enhanced validator config |
| Address Registry | T3 | Name/address mapping |
| Signature Verifier | T3 | Signature verification |
| TIP20 Channel Reserve | T5 | Channel reservations |
| Receive Policy Guard | T6 | Receive policy enforcement |
| Storage Credits | T7 | Storage credit system |
| Current Committee | T8 | Active validator set query |
| Zone Factory | T10 | Zone creation |

### Precompile Security

All precompiles enforce **direct-call-only** (no delegatecall) via the `tempo_precompile!` macro. This prevents proxy contracts from delegating to precompiles and bypassing access controls.

---

## Precompile Details

### TIP-20 Token (`0x20C0...`)

The enshrined token standard. Extends ERC-20 with:

- **Permissioned transfers** — TIP-403 compliance checks
- **Fee sponsorship** — Third-party fee payment
- **Batch transfers** — Multiple recipients in one tx
- **Stablecoin metadata** — Currency, decimals (6 for USD)

Standard ERC-20 functions: `transfer`, `transferFrom`, `approve`, `balanceOf`, `totalSupply`

Extended functions: `sponsorTransfer`, `batchTransfer`, `mint`, `burn` (admin only)

### TIP-403 Registry (`0x403C00...`)

Permissioned token compliance registry:
- Tracks whitelisted addresses
- Enforces transfer restrictions
- Inherited by Zones (read-only against L1 state)

### Stablecoin DEX / Fee AMM (`0xDEC000...`)

Automated market maker for fee token conversion:
- Converts user-paid stablecoins to validator-preferred tokens
- Pairwise liquidity pools (e.g., pathUSD/AlphaUSD)
- Initialized at genesis with minted liquidity

### Nonce Manager (`0x4E4F4E4345...`)

2D nonce system:
- **Sequential nonce** — Standard Ethereum-style nonce
- **Capacity nonce** — Expiring capacity for parallel execution
- Supports account abstraction batch calls

### Validator Config V2 (`0xCCCCCCCC...01`)

Validator set management (T2+):
- Register/unregister validators
- Update network endpoints
- Rotate signing keys
- Requires Ed25519 signature for registration

### Account Keychain (`0xB10C00...`)

Passkey/WebAuthn key management:
- Register P256 public keys
- Associate keys with accounts
- Enable passkey authentication for transactions

### Zone Factory (`0x5AF200...`)

Zone creation (T10+):
- `createZone()` — Deploy new zone
- Configurable zone parameters
- Owner-controlled (ZoneFactory owner set at genesis)

---

## Predeployed Ethereum Contracts

In addition to custom precompiles, Tempo predeploys standard Ethereum contracts:

| Contract | Address | Notes |
|----------|---------|-------|
| **Multicall3** | `0xcA11bde05977b3631167028862bE2a173976CA11` | Standard Ethereum address |
| **CreateX** | `0xba5Ed099633D3B313e4D5F7bdc1305d3c28ba5Ed` | Deterministic deployment |
| **Safe Deployer** | `0x914d7Fec6aaC8cd542e72Bca78B30650d45643d7` | Gnosis Safe deployment |
| **Permit2** | `0x000000000022d473030f116ddee9f6b43ac78ba3` | Deployed via Arachnid CREATE2 |
| **Arachnid CREATE2 Factory** | `0x4e59b44847b379578588920cA78FbF26c0B4956C` | Deterministic address deployment |

Bytecode hashes are verified against Ethereum mainnet to ensure compatibility.

---

## Transaction Types

Tempo supports multiple transaction signature types via `TempoTxEnvelope`:

| Type | Signature | Use Case |
|------|-----------|----------|
| Legacy / EIP-2930 / EIP-1559 | secp256k1 | Standard EOA transactions |
| P256 / WebAuthn | P256 (passkeys) | Account abstraction with passkey auth |
| AA-signed | Custom | Batch calls, fee sponsorship |

### 2D Nonce System

Tempo's transaction pool (`TempoTransactionPool`) supports 2D nonces:

1. **Sequential nonce** — Standard Ethereum nonce (prevents replay)
2. **Capacity nonce** — Expiring capacity for parallel execution (introduced at T11)

The `AA2dPool` handles account abstraction transactions with 2D nonce validation.

---

## Block Structure

### TempoHeader

Extends standard Ethereum header with:

| Field | Type | Description |
|-------|------|-------------|
| `general_gas_limit` | u64 | Gas limit for non-payment transactions |
| `shared_gas_limit` | u64 | Shared gas limit (T4+) |
| `timestamp_millis_part` | u64 | Millisecond-precision timestamp |
| `consensus_context` | struct | Epoch, view, parent_view, proposer public key |

### TempoBlockEnv

Extends standard block environment with:

| Field | Description |
|-------|-------------|
| `timestamp_millis_part` | Millisecond precision |
| `epoch_length` | Blocks per epoch |
| `proposer_public_key` | Block proposer's Ed25519 key |

---

## Key Constraints

1. **No delegatecall to precompiles** — All precompiles enforce direct-call-only
2. **TIP-20 prefix matching** — Addresses matching `0x20C0...` are dynamically dispatched
3. **Osaka EVM target** — All EVM features target Ethereum Osaka hardfork
4. **No ommers** — Tempo does not support uncle blocks
5. **T10 hardfork required for Zones** — Zone Factory activates at T10
