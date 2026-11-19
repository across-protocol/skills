# Chains, Tokens, Routes and Limits

`GET /v1/capabilities` lists the chains and tokens available to move between,
flags anything unavailable with a reason, and, for one fully specified route,
returns how it can be executed and its indicative limits. It is the only
discovery endpoint; do not use legacy ones.

## GET /v1/capabilities

Needs an API key. Request access at https://docs.across.to/introduction#get-your-api-key.

```bash
curl -H "Authorization: Bearer $ACROSS_API_KEY" \
  "https://api.across.to/v1/capabilities?originChainId=42161&destinationChainId=8453&include=tokens"
```

### Query parameters

| Parameter | Description |
|---|---|
| `include` | `chains`, `tokens`. Defaults to both |
| `originChainId`, `destinationChainId` | Chain ids |
| `originToken`, `destinationToken` | Token address or symbol. Requires the matching chain id |
| `reachableOnly` | `true` returns only tokens that can be quoted against the opposite selection (`available` or `degraded`). Default `false` |
| `search` | Free-text match on symbol, name or address |
| `capabilities.operations` | Operations your client can perform, as `kind:ecosystem` (for example `approve_token:evm`) or `submit_execution`. Repeat or comma-separate. Tokens none of them can fund come back `unavailable` with `unsupported_funding_method` |
| `rails` | Evaluate routes on one rail only: `across`, `cctp`, `oft`, `paxos`. `capabilities.limits` is not returned with it |
| `limit` | Page size per token side, 1 to 100, default 10 |
| `originCursor`, `destinationCursor` | Cursors from `pagination.origin.nextCursor` / `pagination.destination.nextCursor` |

The query is strict: an unrecognised parameter returns `400`.

### Response

| Field | Contents |
|---|---|
| `chains[]` | `chainId`, `name`, `displayName`, `namespace` (`evm`, `solana`, `tron`), `canBeOrigin`, `canBeDestination`, `nativeToken`, `explorerUrl`, `logoUrl`. Not paginated |
| `originTokens[]`, `destinationTokens[]` | `chainId`, `address`, `symbol`, `displaySymbol`, `name`, `decimals`, `priceUsd` (advisory, display only), `verified`, `availability` |
| `pagination.origin`, `pagination.destination` | `nextCursor` (absent on the last page), `total` |
| `capabilities` | Present only when `originChainId`, `originToken`, `destinationChainId` and `destinationToken` are all set and the route resolves |
| `meta.unresolvedAnchors` | Anchors that could not be resolved, so you can tell an empty result from a typo |

Token `availability.status`:
- `available`: selectable as things stand.
- `degraded`: selectable and still quotes, but at risk; `reasons` says why.
- `incompatible`: a route exists only after clearing the opposite chain and token.
- `unavailable`: no usable route even then.

### The `capabilities` block (one route)

| Field | Meaning |
|---|---|
| `availability` | `available`, `degraded` (still quotes) or `unavailable` (does not quote), with `reasons` |
| `amountTypes` | Quote modes the route serves |
| `executionOptions[].requiredOperations` | Each option is one way to execute the pair and needs every operation it lists. Narrowed by the `capabilities.operations` you sent |
| `limits.min`, `limits.max` | Indicative bounds in the origin token. `limits.destination.max` gives the bound in the destination token for output-based quotes |
| `limits.validUntil` | When the snapshot goes stale |
| `gasTopUp` | Whether a `gasTopUp` is supported, and `limits.max.amount` for it |

## Rules

- **Availability means selectable, not priceable.** `POST /v1/quotes` is the
  binding check for a specific amount.
- **Limits are a snapshot.** An absent bound means unknown, not unlimited.
  Re-read after `validUntil`.
- **Take `decimals` from the response.** The same asset can have different
  decimals on different chains.
- **Use the token addresses the API returns**, including for native tokens.
  Addresses come back lowercased; compare case-insensitively.
- **Follow cursors** until `nextCursor` is absent; a page can hold fewer rows
  than `limit`, or none, and still continue. `409 conflict.cursor_expired`
  means start again from the first page.
- **Don't hardcode chain or token lists.** Query the API.
