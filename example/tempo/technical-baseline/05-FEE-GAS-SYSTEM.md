# 05 — Fee & Gas System

## Overview

Tempo's fee system is one of its most distinctive features. Unlike Ethereum where gas is paid in ETH (a volatile asset), Tempo charges fees in **USD stablecoins** (pathUSD at launch), providing predictable transaction costs.

---

## Fee Architecture

### Stablecoin-Denominated Fees

Users pay gas in USD-stablecoins. The `TempoFeeManager` precompile resolves which token to charge for each transaction.

**Flow:**
```
User submits tx → Fee resolved in pathUSD (attodollars)
                → Execution consumes gas
                → Fee deducted from user's pathUSD balance
                → Fee AMM converts to validator-preferred token (if needed)
                → Validator receives fee
```

### Attodollar Precision

Fees are denominated in **attodollars** (10^-18 USD precision):

| Unit | Value | Example |
|------|-------|---------|
| 1 attodollar | 10^-18 USD | Smallest unit |
| 1 microdollar | 10^-12 USD | 1,000,000 attodollars |
| 1 millidollar | 10^-15 USD | 1,000,000,000 attodollars |
| 1 USD | 10^18 attodollars | Base unit |

---

## Base Fee

### T0 (Genesis)

- Base fee: **10 billion attodollars** (0.00000001 USD per gas unit)
- Standard transfer (~50,000 gas): ~0.0005 USD

### T1+ (First Hardfork)

- Base fee: **20 billion attodollars** (0.00000002 USD per gas unit)
- Standard transfer (~50,000 gas): ~0.001 USD (sub-millidollar)

### T7+ (TIP-1067): Dynamic Base Fee

At T7, Tempo implements an **EIP-1559-style dynamic base fee**:

| Parameter | Value |
|-----------|-------|
| Target gas | 10,000,000 |
| Min base fee | 600,000,000 attodollars |
| Max base fee | 12,000,000,000,000 attodollars |
| Adjustment formula | EIP-1559 (8th-power clamp) |

The base fee adjusts based on block utilization:
- Block gas > target → base fee increases
- Block gas < target → base fee decreases

---

## Gas Limits

### T0 (Genesis)

| Limit | Value | Description |
|-------|-------|-------------|
| Block gas limit | 500,000,000 | Total gas per block |

### T1+ (First Hardfork)

| Limit | Value | Description |
|-------|-------|-------------|
| General gas limit | 30,000,000 | For non-payment transactions |
| Shared gas limit | 50,000,000 | Block gas limit / 10 |
| Per-tx gas cap | 30,000,000 | Max gas per transaction (T1A+) |

### T4+

| Limit | Value | Description |
|-------|-------|-------------|
| Shared gas limit | 0 | Disabled (previously block_gas_limit / 10) |

---

## TIP-1000: Custom Gas Costs (T1+)

TIP-1000 introduces custom gas parameters for state-creating operations:

| Operation | Gas Cost | Notes |
|-----------|----------|-------|
| SSTORE (create) | 250,000 | Creating new storage slot |
| Contract create | 500,000 | Deploying new contract |
| New account | 250,000 | Creating new account |
| Code deposit | 1,000 per byte | Contract code storage |

**Rationale:** State-creating operations impose long-term storage costs. Higher gas prices discourage state bloat.

---

## TIP-1016: Gas Split (T4+, Currently Disabled)

TIP-1016 splits gas into two components:

1. **Regular gas** — Computational cost (CPU, memory)
2. **State gas** — Permanent storage cost (state growth)

**Status:** Currently disabled (TODO in code). The infrastructure exists but is not active.

---

## TIP-1060: Storage Credits (T7+)

TIP-1060 introduces a **storage credit system** to incentivize state cleanup:

### Mechanism

1. **SSTORE creation cost**: 250,000 gas (same as TIP-1000)
2. **Residual cost**: 5,000 gas (immediate cost)
3. **Credit**: 245,000 gas (credited via StorageCredits precompile)

