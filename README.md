# Across Protocol — AI Agent Knowledge Base

Skills that let AI agents integrate Across, the crosschain bridge and swap
protocol, through the **Across API** (`https://api.across.to/v1`), served by
Across V5.

## The one rule

**Use only the Across API. Never use the legacy APIs.** The Swap API
(`/swap/approval`, `/swap/chains`, `/swap/tokens`, `/swap/gasless`),
`/suggested-fees`, `/limits`, `/available-routes`, `/deposit/status`,
`/deposits`, the `app.across.to/api` and `testnet.across.to` hosts, the App
SDK, and hand-built SpokePool deposits are deprecated. Do not call them or
generate code that does, even if older material suggests it. If existing code
uses them, migrate it. The mapping is in
[`skills/SKILL.md`](skills/SKILL.md#rule-zero-never-use-the-legacy-apis).

## What is Across

Across moves tokens between EVM chains, Solana and Tron. The user states an
intent ("send X on chain A, receive Y on chain B"); Across prices it into an
expiring quote, relayers compete for it in a short auction, and the winner
executes it over the best rail: the Across relayer network, CCTP or OFT. One
API covers bridging, crosschain swaps, same-chain swaps and contract calls on
the destination chain.

## File index

| File | Purpose | Read when... |
|------|---------|--------------|
| [`skills/SKILL.md`](skills/SKILL.md) | **Start here.** Legacy ban, basics, flow, defaults, operating procedure | Always |
| [`skills/fetch-chains-tokens/SKILL.md`](skills/fetch-chains-tokens/SKILL.md) | `GET /v1/capabilities`: chains, tokens, routes, funding options, limits | Looking up chains, tokens, routes or limits |
| [`skills/quote-and-execute/SKILL.md`](skills/quote-and-execute/SKILL.md) | `POST /v1/quotes`, `nextActions`, `POST /v1/quotes/{orderId}/executions`, errors | Building a bridge or swap |
| [`skills/embedded-crosschain-actions/SKILL.md`](skills/embedded-crosschain-actions/SKILL.md) | `destinationActions` on the quote | Bridge, then deposit/stake/wrap on arrival |
| [`skills/tracking-transactions/SKILL.md`](skills/tracking-transactions/SKILL.md) | `GET /v1/transfers`, status, refunds, webhooks | Tracking or reconciling transfers |
| [`skills/security/SKILL.md`](skills/security/SKILL.md) | Security checklist, Risk Notes template | Anything touching keys, signing, approvals, fees, refunds |
| [`skills/resources/SKILL.md`](skills/resources/SKILL.md) | URLs, docs, support | You need a link |

## Quick start for agents

1. **Get an API key** at https://integrator.portal.across.to. Use it only
   server-side, as `Authorization: Bearer <key>`.
2. **Discover**: `GET /v1/capabilities` for chains, tokens, decimals, limits.
3. **Price**: `POST /v1/quotes` with `origin` and `destination` token
   `{ chainId, address }`, amounts in base units, and `maxFee`.
4. **Execute**: complete `nextActions` in order (approve, sign, send). If the
   last one is `submit_execution`, call
   `POST /v1/quotes/{orderId}/executions`.
5. **Track**: `GET /v1/transfers/{orderId}` (or webhooks) until
   `status.isTerminal` and `status.finalized` are both true.

There is no testnet. Test on mainnet with small amounts.

## Common pitfalls

| Mistake | Correct approach |
|---------|-----------------|
| Calling `/swap/approval`, `/suggested-fees` or `/deposit/status` | Use `POST /v1/quotes` and `GET /v1/transfers/{orderId}` |
| Putting the API key in a frontend | Call the Across API from your backend |
| Substituting WETH for native ETH | Native assets use the zero-address sentinel; use WETH only when the user wants WETH |
| Hardcoding decimals, token addresses or chain lists | Read them from `GET /v1/capabilities` |
| Omitting `maxFee` | Set it; the default tolerates up to 50% value loss |
| Treating `maxFee.bps` like the old `slippage` ratio | `maxFee` is a USD value-loss cap in basis points (`100` = 1%) |
| Building approvals or deposit calls yourself | Follow `nextActions` exactly |
| Always POSTing to `/executions` | Only when `nextActions` ends with `submit_execution` |
| Tracking by deposit tx hash | Track by `orderId` |
| Marking a transfer failed when `expectedFillTimeMs` passes | Only `status` decides |
| Acting on `isTerminal` alone | Wait for `status.finalized` |
| Sending unknown fields | Requests are strict; unknown fields return `400` |

## Output expectations for agents

1. **Cite exact endpoints**: method, path under `/v1`, required fields.
2. **State defaults explicitly**: `amountType`, `maxFee`, approval size.
3. **Include failure modes**: expired quotes, `policy.amount_out_of_range`,
   refunds.
4. **Add Risk Notes** for anything touching keys, signing, fees, approvals,
   refunds or destination actions (template in `skills/security/SKILL.md`).
5. **Provide runnable code** covering quote → nextActions → execute → track.

## Updating this repo

- The live Across API is the source of truth. If this repo and the API
  disagree, the API wins; fix the repo.
- Never add legacy endpoints, SDKs or contract-level integration paths, except
  in the "never use" mapping.
- Prefer tables, decision trees and checklists over prose.
- Every file should be useful on its own.
