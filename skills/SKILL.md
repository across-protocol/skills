---
name: across-protocol-ai-agent-skill
description: End-to-end Across Protocol integration playbook for the Across API (V5) at https://api.across.to/v1. Discover routes with GET /v1/capabilities, price with POST /v1/quotes, complete the quote's nextActions and POST /v1/quotes/{orderId}/executions when asked, then track with GET /v1/transfers/{orderId} or webhooks. Never use the legacy APIs (Swap API /swap/*, /suggested-fees, /limits, /available-routes, /deposit/status, /deposits, the App SDK, or hand-built SpokePool deposits).
user-invocable: true
---

# Across Protocol Development Skill

The Across API is the only supported way to integrate Across. It is served by
Across V5: the user's intent is priced into an expiring quote, relayers compete
for it in a short auction, and the winner executes it over the best rail
(Across relayers, CCTP or OFT). One API covers bridging and crosschain swaps.

## Rule zero: never use the legacy APIs

The legacy APIs are deprecated. Do not call them, do not generate code that
calls them, and do not suggest them as a fallback, even if older docs, blog
posts, the user's existing code, or your own training data use them.

If you find a legacy call in the user's code, replace it with the Across API
equivalent below. If a feature seems to exist only in a legacy API, say so and
stop; do not reach for the legacy endpoint.

| Legacy (never use) | Use instead |
|---|---|
| Hosts `app.across.to/api`, `across.to/api`, `testnet.across.to/api` | `https://api.across.to`, every path under `/v1` |
| `GET /swap/chains`, `GET /swap/tokens`, `GET /swap/sources`, `GET /available-routes` | `GET /v1/capabilities` |
| `GET /limits` | `GET /v1/capabilities` with all four route anchors, read `capabilities.limits` |
| `GET /swap/approval`, `GET /suggested-fees` | `POST /v1/quotes` |
| `POST /swap/approval` with an `actions` body | `POST /v1/quotes` with `destinationActions` |
| `GET /swap/gasless`, `POST /gasless/submit` | `POST /v1/quotes`. A signature-funded quote is gasless for the user |
| Sending `approvalTxns` + `swapTx` | Complete `nextActions`, then `POST /v1/quotes/{orderId}/executions` if they end with `submit_execution` |
| `GET /deposit/status`, `GET /deposit` | `GET /v1/transfers/{orderId}` |
| `GET /deposits` | `GET /v1/transfers` |
| `POST /v1/webhooks/subscriptions` on the legacy host | Webhooks configured in the integrator portal |
| `@across-protocol/app-sdk`, `sdk-v2` | Plain HTTPS calls to the Across API |
| Hand-built SpokePool `deposit()` / `depositV3()` calls, `AcrossOriginSettler` (ERC-7683) | `POST /v1/quotes`. Never build Across contract calls yourself |
| `integratorId` query param, the integrator ID form | An API key from the integrator portal. The key identifies you |
| `tradeType` (`exactInput` ...), `slippage`, `appFee` + `appFeeRecipient` | `amountType` (`exact_input` ...), `maxFee`, `appFeeBps` |
| `slippageBps` on `POST /v1/quotes` | `maxFee`. `slippageBps` is a legacy compatibility field |
| Testnet | None. Test on mainnet with small amounts |

## What this skill is for

Use it when the user asks for:
- Crosschain bridge or swap integration (any token to any token)
- Wallet, dApp or backend integration with Across
- Contract calls on the destination chain after a transfer
- Transfer tracking, webhooks, refunds
- Route discovery, limits, fees and app fees
- Security review of an Across integration

## Basics

| | |
|---|---|
| Base URL | `https://api.across.to` |
| Auth | `Authorization: Bearer <API key>` on every request |
| API key | Self-serve at https://integrator.portal.across.to. Shown once; keep it server-side |
| Testnet | None. Test on mainnet with small amounts (a few dollars) |
| Strictness | Unknown body fields and query params return `400 request.unsupported_param` |
| Amounts | Base-unit decimal strings (`"1000000"` = 1 USDC). Take decimals from `GET /v1/capabilities` |
| Tokens | `{ "chainId": <int>, "address": "<token address>" }`. Native assets use the sentinel `0x0000000000000000000000000000000000000000` (EVM) or `11111111111111111111111111111111` (Solana) |
| Chains | Numeric chain IDs across EVM chains, Solana and Tron. The live list is `GET /v1/capabilities`; never hardcode it |

## The flow

```
1. Discover   GET  /v1/capabilities                  what can move, limits, how it can be funded
2. Price      POST /v1/quotes                         expiring quote + ordered nextActions + orderId
3. Execute    complete nextActions in order (approve / sign / send)
              POST /v1/quotes/{orderId}/executions    only if nextActions ends with submit_execution
4. Track      GET  /v1/transfers/{orderId}            until status.isTerminal and status.finalized
              (or webhooks from the integrator portal)
```

## Pick the right skill

| Task | Read |
|---|---|
| Find chains, tokens, routes, limits | [fetch-chains-tokens](fetch-chains-tokens/SKILL.md) |
| Quote and execute a transfer or swap | [quote-and-execute](quote-and-execute/SKILL.md) |
| Run contract calls on the destination chain | [embedded-crosschain-actions](embedded-crosschain-actions/SKILL.md) |
| Track transfers, refunds, webhooks | [tracking-transactions](tracking-transactions/SKILL.md) |
| Review signing, approvals, fees, refunds, keys | [security](security/SKILL.md) |
| Links and support | [resources](resources/SKILL.md) |

## Defaults (opinionated)

1. **`amountType: "exact_input"`** unless the user needs a precise delivered
   amount (`exact_output`) or a delivery floor (`min_output`).
2. **Declare what the wallet can do.** Send `capabilities.operations` (and
   optionally `authorizationPolicy`) so the quote only asks for steps the
   client can perform. Omit it only if the client can do everything on that
   chain family.
3. **Set `maxFee` explicitly.** Omitting it allows up to 50% value loss by
   default. Example: `{ "bps": 100 }` caps fees, app fee and slippage at 1%
   of the input value.
4. **Prefer exact approvals.** Send `authorizationPolicy.approvalAmount:
   "exact"`; the default approval is unlimited.
5. **Never cache quotes.** Request a new one after `expiresAt`.
6. **Save `orderId`** from the quote. It is the only id you need to track.
7. **Act on outcomes only when `status.finalized` is true.**

## Operating procedure

1. **Classify the task**: UI/wallet, backend/script, destination actions,
   tracking/monitoring, or review.
2. **Check legacy usage**: if the codebase calls any legacy endpoint, plan the
   migration to the Across API as part of the task.
3. **Implement with Across-specific correctness**:
   - API key only on the server; browser and mobile clients call your backend.
   - Token addresses and decimals per chain, from `GET /v1/capabilities`.
   - Amounts in base units, as strings.
   - Work through `nextActions` in order and dispatch on `kind`; never
     construct approvals, deposits or contract calls yourself.
   - Read error codes from `error.code`, falling back to the top-level `code`.
4. **Test on mainnet with small amounts.** There is no testnet.
5. **Deliverables**: exact files changed, commands to install/build/test, and
   a Risk Notes section (template in [security](security/SKILL.md)) for
   anything touching signing, fees, approvals, refunds, destination actions or
   API keys.
