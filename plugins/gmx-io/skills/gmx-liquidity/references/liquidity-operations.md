# Liquidity operations

Use `GmxApiSdk` for the reads in [the skill](../SKILL.md). GM/GLV writes are not exposed by that client in `@gmx-io/sdk@2.1.1`; use its exported ABIs, contract registry, gas keys and calculation helpers with viem. The following examples build transactions; broadcasting is a separate authorized step.

## Setup and current deployment

TypeScript snippets share this setup. Put awaited code inside an async function and compile to CommonJS (or use equivalent `.cjs` imports). The RPC and API must refer to the same chain.

```typescript
import { GmxApiSdk } from "@gmx-io/sdk/v2";
import { getContract } from "@gmx-io/sdk/configs/contracts";
import { GAS_LIMITS_STATIC_CONFIG } from "@gmx-io/sdk/configs/gasLimits";
import * as keys from "@gmx-io/sdk/configs/dataStore";
import { estimateExecuteDepositGasLimit, estimateDepositOraclePriceCount,
  getExecutionFee, type GasLimitsConfig } from "@gmx-io/sdk/utils/fees";
import exchangeRouterAbi from "@gmx-io/sdk/abis/ExchangeRouter";
import glvRouterAbi from "@gmx-io/sdk/abis/GlvRouter";
import glvReaderAbi from "@gmx-io/sdk/abis/GlvReader";
import dataStoreAbi from "@gmx-io/sdk/abis/DataStore";
import { createPublicClient, http, encodeFunctionData, erc20Abi, zeroAddress,
  type Abi, type Address, type Hex } from "viem";
import { arbitrum } from "viem/chains";

const chainId = 42161;
const sdk = new GmxApiSdk({ chainId });
const rpcUrl = process.env.GMX_RPC_URL;
if (!rpcUrl) throw new Error("GMX_RPC_URL is required for contract reads");
const publicClient = createPublicClient({ chain: arbitrum, transport: http(rpcUrl) });
const exchangeRouter = getContract(chainId, "ExchangeRouter");
const glvRouter = getContract(chainId, "GlvRouter");
const dataStore = getContract(chainId, "DataStore");
```

See [contract addresses](contract-addresses.md) for the checked deployment snapshot. `SyntheticsRouter` spends approved ERC-20s for both routers. The custody addresses `DepositVault`, `WithdrawalVault`, `ShiftVault` and `GlvVault` are distinct from market/GLV share-token addresses.

For modern TypeScript, type-check with `module: "ESNext"` and `moduleResolution: "bundler"`; Node16 resolution encounters the SDK's declaration packaging issues. A Node-compatible bundle can keep package imports external so the SDK uses its working CommonJS entry point:

```bash
npx esbuild script.ts --bundle --platform=node --format=cjs --packages=external --outfile=script.cjs
node script.cjs
```

## Discover GLVs and balances

```typescript
const page = await publicClient.readContract({
  address: getContract(chainId, "GlvReader"),
  abi: glvReaderAbi,
  functionName: "getGlvInfoList",
  args: [dataStore, 0n, 100n],
});
console.log(page);
```

Paginate with increasing `start`/`end` until a short/empty page. Each entry contains `glv.glvToken`, `glv.longToken`, `glv.shortToken` and `markets`. Choose the user's vault from this data, then re-fetch `getGlvInfo(dataStore, glvToken)` before quoting. Use the actual `markets.length` when estimating GLV execution fees; never hardcode historical counts like 53 or 20. Check the selected constituent's GLV deposit/withdrawal configuration, capacity, enabled status, and current holdings as applicable. A listed constituent is not necessarily available for every operation.

For a specific GM/GLV token, read `balanceOf(account)` and `decimals()` using `erc20Abi`; API wallet balances may not include all share tokens. `GlvVault` is not the token address. GM-to-GLV deposits require the GM market to be a constituent; GM-to-GM shifts require compatible collateral tokens and enabled source/destination markets.

