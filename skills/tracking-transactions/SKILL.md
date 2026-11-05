# Tracking Transfers

Track every transfer by the `orderId` from its quote, with
`GET /v1/transfers/{orderId}` or with webhooks. Never use the legacy
`/deposit/status`, `/deposit` or `/deposits`, and never track by deposit
transaction hash.

## GET /v1/transfers/{orderId}

```
GET https://api.across.to/v1/transfers/{orderId}
Authorization: Bearer <API key>
```

No query parameters are accepted. A `404 not_found.transfer` means the order
does not exist or belongs to another integrator; the two are indistinguishable.

### Status

| Field | Meaning |
|---|---|
| `status.state` | `pending`, `completed`, `refunded`, `failed`. Branch your logic on this |
| `status.reason` | Detail: `finding_relayer`, `awaiting_execution`, `filled`, `filled_destination_action_successful`, `filled_destination_action_failed`, `refund_confirmed`, `expired`, `origin_transfer_failed`, `submission_rejected` |
| `status.description` | Support-safe sentence you can show users as-is. Never branch on it |
| `status.isTerminal` | The outcome is known. **Can flip back to `false`** until `finalized` |
| `status.finalized` | The outcome can no longer change |

Show progress from `isTerminal`. Act on the outcome in any way you cannot undo
(crediting a user, releasing goods, re-sending funds) only when `finalized` is
`true`, typically minutes after the fill.

### Other fields

- `origin`: chain, input token and amount, `depositor`
- `destination`: chain, output token and amount, `recipient` (absent until
  delivery is observed)
- `refund`: refund chain, token, `minAmount` (the guaranteed floor, not the
  received amount) and recipient; any of these can be `null`
- `transactions[]`: `deposit`, `fill`, `refund` (and `intermediate`) entries
  with `txnRef`, `chainId`, `blockTimestamp`, `explorerUrl`
- `submittedAt`, `filledAt`, `fillTimeMs`
- `fundingMethod`: `approval`, `permit2`, `erc3009`, `spl_delegate`,
  `hypercore_action`, ...
- `quoted`: the terms the quote promised
- `appFee`: `feeUsd`, `integratorEarningUsd`; `externalId` if you set one

## Polling pattern

```ts
const BASE_URL = "https://api.across.to";
const API_KEY = process.env.ACROSS_API_KEY!;

async function track(orderId: string) {
  while (true) {
    const res = await fetch(`${BASE_URL}/v1/transfers/${orderId}`, {
      headers: { Authorization: `Bearer ${API_KEY}` },
    });
    if (res.ok) {
      const transfer = await res.json();
      const { isTerminal, finalized } = transfer.status;
      if (isTerminal && finalized) return transfer;
    }
    await new Promise((r) => setTimeout(r, 5_000));
  }
}
```

- Poll about every 5 seconds.
- Treat a `404` right after execution as not yet visible and keep polling for
  a bounded time. A `404` that persists means a wrong `orderId` or another
  integrator's key.
- **Do not mark a transfer failed on a timer.** Neither `expectedFillTimeMs` nor
  `originValidUntil` means failure on a crosschain route; late deliveries still
  complete. Only `status` decides.
- A transfer whose deposit landed stays `pending` until it completes or is
  refunded; it does not become `expired`. If it is still `pending` well past
  `originValidUntil` plus `refundPolicy.expectedRefundTimeMs`, contact Across
  with the `orderId`.

## Refund outcomes

- `refunded` + `refund_confirmed`: the refund executed. Read `refund` and the
  `transactions[]` entry with `type: "refund"`.
- `failed` + `expired` or `origin_transfer_failed`: the funds never moved.
- A destination refund pays the **bridged** token (for example USDC on the
  destination when the user wanted WETH), to `destination.refundAddress` or
  the recipient.
- Refunds are automatic: no signature, no gas, no fee for the user.

## GET /v1/transfers (list)

Newest first, keyset-paginated. Filters are repeatable (values OR within one
filter, AND across filters), and unknown params return `400`.

| Parameter | Description |
|---|---|
| `limit` | 1 to 100, default 50 |
| `cursor` | `nextCursor` from the previous page |
| `fromSubmittedAt`, `toSubmittedAt` | RFC 3339 range on `submittedAt` |
| `orderId`, `externalId`, `txnRef` | Reconciliation lookups. `txnRef` matches any transaction in the lifecycle |
| `statusState`, `statusReason` | Status filters |
| `depositor`, `recipient`, `refundAddress`, `depositAddress` | Address filters |
| `originChainId`, `destinationChainId`, `originTokenAddress`, `destinationTokenAddress` | Route filters |

Results are scoped to your API key's integrator.

## Webhooks (push instead of poll)

Set up in the integrator portal (https://integrator.portal.across.to) under
**Webhooks**: an HTTPS endpoint that answers `2xx` within 10 seconds, and the
events to receive. The `whsec_...` signing secret is shown once.

| Event | `data` | Sent when |
|---|---|---|
| `transfer.status` | `orderId`, `externalId`, `status` | Any `status` field changes |
| `transfer.data` | The full transfer, as from `GET /v1/transfers/{orderId}` | Any transfer field changes |

Body shape: `{ "version": 1, "type": "transfer.status", "payload": { "id", "timestamp", "data" } }`.
Headers: `webhook-id`, `webhook-timestamp`, `webhook-signature`.

Handling rules:
- **Verify** every delivery with a [Standard Webhooks](https://www.standardwebhooks.com)
  library, against the raw body, before parsing.
- **Acknowledge first** with a `2xx`, then process. Failures are retried with
  backoff.
- **Deduplicate on `webhook-id`**; delivery is at least once.
- **Don't rely on order**; read `status` from each event.
- **Wait for `finalized: true`** before irreversible actions.
- No test events are sent, and changes made while a webhook is paused are not
  replayed.

Do not use the legacy `POST /v1/webhooks/subscriptions` on the old host; it
belongs to the deprecated deposit-address API.
