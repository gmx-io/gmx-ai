---
name: gmx-trading
description: Trade perpetuals and swap tokens on GMX V2 — a decentralized exchange with oracle-based pricing on Arbitrum, Avalanche, and Botanix. Supports market/limit/stop orders, leverage up to 100x, and programmable position management via TypeScript SDK or REST API.
license: MIT
metadata:
  author: gmx-io
  version: "0.1"
  chains: "arbitrum, avalanche, botanix"
---

# GMX Trading Skill

## Overview

GMX V2 is a decentralized perpetual and spot exchange using oracle-based pricing (Chainlink Data Streams) instead of AMM curves. Traders get CEX-like execution with onchain settlement.

**Supported chains:**
| Chain | Chain ID | Native Token |
|-------|----------|-------------|
| Arbitrum | 42161 | ETH |
| Avalanche | 43114 | AVAX |
| Botanix | 3637 | BTC |

**Two integration paths:**
- **SDK** (`@gmx-io/sdk`) — Full read + write: fetch markets, create orders, manage positions
- **REST API** — Read-only: prices, markets, positions, trade history

**Trading modes:**
- **Classic** — User signs each transaction directly
- **Express** — Gelato relay pays gas, user signs EIP-712 message (frontend only)
- **Express + One-Click** — Subaccount delegates signing for instant execution (frontend only)

## SDK Quick Start

Install the SDK:

```bash
npm install @gmx-io/sdk viem
```

> **Import note:** The SDK's ESM build has broken imports (missing file extensions). Use CommonJS require or configure your bundler to resolve extensionless imports. In Node.js scripts, use `const { GmxSdk } = require("@gmx-io/sdk")`.

Create an SDK instance:

```typescript
const { GmxSdk } = require("@gmx-io/sdk");

const sdk = new GmxSdk({
  chainId: 42161,
  rpcUrl: "https://arb1.arbitrum.io/rpc",
  oracleUrl: "https://arbitrum-api.gmxinfra.io",
  subsquidUrl: "https://gmx.squids.live/gmx-synthetics-arbitrum:prod/api/graphql",
});
```

**Chain configuration:**

| Chain | chainId | oracleUrl | subsquidUrl |
|-------|---------|-----------|-------------|
| Arbitrum | 42161 | `https://arbitrum-api.gmxinfra.io` | `https://gmx.squids.live/gmx-synthetics-arbitrum:prod/api/graphql` |
| Avalanche | 43114 | `https://avalanche-api.gmxinfra.io` | `https://gmx.squids.live/gmx-synthetics-avalanche:prod/api/graphql` |
| Botanix | 3637 | `https://botanix-api.gmxinfra.io` | `https://gmx.squids.live/gmx-synthetics-botanix:prod/api/graphql` |

**Set up a wallet for write operations:**

```typescript
const { createWalletClient, http } = require("viem");
const { privateKeyToAccount } = require("viem/accounts");
const { arbitrum } = require("viem/chains");

const account = privateKeyToAccount("0xYOUR_PRIVATE_KEY");
const walletClient = createWalletClient({
  account,
  chain: arbitrum,
  transport: http("https://arb1.arbitrum.io/rpc"),
});

const sdk = new GmxSdk({
  chainId: 42161,
  rpcUrl: "https://arb1.arbitrum.io/rpc",
  oracleUrl: "https://arbitrum-api.gmxinfra.io",
  subsquidUrl: "https://gmx.squids.live/gmx-synthetics-arbitrum:prod/api/graphql",
  account: account.address,
  walletClient,
});
```

**Lightweight alternative (read-only, no RPC needed):**

```typescript
const { GmxApiSdk } = require("@gmx-io/sdk/v2");

const apiSdk = new GmxApiSdk({ chainId: 42161 });
const { marketsInfoData } = await apiSdk.fetchMarketsInfo();
```

## Order Helpers

The SDK provides convenience methods that handle amount calculation and transaction submission automatically.