## Quote and execution fee

Obtain expected outputs using current reader/pricing logic, token prices, supply, pool state, fees and price impact. `SyntheticsReader.getDepositAmountOut`, `getWithdrawalAmountOut` and `getMarketTokenPrice`, plus `GlvReader.getGlvValue` / `getGlvTokenPrice`, are relevant entry points; inspect their ABI signatures and maximize/PnL-factor parameters. A simple TVL/supply division is not an executable quote. GLV raw-token deposits involve both the constituent GM calculation and GLV share conversion. See [Reader](https://docs.gmx.io/docs/api/contracts/reader/) and [GLV Reader](https://docs.gmx.io/docs/api/contracts/glv-reader/).

Apply the user's slippage tolerance to expected outputs using bigint arithmetic. GM deposit/shift minimums use destination GM units, GLV deposits use GLV units, and withdrawal minimums use each output token's decimals. Do not fill an unknown minimum with zero or silently increase slippage on failure.

Keeper execution fees are separate from the wallet's gas cost for creating a request. Read current gas parameters from `DataStore` with SDK-exported keys; no `GmxSdk` initialization is needed:

```typescript
async function readGasLimits(): Promise<GasLimitsConfig> {
  const gasKeys = {
    depositToken: keys.depositGasLimitKey(),
    withdrawalMultiToken: keys.withdrawalGasLimitKey(),
    shift: keys.shiftGasLimitKey(),
    singleSwap: keys.singleSwapGasLimitKey(),
    swapOrder: keys.swapOrderGasLimitKey(),
    increaseOrder: keys.increaseOrderGasLimitKey(),
    decreaseOrder: keys.decreaseOrderGasLimitKey(),
    estimatedGasFeeBaseAmount: keys.ESTIMATED_GAS_FEE_BASE_AMOUNT_V2_1,
    estimatedGasFeePerOraclePrice: keys.ESTIMATED_GAS_FEE_PER_ORACLE_PRICE,
    estimatedFeeMultiplierFactor: keys.ESTIMATED_GAS_FEE_MULTIPLIER_FACTOR,
    gelatoRelayFeeMultiplierFactor: keys.GELATO_RELAY_FEE_MULTIPLIER_FACTOR_KEY,
    glvDepositGasLimit: keys.GLV_DEPOSIT_GAS_LIMIT,
    glvWithdrawalGasLimit: keys.GLV_WITHDRAWAL_GAS_LIMIT,
    glvPerMarketGasLimit: keys.GLV_PER_MARKET_GAS_LIMIT,
  };
  const blockNumber = await publicClient.getBlockNumber();
  const entries = await Promise.all(Object.entries(gasKeys).map(async ([name, key]) => [
    name,
    await publicClient.readContract({
      address: dataStore, abi: dataStoreAbi, functionName: "getUint",
      args: [key as Hex], blockNumber,
    }),
  ]));
  return { ...GAS_LIMITS_STATIC_CONFIG[chainId], ...Object.fromEntries(entries) } as GasLimitsConfig;
}

async function quoteGmDepositExecutionFee() {
  const [gasLimits, gasPrice, tokens] = await Promise.all([
    readGasLimits(), publicClient.getGasPrice(), sdk.fetchTokensData(),
  ]);
  const tokensData = Object.fromEntries(tokens.map((token) => [token.address, token]));
  const fee = getExecutionFee(
    chainId, gasLimits, tokensData,
    estimateExecuteDepositGasLimit(gasLimits, { swapsCount: 0, callbackGasLimit: 0n }),
    gasPrice, estimateDepositOraclePriceCount(0),
  );
  if (!fee) throw new Error("Native-token price or gas data unavailable");
  return fee.feeTokenAmount;
}
```

This example estimates a GM deposit with no swaps/callback. The exported helper applies the live DataStore gas multipliers and chain minimums; the supplied gas price must also reflect the network pricing/buffer chosen for submission. Use the matching helpers for other requests, and account for every swap and callback:

| Operation | Gas helper from `@gmx-io/sdk/utils/fees` | Oracle-count helper |
|-----------|------------------------------------------------|---------------------|
| GM deposit | `estimateExecuteDepositGasLimit(gasLimits, { swapsCount, callbackGasLimit })` | `estimateDepositOraclePriceCount(swapsCount)` |
| GM withdrawal | `estimateExecuteWithdrawalGasLimit(gasLimits, { swapsCount, callbackGasLimit })` | `estimateWithdrawalOraclePriceCount(swapsCount)` |
| GM shift | `estimateExecuteShiftGasLimit(gasLimits, { callbackGasLimit })` | `estimateShiftOraclePriceCount()` |
| GLV deposit | `estimateExecuteGlvDepositGasLimit(gasLimits, { marketsCount, isMarketTokenDeposit, swapsCount })` | `estimateGlvDepositOraclePriceCount(marketsCount, swapsCount)` |
| GLV withdrawal | `estimateExecuteGlvWithdrawalGasLimit(gasLimits, { marketsCount, swapsCount })` | `estimateGlvWithdrawalOraclePriceCount(marketsCount, swapsCount)` |

Use bigint arguments where the helper types require them. The GLV examples below use zero callback gas. Refresh the estimate before submission and keep enough native balance for both the execution fee and creation transaction gas. Gas estimation of the creation transaction alone does not calculate the keeper fee.

## Atomic funding and request creation

For ERC-20 inputs, first check allowance to `SyntheticsRouter`. `sdk.buildApproveTransaction({ tokenAddress, spender: "router", amount })` can build the approval for underlying, GM, or GLV tokens. Send only the authorized allowance and wait for a successful receipt before creating a request.

**Funding and creation must share one transaction.** Never transfer tokens to a request vault in a separate transaction. The examples below use one `multicall` to wrap/send native coin, transfer ERC-20s, then create the request.

```typescript
function buildLiquidityRequest(
  router: Address,
  abi: Abi,
  vault: Address,
  inputs: { token: Address; amount: bigint }[],
  createData: Hex,
  executionFee: bigint,
  nativeInput = 0n,
) {
  const value = executionFee + nativeInput;
  const calls: Hex[] = [encodeFunctionData({
    abi, functionName: "sendWnt", args: [vault, value],
  })];
  for (const input of inputs) {
    if (input.amount > 0n) calls.push(encodeFunctionData({
      abi, functionName: "sendTokens", args: [input.token, vault, input.amount],
    }));
  }
  calls.push(createData);
  return { to: router, data: encodeFunctionData({ abi, functionName: "multicall", args: [calls] }), value };
}
```

The examples below use wrapped/ERC-20 inputs. For native input, put its amount in `nativeInput` and omit that amount from `inputs`; the selected initial token must be the chain's wrapped native token. `sendWnt` performs wrapping. The `shouldUnwrapNativeToken` flag controls refunds/outputs; it does **not** determine whether input is funded from native coin. Aggregate duplicate-token inputs for same-collateral markets before building calls.

## GM deposit

Inputs and the nonzero minimum output come from the selected market and a fresh quote. The builder does not send a transaction.

```typescript
function buildGmDeposit(p: {
  receiver: Address; market: Address; longToken: Address; shortToken: Address;
  longAmount: bigint; shortAmount: bigint; minMarketTokens: bigint; executionFee: bigint;
}) {
  const createData = encodeFunctionData({
    abi: exchangeRouterAbi, functionName: "createDeposit", args: [{
      addresses: {
        receiver: p.receiver, callbackContract: zeroAddress, uiFeeReceiver: zeroAddress,
        market: p.market, initialLongToken: p.longToken, initialShortToken: p.shortToken,
        longTokenSwapPath: [], shortTokenSwapPath: [],
      },
      minMarketTokens: p.minMarketTokens, shouldUnwrapNativeToken: false,
      executionFee: p.executionFee, callbackGasLimit: 0n, dataList: [],
    }],
  });
  return buildLiquidityRequest(exchangeRouter, exchangeRouterAbi, getContract(chainId, "DepositVault"), [
    { token: p.longToken, amount: p.longAmount }, { token: p.shortToken, amount: p.shortAmount },
  ], createData, p.executionFee);
}
```

## GM withdrawal

Approve the GM token and send it to `WithdrawalVault`. The amount is taken from the funded vault balance, not a field in `createWithdrawal`.

```typescript
function buildGmWithdrawal(p: {
  receiver: Address; market: Address; amount: bigint;
  minLong: bigint; minShort: bigint; executionFee: bigint;
}) {
  const createData = encodeFunctionData({
    abi: exchangeRouterAbi, functionName: "createWithdrawal", args: [{
      addresses: {
        receiver: p.receiver, callbackContract: zeroAddress, uiFeeReceiver: zeroAddress,
        market: p.market, longTokenSwapPath: [], shortTokenSwapPath: [],
      },
      minLongTokenAmount: p.minLong, minShortTokenAmount: p.minShort,
      shouldUnwrapNativeToken: false, executionFee: p.executionFee, callbackGasLimit: 0n, dataList: [],
    }],
  });
  return buildLiquidityRequest(exchangeRouter, exchangeRouterAbi, getContract(chainId, "WithdrawalVault"), [
    { token: p.market, amount: p.amount },
  ], createData, p.executionFee);
}
```

## GM shift

The input is source GM; the output minimum is destination GM. Check matching long/short tokens and target deposit/source withdrawal constraints before quoting. A shift has no `shouldUnwrapNativeToken` field.

```typescript
function buildGmShift(p: {
  receiver: Address; fromMarket: Address; toMarket: Address;
  amount: bigint; minMarketTokens: bigint; executionFee: bigint;
}) {
  const createData = encodeFunctionData({
    abi: exchangeRouterAbi, functionName: "createShift", args: [{
      addresses: {
        receiver: p.receiver, callbackContract: zeroAddress, uiFeeReceiver: zeroAddress,
        fromMarket: p.fromMarket, toMarket: p.toMarket,
      },
      minMarketTokens: p.minMarketTokens, executionFee: p.executionFee, callbackGasLimit: 0n, dataList: [],
    }],
  });
  return buildLiquidityRequest(exchangeRouter, exchangeRouterAbi, getContract(chainId, "ShiftVault"), [
    { token: p.fromMarket, amount: p.amount },
  ], createData, p.executionFee);
}
```

## GLV deposit: raw tokens or GM

Use `GlvRouter` and `GlvVault`. `market` must be an eligible constituent. A GM-token deposit sets `isMarketTokenDeposit: true` and funds the market token; a raw-token deposit funds the selected long/short tokens and sets it to false.

```typescript
function buildGlvDeposit(p: {
  receiver: Address; glv: Address; market: Address; longToken: Address; shortToken: Address;
  funding: { kind: "gm"; amount: bigint } | { kind: "tokens"; longAmount: bigint; shortAmount: bigint };
  minGlvTokens: bigint; executionFee: bigint;
}) {
  const isGm = p.funding.kind === "gm";
  const inputs = p.funding.kind === "gm"
    ? [{ token: p.market, amount: p.funding.amount }]
    : [{ token: p.longToken, amount: p.funding.longAmount }, { token: p.shortToken, amount: p.funding.shortAmount }];
  const createData = encodeFunctionData({
    abi: glvRouterAbi, functionName: "createGlvDeposit", args: [{
      addresses: {
        glv: p.glv, market: p.market, receiver: p.receiver,
        callbackContract: zeroAddress, uiFeeReceiver: zeroAddress,
        initialLongToken: isGm ? zeroAddress : p.longToken,
        initialShortToken: isGm ? zeroAddress : p.shortToken,
        longTokenSwapPath: [], shortTokenSwapPath: [],
      },
      minGlvTokens: p.minGlvTokens, executionFee: p.executionFee, callbackGasLimit: 0n,
      shouldUnwrapNativeToken: false, isMarketTokenDeposit: isGm, dataList: [],
    }],
  });
  return buildLiquidityRequest(glvRouter, glvRouterAbi, getContract(chainId, "GlvVault"), inputs, createData, p.executionFee);
}
```

## GLV withdrawal

Approve/fund the GLV share token, not the constituent GM token or request custody vault. Minimum outputs are the constituent's underlying tokens.

```typescript
function buildGlvWithdrawal(p: {
  receiver: Address; glv: Address; market: Address; amount: bigint;
  minLong: bigint; minShort: bigint; executionFee: bigint;
}) {
  const createData = encodeFunctionData({
    abi: glvRouterAbi, functionName: "createGlvWithdrawal", args: [{
      addresses: {
        receiver: p.receiver, callbackContract: zeroAddress, uiFeeReceiver: zeroAddress,
        market: p.market, glv: p.glv, longTokenSwapPath: [], shortTokenSwapPath: [],
      },
      minLongTokenAmount: p.minLong, minShortTokenAmount: p.minShort,
      shouldUnwrapNativeToken: false, executionFee: p.executionFee, callbackGasLimit: 0n, dataList: [],
    }],
  });
  return buildLiquidityRequest(glvRouter, glvRouterAbi, getContract(chainId, "GlvVault"), [
    { token: p.glv, amount: p.amount },
  ], createData, p.executionFee);
}
```

## Submit, follow execution, cancel

1. Review the built transaction, input allowances/balances, quote minimums and fee against the authorized operation. Simulate the creation multicall from the real account and estimate its transaction gas before sending through the connected wallet/signer. A successful creation simulation does not guarantee keeper execution at later prices.
2. Save the transaction hash and wait for a successful receipt. Wallet writes return a hash, **not** the Solidity `bytes32` request key. Decode the matching `DepositCreated`, `WithdrawalCreated`, `ShiftCreated`, `GlvDepositCreated` or `GlvWithdrawalCreated` event in `EventEmitter` logs to obtain the request key. These named events are carried by the emitter's generic `EventLog*` ABI; inspect its event name and data items rather than assuming a standalone ABI event named `DepositCreated`.
3. Follow that key to the corresponding `*Executed` or `*Cancelled` event, then refresh LP/underlying balances. Record cancellation reasons and refunds. `fetchOrderStatus()` tracks API order requests, not these direct liquidity requests. Use a bounded wait and leave unresolved requests pending; reconcile before resubmitting.
4. If cancellation is requested and allowed by the current request age/state, send `cancelDeposit(key)`, `cancelWithdrawal(key)` or `cancelShift(key)` to `ExchangeRouter`; GLV requests use `cancelGlvDeposit(key)` / `cancelGlvWithdrawal(key)` on `GlvRouter`. The creator must authorize it. Check the live cancellation delay and race with keeper execution; verify the cancellation receipt/event and actual refund recipient.

Do not assume tokens or excess execution fees always return to `receiver`: cancellation/output routing depends on the request type and stored account/receiver fields. Inspect those fields and events. An executed shift burns source GM and mints destination GM atomically in keeper execution; request creation is still asynchronous.

Sources: [SDK ABIs](https://github.com/gmx-io/gmx-interface/tree/release/sdk/src/abis), [SDK gas helpers](https://github.com/gmx-io/gmx-interface/tree/release/sdk/src/utils/fees), [ExchangeRouter](https://docs.gmx.io/docs/api/contracts/exchange-router/), [GlvRouter](https://docs.gmx.io/docs/api/contracts/glv-router/), [event monitoring](https://docs.gmx.io/docs/api/contracts/events/).
