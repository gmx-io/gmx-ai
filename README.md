# GMX Agent Skills

Agent skills for trading on [GMX V2](https://app.gmx.io).

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

## Links

- [GMX Documentation](https://docs.gmx.io)
- [GMX App](https://app.gmx.io)
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk)
- [gmx-io/gmx-interface](https://github.com/gmx-io/gmx-interface)

## License

MIT