> **Important:** Never hardcode market or token addresses. Always fetch them dynamically — addresses differ per chain and can change between deployments.

### Step 1: Resolve market and token addresses

```typescript
// Fetch all markets and tokens
const { marketsInfoData, tokensData } = await sdk.markets.getMarketsInfo();

// Find ETH/USD market by index token symbol
const ethUsdMarket = Object.values(marketsInfoData).find(
  (m) => tokensData[m.indexTokenAddress]?.symbol === "ETH" && !m.isSpotOnly
);

// Get token addresses from market
const marketAddress = ethUsdMarket.marketTokenAddress;
const longToken = ethUsdMarket.longTokenAddress;   // e.g., WETH
const shortToken = ethUsdMarket.shortTokenAddress;  // e.g., USDC

// Find a token by symbol
const usdcAddress = Object.values(tokensData).find((t) => t.symbol === "USDC")?.address;
```

### Step 2: Place orders

**Open a long position:**

```typescript
await sdk.orders.long({
  marketAddress,                         // from step 1
  payTokenAddress: usdcAddress,          // token you're paying with
  collateralTokenAddress: longToken,     // WETH as collateral for longs
  payAmount: 100000000n,                 // 100 USDC (6 decimals)
  leverage: 50000n,                      // 5x leverage (basis points)
  allowedSlippageBps: 100,               // 1% slippage
});
```

**Open a short position:**

```typescript
await sdk.orders.short({
  marketAddress,
  payTokenAddress: usdcAddress,
  collateralTokenAddress: shortToken,    // USDC as collateral for shorts
  payAmount: 100000000n,
  leverage: 50000n,
});
```

**Swap tokens:**

```typescript
const arbAddress = Object.values(tokensData).find((t) => t.symbol === "ARB")?.address;
const linkAddress = Object.values(tokensData).find((t) => t.symbol === "LINK")?.address;

await sdk.orders.swap({
  fromTokenAddress: arbAddress,
  toTokenAddress: linkAddress,
  fromAmount: 1000000000000000000n, // 1 ARB (18 decimals)
  allowedSlippageBps: 100,
});
```

**Limit orders** — add `limitPrice` for positions or `triggerPrice` for swaps:

```typescript
await sdk.orders.long({
  marketAddress,
  payTokenAddress: usdcAddress,
  collateralTokenAddress: longToken,
  payAmount: 100000000n,
  leverage: 50000n,
  limitPrice: 3000000000000000000000000000000000n, // $3000 (30 decimals)
});
```

**Key parameters:**
- `leverage` — In basis points: `10000n` = 1x, `50000n` = 5x, `1000000n` = 100x
- `allowedSlippageBps` — Default `100` (1%). Range: 1-500
- `payAmount` — Pay this much collateral. Alternative: use `sizeAmount` to specify position size
- `fromAmount` / `toAmount` — For swaps, specify input or desired output amount

## SDK Modules

| Module | Key Methods | Description |
|--------|------------|-------------|
| `sdk.markets` | `getMarkets()`, `getMarketsInfo()`, `getDailyVolumes()` | Market data and liquidity info |
| `sdk.tokens` | `getTokens()`, `getTokenBalances()` | Token metadata, prices, balances |
| `sdk.positions` | `getPositions()`, `getPositionsInfo()`, `getPositionsConstants()` | Open position data |
| `sdk.orders` | `long()`, `short()`, `swap()`, `getOrders()`, `cancelOrders()` | Order creation and management |
| `sdk.trades` | `getTradeHistory()` | Historical trade actions |
| `sdk.utils` | `getGasLimits()`, `getGasPrice()`, `getExecutionFee()` | Gas and fee estimation |
| `sdk.oracle` | `getTickers()`, `getMarkets()`, `getTokens()` | Direct oracle data access |

**Typical read flow:**

```typescript
const { marketsInfoData, tokensData } = await sdk.markets.getMarketsInfo();
const { positionsInfoData } = await sdk.positions.getPositionsInfo({
  marketsInfoData, tokensData, showPnlInLeverage: false,
});
const { ordersInfoData } = await sdk.orders.getOrders({
  marketsInfoData, tokensData,
});
```

