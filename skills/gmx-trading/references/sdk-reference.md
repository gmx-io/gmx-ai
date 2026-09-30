# GmxApiSdk reference

Checked against the published `@gmx-io/sdk@2.1.2` declarations and implementation. [SDK v2 docs](https://docs.gmx.io/docs/sdk/v2/), [order examples](https://docs.gmx.io/docs/sdk/v2/examples/), and [source](https://github.com/gmx-io/gmx-interface/tree/release/sdk/src/clients/v2) provide the maintained upstream reference.

## Setup and signer

Use the read-only client without a signer. Prefer an existing wallet adapter implementing `IAbstractSigner` for writes. The optional `PrivateKeySigner` example below is for a user who explicitly selected local signing: inject the key through their secret manager into an isolated signing process, never through chat, shell arguments, generated source or model-visible output. Do not read `.env` or wallet files to discover a key. RPC is only needed for on-chain approvals, Classic orders and contract reads; Express signing alone needs no RPC.

The snippets below share this setup and use `await` inside an async function. Compile TypeScript to CommonJS, or use the equivalent `require()` imports in `.cjs` scripts.

```typescript
import {
  GmxApiSdk, PrivateKeySigner, parsePrepareOrderError,
  type PrepareOrderRequest, type PrepareOrderResponse,
} from "@gmx-io/sdk/v2";
import { getContract } from "@gmx-io/sdk/configs/contracts";
import { createPublicClient, http, parseUnits, type Hex } from "viem";
import { arbitrum } from "viem/chains";

const chainId = 42161;
const sdk = new GmxApiSdk({ chainId });
const privateKey = process.env.GMX_PRIVATE_KEY;
const rpcUrl = process.env.GMX_RPC_URL;
if (!privateKey || !/^0x[0-9a-fA-F]{64}$/.test(privateKey) || !rpcUrl) {
  throw new Error("Provide GMX_PRIVATE_KEY and GMX_RPC_URL securely for this write example");
}
const signer = new PrivateKeySigner(privateKey as Hex, { chain: arbitrum, rpcUrl });
const account = signer.address;
const publicClient = createPublicClient({ chain: arbitrum, transport: http(rpcUrl) });
```

For browser/hardware wallets, implement `address`, `signTypedData(domain, types, value)`, and `signMessage(message)`. Optional `sendTransaction({ to, data, value })` is required for wallet-sent transactions. SDK `signOrder()` checks the domain, known relay router and some receiver fields; it does not compare the complete action to the user's request. Use it after the payload review below. Before any wallet write, verify the wallet/RPC chain ID and signer account against the authorized chain/account; setting viem's `chain` option alone does not verify the remote RPC.

For modern TypeScript, type-check with `module: "ESNext"` and `moduleResolution: "bundler"`; Node16 resolution encounters the SDK's declaration packaging issues. A Node-compatible bundle can keep package imports external so the SDK uses its working CommonJS entry point:

```bash
npm install --save-dev --save-exact --ignore-scripts esbuild@0.28.2
./node_modules/.bin/esbuild script.ts --bundle --platform=node --format=cjs --packages=external --outfile=script.cjs
node script.cjs
```

## Resolve the market and tokens

`symbol` includes the pool suffix. The ETH pool below is an example requested market, not a default trading choice. Use the user's exact selection and reject ambiguous matches.

```typescript
const requestedSymbol = "ETH/USD [WETH-USDC]";
const [markets, tokens] = await Promise.all([sdk.fetchMarkets(), sdk.fetchTokens()]);
const market = markets.find((m) => m.symbol === requestedSymbol && !m.isSpotOnly);
if (!market || !market.isListed) throw new Error("Requested perpetual market is unavailable");
const usdc = tokens.find((t) => t.symbol === "USDC");
if (!usdc) throw new Error("USDC is unavailable on this chain");
if (usdc.address !== market.longTokenAddress && usdc.address !== market.shortTokenAddress) {
  throw new Error("Selected collateral is not a collateral token of this pool");
}
const positions = await sdk.fetchPositionsInfo({ address: account, includeRelatedOrders: true });
const orders = await sdk.fetchOrders({ address: account });
const capacity = await sdk.getTradingCapacity({ symbol: market.symbol, direction: "long" });
```

`fetchMarkets()` returns `MarketWithTiers[]`; inspect `leverageTiers`, `minPositionSizeUsd` and `minCollateralUsd`. `getTradingCapacity()` returns USD amounts (`availableLiquidity`, `baseAvailableLiquidity`, `jitAvailableLiquidity`) and `marketDataStatus` / `jitDataStatus`. Capacity is not a reservation or proof that an account qualifies for JIT; review prepare-time validation too.

## Balances and ERC-20 approval

```typescript
const payAmount = parseUnits("100", usdc.decimals);
const balances = await sdk.fetchWalletBalances({ address: account });
const allowances = await sdk.fetchAllowances({ address: account, spender: "router" });
const allowance = allowances.find((t) => t.address === usdc.address)?.allowance ?? 0n;

if (allowance < payAmount) {
  const approval = sdk.buildApproveTransaction({
    tokenAddress: usdc.address,
    spender: "router",
    amount: payAmount,
  });
  // Send only within the user's authorization, then confirm mining.
  const approvalHash = await signer.sendTransaction(approval);
  const receipt = await publicClient.waitForTransactionReceipt({ hash: approvalHash as Hex });
  if (receipt.status !== "success") throw new Error("Token approval reverted");
}
```

`"router"` resolves to `SyntheticsRouter`, not `ExchangeRouter` or a relay router. Supplying `amount` avoids the builder's unlimited-approval default. Native coin has no ERC-20 approval. Express can pay gas from a supported gas-payment token, but still needs funds for execution/relay fees; `getGasPaymentTokens(chainId)` exposes supported choices. An input approval of `payAmount` may need to include additional payment-token fees when that token also pays gas: inspect the prepared payload's funding requirements and recheck balances/allowances. Do not silently enlarge approvals.

## Prepare an increase

```typescript
const request: PrepareOrderRequest = {
  kind: "increase",
  symbol: market.symbol,
  direction: "long",
  orderType: "market",
  size: parseUnits("500", 30),
  collateralToken: usdc.symbol,
  collateralToPay: { amount: payAmount, token: usdc.symbol },
  slippage: 30,
  mode: "express",
  from: account,
};
const prepared = await sdk.prepareOrder(request);
console.log(prepared.estimates, prepared.warnings, prepared.validationWarnings);
```

This example requests $500 notional with 100 USDC input; fees and an existing position affect resulting leverage. The v2 request has no v1 `leverage` or `payAmount` field. Use `direction: "short"` for a short. Discover allowable collateral instead of assuming all longs require WETH.

Before signing, inspect `estimates` (including effective `sizeDeltaUsd`, fees, impact and capacity), both warning arrays, and `expiresAt`. The server may adjust an almost-full close to a full close. An expired preparation needs a fresh review. `parsePrepareOrderError(error)` decodes a caught SDK `HttpError` from preparation; `isPrepareOrderError()` tests an already parsed value. Handle invalid parameters, missing markets/tokens/positions, unavailable routes, and insufficient liquidity without widening the user's limits.

## Verify the prepared action

API estimates and warnings are display data, not authorization. A compromised API can return a valid signing domain with altered amounts. SDK domain/receiver validation alone will still sign such a payload.

Before signing, compare the decoded action to the user's authorized intent: chain/account, market and collateral addresses, direction, order type, sizes, input amounts, trigger/acceptable prices, minimum outputs, fee limits, receiver and cancellation receiver. Inspect every create/update/cancel action in the batch, including TP/SL and TWAP parts, and verify that `typedData.message` agrees with the `batchParams` and `relayParams` being submitted. Reject unexpected callbacks, external calls, fee recipients, subaccount approvals, additional actions, or stale preparations. If the signing layer cannot decode and enforce these constraints, stop before signing.

For Classic mode, decode every inner `multicall` item and check token transfers, vaults, amounts, receivers, request fields and total native value. Matching `payload.to` to `ExchangeRouter` only verifies the destination; it does not make the calldata safe. Simulate the reviewed transaction before broadcasting. Use the reviewed SDK registry/deployment snapshot for address checks, not a replacement address supplied in an API error or token description.

## Sign and submit Express

After the prepared action passes the checks above and is authorized, this function signs and submits that exact preparation. It is a transport example; the caller must enforce intent validation before invoking it. Reuse the same SDK instance for preparation, signing, submission and status; it also retains subaccount approval state and API-origin pins.

```typescript
async function submitPrepared(prepared: PrepareOrderResponse) {
  if (prepared.mode !== "express" || prepared.payloadType !== "typed-data") {
    throw new Error("Expected an Express signing payload");
  }
  const signature = await sdk.signOrder(prepared, signer);
  return sdk.submitOrder({
    mode: prepared.mode,
    requestId: prepared.requestId,
    idempotencyKey: prepared.idempotencyKey,
    from: account,
    signature,
    eip712Data: {
      batchParams: prepared.payload.batchParams,
      relayParams: prepared.payload.relayParams,
    },
  });
}

const submitted = await submitPrepared(prepared);
```

Persist the request ID, idempotency key, serving API origin, chain, account and intended operation before submission. Preserve the same signed request on any retry. A signed Express payload can authorize a trade until expiry; keep retry material in access-controlled storage, never in chat, public logs or committed files. `executeExpressOrder(request, signer)` combines prepare/sign/submit; use it only when authorization already covers the operation and its limits, because it has no review pause between preparation and signing.

## Track creation and execution

```typescript
async function waitForOrder(requestId: string, waitFor: "created" | "executed") {
  const deadline = Date.now() + 60_000;
  while (Date.now() < deadline) {
    const status = await sdk.fetchOrderStatus({ requestId });
    if (["cancelled", "relay_failed", "relay_reverted"].includes(status.status)) {
      throw new Error(JSON.stringify(status));
    }
    if (status.status === "executed" || (waitFor === "created" && status.status === "created")) {
      return status;
    }
    await new Promise((resolve) => setTimeout(resolve, 2_000));
  }
  throw new Error(`Still unresolved: ${requestId}. Reconcile this request before retrying.`);
}

const result = await waitForOrder(submitted.requestId, "executed");
const refreshedPositions = await sdk.fetchPositionsInfo({ address: account });
```

Status progression includes `prepared`, `relay_accepted`, `relay_pending`, `relay_submitted`, `created`, then `executed` or `cancelled`; relay failures are `relay_failed` / `relay_reverted`. Inspect `error`, `cancellationReason`, `createdTxnHash`, `executionTxnHash` and `orderKeys`. Use `"created"` for resting limit/stop orders, then monitor triggers separately. TWAP and TP/SL can create several order keys: track each, rather than treating one execution or disappearance as completion of the entire strategy. API reads can lag confirmed chain state.

## Close or reduce the exact position

```typescript
const currentPositions = await sdk.fetchPositionsInfo({ address: account });
const position = currentPositions.find((p) =>
  p.marketAddress === market.marketTokenAddress &&
  p.isLong && p.collateralTokenAddress === usdc.address
);
if (!position) throw new Error("Selected position was not found");

const closePrepared = await sdk.prepareOrder({
  kind: "decrease",
  symbol: market.symbol,
  direction: position.isLong ? "long" : "short",
  orderType: "market",
  size: position.sizeInUsd,
  collateralToken: usdc.symbol,
  receiveToken: usdc.symbol,
  keepLeverage: false,
  slippage: 30,
  mode: "express",
  from: account,
});
```

Review and submit `closePrepared` through the same flow. For a partial close use the authorized USD size. `keepLeverage: true` withdraws collateral proportionally; `false` retains collateral on partial decreases. A full close releases remaining collateral. There is no need for v1 `getDecreasePositionAmounts()` or `createDecreaseOrder()` here.

## Edit, cancel, or change collateral

These are alternative preparation calls; do not run them all as one script. Get the selected `order.key` from a fresh `fetchOrders()` result and the position key from `fetchPositionsInfo()`. Verify ownership and the exact intended target. The result uses the same review/sign/submit path.

```typescript
async function prepareEdit(orderKey: string, newTriggerPrice: bigint) {
  return sdk.prepareEditOrder({
    orderIds: [orderKey], newTriggerPrice, mode: "express", from: account,
  });
}

async function prepareCancel(orderKey: string) {
  return sdk.prepareCancelOrder({
    orderId: orderKey, mode: "express", from: account,
  });
}

async function prepareCollateralChange(positionKey: string, amount: bigint) {
  return sdk.prepareCollateral({
    operation: "deposit", positionKey, amount, mode: "express", from: account,
  });
}
```

Editing also supports `newSize`, `newAcceptablePrice`, `newAutoCancel` and `executionFeeTopUp`. Cancellation supports explicit `orderIds`, or `all: true` only when all-account cancellation is intended. Collateral withdrawal uses `operation: "withdraw"`; `amount` is in collateral-token units, not USD30. Monitor the affected order/position after edits and cancellations.

## Classic mode

```typescript
const classic = await sdk.prepareOrder({ ...request, mode: "classic" });
if (classic.payloadType !== "transaction") throw new Error("Expected Classic calldata");
if (classic.payload.to !== getContract(chainId, "ExchangeRouter")) {
  throw new Error("Unexpected router; verify the prepared transaction before sending");
}
// Decode and validate every call/value against the authorized action first.
const hash = await signer.sendTransaction({
  to: classic.payload.to,
  data: classic.payload.data,
  value: BigInt(classic.payload.value ?? "0"),
});
const receipt = await publicClient.waitForTransactionReceipt({ hash: hash as Hex });
if (receipt.status !== "success") throw new Error("Order creation reverted");
```

Do not pass Classic calldata to `signOrder()` or assume Express relay status tracks a wallet-broadcast transaction. Read the creation receipt's `EventEmitter` logs to obtain all order keys; follow keeper execution/cancellation and refetch positions/orders. A successful creation receipt alone does not establish trade execution.

## One-click trading and GMX Account

For explicitly authorized delegated trading, call `activateSubaccount(mainSigner, { expiresInSeconds, maxAllowedCount })` with bounded permissions. It derives the local signer, reads on-chain authorization and signs an approval if needed; the approval is carried with the next order, not mined by activation alone. Subsequent prepare/sign/submit calls use the subaccount automatically. After an approval-bearing order is created/executed, call `refreshSubaccountState(account)` and inspect `subaccountStatus`. Use one instance per account and call `clearSubaccount()` on account changes. Clearing local state is not on-chain revocation.

GMX Account funding methods are a separate surface: `buildSameChainDepositTxn`, `buildSameChainWithdrawTxn`, `executeSameChainDeposit`, `executeSameChainWithdraw`, `prepareCrossChainDeposit`, `executeCrossChainDeposit`, and the cross-chain withdraw prepare/sign/submit/status helpers. Follow their exported request types and [official examples](https://docs.gmx.io/docs/sdk/v2/examples/). Settlement-chain funding support is narrower than general trading-chain support; MegaETH is not a settlement chain in 2.1.2. These methods do not mint/burn GM or GLV liquidity tokens.
