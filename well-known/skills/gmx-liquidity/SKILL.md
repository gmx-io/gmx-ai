---
name: gmx-liquidity
description: Provide liquidity on GMX V2 — deposit into GM pools and GLV vaults to earn fees from trading activity on Arbitrum, Avalanche, and Botanix. Covers pool deposits, withdrawals, shifts between pools, and GLV vault operations via direct contract interaction.
license: MIT
metadata:
  author: gmx-io
  version: "0.1"
  chains: "arbitrum, avalanche, botanix"
---

# GMX Liquidity Skill

## Overview

GMX V2 liquidity providers deposit tokens into **GM pools** (individual market pools) or **GLV vaults** (diversified multi-pool vaults) to earn trading fees, borrowing fees, and funding fees.

**Pool types:**
- **GM pools** — Single-market liquidity pools backing one trading pair (e.g., ETH/USD). LPs receive GM tokens representing their share. Each pool holds a long token and a short token (typically a stablecoin).
- **GLV vaults** — Diversified vaults that auto-allocate across multiple GM pools. LPs receive GLV tokens. GLV rebalances between underlying pools to optimize yield.

**Supported chains:** Arbitrum (42161), Avalanche (43114), Botanix (3637)

**Integration path:** Direct contract calls via viem. LP operations are not yet in `@gmx-io/sdk` — use ExchangeRouter for GM operations and GlvRouter for GLV operations. ABIs are available from `@gmx-io/sdk/abis`.

**Two-phase execution:** Like trading orders, LP operations follow create → execute. The user submits a deposit/withdrawal, then a keeper executes it with oracle prices (1–5 seconds).

## GM Pool Deposits

Deposit long tokens, short tokens, or both into a GM pool to mint GM tokens.

### Transaction Pattern

All LP transactions use a multicall pattern on ExchangeRouter:

```typescript
const { encodeFunctionData } = require("viem");
const { ExchangeRouterAbi } = require("@gmx-io/sdk/abis");

// Step 1: Send execution fee (native token) to DepositVault
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendWnt",
  args: [depositVaultAddress, executionFee + nativeTokenDepositAmount],
});

// Step 2: Send ERC20 tokens to DepositVault (if not native)
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendTokens",
  args: [tokenAddress, depositVaultAddress, amount],
});

// Step 3: Create deposit order
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "createDeposit",
  args: [{
    addresses: {
      receiver: accountAddress,
      callbackContract: "0x0000000000000000000000000000000000000000",
      uiFeeReceiver: "0x0000000000000000000000000000000000000000",
      market: marketTokenAddress,
      initialLongToken: longTokenAddress,
      initialShortToken: shortTokenAddress,
      longTokenSwapPath: [],
      shortTokenSwapPath: [],
    },
    minMarketTokens: 0n,      // Minimum GM tokens to receive (set for slippage protection)
    shouldUnwrapNativeToken: false,
    executionFee: executionFee,
    callbackGasLimit: 0n,
    dataList: [],
  }],
});
```

### CreateDepositParams

```typescript
type CreateDepositParams = {
  addresses: {
    receiver: string;              // Who receives the GM tokens
    callbackContract: string;      // zeroAddress for no callback
    uiFeeReceiver: string;         // zeroAddress for no UI fee
    market: string;                // GM market token address
    initialLongToken: string;      // Long token to deposit
    initialShortToken: string;     // Short token to deposit
    longTokenSwapPath: string[];   // Swap route (empty for direct)
    shortTokenSwapPath: string[];  // Swap route (empty for direct)
  };
  minMarketTokens: bigint;         // Slippage protection
  shouldUnwrapNativeToken: boolean; // Unwrap WETH to ETH on refund
  executionFee: bigint;            // Keeper gas cost in native token
  callbackGasLimit: bigint;        // 0n for no callback
  dataList: string[];              // Empty array []
};
```

### Key Details

- **Both tokens optional:** You can deposit only long token, only short token, or both
- **Native token handling:** If depositing ETH/AVAX/BTC, add the amount to `sendWnt` value instead of `sendTokens`
- **`minMarketTokens`:** Set this based on expected GM tokens minus slippage. Use `0n` only for testing.
- **Swap paths:** Empty arrays for direct deposits. Populate if depositing a token that isn't the pool's long/short token (routes through other markets).

