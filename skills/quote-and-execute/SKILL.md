# Quote and Execute

`POST /v1/quotes` prices one transfer and returns everything needed to execute
it: the expected and guaranteed amounts, an itemised fee, `expiresAt`, and the
ordered `nextActions`. Complete `nextActions` in order; only when they end
with `submit_execution`, post the results to
`POST /v1/quotes/{orderId}/executions`.

A quote is an offer, not a commitment. Never use legacy endpoints for any of
this (see [SKILL.md](../SKILL.md#rule-zero-use-only-these-endpoints)).

## Base URL and auth
- `https://api.across.to`
- `Authorization: Bearer <API key>`, `Content-Type: application/json`
- The body is strict: an unrecognised field returns `400 request.unsupported_param`.

## POST /v1/quotes

### Request body

| Field | Required | Description |
|---|---|---|
| `origin.token` | yes | `{ chainId, address }` of the input token |
| `origin.depositor` | yes | Wallet that funds the transfer |
| `origin.amount` | `exact_input` | Input in base units (uint256 decimal string) |
| `destination.token` | yes | `{ chainId, address }` of the output token |
| `destination.recipient` | yes | Address that receives the output |
| `destination.amount` | `exact_output`, `min_output` | Output in base units |
| `destination.refundAddress` | no | Who receives a destination-chain refund (paid in the bridged token). Defaults to the recipient. EVM destinations only (HyperCore and Lighter included) |
| `amountType` | no | `exact_input` (default), `exact_output`, `min_output` |
| `capabilities.operations` | no | What your client can do: `{ "kind": "approve_token" \| "sign_typed_data" \| "send_transaction" \| "sign_transaction", "ecosystem": "evm" \| "solana" \| "tron" }` entries, plus `{ "kind": "submit_execution" }`. The server derives the funding methods from these. Omit only if the client can perform every operation on the origin chain's ecosystem |
| `authorizationPolicy.allowedTypes` | no | Which authorizations you permit: `approval`, `permit2`, `erc3009`, `hypercore_action`, `spl_delegate`. `permit2` also needs `approval` |
| `authorizationPolicy.approvalAmount` | no | `exact` (the funded cap) or `infinite` (default, `MAX_UINT256`) |
| `maxFee` | no | `{ bps, fixedUsd }`: the most USD value the user can lose between expected input and guaranteed output (fees, app fee, swap impact and slippage). `bps` defaults to 5000 (50%) without an integrator override; `fixedUsd` defaults to `0`, max `99.99`. Over the limit: `400 policy.amount_out_of_range` |
| `appFeeBps` | no | Your fee in basis points of the input value, taken in the destination token (`10` on 1,000 USDC = 1 USDC). Needs activation by Across; above your entitlement returns `400 request.invalid_field`. Not on same-chain swaps |
| `gasTopUp.amount` | no | Destination native gas (wei) delivered to the recipient. Check `capabilities.gasTopUp` for support and the maximum |
| `destinationActions` | no | Contract calls on the destination chain. See [embedded-crosschain-actions](../embedded-crosschain-actions/SKILL.md) |
| `externalId` | no | Your reconciliation id (1 to 256 chars), echoed on the transfer |
| `metadata` | no | Up to 16 string key/value pairs |
| `attestation.verifier` | no | Opt in to an Across-signed EIP-712 attestation over the quote's terms |
| `origin.minInputAmount` | no | Minimum-input quote for integrators that run their own origin action first. Needs the `min-input-quote` permission (`403 permission.not_entitled` otherwise) |
| `origin.feePayer` | no | Solana origins only: account that pays the fee and deposit rent. Defaults to the depositor |

`slippageBps` is a legacy field kept for compatibility. Do not use it; use
`maxFee` (the two cannot be combined).

### Response (key fields)

| Field | Meaning |
|---|---|
| `orderId` | Path id for the executions call and the id to track. Save it |
| `expiresAt` | Quote expiry (RFC 3339, UTC). Quotes expire about 15 minutes after issue |
| `originValidUntil` | Last instant the origin deposit may land. On crosschain routes the destination can still execute after it, so it is not a failure deadline |
| `expectedFillTimeMs` | Indicative, not an SLA, and not proof of failure when it passes |
| `origin.amount`, `origin.maxAmount` | Expected spend; `maxAmount` is what the funding authorization covers |
| `destination.amount` | Expected delivery |
| `destination.minAmount` | Guaranteed floor. Show this as the minimum the user receives |
| `fee` | `transferFee`, `appFee`, `totalFee` (expected) and `maxTransferFee`, `maxTotalFee` (worst case), in the destination token with `amount` and `amountUsd`. A missing `amount` is never a zero |
| `gasEstimates[]` | `paid_by_wallet` (gas the user's wallet pays on top) vs `included_in_transfer_fee`. `[]` does not mean free |
| `refundPolicy` | `supported: true` with refund `token`, `refundAddress`, `expectedRefundTimeMs`, or `supported: false` with a `reason` (`irreversible_rail`, ...). Show it before the user signs |
| `refundTo[]` | Every token/chain/address a refund could arrive in (advisory) |
| `nextActions[]` | The ordered steps to complete |
| `gasTopUp`, `attestation` | Present only when requested |

The auction sets the final price: delivery can land above
`destination.amount`, never below `destination.minAmount`.

## nextActions

Check the whole list before the first wallet prompt, so you never start a plan
the client cannot finish. Then work through it in order, dispatching on `kind`:

| `kind` | `id` | What to do |
|---|---|---|
| `approve_token` | `approval`, `approval-reset` | ERC-20 `approve(spender, amount)` on `tokenAddress`, on `chainId`. `approval-reset` is the reset to zero that USDT-class tokens need first. Wait for the receipt |
| `sign_typed_data` | `funding`, `hypercore-transfer` | Sign `typedData` verbatim with `eth_signTypedData_v4`. Keep the signature keyed by the action's `id` (read it off the action, don't hardcode it). `signerRequirement: "eoa"` means a contract-account signature is rejected |
| `send_transaction` (EVM) | `prefunded-store`, `swap-execute` | Broadcast `{ to, data, value }` on `chainId`. `prefunded-store` deposits an approval-funded transfer; `swap-execute` is a same-chain swap that completes when it lands |
| `send_transaction` (`ecosystem: "solana"`) | `spl-delegate-approve` | Wallet-paid Solana transaction built from `instructions`, with `feePayer`. Sign and broadcast it; keep the signature |
| `sign_transaction` | `spl-delegate-approve` | Sponsored Solana flow (`sponsoredBy: "across"`): add the depositor's signature to `transaction` (base64) without modifying it, and do not broadcast it. Re-quote after `lastValidBlockHeight` |
| `submit_execution` | `submit-execution` | Always last. `inputs` lists the ids of earlier actions whose results to submit. Call the executions endpoint |

No `submit_execution` means nothing to POST. An approval-funded quote ends
with a `prefunded-store` transaction; once it lands, Across picks up the
deposit on chain and the transfer starts. A same-chain `swap-execute` is not
submitted either. Track `orderId` in both cases.

## POST /v1/quotes/{orderId}/executions

Call it only when `nextActions` ends with `submit_execution`, and before
`expiresAt`. Send one `authorization` arm:

| Arm | When |
|---|---|
| `{ "signatures": { "<action id>": "0x..." } }` | The quote asked for signatures. One entry for exactly each `sign_typed_data` action, keyed by its `id`. An empty map is correct when `submit_execution` has empty `inputs` |
| `{ "txnRef": "..." }` | A transaction reference: the Solana signature for an `spl_delegate` quote (base58 or 0x-hex). Optional for a `prefunded-store` quote, which starts on its own |
| `{ "signedTransaction": "<base64>" }` | Sponsored Solana quotes (`sponsoredBy: "across"`) |

Response: `{ orderId, acceptedAt }`. A 2xx means accepted, not arrived; track
it with `GET /v1/transfers/{orderId}`.

- Quote expired: request a new quote; do not retry the execution.
- `400 execution.invalid_signature`: the message names the action ids the
  quote expects.
- `400 SPONSORED_TRANSACTION_EXPIRED` (sponsored Solana): re-quote.
- `409 conflict.submission_state_unknown`: retry the same request with
  backoff; never re-quote.

## End-to-end example: 1 USDC Arbitrum → Base (viem)

```ts
import { createWalletClient, createPublicClient, http, erc20Abi } from "viem";
import { arbitrum } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const BASE_URL = "https://api.across.to";
const API_KEY = process.env.ACROSS_API_KEY!; // server-side only
const USDC_ARBITRUM = "0xaf88d065e77c8cC2239327C5EDb3A432268e5831";
const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
const wallet = createWalletClient({ account, chain: arbitrum, transport: http() });
const client = createPublicClient({ chain: arbitrum, transport: http() });

async function across(path: string, init: RequestInit = {}) {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });
  const body = await res.json();
  if (!res.ok) {
    const code = body?.error?.code ?? body?.code;
    throw new Error(`${res.status} ${code}: ${body?.error?.message ?? body?.message}`);
  }
  return body;
}

// 1. Price
const quote = await across("/v1/quotes", {
  method: "POST",
  body: JSON.stringify({
    origin: {
      token: { chainId: 42161, address: USDC_ARBITRUM },
      amount: "1000000", // 1 USDC, 6 decimals
      depositor: account.address,
    },
    destination: {
      token: { chainId: 8453, address: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" }, // USDC
      recipient: account.address,
    },
    amountType: "exact_input",
    maxFee: { bps: 100 }, // at most 1% value loss
    capabilities: {
      operations: [
        { kind: "approve_token", ecosystem: "evm" },
        { kind: "sign_typed_data", ecosystem: "evm" },
        { kind: "send_transaction", ecosystem: "evm" },
        { kind: "submit_execution" },
      ],
    },
    authorizationPolicy: { approvalAmount: "exact" },
  }),
});

// 2. Check every action before the first wallet prompt: a kind this client supports,
//    on the wallet's chain, approving only the quoted token up to origin.maxAmount
const supported = new Set(["approve_token", "sign_typed_data", "send_transaction", "submit_execution"]);
for (const a of quote.nextActions) {
  if (!supported.has(a.kind) || (a.ecosystem && a.ecosystem !== "evm")) {
    throw new Error(`unsupported action ${a.kind}/${a.ecosystem}`);
  }
  if (a.kind !== "submit_execution" && a.chainId !== arbitrum.id) {
    throw new Error(`action ${a.id} targets chain ${a.chainId}, wallet is on ${arbitrum.id}`);
  }
  if (
    a.kind === "approve_token" &&
    (a.tokenAddress.toLowerCase() !== USDC_ARBITRUM.toLowerCase() ||
      BigInt(a.amount) > BigInt(quote.origin.maxAmount))
  ) {
    throw new Error(`approval ${a.id} does not match the quoted token and amount`);
  }
}

// 3. Complete nextActions in order
const signatures: Record<string, `0x${string}`> = {};
let submit = false;
for (const a of quote.nextActions) {
  if (a.kind === "approve_token") {
    const hash = await wallet.writeContract({
      address: a.tokenAddress,
      abi: erc20Abi,
      functionName: "approve",
      args: [a.spender, BigInt(a.amount)],
    });
    await client.waitForTransactionReceipt({ hash });
  } else if (a.kind === "sign_typed_data") {
    signatures[a.id] = await wallet.signTypedData(a.typedData);
  } else if (a.kind === "send_transaction") {
    const hash = await wallet.sendTransaction({ to: a.to, data: a.data, value: BigInt(a.value) });
    await client.waitForTransactionReceipt({ hash });
  } else if (a.kind === "submit_execution") {
    submit = true;
  }
}

// 4. Submit only when the quote asks for it
if (submit) {
  await across(`/v1/quotes/${quote.orderId}/executions`, {
    method: "POST",
    body: JSON.stringify({ authorization: { signatures } }),
  });
}

// 5. Track quote.orderId (see tracking-transactions)
```

## Errors

Two envelopes. Read `error.code`, and fall back to the top-level `code`:

```json
{ "code": "unauthenticated", "message": "AUTH_INVALID", "details": null }
```
```json
{ "error": { "type": "request", "code": "request.unsupported_param", "message": "unsupported field: amountTyp", "param": "amountTyp", "requestId": "..." } }
```

| Situation | Result |
|---|---|
| Missing or invalid API key | `401`, flat `unauthenticated` |
| Unrecognised body or query field | `400 request.unsupported_param`, naming the field |
| Field not allowed for your key (for example `appFeeBps`) | `400 request.invalid_field` with `param` |
| Amount the route cannot clear, or fee above `maxFee` | `400 policy.amount_out_of_range` |
| Feature needs a permission | `403 permission.not_entitled` |
| Quote past its expiry | Re-quote; do not retry the execution |
| Too many requests | `429`, flat gateway shape |

Quote `error.requestId` in support requests.
