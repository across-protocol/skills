# Chains, Tokens, Routes and Limits

`GET /v1/capabilities` is the single discovery endpoint: supported chains,
origin and destination tokens, route availability, how a route can be funded,
and indicative limits. Never use the legacy `/swap/chains`, `/swap/tokens`,
`/swap/sources`, `/available-routes` or `/limits`.

## GET /v1/capabilities

```
GET https://api.across.to/v1/capabilities?originChainId=42161&destinationChainId=8453&include=tokens
Authorization: Bearer <API key>
```

### Query parameters

| Parameter | Description |
|---|---|
| `include` | `chains`, `tokens` (repeatable). Defaults to both |
| `originChainId`, `destinationChainId` | Narrow each side to one chain |
| `originToken`, `destinationToken` | Token address or symbol. Requires the matching chain id |
| `reachableOnly` | `true` drops rows that cannot be quoted right now |
| `search` | Free-text match on symbol, name or address |
| `rails` | Evaluate routes on one rail only: `across`, `cctp`, `oft`, `paxos` |
| `capabilities.operations` | What the client can do, as `kind:ecosystem` (for example `sign_typed_data:evm`) or `submit_execution`. Repeatable. Tokens nothing you declared can fund come back unavailable |
| `limit` | Page size per token side, 1 to 100, default 10 |
| `originCursor`, `destinationCursor` | Opaque cursors from `pagination.origin.nextCursor` / `pagination.destination.nextCursor` |

The query is strict: an unknown parameter returns `400`. The legacy names
`inputToken` / `outputToken` are now `originToken` / `destinationToken`.

### Response

| Field | Contents |
|---|---|
| `chains[]` | `chainId`, `name`, `displayName`, `namespace` (`evm`, `solana`, `tron`), `canBeOrigin`, `canBeDestination`, `nativeToken`, `availability`, per-rail health |
| `originTokens[]`, `destinationTokens[]` | `chainId`, `address`, `symbol`, `name`, `decimals`, `priceUsd` (advisory), `verified`, `availability` |
| `pagination.origin`, `pagination.destination` | `nextCursor` (absent on the last page), `total` |
| `capabilities` | Present only when all four of `originChainId`, `originToken`, `destinationChainId`, `destinationToken` are set and a route exists |

`availability.status` is `available`, `degraded` (still quotes), `incompatible`
(usable only with a different opposite selection) or `unavailable`, with
`reasons` explaining why.

The `capabilities` block for one route:

| Field | Meaning |
|---|---|
| `availability` | The route's status |
| `amountTypes` | Which of `exact_input`, `exact_output`, `min_output` the route serves |
| `executionOptions` | Funding plans in preference order, as the operations a client must perform. Empty means nothing you declared can fund it |
| `limits` | Indicative min/max in base units. **Absent means unknown, never unbounded.** The quote is the only binding check |
| `gasTopUp` | The most destination native gas you can request with `gasTopUp` |

## Rules

- **Availability means selectable, not guaranteed to price.** `POST /v1/quotes`
  is the binding check for a specific amount.
- **Take `decimals` from the response.** The same asset can have different
  decimals on different chains.
- **Paginate by cursor.** Keep requesting while `nextCursor` is present; a page
  can hold fewer rows than `limit` and still continue.
- **Native assets** use `0x0000000000000000000000000000000000000000` (EVM) or
  `11111111111111111111111111111111` (Solana). Wrapped SOL is an ordinary SPL
  token.
- **Don't hardcode chain lists or token addresses.** Support changes; query the
  API.
- **Don't hardcode Across contract addresses.** Approval spenders and
  transaction targets come from the quote's `nextActions`.

## Reference token addresses

Only for building examples. Verify against `GET /v1/capabilities` before use.

### USDC
| Chain | Chain ID | Address |
|---|---|---|
| Ethereum | 1 | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| Arbitrum | 42161 | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` |
| Base | 8453 | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Optimism | 10 | `0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85` |
| Polygon | 137 | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |
