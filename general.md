Shadowbook is a proprietary orderbook decentralized exchange (Prop DEX) on Cardano. Unlike traditional automated market maker (AMM) DEXes, Shadowbook implements a full orderbook matching engine. The protocol allows makers to instantly place/cancel virtual limit orders that have zero on-chain cost. On the other hand, takers get tight spreads and better prices for assets compared to traditional AMMs.

Shadowbook's mission is to solve the liquidity problem of the Cardano ecosystem by providing technology for maximum capital efficiency.

For performance reasons our tech stack is fully in Rust.

## Cardano DeFi market problem

Cardano's smart contract platform doesn't allow proper implementation of concentrated liquidity market maker pools or any other reasonably capital-efficient solution. That is why the market is full of constant product AMM DEXes, which are the least capital-efficient model in the DeFi landscape.

Due to this technical limitation, the Cardano DeFi market is struggling to gain any traction in the global crypto DeFi market.

Additional challenges include:
- Pool prices are slow to adjust to actual fast market movements.
- Spreads are wide and slippage is high, so retail traders experience poor execution.
- Placing an order on Cardano is expensive, with network (gas) fees of 1–1.5 ADA per order plus additional "batcher" and DEX fees.
- Liquidity is fragmented across multiple pools, venues, and trading pairs.
- Without user segmentation, genuine traders don't get better prices or execution compared to arbitrageurs and informed trading bots.

At the end of the day it's the loyal Cardano users who carry all of these inefficiencies on their shoulders.

## Shadowbook's solution

Shadowbook replaces the AMM model with a hybrid orderbook architecture designed for Cardano's eUTxO ledger. The protocol separates makers and takers into distinct roles, each optimized for their use case.

### Virtual orderbook for makers

Market makers connect directly to the Shadowbook matching engine and place virtual limit orders. These orders live off-chain — they are placed and cancelled instantly with no transaction fees and no on-chain footprint. This removes the cost barrier that prevents active market making on Cardano.

Since virtual orders cost nothing to maintain, makers can quote tighter spreads and update prices rapidly in response to market movements. This is impossible on constant product AMMs where liquidity sits passively on a bonding curve.

### On-chain settlement for takers

When a taker matches against the orderbook, the trade settles on-chain through Shadowbook's smart contracts. Takers submit on-chain orders (limit or market) that the protocol's execution engine fulfills when conditions are met. The on-chain component handles custody — takers retain full self-custody of their funds throughout the process.

### Capital efficiency gains

By concentrating maker liquidity in a real orderbook rather than spreading it across a constant product curve, Shadowbook delivers:

- **Tighter spreads** — makers quote competitive prices at specific levels instead of relying on a mathematical curve.
- **Lower slippage** — aggregated depth at each price level means larger orders execute with less price impact.
- **Faster price discovery** — makers can update quotes instantly in response to market movements, so prices reflect actual market conditions.
- **Reduced cost for traders** — virtual orders eliminate the per-order on-chain cost for makers, and takers benefit from better execution quality.

### Integration approach

Shadowbook does not operate its own trading interface. Instead, it plugs into existing Cardano infrastructure as a liquidity source. Currently integrated with DexHunter, with plans to extend to other DEXes and aggregators. This approach lets traders access Shadowbook's liquidity through the interfaces they already use.