## GM Pool Withdrawals

Burn GM tokens to receive back long and short tokens from the pool.

### Transaction Pattern

```typescript
// Step 1: Send execution fee to WithdrawalVault
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendWnt",
  args: [withdrawalVaultAddress, executionFee],
});

// Step 2: Send GM tokens to WithdrawalVault
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendTokens",
  args: [marketTokenAddress, withdrawalVaultAddress, marketTokenAmount],
});

// Step 3: Create withdrawal order
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "createWithdrawal",
  args: [{
    addresses: {
      receiver: accountAddress,
      callbackContract: "0x0000000000000000000000000000000000000000",
      uiFeeReceiver: "0x0000000000000000000000000000000000000000",
      market: marketTokenAddress,
      longTokenSwapPath: [],
      shortTokenSwapPath: [],
    },
    minLongTokenAmount: 0n,       // Minimum long tokens to receive
    minShortTokenAmount: 0n,      // Minimum short tokens to receive
    shouldUnwrapNativeToken: true, // Unwrap WETH to ETH
    executionFee: executionFee,
    callbackGasLimit: 0n,
    dataList: [],
  }],
});
```

### CreateWithdrawalParams

```typescript
type CreateWithdrawalParams = {
  addresses: {
    receiver: string;
    callbackContract: string;
    uiFeeReceiver: string;
    market: string;                // GM market token address
    longTokenSwapPath: string[];
    shortTokenSwapPath: string[];
  };
  minLongTokenAmount: bigint;      // Slippage protection for long token
  minShortTokenAmount: bigint;     // Slippage protection for short token
  shouldUnwrapNativeToken: boolean;
  executionFee: bigint;
  callbackGasLimit: bigint;
  dataList: string[];
};
```

## Shifts (Move Liquidity Between Pools)

Transfer GM tokens from one pool to another in a single operation, avoiding separate withdraw + deposit.

### Transaction Pattern

```typescript
// Step 1: Send execution fee to ShiftVault
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendWnt",
  args: [shiftVaultAddress, executionFee],
});

// Step 2: Send source GM tokens to ShiftVault
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "sendTokens",
  args: [fromMarketTokenAddress, shiftVaultAddress, fromMarketTokenAmount],
});

// Step 3: Create shift order
encodeFunctionData({
  abi: ExchangeRouterAbi,
  functionName: "createShift",
  args: [{
    addresses: {
      receiver: accountAddress,
      callbackContract: "0x0000000000000000000000000000000000000000",
      uiFeeReceiver: "0x0000000000000000000000000000000000000000",
      fromMarket: fromMarketTokenAddress,
      toMarket: toMarketTokenAddress,
    },
    minMarketTokens: minToMarketTokenAmount,  // Slippage-adjusted
    executionFee: executionFee,
    callbackGasLimit: 0n,
    dataList: [],
  }],
});
```

**Shift benefits:** No swap fees (unlike withdraw + deposit), single transaction, atomic execution.

## GLV Vault Deposits

Deposit tokens into a GLV vault. Supports two modes: deposit underlying tokens (long/short) or deposit GM market tokens directly.

### Transaction Pattern (Token Deposit)

Uses GlvRouter instead of ExchangeRouter:

```typescript
const { GlvRouterAbi } = require("@gmx-io/sdk/abis");

// Step 1: Send execution fee + native deposit to GlvVault (via DepositVault)
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "sendWnt",
  args: [depositVaultAddress, executionFee + nativeTokenAmount],
});

// Step 2: Send ERC20 tokens to DepositVault
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "sendTokens",
  args: [tokenAddress, depositVaultAddress, amount],
});

// Step 3: Create GLV deposit
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "createGlvDeposit",
  args: [{
    addresses: {
      glv: glvTokenAddress,
      market: underlyingMarketAddress,
      receiver: accountAddress,
      callbackContract: "0x0000000000000000000000000000000000000000",
      uiFeeReceiver: "0x0000000000000000000000000000000000000000",
      initialLongToken: longTokenAddress,
      initialShortToken: shortTokenAddress,
      longTokenSwapPath: [],
      shortTokenSwapPath: [],
    },
    minGlvTokens: 0n,
    executionFee: executionFee,
    callbackGasLimit: 0n,
    shouldUnwrapNativeToken: false,
    isMarketTokenDeposit: false,   // Depositing underlying tokens
    dataList: [],
  }],
});
```