**Write flow — use convenience helpers or low-level methods:**

```typescript
// High-level (recommended)
await sdk.orders.long(params);
await sdk.orders.short(params);
await sdk.orders.swap(params);

// Low-level (full control)
await sdk.orders.createIncreaseOrder({ ... });
await sdk.orders.createDecreaseOrder({ ... });
await sdk.orders.createSwapOrder({ ... });
```

## Order Types

| Type | Enum | Behavior |
|------|------|----------|
| Market | `MarketSwap(0)`, `MarketIncrease(2)`, `MarketDecrease(4)` | Execute immediately at current oracle price |
| Limit | `LimitSwap(1)`, `LimitIncrease(3)`, `LimitDecrease(5)` | Execute when oracle price reaches trigger price |
| Stop Increase | `StopIncrease(8)` | Open position when price moves past trigger (breakout entry) |
| Stop-Loss | `StopLossDecrease(6)` | Auto-close position to limit losses |
| Liquidation | `Liquidation(7)` | System-triggered when position falls below maintenance margin |

**Trigger conditions:**
- Long Limit: oracle price <= trigger price (buy the dip)
- Long Stop-Loss: oracle price <= trigger price (exit on drop)
- Long Take-Profit (LimitDecrease): oracle price >= trigger price (exit on rise)
- Short Limit: oracle price >= trigger price (sell the rally)
- Short Stop-Loss: oracle price >= trigger price (exit on rise)
- Short Take-Profit (LimitDecrease): oracle price <= trigger price (exit on drop)

**Auto-cancel limits:** Maximum concurrent auto-cancel orders per position: 11 on Arbitrum, 6 on Avalanche and Botanix.

**Sidecar orders:** Stop-loss and take-profit orders can be attached to increase orders via `createSltpEntries`, `cancelSltpEntries`, and `updateSltpEntries` parameters in `createIncreaseOrder()`.

**TWAP orders:** Split a large order into 2–30 parts executed over a configurable duration. TWAP utilities (`getTwapDurationInSeconds`, `getIsValidTwapParams`) are exported from `@gmx-io/sdk/utils/twap` but full TWAP order creation is only available via the frontend UI.

## Fees

**Position fees:**
- Opening/closing: **0.04%** if the trade balances long/short OI, **0.06%** if it imbalances
- Applied to position size (notional value)

**Swap fees:**
- Standard pairs: **0.05%** (balancing) / **0.07%** (imbalancing)
- Stablecoin pairs: **0.005%** (balancing) / **0.02%** (imbalancing)

**Funding rate:**
- Adaptive rate that flows from the larger open interest side to the smaller side
- Rebalances long/short exposure over time
- Accrues continuously, settled on position changes

**Borrowing rate:**
- Kink model: low rate below utilization threshold, steep above
- Approximately 45–55% APR at 75% utilization
- Paid by all positions proportional to size

**Execution fee:**
- Covers keeper gas cost for executing the order onchain
- Paid upfront in native token (ETH/AVAX/BTC)
- Surplus refunded after execution
- Use `sdk.utils.getExecutionFee()` to estimate

## REST API

### Oracle Endpoints

Base URL: `https://{network}-api.gmxinfra.io`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/prices/tickers` | GET | Current min/max prices for all tokens |
| `/prices/candles` | GET | OHLC price candles (`?tokenSymbol=ETH&period=1h`) |
| `/signed_prices/latest` | GET | Signed oracle prices for order execution |
| `/tokens` | GET | Token list with addresses and decimals |
| `/markets` | GET | Market configuration (index/long/short tokens) |
| `/markets/info` | GET | Extended market info with pool sizes and utilization |

### OpenAPI Endpoints

Base URL: `https://gmx-api-{network}.gmx.io/api/v1`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/markets` | GET | Market metadata |
| `/tickers` | GET | Current prices per market |
| `/tokens` | GET | Token information |
| `/positions?account={addr}` | GET | Open positions for an account |
| `/orders?account={addr}` | GET | Pending orders for an account |
| `/rates` | GET | Funding and borrowing rates |
| `/apy` | GET | Pool APY data |
| `/performance?account={addr}` | GET | Account trading performance |

