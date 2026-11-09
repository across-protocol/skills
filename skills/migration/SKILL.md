# Migration Helper: Legacy APIs → Across API

Use this skill when a codebase still calls Across V4 or its Swap API family,
which are deprecated and must not be used, and the user wants to move to the
Across API (`https://api.across.to/v1`). Work through the steps in order. The
goal is zero legacy calls left, not a mix of old and new.

## Step 0: API key first

Before changing any code, the user needs an API key from
**https://dashboard.across.to**. The legacy `integratorId` does not carry
over; the key alone identifies the integration.

1. Sign up at https://dashboard.across.to and create an organization and an
   integration.
2. Open **API keys**, create a key and copy it. It is shown once.
3. Store it server-side (secrets manager or environment variable, for example
   `ACROSS_API_KEY`). Never put it in frontend code.
4. Check it works:
   ```bash
   curl -H "Authorization: Bearer $ACROSS_API_KEY" \
     "https://api.across.to/v1/capabilities?originChainId=42161&destinationChainId=8453&include=tokens"
   ```
   A `401` means the key is missing, wrong, revoked or expired.

If the user has no key yet, stop here and ask them to create one. Do not
invent a placeholder key in committed code.

## Step 1: Find every legacy call

Search the codebase for these. Each hit must be migrated or removed.

| Search for | What it is |
|---|---|
| `app.across.to/api`, `across.to/api`, `testnet.across.to` | Legacy hosts |
| `/swap/approval`, `/swap/chains`, `/swap/tokens`, `/swap/sources`, `/swap/gasless`, `/gasless/submit`, `/swap/counterfactual` | Swap API |
| `/suggested-fees`, `/limits`, `/available-routes` | Legacy quote and route endpoints |
| `/deposit/status`, `/deposit?`, `/deposits` | Legacy tracking |
| `/deposit-addresses`, `/v1/webhooks/subscriptions` | Legacy deposit-address API |
| `@across-protocol/app-sdk` | App SDK (wraps the legacy APIs) |
| `depositV3`, `SpokePool` | Hand-built deposit calls |
| `integratorId`, `tradeType`, `appFeeRecipient`, `slippage=` | Legacy parameters |

Example search:

```bash
rg -n -e 'across\.to/api' -e '/swap/(approval|chains|tokens|sources|gasless|counterfactual)' \
  -e 'suggested-fees' -e '/available-routes' -e '/limits' -e '/deposit(/status|s)?\b' \
  -e 'gasless/submit' -e 'deposit-addresses' -e 'webhooks/subscriptions' \
  -e '@across-protocol/app-sdk' -e 'depositV3' -e 'integratorId' -e 'tradeType'
```

Show the user the list of hits and the planned replacement for each before
editing.

## Step 2: Map each call

| What the code does | Legacy | Across API |
|---|---|---|
| Find chains, tokens, routes | `GET /swap/chains`, `GET /swap/tokens`, `GET /available-routes` | `GET /v1/capabilities` |
| Check limits | `GET /limits` | `GET /v1/capabilities` with `originChainId`, `originToken`, `destinationChainId`, `destinationToken` all set; read `capabilities.limits` |
| Get a quote and fees | `GET /swap/approval`, `GET /suggested-fees` | `POST /v1/quotes` |
| Send the transfer | Send `approvalTxns`, then `swapTx` | Complete the quote's `nextActions`; if they end with `submit_execution`, call `POST /v1/quotes/{orderId}/executions` |
| Gasless transfers | `GET /swap/gasless`, then `POST /gasless/submit` | Same quote and execution calls. When the token supports it, the quote asks for a signature instead of a transaction |
| Destination contract calls | `POST /swap/approval` with an `actions` body | `destinationActions` on `POST /v1/quotes` (EVM to EVM only) |
| Track one transfer | `GET /deposit/status`, `GET /deposit` | `GET /v1/transfers/{orderId}` |
| List transfers | `GET /deposits` | `GET /v1/transfers` |
| Transfer notifications | `POST /v1/webhooks/subscriptions` | Configure webhooks in https://dashboard.across.to |
| Build deposits through an SDK or contract calls | `@across-protocol/app-sdk`, SpokePool `depositV3` | `POST /v1/quotes` + `nextActions` |

`GET /v1/capabilities` renames `inputToken` / `outputToken` to `originToken` /
`destinationToken`.

## Step 3: Map request fields

| Legacy | Across API (`POST /v1/quotes` body) |
|---|---|
| `inputToken` + `originChainId` | `origin.token: { chainId, address }` |
| `outputToken` + `destinationChainId` | `destination.token: { chainId, address }` |
| `amount` | `origin.amount`, or `destination.amount` for `exact_output` / `min_output` |
| `depositor` | `origin.depositor` |
| `recipient` | `destination.recipient` |
| `tradeType`: `exactInput`, `minOutput`, `exactOutput` | `amountType`: `exact_input`, `min_output`, `exact_output` |
| `appFee` (decimal, `0.01` = 1%) + `appFeeRecipient` | `appFeeBps` (basis points, `100` = 1%). No recipient: Across pays your share monthly |
| `slippage` (`0.005` or `auto`) | `maxFee: { bps, fixedUsd }` |
| `integratorId` | Remove. The API key identifies you |

