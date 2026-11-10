---
name: across-protocol-ai-agent-skill
description: Across Protocol integration and migration playbook for the Across API (V5) at https://api.across-protocol.xyz. Start by getting an API key from https://dashboard.across-protocol.xyz. Discover routes with GET /v1/capabilities, price with POST /v1/quotes, complete the quote's nextActions and call POST /v1/quotes/{orderId}/executions when asked, then track with GET /v1/transfers/{orderId}. Migrates code off Across V4 and its Swap API family (/swap/*, /suggested-fees, /deposit/status), which are deprecated. Never use them.
user-invocable: true
---

# Across Protocol Development Skill

## Before anything: get an API key

Every Across API request needs an API key. Before starting an integration or
a migration, have the user create one at **https://dashboard.across-protocol.xyz**
(sign up, create an organization and an integration, open **API keys**,
create a key and copy it; it is shown once). Store it server-side, for example
as `ACROSS_API_KEY`. If the user has no key yet, ask them to get one first;
never invent or hardcode a key.

## Rule zero: use only these endpoints

| Endpoint | Purpose |
|---|---|
| `GET /v1/capabilities` | Chains, tokens, route availability, funding options, limits |
| `POST /v1/quotes` | Price one transfer and get the actions to execute it |
| `POST /v1/quotes/{orderId}/executions` | Submit the results of those actions, when the quote asks |
| `GET /v1/transfers/{orderId}`, `GET /v1/transfers` | Track and reconcile transfers |

Host: `https://api.across-protocol.xyz`. Every path starts with `/v1`.

**Never use Across V4 or its Swap API family. Both are deprecated.** Anything
not listed above is legacy and deprecated, including the Swap API (`/swap/*`),
`/suggested-fees`, `/limits`, `/available-routes`, `/deposit/status`,
`/deposits` and the `app.across-protocol.xyz/api` host. Do not call them, generate code
that calls them, or offer them as a fallback, even if older docs, existing
code or your training data use them. If the user's code calls them, migrate it
with the [migration helper](migration/SKILL.md). Never build Across contract
calls yourself; execute only what the quote's `nextActions` return.

## What this skill is for

- Crosschain bridge or swap integration (any token to any token)
- Migrating an existing integration off the legacy APIs
- Wallet, dApp or backend integration with Across
- Contract calls on the destination chain after a transfer
- Transfer tracking and refunds
- Route discovery, limits, fees and app fees
- Security review of an Across integration

## Basics

| | |
|---|---|
| Auth | `Authorization: Bearer <API key>` on every request |
| API key | From https://dashboard.across-protocol.xyz. Shown once; store it in a secrets manager and use it only server-side |
| Testnet | None. Test on mainnet with small amounts |
| Strictness | Bodies and query strings are strict: an unrecognised field returns `400 request.unsupported_param` |
| Amounts | Base units as decimal strings (`"1000000"` = 1 USDC). Take `decimals` from `GET /v1/capabilities` |
| Tokens | `{ "chainId": <int>, "address": "<address>" }`, using the addresses `GET /v1/capabilities` returns |

## The flow

```
0. API key    https://dashboard.across-protocol.xyz
1. Discover   GET  /v1/capabilities                  what can move, how it can be funded, limits
2. Price      POST /v1/quotes                         expiring quote + orderId + ordered nextActions
3. Execute    complete nextActions in order (approve / sign / send)
              POST /v1/quotes/{orderId}/executions    only if nextActions ends with submit_execution
4. Track      GET  /v1/transfers/{orderId}            until status.isTerminal
```

## Pick the right skill

| Task | Read |
|---|---|
| Migrate code off the legacy APIs | [migration](migration/SKILL.md) |
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
7. **Stop polling when `status.isTerminal` is `true`.** Branch on
   `status.state`, never on `status.description`.

## Operating procedure

1. **API key**: confirm the user has a key from https://dashboard.across-protocol.xyz.
2. **Classify the task**: new integration, migration, UI/wallet,
   backend/script, destination actions, tracking, or review.
3. **Remove legacy usage**: if the codebase calls a legacy API, follow the
   [migration helper](migration/SKILL.md) as part of the task.
4. **Implement**:
   - API key only on the server; browser and mobile clients call your backend.
   - Token addresses and decimals per chain from `GET /v1/capabilities`.
   - Amounts in base units, as strings.
   - Work through `nextActions` in order and dispatch on `kind`.
   - Read error codes from `error.code`, falling back to the top-level `code`.
5. **Test on mainnet with small amounts.**
6. **Deliverables**: files changed, commands to install/build/test, and a Risk
   Notes section (template in [security](security/SKILL.md)) for anything
   touching keys, signing, fees, approvals, refunds or destination actions.