### Market Token Deposit Mode

Deposit existing GM tokens into a GLV vault:

```typescript
// Set isMarketTokenDeposit: true
// Send GM tokens instead of long/short tokens:
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "sendTokens",
  args: [marketTokenAddress, depositVaultAddress, marketTokenAmount],
});
```

### CreateGlvDepositParams

```typescript
type CreateGlvDepositParams = {
  addresses: {
    glv: string;                   // GLV token address
    market: string;                // Underlying GM market address
    receiver: string;
    callbackContract: string;
    uiFeeReceiver: string;
    initialLongToken: string;
    initialShortToken: string;
    longTokenSwapPath: string[];
    shortTokenSwapPath: string[];
  };
  minGlvTokens: bigint;           // Slippage protection
  executionFee: bigint;
  callbackGasLimit: bigint;
  shouldUnwrapNativeToken: boolean;
  isMarketTokenDeposit: boolean;   // true = deposit GM tokens, false = deposit underlying
  dataList: string[];
};
```

## GLV Vault Withdrawals

Burn GLV tokens to receive underlying tokens.

### Transaction Pattern

```typescript
// Step 1: Send execution fee to WithdrawalVault
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "sendWnt",
  args: [withdrawalVaultAddress, executionFee],
});

// Step 2: Send GLV tokens to WithdrawalVault
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "sendTokens",
  args: [glvTokenAddress, withdrawalVaultAddress, glvTokenAmount],
});

// Step 3: Create GLV withdrawal
encodeFunctionData({
  abi: GlvRouterAbi,
  functionName: "createGlvWithdrawal",
  args: [{
    addresses: {
      receiver: accountAddress,
      callbackContract: "0x0000000000000000000000000000000000000000",
      uiFeeReceiver: "0x0000000000000000000000000000000000000000",
      market: underlyingMarketAddress,
      glv: glvTokenAddress,
      longTokenSwapPath: [],
      shortTokenSwapPath: [],
    },
    minLongTokenAmount: 0n,
    minShortTokenAmount: 0n,
    shouldUnwrapNativeToken: true,
    executionFee: executionFee,
    callbackGasLimit: 0n,
    dataList: [],
  }],
});
```

## Cancellations

Cancel pending deposit, withdrawal, or shift operations using their order key:

```typescript
// GM operations via ExchangeRouter
await exchangeRouter.write.cancelDeposit([depositKey]);
await exchangeRouter.write.cancelWithdrawal([withdrawalKey]);
await exchangeRouter.write.cancelShift([shiftKey]);

// GLV operations via GlvRouter
await glvRouter.write.cancelGlvDeposit([depositKey]);
await glvRouter.write.cancelGlvWithdrawal([withdrawalKey]);
```

## Fees

**Deposit/Withdrawal swap fees:**
- Pool-balancing deposit: **0.05%** (improves long/short ratio)
- Pool-imbalancing deposit: **0.07%**
- Stablecoin pools: **0.005%** / **0.02%**
- Shifts between pools: **no swap fees**

**Fee calculation:**
- Fees are deducted from the deposit amount before minting GM/GLV tokens
- Withdrawal fees are deducted from the received token amounts
- Price impact applies when deposits change the pool's long/short balance

**Execution fee:**
- Same as trading: covers keeper gas, paid in native token, surplus refunded

## GM Token Pricing

GM token price is derived from pool value:

```
GM Price = Pool Value / GM Total Supply
Pool Value = (Long Token Amount × Long Price) + (Short Token Amount × Short Price) + Pending PnL
```

To estimate GM tokens received from a deposit:

```
GM Tokens = (Deposit Value in USD × GM Supply) / Pool Value
```

## Reading Pool Data

Use the SDK to fetch market and pool information:

```typescript
const { GmxSdk } = require("@gmx-io/sdk");

const sdk = new GmxSdk({ chainId: 42161, rpcUrl, oracleUrl, subsquidUrl });
const { marketsInfoData, tokensData } = await sdk.markets.getMarketsInfo();

// Each market in marketsInfoData contains:
// - longPoolAmount, shortPoolAmount (pool liquidity)
// - poolValueMax (total pool value in USD, 30 decimals)
// - longInterestUsd, shortInterestUsd (open interest)
// - swapFeeFactorForBalanceWasImproved, swapFeeFactorForBalanceWasNotImproved
// - borrowingFactorPerSecondForLongs, borrowingFactorPerSecondForShorts
```

Use the REST API for pool APY:

```
GET https://gmx-api-arbitrum.gmx.io/api/v1/apy
GET https://gmx-api-arbitrum.gmx.io/api/v1/rates
```

## Contract Addresses

### ExchangeRouter (GM operations)

| Chain | ExchangeRouter | DepositVault | WithdrawalVault | ShiftVault |
|-------|---------------|--------------|-----------------|------------|
| Arbitrum | `0x1C3fa76e6E1088bCE750f23a5BFcffa1efEF6A41` | `0xF89e77e8Dc11691C9e8757e84aaFbCD8A67d7A55` | `0x0628D46b5D145f183AdB6Ef1f2c97eD1C4701C55` | `0xfe99609C4AA83ff6816b64563Bdffd7fa68753Ab` |
| Avalanche | `0x8f550E53DFe96C055D5Bdb267c21F268fCAF63B2` | `0x90c670825d0C62ede1c5ee9571d6d9a17A722DFF` | `0xf5F30B10141E1F63FC11eD772931A8294a591996` | `0x7fC46CCb386e9bbBFB49A2639002734C3Ec52b39` |
| Botanix | `0xBCB5eA3a84886Ce45FBBf09eBF0e883071cB2Dc8` | `0x4D12C3D3e750e051e87a2F3f7750fBd94767742c` | `0x46BAeAEdbF90Ce46310173A04942e2B3B781Bf0e` | `0xa7EE2737249e0099906cB079BCEe85f0bbd837d4` |

### GlvRouter (GLV operations)

| Chain | GlvRouter | GlvReader | GlvVault |
|-------|-----------|-----------|----------|
| Arbitrum | `0x7EAdEE2ca1b4D06a0d82fDF03D715550c26AA12F` | `0x2C670A23f1E798184647288072e84054938B5497` | `0x393053B58f9678C9c28c2cE941fF6cac49C3F8f9` |
| Avalanche | `0x7E425c47b2Ff0bE67228c842B9C792D0BCe58ae6` | `0x5C6905A3002f989E1625910ba1793d40a031f947` | `0x527FB0bCfF63C47761039bB386cFE181A92a4701` |
| Botanix | `0xC92741F0a0D20A95529873cBB3480b1f8c228d9F` | `0x955Aa50d2ecCeffa59084BE5e875eb676FfAFa98` | `0xd336087512BeF8Df32AF605b492f452Fd6436CD8` |

## Limitations

- **No SDK helpers:** Unlike trading (`sdk.orders.long()`), LP operations require manual multicall construction with viem
- **No atomic deposits:** Deposits always go through the two-phase keeper flow
- **Atomic withdrawals exist** (`executeAtomicWithdrawal`) but require signed oracle prices — not practical for external integrators
- **GLV vault selection:** GLV deposits route through a specific underlying GM market — the `market` address in params determines which pool receives the tokens before GLV allocation

## References

- [Contract Addresses](references/contract-addresses.md) — Full address tables per chain
- [GMX Documentation](https://docs.gmx.io) — Official protocol docs
- [GMX App — Earn](https://app.gmx.io/#/pools) — LP interface
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk) — SDK with ABIs
- Source: `gmx-interface/src/domain/synthetics/markets/create*Txn.ts`
