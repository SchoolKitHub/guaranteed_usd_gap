# GMX V1 Vault - Guaranteed USD Accounting Gap (P6)

[![Foundry](https://img.shields.io/badge/Built%20with-Foundry-FFDB1C.svg)](https://getfoundry.sh/)
[![Solidity](https://img.shields.io/badge/Solidity-0.6.12-363636.svg)](https://soliditylang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## ⚠️ Critical Vulnerability POC

This repository contains a **Proof of Concept** demonstrating a critical accounting vulnerability in the GMX V1 Vault on Arbitrum. The vulnerability causes LP fund lockup due to near-100% pool utilization.

**Status:** 🔴 ACTIVE ON MAINNET (Block 426581531 and persists to current block)

---

## Quick Start

### Prerequisites

- [Foundry](https://getfoundry.sh/) installed
- Arbitrum RPC URL (Infura, Alchemy, or public)

### 1. Clone Repository

```bash
git clone https://github.com/SchoolKitHub/guaranteed_usd_gap.git
cd guaranteed_usd_gap
```

### 2. Install Dependencies

```bash
forge install
```

### 3. Set Environment Variables

```bash
export ARBITRUM_RPC_URL="https://arbitrum-mainnet.infura.io/v3/YOUR_KEY"
```

Or create a `.env` file:
```bash
echo 'ARBITRUM_RPC_URL=https://arbitrum-mainnet.infura.io/v3/YOUR_KEY' > .env
source .env
```

### 4. Run Tests

```bash
forge test --fork-url $ARBITRUM_RPC_URL --fork-block-number 426581531 -vvv
```

### Expected Output

```
Running 6 tests for test/GuaranteedUsdInsolvencyReplication.t.sol:GuaranteedUsdInsolvencyReplication
[PASS] test_1_VerifyInsolvencyBlockState() (gas: 62999)
[PASS] test_2_AnalyzeGuaranteedUsdConsistency() (gas: 127372)
[PASS] test_3_TestLPRedemptionAtStress() (gas: 110143)
[PASS] test_4_CalculateFinancialImpact() (gas: 1611358)
[PASS] test_5_SimulateLiquidationCascadeEffect() (gas: 79082)
[PASS] test_6_ProveInvariantViolationPath() (gas: 36235)

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 1.16s
```

---

## Vulnerability Summary

### The Problem

The GMX V1 Vault has non-atomic state updates between:
- `_decreaseGuaranteedUsd()` 
- `_decreaseReservedAmount()`

This allows `reservedAmounts` to approach `poolAmounts`, creating a state where:

```
Utilization = reservedAmounts / poolAmounts = 99.99999985%
```

### Impact

| Impact | Severity |
|--------|----------|
| Permanent freezing of LP funds | Critical |
| Protocol insolvency | Critical |
| GLP redemption failure | Critical |

### Live Evidence

```bash
# Verify on mainnet RIGHT NOW:
cast call 0x489ee077994B6658eAfA855C308275EAd8097C4A \
  "poolAmounts(address)(uint256)" 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1 \
  --rpc-url $ARBITRUM_RPC_URL
# Returns: 6575202314069967790 (6.575 ETH)

cast call 0x489ee077994B6658eAfA855C308275EAd8097C4A \
  "reservedAmounts(address)(uint256)" 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1 \
  --rpc-url $ARBITRUM_RPC_URL
# Returns: 6575202304473702214 (6.575 ETH - nearly identical!)
```

---

## Test Descriptions

| Test | Purpose |
|------|---------|
| `test_1_VerifyInsolvencyBlockState` | Confirms 99.99% WETH utilization at Block 426581531 |
| `test_2_AnalyzeGuaranteedUsdConsistency` | Checks guaranteedUsd accounting consistency |
| `test_3_TestLPRedemptionAtStress` | Proves LP redemption collateral is near-zero |
| `test_4_CalculateFinancialImpact` | Quantifies TVL at risk (~$1.09M) |
| `test_5_SimulateLiquidationCascadeEffect` | Models cascade drift in guaranteedUsd |
| `test_6_ProveInvariantViolationPath` | Proves system at invariant violation edge |

---

## File Structure

```
guaranteed_usd_gap/
├── README.md                           # This file
├── foundry.toml                        # Foundry configuration (solc 0.6.12)
├── GMX_GUARANTEED_USD_GAP_SUBMISSION.md # Full Immunefi submission
├── test/
│   └── GuaranteedUsdInsolvencyReplication.t.sol  # Main POC (6 tests)
└── lib/
    └── forge-std/                      # Foundry test library
```

---

## Contract Addresses (Arbitrum)

| Contract | Address |
|----------|---------|
| Vault | `0x489ee077994B6658eAfA855C308275EAd8097C4A` |
| GLP Manager | `0x321F653eED006AD1C29D174e17d96351BDe22649` |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |
| WBTC | `0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f` |

---

## Tenderly Simulations

Three simulations were saved at Block 426581531:

| Function | Simulation ID |
|----------|---------------|
| `poolAmounts(WETH)` | `609b4f40-dff5-4876-8f90-0ecf2003f232` |
| `reservedAmounts(WETH)` | `4a05f754-5a14-4bf5-ae3b-c1c5443a5147` |
| `getRedemptionCollateral(WETH)` | `9ca0f246-ada1-4d55-9fa4-b957f2594d58` |

---

## Remediation

1. **Add Utilization Circuit Breaker**
```solidity
require(reservedAmounts[_token] <= poolAmounts[_token].mul(95).div(100));
```

2. **Atomic State Updates**
```solidity
function _updatePositionState(address _token, uint256 _reserveDelta, uint256 _guaranteedDelta) private {
    reservedAmounts[_token] = reservedAmounts[_token].sub(_reserveDelta);
    guaranteedUsd[_token] = guaranteedUsd[_token].sub(_guaranteedDelta);
}
```

---

## Disclaimer

This POC is for **educational and responsible disclosure purposes only**. All testing was performed on local forks. No mainnet exploitation was performed.

---

## License

MIT
