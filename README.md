# Across Protocol — AI Agent Knowledge Base

Skills that let AI agents integrate Across, the crosschain bridge and swap
protocol, through the **Across API** (V5) at `https://api.across.to`, and help
migrate existing integrations off the legacy APIs.

## Start here: get an API key

Before starting an Across integration or migration, get an API key from
**https://dashboard.across.to**. Every request needs it, as
`Authorization: Bearer <key>`, sent from your server only.

## The one rule

Use only these endpoints:

| Endpoint | Purpose |
|---|---|
| `GET /v1/capabilities` | Chains, tokens, routes, funding options, limits |
| `POST /v1/quotes` | Price a transfer and get the actions to execute it |
| `POST /v1/quotes/{orderId}/executions` | Submit those actions' results, when the quote asks |
| `GET /v1/transfers/{orderId}`, `GET /v1/transfers` | Track and reconcile transfers |

**Never use the legacy APIs.** Anything else, including the Swap API
(`/swap/*`), `/suggested-fees`, `/limits`, `/available-routes`,
`/deposit/status`, `/deposits` and the `app.across.to/api` host, is
deprecated. Do not call it or generate code that does, even if older material
suggests it. If existing code uses it, migrate with
[`skills/migration/SKILL.md`](skills/migration/SKILL.md).

## What is Across

Across moves tokens between chains. You describe the transfer, the Across API
prices it into an expiring quote with a guaranteed minimum, relayers compete
for the order, and most transfers fill in seconds. One API covers bridging,
crosschain swaps, same-chain swaps and contract calls on the destination
chain.

## File index

| File | Purpose | Read when... |
|------|---------|--------------|
| [`skills/SKILL.md`](skills/SKILL.md) | **Start here.** API key, endpoints, legacy ban, basics, flow, defaults | Always |
| [`skills/migration/SKILL.md`](skills/migration/SKILL.md) | Migration helper: find legacy calls, map endpoints and fields, before/after code, verification | Code calls `/swap/*`, `/suggested-fees`, `/deposit/status` or other legacy APIs |
| [`skills/fetch-chains-tokens/SKILL.md`](skills/fetch-chains-tokens/SKILL.md) | `GET /v1/capabilities` | Looking up chains, tokens, routes or limits |
| [`skills/quote-and-execute/SKILL.md`](skills/quote-and-execute/SKILL.md) | `POST /v1/quotes`, `nextActions`, `POST /v1/quotes/{orderId}/executions`, errors | Building a bridge or swap |
| [`skills/embedded-crosschain-actions/SKILL.md`](skills/embedded-crosschain-actions/SKILL.md) | `destinationActions` on `POST /v1/quotes` | Bridge, then deposit/stake/wrap on arrival |
| [`skills/tracking-transactions/SKILL.md`](skills/tracking-transactions/SKILL.md) | `GET /v1/transfers/{orderId}`, `GET /v1/transfers`, refunds | Tracking or reconciling transfers |
| [`skills/security/SKILL.md`](skills/security/SKILL.md) | Checklist, Risk Notes template | Anything touching keys, signing, approvals, fees, refunds |
| [`skills/resources/SKILL.md`](skills/resources/SKILL.md) | Links | You need a URL |

## Quick start for agents

1. **API key**: get one at https://dashboard.across.to. Use it only
   server-side, as `Authorization: Bearer <key>`.
2. **Discover**: `GET /v1/capabilities` for chains, tokens, decimals, limits.
3. **Price**: `POST /v1/quotes` with `origin` and `destination` tokens as
   `{ chainId, address }`, amounts in base units, and `maxFee`.
4. **Execute**: complete `nextActions` in order. If the last one is
   `submit_execution`, call `POST /v1/quotes/{orderId}/executions`.
5. **Track**: `GET /v1/transfers/{orderId}` until `status.isTerminal` is
   true.

Migrating? Do step 1, then follow
[`skills/migration/SKILL.md`](skills/migration/SKILL.md).

There is no testnet. Test on mainnet with small amounts.

## Common pitfalls

| Mistake | Correct approach |
|---------|-----------------|
| Starting without an API key | Get one at https://dashboard.across.to first |
| Calling a legacy endpoint | Use only the `/v1` endpoints above |
| Copying legacy `slippage` into `maxFee` | `maxFee` is a total USD value-loss cap, not slippage; set a bound the user agrees to |
| Putting the API key in a frontend | Call the Across API from your backend |
| Hardcoding decimals, token addresses or chain lists | Read them from `GET /v1/capabilities` |
| Omitting `maxFee` | Set it; the default can allow up to 50% value loss |
| Building approvals or contract calls yourself | Follow `nextActions` exactly |
| Always calling the executions endpoint | Only when `nextActions` ends with `submit_execution` |
| Tracking by transaction hash | Track by `orderId` |
| Marking a transfer failed when `expectedFillTimeMs` passes | Only `status` decides |
| Sending unknown fields | Requests are strict; unknown fields return `400` |

## Output expectations for agents

1. Cite exact endpoints: method, `/v1` path, required fields.
2. State defaults explicitly: `amountType`, `maxFee`, approval size.
3. Include failure modes: expired quotes, `policy.amount_out_of_range`,
   refunds.
4. Add Risk Notes (template in `skills/security/SKILL.md`).
5. Provide runnable code covering quote → nextActions → execute → track.

## Updating this repo

- Reflect only the public Across API documentation (https://docs.across.to).
- Document only the `/v1` endpoints above. Legacy endpoints appear only in the
  migration helper, as things to replace.
- Prefer tables and checklists over prose; keep every file useful on its own.