Swagger spec: `{base_url}/swagger.json`

Network slugs: `arbitrum`, `avalanche`, `botanix`

### GraphQL (Subsquid)

Base URL: `https://gmx.squids.live/gmx-synthetics-{network}:prod/api/graphql`

Example — fetch recent trade actions:

```graphql
query {
  tradeActions(
    where: { account_eq: "0x..." }
    orderBy: transaction_timestamp_DESC
    limit: 10
  ) {
    id
    eventName
    orderType
    sizeDeltaUsd
    transaction { timestamp, hash }
  }
}
```

### Fallback URLs

| Chain | Primary | Fallback 1 | Fallback 2 |
|-------|---------|------------|------------|
| Arbitrum | `arbitrum-api.gmxinfra.io` | `arbitrum-api-fallback.gmxinfra.io` | `arbitrum-api-fallback.gmxinfra2.io` |
| Avalanche | `avalanche-api.gmxinfra.io` | `avalanche-api-fallback.gmxinfra.io` | `avalanche-api-fallback.gmxinfra2.io` |
| Botanix | `botanix-api.gmxinfra.io` | `botanix-api-fallback.gmxinfra.io` | `botanix-api-fallback.gmxinfra2.io` |

## Key Concepts

**Oracle-based pricing:** GMX does not use an AMM. Prices come from Chainlink Data Streams, giving traders zero-slippage execution at the oracle price (subject to price impact from pool utilization).

**Two-phase execution:** Orders follow a create → execute pattern. The user submits an order transaction, then a keeper executes it with fresh oracle prices. This typically takes 1–5 seconds.

**BigInt amounts:** All amounts use BigInt. Prices are scaled to 30 decimals (`1 USD = 10^30`). Token amounts use their native decimals (e.g., USDC = 6, ETH = 18).

**Multicall batching:** The SDK batches RPC calls automatically. Production chains use `batchSize: 1024 * 1024` bytes per multicall with no waiting. This is configured per-chain in `BATCH_CONFIGS`.

**GMX Account:** Cross-chain trading from Ethereum, Base, or BNB Chain via LayerZero/Stargate bridge. Users can trade on Arbitrum/Avalanche without bridging manually.

**Subaccounts:** Delegate trading to a subaccount address for one-click trading. The subaccount can execute orders without requiring the main wallet signature each time.

## Limitations

- **GM pool deposits/withdrawals:** Not available in SDK. Use the app at `app.gmx.io` or wait for SDK v2.
- **GLV vault operations:** Not available in SDK.
- **Express orders:** Frontend-only via Gelato relay. Not exposed in the SDK.
- **TWAP orders:** Utility functions available (`@gmx-io/sdk/utils/twap`) but no SDK method to create TWAP orders programmatically.
- **Order updates:** Orders cannot be modified. Cancel and recreate instead (`sdk.orders.cancelOrders()`).
- **Trade simulation:** The `skipSimulation` parameter exists but simulation is deprecated. Set `skipSimulation: true`.

## References

- [SDK Reference](references/sdk-reference.md) — Full module and method documentation
- [API Endpoints](references/api-endpoints.md) — Oracle, OpenAPI, and GraphQL endpoint details
- [Contract Addresses](references/contract-addresses.md) — Deployed contracts per chain
- [Order Types](references/order-types.md) — Detailed order type behavior and trigger logic
- [GMX Documentation](https://docs.gmx.io) — Official protocol documentation
- [GMX App](https://app.gmx.io) — Trading interface
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk) — SDK package
- [gmx-io/gmx-interface](https://github.com/gmx-io/gmx-interface) — Frontend source code
