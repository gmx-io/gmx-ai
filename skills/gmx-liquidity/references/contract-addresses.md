# Contract Addresses — Liquidity Operations

Contracts used for GM pool and GLV vault operations. Addresses sourced from `@gmx-io/sdk` `configs/contracts.ts`.

> **Note:** Most contract addresses may change on upgrades. `DataStore` and `EventEmitter` are permanent.

## Arbitrum (42161)

### GM Pool Operations (ExchangeRouter)

| Contract | Address |
|----------|---------|
| ExchangeRouter | `0x1C3fa76e6E1088bCE750f23a5BFcffa1efEF6A41` |
| SyntheticsRouter | `0x7452c558d45f8afC8c83dAe62C3f8A5BE19c71f6` |
| SyntheticsReader | `0x470fbC46bcC0f16532691Df360A07d8Bf5ee0789` |
| DataStore | `0xFD70de6b91282D8017aA4E741e9Ae325CAb992d8` |

### Vaults (token transfer targets)

| Contract | Address |
|----------|---------|
| DepositVault | `0xF89e77e8Dc11691C9e8757e84aaFbCD8A67d7A55` |
| WithdrawalVault | `0x0628D46b5D145f183AdB6Ef1f2c97eD1C4701C55` |
| OrderVault | `0x31eF83a530Fde1B38EE9A18093A333D8Bbbc40D5` |
| ShiftVault | `0xfe99609C4AA83ff6816b64563Bdffd7fa68753Ab` |

### GLV Operations (GlvRouter)

| Contract | Address |
|----------|---------|
| GlvRouter | `0x7EAdEE2ca1b4D06a0d82fDF03D715550c26AA12F` |
| GlvReader | `0x2C670A23f1E798184647288072e84054938B5497` |
| GlvVault | `0x393053B58f9678C9c28c2cE941fF6cac49C3F8f9` |

### Other

| Contract | Address |
|----------|---------|
| Multicall | `0xe79118d6D92a4b23369ba356C90b9A7ABf1CB961` |
| NATIVE_TOKEN (WETH) | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |

---

## Avalanche (43114)

### GM Pool Operations (ExchangeRouter)

| Contract | Address |
|----------|---------|
| ExchangeRouter | `0x8f550E53DFe96C055D5Bdb267c21F268fCAF63B2` |
| SyntheticsRouter | `0x820F5FfC5b525cD4d88Cd91aCf2c28F16530Cc68` |
| SyntheticsReader | `0x62Cb8740E6986B29dC671B2EB596676f60590A5B` |
| DataStore | `0x2F0b22339414ADeD7D5F06f9D604c7fF5b2fe3f6` |

### Vaults

| Contract | Address |
|----------|---------|
| DepositVault | `0x90c670825d0C62ede1c5ee9571d6d9a17A722DFF` |
| WithdrawalVault | `0xf5F30B10141E1F63FC11eD772931A8294a591996` |
| OrderVault | `0xD3D60D22d415aD43b7e64b510D86A30f19B1B12C` |
| ShiftVault | `0x7fC46CCb386e9bbBFB49A2639002734C3Ec52b39` |

### GLV Operations (GlvRouter)

| Contract | Address |
|----------|---------|
| GlvRouter | `0x7E425c47b2Ff0bE67228c842B9C792D0BCe58ae6` |
| GlvReader | `0x5C6905A3002f989E1625910ba1793d40a031f947` |
| GlvVault | `0x527FB0bCfF63C47761039bB386cFE181A92a4701` |

### Other

| Contract | Address |
|----------|---------|
| Multicall | `0x50474CAe810B316c294111807F94F9f48527e7F8` |
| NATIVE_TOKEN (WAVAX) | `0xB31f66AA3C1e785363F0875A1B74E27b85FD66c7` |

---

## Botanix (3637)

### GM Pool Operations (ExchangeRouter)

| Contract | Address |
|----------|---------|
| ExchangeRouter | `0xBCB5eA3a84886Ce45FBBf09eBF0e883071cB2Dc8` |
| SyntheticsRouter | `0x3d472afcd66F954Fe4909EEcDd5c940e9a99290c` |
| SyntheticsReader | `0x922766ca6234cD49A483b5ee8D86cA3590D0Fb0E` |
| DataStore | `0xA23B81a89Ab9D7D89fF8fc1b5d8508fB75Cc094d` |

### Vaults

| Contract | Address |
|----------|---------|
| DepositVault | `0x4D12C3D3e750e051e87a2F3f7750fBd94767742c` |
| WithdrawalVault | `0x46BAeAEdbF90Ce46310173A04942e2B3B781Bf0e` |
| OrderVault | `0xe52B3700D17B45dE9de7205DEe4685B4B9EC612D` |
| ShiftVault | `0xa7EE2737249e0099906cB079BCEe85f0bbd837d4` |

### GLV Operations (GlvRouter)

| Contract | Address |
|----------|---------|
| GlvRouter | `0xC92741F0a0D20A95529873cBB3480b1f8c228d9F` |
| GlvReader | `0x955Aa50d2ecCeffa59084BE5e875eb676FfAFa98` |
| GlvVault | `0xd336087512BeF8Df32AF605b492f452Fd6436CD8` |

### Botanix-Specific Tokens

| Token | Address |
|-------|---------|
| NATIVE_TOKEN (PBTC) | `0x0D2437F93Fed6EA64Ef01cCde385FB1263910C56` |
| StBTC | `0xF4586028FFdA7Eca636864F80f8a3f2589E33795` |

### Other

| Contract | Address |
|----------|---------|
| Multicall | `0x4BaA24f93a657f0c1b4A0Ffc72B91011E35cA46b` |

---

## ABI Import Paths

```typescript
const { ExchangeRouterAbi } = require("@gmx-io/sdk/abis/ExchangeRouter");
const { GlvRouterAbi } = require("@gmx-io/sdk/abis/GlvRouter");
```

## Token Approval

Before depositing ERC20 tokens, approve the SyntheticsRouter (for GM operations) or GlvRouter (for GLV operations) to spend your tokens:

```typescript
// For GM deposits — approve SyntheticsRouter
await tokenContract.write.approve([syntheticsRouterAddress, amount]);

// For GLV deposits — approve GlvRouter
await tokenContract.write.approve([glvRouterAddress, amount]);
```

## Source

Contract addresses are defined in `@gmx-io/sdk` at `sdk/src/configs/contracts.ts`.
