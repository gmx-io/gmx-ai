# GMX Agent Skills

Agent skills for trading and providing liquidity on [GMX](https://app.gmx.io), built around `GmxApiSdk` from `@gmx-io/sdk/v2`.

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

Trade perpetuals and swap tokens on Arbitrum, Avalanche, and MegaETH using the GMX API SDK:

- Discover markets, prices, trading capacity, wallet balances, positions and orders.
- Prepare, sign, submit and track Express orders; prepare wallet-sent Classic transactions.
- Open and close positions, swap, edit/cancel orders, and change collateral.
- Use limit/stop orders, TP/SL, TWAP and authorized subaccounts.

References cover SDK examples, HTTP routes and retry behavior, order semantics, and current contract addresses.

### gmx-liquidity

Read GM pool and GLV vault data through the API SDK, and use current contracts for liquidity writes:

- Query pool state, APY, historical performance and GM fee earnings.
- Discover GLV tokens and their current constituent markets.
- Build GM/GLV deposits and withdrawals, and GM-to-GM shifts.
- Estimate keeper fees, apply output minimums and track asynchronous execution.

The API SDK does not yet build GM/GLV liquidity transactions. The skill provides contract workflows using the package's ABIs and address registry. GMX Account funding is a separate capability. Both skills are self-contained and include their own contract-address reference.

## Compatibility

The examples and address snapshot were checked against published `@gmx-io/sdk@2.1.1` on 2026-09-29. Install `@gmx-io/sdk@latest`, record the resolved version and consult its types when upgrading. For plain Node scripts with 2.1.1, use CommonJS; native ESM encounters extensionless SDK imports.

Skill files are synchronized across `skills/`, `.well-known/skills/`, and `plugins/gmx-io/skills/`.

## Links

- [GMX SDK documentation](https://docs.gmx.io/docs/sdk/v2/)
- [GMX API integration guide](https://docs.gmx.io/docs/api/integration-guide/)
- [Contract deployments](https://docs.gmx.io/docs/api/contracts/addresses/)
- [`@gmx-io/sdk` on npm](https://www.npmjs.com/package/@gmx-io/sdk)
- [gmx-io/gmx-interface](https://github.com/gmx-io/gmx-interface)

## License

MIT
