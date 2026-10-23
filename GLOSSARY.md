# Glossary

- **Intent:** a signed statement of desired outcome with constraints.
- **Solver / filler / resolver:** party that executes intents.
- **Settlement contract:** on-chain contract enforcing order constraints.
- **Vault relayer:** separate contract that holds user allowances and only pulls tokens for verified orders.
- **Interaction:** solver-supplied call the settlement contract executes (e.g. a DEX swap).
- **Surplus:** improvement over the user's limit price returned to the user.
- **Batch auction:** auction settling many orders at uniform prices.
- **Dutch auction (intents):** order whose offered price decays over time until a filler accepts.
- **RFQ:** request for quote; market makers quote prices for a specific order.
- **Witness (Permit2):** extra signed data binding a transfer permission to an order.
- **Type hash:** `keccak256` of an EIP-712 type string; fixes which fields a signature covers.
- **Last look:** a solver's option to decline after winning.
- **Commit–reveal:** bidding where hashed bids are published before bids are revealed.
- **HHI:** Herfindahl–Hirschman index; sum of squared market shares, used here for solver win concentration.
- **Settlement oracle:** mechanism that proves a cross-chain fill so the filler can be repaid.