### Clearing Storage

When a user clears an occupied storage slot (sets to zero):
- Receives storage credits
- Credits can be used to offset future storage costs

### StorageCredits Precompile (`0x106000...`)

Activates at T7. Provides:
- Query credit balance
- Apply credits to transactions
- Track credit lifecycle

**Rationale:** Users who clean up state are rewarded, reducing long-term storage burden.

---

## Fee AMM (Automated Market Maker)

### StablecoinDEX Precompile (`0xDEC000...`)

The Fee AMM converts user-paid stablecoins to the validator's preferred stablecoin.

**Flow:**
```
User pays fee in pathUSD
  → Fee AMM checks validator's preferred token
  → If different, swap pathUSD → preferred token
  → Validator receives preferred token
```

### Liquidity Pools

Pairwise liquidity pools are initialized at genesis:
- pathUSD / AlphaUSD
- pathUSD / BetaUSD
- pathUSD / ThetaUSD

Liquidity is minted from pre-funded accounts (50,000 accounts generated from mnemonic).

### AMM Formula

Constant-product formula (x * y = k):
- Price impact based on pool depth
- Slippage protection via min-output parameters

---

## Fee Token Configuration

### Genesis Configuration

Fee tokens are configured at genesis via `genesis_args.rs`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `gas_token_name` | "pathUSD" | Token name |
| `gas_token_symbol` | "USD" | Token symbol |
| `gas_token_currency` | "USD" | Currency type (must be "USD") |
| `pathUSD_admin` | Genesis admin | Token admin address |
| `pathUSD_mint_amount` | Pre-funded | Initial mint amount |

### Fee Token Constraints

From memory (`tempo-gas-fee-constraints.md`):

1. **Currency must be "USD"** — Fee tokens must have `currency="USD"`
2. **FeeAMM for price discovery** — No oracle; AMM determines conversion rates
3. **30 bps fixed fee** — Protocol fee on swaps
4. **`--tempo.fee-token pathUSD` for cast** — Required for Foundry's `cast` tool
5. **Ledger workaround** — Omit `--tempo.fee-token` for type 2 transactions (EIP-1559)

---

## Fee Calculation Example

### Standard TIP-20 Transfer (T1+)

| Component | Value |
|-----------|-------|
| Gas used | ~50,000 |
| Base fee | 20,000,000,000 attodollars |
| Gas price | Base fee + priority fee |
| Total fee | 50,000 × 20B = 1,000,000,000,000,000 attodollars |
| In USD | 0.001 USD (1 millidollar) |

### Contract Deployment

| Component | Value |
|-----------|-------|
| Gas used | ~1,000,000 (varies) |
| Base fee | 20,000,000,000 attodollars |
| SSTORE creates | 10 × 250,000 = 2,500,000 gas |
| Total gas | ~3,500,000 |
| Total fee | 3.5M × 20B = 70,000,000,000,000,000 attodollars |
| In USD | 0.07 USD |

---

## Fee Recipient

### Validator Fee Recipient

Each validator specifies a `feeRecipient` address during registration. Block fees are sent to this address.

### Subblock Fee Recipient (Future)

The `TempoBlockExecutor` includes logic for subblock fee recipients, though subblocks are currently disabled (`with_subblocks: false`).

---

## Protocol Fees

The `ProtocolFeeManager` handles internal protocol fee hooks:
- Burns a portion of fees (deflationary)
- Funds protocol operations
- Configurable via governance

---

## Key Constraints

1. **Stablecoin-only fees** — Gas must be paid in USD stablecoins (pathUSD at launch)
2. **FeeAMM required** — No oracle; AMM provides price discovery
3. **30 bps fixed fee** — Protocol fee on AMM swaps
4. **T7 for dynamic base fee** — EIP-1559-style adjustment activates at T7
5. **Storage credits at T7** — TIP-1060 storage credit system activates at T7
6. **Ledger workaround** — Omit `--tempo.fee-token` for type 2 transactions
