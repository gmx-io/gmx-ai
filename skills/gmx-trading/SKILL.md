---
name: gmx-trading
description: Trade GMX perpetuals and swap tokens using GmxApiSdk. Use for market, limit, stop, TP/SL and TWAP orders, position management, and market or account reads on Arbitrum, Avalanche, and MegaETH.
license: MIT
metadata:
  author: gmx-io
  version: "0.3.0"
  chains: "arbitrum, avalanche, megaeth"
---

# GMX Trading

Use `GmxApiSdk` from `@gmx-io/sdk/v2` for new integrations. It reads the GMX API and prepares trading transactions; Express orders use EIP-712 signatures and GMX Relay, while Classic orders are sent by the user's wallet. SDK v1 (`GmxSdk`) is in maintenance and is not the default for this skill.

Validated against published `@gmx-io/sdk@2.1.1` on 2026-09-29. Install the package, not the `/v2` import subpath:

```bash
npm install @gmx-io/sdk@latest viem
npm ls @gmx-io/sdk
```

Recheck the installed types and [SDK docs](https://docs.gmx.io/docs/sdk/v2/) when using a newer version. The SDK requires Node.js >=18. In 2.1.1, native Node ESM encounters extensionless internal imports; use `.cjs` for plain Node scripts, or compile TypeScript to CommonJS. TypeScript examples in the references use that compilation model.

## Start with reads

No wallet, RPC, oracle URL, or Subsquid URL is required:

```javascript
// Save as markets.cjs and run: node markets.cjs
const { GmxApiSdk } = require("@gmx-io/sdk/v2");

async function main() {
  const sdk = new GmxApiSdk({ chainId: 42161 });
  const markets = await sdk.fetchMarkets();
  console.table(markets.map((m) => ({
    symbol: m.symbol,
    address: m.marketTokenAddress,
    spotOnly: m.isSpotOnly,
  })));
}

main().catch((error) => { console.error(error); process.exitCode = 1; });
```

| Network | Chain ID | Native gas token | API host |
|---------|----------|------------------|----------|
| Arbitrum | 42161 | ETH | `https://arbitrum.gmxapi.io` |
| Avalanche | 43114 | AVAX | `https://avalanche.gmxapi.io` |
| MegaETH | 4326 | ETH | `https://megaeth.gmxapi.io` |

## Trading workflow

1. Establish the user's chain, account, exact market, direction, size, collateral, order type, and slippage. Discover markets with `fetchMarkets()` and tokens with `fetchTokens()`. Use the returned full market `symbol`, including its pool suffix; the same index can have multiple pools. Preserve address casing.
2. Fetch current positions/orders and wallet balances. Check the market's leverage tiers and minimums. For an increase, use `getTradingCapacity({ symbol, direction })` and its freshness/JIT flags; legacy `availableLiquidityLong/Short` fields exclude JIT capacity.
3. For ERC-20 input, check `fetchAllowances({ address, spender: "router" })`. `buildApproveTransaction()` builds approval calldata for `SyntheticsRouter`; submitting it requires a wallet and a confirmed receipt. Allowance, spend balance, and gas payment balance are separate checks.
4. Call `prepareOrder()` (or the edit, cancel, or collateral preparation method). Review size, fee and price-impact estimates, warnings, expiry, account, chain, and transaction/signing payload against the requested action. Preparation does not execute a trade. Honor the user's existing authorization; preparation alone grants no permission to sign or submit.
5. For Express, call `signOrder()` and `submitOrder()` on the same SDK instance, preserving the request ID, idempotency key and prepared payload. For Classic, send the returned transaction through the wallet. See [SDK reference](references/sdk-reference.md) for both flows.
6. Track the result. Relay acceptance or a successful creation transaction is not keeper execution. Conditional orders normally rest at `created`; market orders require `executed`. Check every returned order key for TP/SL or TWAP, then refresh account state.

Do not prepare a replacement trade just because submission timed out. Reconcile the original request/transaction first. Report an unresolved timeout as pending, not failed or completed.

## Units and execution

- Increase/decrease `size` and API `triggerPrice` use 30-decimal USD `bigint`. `size` is not token quantity or leverage. Derive the notional from the user's collateral/leverage request and inspect the prepared result, including fees and any existing position.
- `collateralToPay.amount`, swap `size`, and collateral deposit/withdrawal `amount` use the relevant token's decimals. `slippage` is a number in basis points: `30` means 0.3%.
- SDK responses mix `bigint`, numbers, strings and nullable analytics. Use each method's declared type. Never run large amounts through JavaScript `Number`; stringify bigint explicitly when logging JSON.
- Prices, fees, capacity, and supported leverage change. Use live data and preparation estimates instead of fixed fee tables, a universal leverage cap, or automatic slippage increases.
- A decrease identifies a position by account, market, side, and collateral token. Re-fetch it immediately before preparing a close. Full close uses the current `sizeInUsd`; partial close must respect remaining-position minimums.

## References

- [SDK reference](references/sdk-reference.md): signer setup, reads, approval, prepare/sign/submit, closing, editing, cancellation, collateral and subaccounts.
- [Order types](references/order-types.md): trigger directions, TP/SL, TWAP, and unit differences.
- [API endpoints](references/api-endpoints.md): HTTP routes, peer hosts, retries, and raw JSON representation.
- [Contract addresses](references/contract-addresses.md): current routers/readers/vaults and runtime address lookup.
- [Official order examples](https://docs.gmx.io/docs/sdk/v2/examples/): additional supported workflows, including GMX Account funding. GMX Account funding is not GM/GLV liquidity provision.
