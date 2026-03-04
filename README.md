# GMX Agent Skills

Agent skills for trading and providing liquidity on [GMX V2](https://app.gmx.io).

## Install

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

Provide liquidity on GMX V2 — deposit into GM pools and GLV vaults to earn trading fees.

**Capabilities:**
- Deposit into GM pools (single or dual token)
- Withdraw from GM pools
- Shift liquidity between GM pools (no swap fees)
- GLV vault deposits (token or GM token mode)
- GLV vault withdrawals
- Cancel pending LP operations

**Reference files included:**
- Contract addresses for ExchangeRouter, GlvRouter, and all vaults per chain

## Links

- [GMX Documentation](https://docs.gmx.io)
- [GMX App](https://app.gmx.io)
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk)
- [gmx-io/gmx-interface](https://github.com/gmx-io/gmx-interface)

## License

MIT