Rules while mapping:
- **Don't copy `slippage` into `maxFee`.** `maxFee` caps the user's total USD
  value loss (fees, app fee and slippage combined). Omitting it, like
  `slippage=auto`, uses the default, which can allow up to 50%; set an
  explicit bound the user agrees to.
- **Convert `appFee` units.** `0.0025` becomes `appFeeBps: 25`. App fees need
  activation by Across.
- **The body is strict.** Drop every legacy field; an unknown field returns
  `400 request.unsupported_param`.
- **GET query → POST JSON body.** `/swap/approval` took query parameters;
  `/v1/quotes` takes a JSON body with nested `origin` / `destination`.

## Step 4: Map response handling

| Legacy | Across API |
|---|---|
| `expectedOutputAmount`, `minOutputAmount` | `destination.amount`, `destination.minAmount` |
| `fees` | `fee`: `totalFee`, `transferFee`, `appFee` (plus `maxTotalFee`, `maxTransferFee`) |
| `approvalTxns`, `swapTx` | `nextActions`: an ordered list of what to approve, sign or send |
| Deposit tx hash as the transfer id | `orderId` from the quote |
| `status`: `pending`, `filled`, `expired`, `refunded`, ... | `status.state`: `pending`, `completed`, `refunded`, `failed`, with `status.reason` for detail |

Rewrite the execution code as a loop over `nextActions` that dispatches on
`kind` (see [quote-and-execute](../quote-and-execute/SKILL.md)). Do not keep
code that sends `approvalTxns` and `swapTx`.

## Step 5: Migrate tracking and storage

- **Store `orderId`** for every transfer (in the database row, job payload or
  UI state) where the code used to store the deposit tx hash or `depositId`.
- Replace `/deposit/status` polling with `GET /v1/transfers/{orderId}`; stop
  polling when `status.isTerminal` is `true`.
- Replace `status === "filled"` style checks with `status.state` checks, and
  show `status.description` to users.
- Replace `/deposits` reconciliation with `GET /v1/transfers` and its filters
  (`externalId`, `txnRef`, `depositor`, `recipient`, chain and token filters).
  Set `externalId` on quotes to match transfers to your own records.

## Before and after

Legacy (remove):

```ts
const q = await fetch(
  `https://app.across.to/api/swap/approval?tradeType=exactInput&amount=1000000` +
    `&inputToken=${USDC_ARB}&originChainId=42161&outputToken=${USDC_BASE}` +
    `&destinationChainId=8453&depositor=${user}&integratorId=0xdead&slippage=auto`,
).then((r) => r.json());
for (const tx of q.approvalTxns ?? []) await wallet.sendTransaction(tx);
const hash = await wallet.sendTransaction(q.swapTx);
// later: GET /deposit/status?depositTxnRef=${hash}
```

Across API:

```ts
const quote = await fetch("https://api.across.to/v1/quotes", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.ACROSS_API_KEY}`, // from dashboard.across.to
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    origin: { token: { chainId: 42161, address: USDC_ARB }, amount: "1000000", depositor: user },
    destination: { token: { chainId: 8453, address: USDC_BASE }, recipient: user },
    amountType: "exact_input",
    maxFee: { bps: 100 },
  }),
}).then((r) => r.json());

// Complete quote.nextActions in order; POST /v1/quotes/{orderId}/executions
// only if they end with submit_execution (full loop in quote-and-execute).
// Save quote.orderId, then poll GET /v1/transfers/{orderId} until status.isTerminal.
```

## Step 6: Verify

- [ ] The Step 1 search returns no hits.
- [ ] Every request goes to `https://api.across.to/v1/...` with
      `Authorization: Bearer <key>`, from the server.
- [ ] `integratorId`, `tradeType`, `slippage` and `appFeeRecipient` are gone.
- [ ] Token addresses and decimals come from `GET /v1/capabilities`.
- [ ] Execution loops over `nextActions` and calls the executions endpoint only
      when the list ends with `submit_execution`.
- [ ] Transfers are stored and tracked by `orderId`.
- [ ] One small mainnet transfer (a few dollars) completes end to end. There
      is no testnet.

## Notes
- No testnet: test on mainnet with small amounts.
- Never fall back to a legacy endpoint during or after the migration, even for
  a feature that seems missing. If something has no Across API equivalent, say
  so and ask the user.
- Legacy behaviour is documented at https://legacy.docs.across.to, for reading
  existing code only. Do not build anything new from it.
- Public guide: https://docs.across.to/introduction/migration-guide
