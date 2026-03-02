# AQUARI Subscription & Affiliate System

Recurring AQUARI token accumulation system with affiliate program on Base. Users commit to monthly purchases (6/12 months), receive AQUARI directly to their Privy wallet each month, and earn a 5% bonus on completion.

## Documents

| Document | Description |
|----------|-------------|
| [Proposal A — Smart Contract](./proposal-a-smart-contract.md) | Vault + atomic swap + affiliate + bonus in 1 tx per month. Full flow diagrams, tx counts, admin dashboard, pros/cons. |
| [Proposal B — Backend-Only](./proposal-b-backend-only.md) | No custom contract. Backend calls Uniswap V2 directly. 1-3 txs per month. Full flow diagrams, tx counts, pros/cons. |
| [Notes & Recommendations](./notes-and-recommendations.md) | Inflow dependency risks, referral split suggestion (75/25), full-stack ownership argument, timeline estimate. |

## Quick Summary

- **Auth**: Privy wallet (Base)
- **Payments**: Fiat (Inflow Pay) + Crypto (vault deposit)
- **Swap**: Uniswap V2 on Base (fee-on-transfer compatible)
- **Token**: AQUARI (5% buy tax, 5% sell tax — not modifiable)
- **Affiliate**: 5% reward from treasury, single-level
- **Completion Bonus**: 5% of total AQUARI accumulated
- **Estimated Timeline**: 3-4 weeks + Inflow buffer
