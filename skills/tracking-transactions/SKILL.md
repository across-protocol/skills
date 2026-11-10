# Tracking Transfers

Track every transfer by the `orderId` from its quote with
`GET /v1/transfers/{orderId}`, and list or reconcile transfers with
`GET /v1/transfers`. Do not use legacy status endpoints, and do not track by
deposit transaction hash.

## GET /v1/transfers/{orderId}

```
GET https://api.across-protocol.xyz/v1/transfers/{orderId}
Authorization: Bearer <API key>
```

Takes no query parameters (any returns `400 request.unsupported_param`). The
id is case-sensitive: use it exactly as the quote returned it.
`404 not_found.transfer` covers an unknown id, a malformed id and another
integrator's transfer alike. A missing or invalid key returns `401` with a
nested `auth.missing_credentials` or `auth.invalid_credentials`.

### Status

| Field | Meaning |
|---|---|
| `status.state` | `pending`, `completed`, `refunded`, `failed`. Branch your logic on this |
| `status.reason` | `finding_relayer`, `awaiting_execution` (pending); `filled`, `filled_destination_action_successful`, `filled_destination_action_failed` (completed); `refund_confirmed` (refunded); `expired`, `origin_transfer_failed`, `submission_rejected` (failed) |
| `status.description` | Support-safe sentence to show users as-is. Never branch on it |
| `status.isTerminal` | `true` when the state is `completed`, `refunded` or `failed`: the outcome is known. **Stop polling when it is `true`** |
| `status.finalized` | Whether the chains involved are final. Follows `isTerminal`, typically minutes later. Informational: you don't need to wait for it |

### Other fields

- `origin`: `chain`, `input` (token and amount), `depositor`
- `destination`: `chain`, `output`, `recipient`; omitted until delivery is
  observed
- `refund`: `chain`, `output` (token and `minAmount`, the quoted floor, not the
  amount received), `recipient`; any part can be `null`
- `transactions[]`: `deposit`, `fill`, `refund` entries, each with a
  `transaction` (`txnRef`, chain, explorer link). Key on (`type`, `txnRef`)
- `submittedAt`, `filledAt`, `fillTimeMs`
- `fundingMethod`, `quoted` (the promised terms), `externalId`
- `appFee` (`feeUsd`, `integratorEarningUsd`), `gasTopUp`

## Polling

```ts
const BASE_URL = "https://api.across-protocol.xyz";
const API_KEY = process.env.ACROSS_API_KEY!;

async function track(orderId: string, notFoundTimeoutMs = 120_000) {
  const start = Date.now();
  while (true) {
    const res = await fetch(`${BASE_URL}/v1/transfers/${orderId}`, {
      headers: { Authorization: `Bearer ${API_KEY}` },
    });
    if (res.ok) {
      const transfer = await res.json();
      if (transfer.status.isTerminal) return transfer;
    } else if (res.status === 404) {
      // Give up if the transfer stays unknown
      if (Date.now() - start > notFoundTimeoutMs) throw new Error(`transfer ${orderId} not found`);
    } else if (res.status !== 429 && res.status < 500) {
      // 400, 401 (missing, revoked or expired key), 403: retrying will not help
      const body = await res.json().catch(() => null);
      throw new Error(`${res.status} ${body?.error?.code ?? body?.code}`);
    }
    // pending, 429 or 5xx: wait and poll again, slower when throttled or erroring
    const delay = res.status === 429 || res.status >= 500 ? 30_000 : 5_000;
    await new Promise((r) => setTimeout(r, delay));
  }
}
```

- Poll about every 5 seconds. Back off on `429` and `5xx`; stop on any other
  `4xx` (for example a `401` from a missing or revoked key) instead of
  retrying forever.
- A `404` that persists means a wrong `orderId` (the lookup is
  case-sensitive) or another integrator's key.
- **Never mark a transfer failed on a timer.** Neither `expectedFillTimeMs`
  nor `originValidUntil` means failure on a crosschain route. Only `status`
  decides.

## Refunds

- Refunds are automatic: the user signs nothing, pays no gas, and Across adds
  no fee.
- `refunded` + `refund_confirmed`: the refund executed. Read `refund` and the
  `transactions[]` entry with `type: "refund"`.
- `failed` + `expired` or `origin_transfer_failed`: the funds never moved.
- A transfer whose deposit was made but has not completed or been refunded
  stays `pending`; it does not become `expired`. If it is still `pending` well
  past `originValidUntil` plus the quote's `refundPolicy.expectedRefundTimeMs`,
  contact Across with the `orderId`.
- A destination refund pays the bridged token (for example USDC on the
  destination when the user wanted WETH), to `destination.refundAddress` or
  the recipient.
- Until a refund executes, a late delivery can still complete the transfer.

## GET /v1/transfers

Your transfers, newest first. Repeatable filters match any of their values;
different filters combine. Unknown parameters return `400`.

| Parameter | Description |
|---|---|
| `limit` | 1 to 100, default 50 |
| `cursor` | `pagination.nextCursor` from the previous page |
| `fromSubmittedAt`, `toSubmittedAt` | Inclusive RFC 3339 bounds with a zone (`Z` or an offset) |
| `statusState`, `statusReason` | Status filters |
| `orderId`, `externalId` | Your ids (up to 100 each) |
| `txnRef` | Matches any transaction in the lifecycle; other hops come back as an `intermediate` ledger entry |
| `depositor`, `recipient` | Address filters |
| `originChainId`, `destinationChainId`, `originTokenAddress`, `destinationTokenAddress` | Route filters |
