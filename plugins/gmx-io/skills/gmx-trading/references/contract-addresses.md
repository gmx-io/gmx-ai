# Contract Addresses

Deployed GMX V2 Synthetics contracts per chain. Addresses sourced from [`sdk/src/configs/contracts.ts`](https://github.com/gmx-io/gmx-interface/blob/release/sdk/src/configs/contracts.ts) in the gmx-interface repository.

> **Note:** Most contract addresses may change on upgrades. `DataStore` and `EventEmitter` are permanent.

## Arbitrum (42161)

### Core Synthetics

| Contract | Address |
|----------|---------|
| DataStore | `0xFD70de6b91282D8017aA4E741e9Ae325CAb992d8` |
| EventEmitter | `0xC8ee91A54287DB53897056e12D9819156D3822Fb` |
| ExchangeRouter | `0x7dE39FF2e232A2203196788d37e234cF8F1b83f1` |
| SyntheticsRouter | `0x7452c558d45f8afC8c83dAe62C3f8A5BE19c71f6` |
| SyntheticsReader | `0xfA26cBb46e2614609406de08CA1Dc7f70a684184` |

### Vaults

| Contract | Address |
|----------|---------|
| DepositVault | `0xF89e77e8Dc11691C9e8757e84aaFbCD8A67d7A55` |
| WithdrawalVault | `0x0628D46b5D145f183AdB6Ef1f2c97eD1C4701C55` |
| OrderVault | `0x31eF83a530Fde1B38EE9A18093A333D8Bbbc40D5` |
| ShiftVault | `0xfe99609C4AA83ff6816b64563Bdffd7fa68753Ab` |

### Relay & Subaccounts

| Contract | Address |
|----------|---------|
| SubaccountRouter | `0x9c05880A2AaD7530c69e18e342eDC9E06cc757db` |
| GelatoRelayRouter | `0x5503b99308dB6923758F9A22d118207D633c4e87` |
| SubaccountGelatoRelayRouter | `0xfD0596f708d9D950E0eF7b5d191e5F8e55b8a67f` |

### GLV (Liquidity Vaults)

| Contract | Address |
|----------|---------|
| GlvReader | `0x85fcBD684D08053f1efAB302dCb04F22E20E65B1` |
| GlvRouter | `0x167540D2DFF14120365CfDDF2F86e3045D4fa712` |
| GlvVault | `0x393053B58f9678C9c28c2cE941fF6cac49C3F8f9` |

### Multichain (GMX Account)

| Contract | Address |
|----------|---------|
| MultichainOrderRouter | `0xABFC734f7CFc9352AED7a97b1F6a236eae831e8A` |
| MultichainVault | `0xCeaadFAf6A8C489B250e407987877c5fDfcDBE6E` |
| LayerZeroProvider | `0x0B33EBA531e5a5A331a3Ff9F418B8205F01C2869` |

### Other

| Contract | Address |
|----------|---------|
| ReferralStorage | `0xe6fab3f0c7199b0d34d7fbe83394fc0e0d06e99d` |
| Multicall | `0xe79118d6D92a4b23369ba356C90b9A7ABf1CB961` |
| NATIVE_TOKEN (WETH) | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` |

---

## Avalanche (43114)

### Core Synthetics

| Contract | Address |
|----------|---------|
| DataStore | `0x2F0b22339414ADeD7D5F06f9D604c7fF5b2fe3f6` |
| EventEmitter | `0xDb17B211c34240B014ab6d61d4A31FA0C0e20c26` |
| ExchangeRouter | `0xc002Db96E682FFF6675966F959677285a0C45Efa` |
| SyntheticsRouter | `0x820F5FfC5b525cD4d88Cd91aCf2c28F16530Cc68` |
| SyntheticsReader | `0xa34320a507493C71Fe35E982e496F7C5d1a7fa02` |

### Vaults

| Contract | Address |
|----------|---------|
| DepositVault | `0x90c670825d0C62ede1c5ee9571d6d9a17A722DFF` |
| WithdrawalVault | `0xf5F30B10141E1F63FC11eD772931A8294a591996` |
| OrderVault | `0xD3D60D22d415aD43b7e64b510D86A30f19B1B12C` |
| ShiftVault | `0x7fC46CCb386e9bbBFB49A2639002734C3Ec52b39` |

### Relay & Subaccounts

| Contract | Address |
|----------|---------|
| SubaccountRouter | `0xAda708aFf0f1D784D28cd8Ff4d6D977fF9599e5D` |
| GelatoRelayRouter | `0x51fe0b7919e1208a717E9B16a097C1C3D70eFbf6` |
| SubaccountGelatoRelayRouter | `0xa62BD1cFE2066c5bF4180b4125BBb5116eEA26c9` |

### GLV (Liquidity Vaults)

| Contract | Address |
|----------|---------|
| GlvReader | `0x321EB66dD95ad33715ee615AAb8dAC6394E7b3F9` |
| GlvRouter | `0x603B3D3aB077CA433b888c05fa59c777d5b6dCAD` |
| GlvVault | `0x527FB0bCfF63C47761039bB386cFE181A92a4701` |

### Multichain (GMX Account)

| Contract | Address |
|----------|---------|
| MultichainOrderRouter | `0x204CC947Fddd11c90e302db2A5ac3865021D1618` |
| MultichainVault | `0x6D5F3c723002847B009D07Fe8e17d6958F153E4e` |
| LayerZeroProvider | `0x74eECe8cC29b3d549db97F566a4445F48ed62a0d` |

### Other

| Contract | Address |
|----------|---------|
| ReferralStorage | `0x827ed045002ecdabeb6e2b0d1604cf5fc3d322f8` |
| Multicall | `0x50474CAe810B316c294111807F94F9f48527e7F8` |
| NATIVE_TOKEN (WAVAX) | `0xB31f66AA3C1e785363F0875A1B74E27b85FD66c7` |

---

## Botanix (3637)

### Core Synthetics

| Contract | Address |
|----------|---------|
| DataStore | `0xA23B81a89Ab9D7D89fF8fc1b5d8508fB75Cc094d` |
| EventEmitter | `0xAf2E131d483cedE068e21a9228aD91E623a989C2` |
| ExchangeRouter | `0xBCB5eA3a84886Ce45FBBf09eBF0e883071cB2Dc8` |
| SyntheticsRouter | `0x3d472afcd66F954Fe4909EEcDd5c940e9a99290c` |
| SyntheticsReader | `0x922766ca6234cD49A483b5ee8D86cA3590D0Fb0E` |

### Vaults

| Contract | Address |
|----------|---------|
| DepositVault | `0x4D12C3D3e750e051e87a2F3f7750fBd94767742c` |
| WithdrawalVault | `0x46BAeAEdbF90Ce46310173A04942e2B3B781Bf0e` |
| OrderVault | `0xe52B3700D17B45dE9de7205DEe4685B4B9EC612D` |
| ShiftVault | `0xa7EE2737249e0099906cB079BCEe85f0bbd837d4` |

### Relay & Subaccounts

| Contract | Address |
|----------|---------|
| SubaccountRouter | `0xa1793126B6Dc2f7F254a6c0E2F8013D2180C0D10` |
| GelatoRelayRouter | `0x98e86155abf8bCbA566b4a909be8cF4e3F227FAf` |
| SubaccountGelatoRelayRouter | `0xd6b16f5ceE328310B1cf6d8C0401C23dCd3c40d4` |

### GLV (Liquidity Vaults)

| Contract | Address |
|----------|---------|
| GlvReader | `0x955Aa50d2ecCeffa59084BE5e875eb676FfAFa98` |
| GlvRouter | `0xC92741F0a0D20A95529873cBB3480b1f8c228d9F` |
| GlvVault | `0xd336087512BeF8Df32AF605b492f452Fd6436CD8` |

### Multichain (GMX Account)

| Contract | Address |
|----------|---------|
| MultichainOrderRouter | `0xbC074fF8b85f9b66884E1EdDcE3410fde96bd798` |
| MultichainVault | `0x9a535f9343434D96c4a39fF1d90cC685A4F6Fb20` |
| LayerZeroProvider | `0x9E721ef9b908B4814Aa18502692E4c5666d1942e` |

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

## Source

For the latest addresses, check these files in the [gmx-interface](https://github.com/gmx-io/gmx-interface/tree/release) repository:

- **Contract addresses**: [`sdk/src/configs/contracts.ts`](https://github.com/gmx-io/gmx-interface/blob/release/sdk/src/configs/contracts.ts)
- **Token addresses**: [`sdk/src/configs/tokens.ts`](https://github.com/gmx-io/gmx-interface/blob/release/sdk/src/configs/tokens.ts)
- **Market addresses**: [`sdk/src/configs/markets.ts`](https://github.com/gmx-io/gmx-interface/blob/release/sdk/src/configs/markets.ts)
