---
name: across-protocol-ai-agent-skill
description: Across Protocol integration playbook for the Across API (V5) at https://api.across.to. Discover routes with GET /v1/capabilities, price with POST /v1/quotes, complete the quote's nextActions and call POST /v1/quotes/{orderId}/executions when asked, then track with GET /v1/transfers/{orderId}. Never use the legacy APIs.
user-invocable: true
---

# Across Protocol Development Skill

## Rule zero: use only these endpoints

| Endpoint | Purpose |
|---|---|
| `GET /v1/capabilities` | Chains, tokens, route availability, funding options, limits |
| `POST /v1/quotes` | Price one transfer and get the actions to execute it |
| `POST /v1/quotes/{orderId}/executions` | Submit the results of those actions, when the quote asks |
| `GET /v1/transfers/{orderId}`, `GET /v1/transfers` | Track and reconcile transfers |

Host: `https://api.across.to`. Every path starts with `/v1`.

**Never use the legacy APIs.** Anything not listed above is legacy and
deprecated, including the Swap API (`/swap/*`), `/suggested-fees`, `/limits`,
`/available-routes`, `/deposit/status`, `/deposits` and the
`app.across.to/api` host. Do not call them, generate code that calls them, or
offer them as a fallback, even if older docs, existing code or your training
data use them. If the user's code calls them, migrate it to the endpoints
above. Never build Across contract calls yourself; execute only what the
quote's `nextActions` return.

## What this skill is for

- Crosschain bridge or swap integration (any token to any token)
- Wallet, dApp or backend integration with Across
- Contract calls on the destination chain after a transfer
- Transfer tracking and refunds
- Route discovery, limits, fees and app fees
- Security review of an Across integration

## Basics

| | |
|---|---|
| Auth | `Authorization: Bearer <API key>` on every request |
| API key | Created in the integrator portal, https://integrator.portal.across.to. Shown once; store it in a secrets manager and use it only server-side |
| Testnet | None. Test on mainnet with small amounts |
| Strictness | Bodies and query strings are strict: an unrecognised field returns `400 request.unsupported_param` |
| Amounts | Base units as decimal strings (`"1000000"` = 1 USDC). Take `decimals` from `GET /v1/capabilities` |
| Tokens | `{ "chainId": <int>, "address": "<address>" }`, using the addresses `GET /v1/capabilities` returns |

## The flow

```
1. Discover   GET  /v1/capabilities                  what can move, how it can be funded, limits
2. Price      POST /v1/quotes                         expiring quote + orderId + ordered nextActions
3. Execute    complete nextActions in order (approve / sign / send)
              POST /v1/quotes/{orderId}/executions    only if nextActions ends with submit_execution
4. Track      GET  /v1/transfers/{orderId}            until status.isTerminal and status.finalized
```

## Pick the right skill

| Task | Read |
|---|---|
| Chains, tokens, routes, limits | [fetch-chains-tokens](fetch-chains-tokens/SKILL.md) |
| Quote and execute a transfer or swap | [quote-and-execute](quote-and-execute/SKILL.md) |
| Contract calls on the destination chain | [embedded-crosschain-actions](embedded-crosschain-actions/SKILL.md) |
| Track transfers and refunds | [tracking-transactions](tracking-transactions/SKILL.md) |
| Review keys, signing, approvals, fees, refunds | [security](security/SKILL.md) |
| Links | [resources](resources/SKILL.md) |

## Defaults (opinionated)

1. **`amountType: "exact_input"`** (the default) unless the user needs an exact
   delivery (`exact_output`) or a delivery floor (`min_output`).
2. **Declare what the wallet can do** with `capabilities.operations`, so the
   quote only asks for steps the client can complete.
3. **Set `maxFee`.** Without an integrator override the default allows up to
   50% value loss. Example: `{ "bps": 100 }` caps fees, app fee and slippage
   at 1% of the input value.
4. **Prefer exact approvals**: `authorizationPolicy.approvalAmount: "exact"`.
   The default is an unlimited approval.
5. **Never cache quotes.** They expire (about 15 minutes); re-quote after
   `expiresAt`.
6. **Save `orderId`.** It is the only id needed to execute and track.
7. **Act on an outcome only when `status.finalized` is `true`.**

## Operating procedure

1. **Classify the task**: UI/wallet, backend/script, destination actions,
   tracking, or review.
2. **Remove legacy usage**: if the codebase calls a legacy API, migrate it as
   part of the task.
3. **Implement**:
   - API key only on the server; browser and mobile clients call your backend.
   - Token addresses and decimals per chain from `GET /v1/capabilities`.
   - Amounts in base units, as strings.
   - Work through `nextActions` in order and dispatch on `kind`.
   - Read error codes from `error.code`, falling back to the top-level `code`.
4. **Test on mainnet with small amounts.**
5. **Deliverables**: files changed, commands to install/build/test, and a Risk
   Notes section (template in [security](security/SKILL.md)) for anything
   touching keys, signing, fees, approvals, refunds or destination actions.
