# GMX Agent Skills

Agent skills for trading and providing liquidity on [GMX V2](https://app.gmx.io).

## Install

**Claude Code (plugin marketplace):**
```
/plugin marketplace add gmx-io/gmx-ai
/plugin install gmx-io@gmx-ai
```

**Vercel Skills CLI:**
```bash
npx skills add gmx-io/gmx-ai
```

## Skills

### gmx-trading

Trade perpetuals and swap tokens on GMX V2 across Arbitrum, Avalanche, and Botanix.

**Capabilities:**
- Open long/short positions with up to 100x leverage
- Swap tokens at oracle prices
- Market, limit, stop-loss, and take-profit orders
- Query positions, markets, and trade history
- Full TypeScript SDK (`@gmx-io/sdk`) and REST API reference

**Reference files included:**
- SDK method signatures and types
- Oracle, OpenAPI, and GraphQL endpoint documentation
- Contract addresses for all supported chains
- Order type behavior and trigger logic

### gmx-liquidity

Provide liquidity on GMX V2 across Arbitrum, Avalanche, and Botanix.

**Capabilities:**
- Query GM pool data (TVL, composition, utilization, capacity)
- Deposit into GM pools (mint GM tokens)
- Withdraw from GM pools (burn GM tokens)
- Deposit into / withdraw from GLV vaults (multi-market auto-rebalancing)
- Shift liquidity between GM pools atomically
- Full contract-level viem examples with multicall patterns

**Reference files included:**
- Contract struct definitions and execution flows
- Gas estimation formulas per operation type
- GLV vault addresses and constituent pool details
- Shares SDK, API, and contract address references with gmx-trading

## Links

- [GMX Documentation](https://docs.gmx.io)
- [GMX App](https://app.gmx.io)
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk)
- [gmx-io/gmx-interface](https://github.com/gmx-io/gmx-interface)

## License

MIT
