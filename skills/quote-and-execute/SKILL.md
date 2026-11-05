# Quote and Execute

`POST /v1/quotes` prices one transfer (bridge, crosschain swap or same-chain
swap) and returns everything needed to execute it. You then complete the
quote's `nextActions` in order and, when they end with `submit_execution`,
post the results to `POST /v1/quotes/{orderId}/executions`.

Never use the legacy `/swap/approval`, `/suggested-fees` or `/swap/gasless`
for this. See [SKILL.md](../SKILL.md#rule-zero-never-use-the-legacy-apis).

## Base URL and auth
- `https://api.across.to`
- `Authorization: Bearer <API key>` and `Content-Type: application/json`
- Bodies are strict: an unknown field returns `400 request.unsupported_param`.

## POST /v1/quotes

### Request body

| Field | Required | Description |
|---|---|---|
| `origin.token` | yes | `{ chainId, address }` of the input token. Native: `0x0000000000000000000000000000000000000000` (EVM), `11111111111111111111111111111111` (Solana) |
| `origin.depositor` | yes | Wallet that funds the transfer. It must also be the wallet that sends any `send_transaction` action |
| `origin.amount` | for `exact_input` | Input in base units, decimal string |
| `destination.token` | yes | `{ chainId, address }` of the output token |
| `destination.recipient` | yes | Address receiving the output |
| `destination.amount` | for `exact_output` / `min_output` | Output in base units |
| `destination.refundAddress` | no | Who receives a destination-chain refund (the bridged token). Defaults to the recipient. EVM destinations only |
| `amountType` | no | `exact_input` (default), `exact_output`, `min_output` |
| `capabilities.operations` | no | What the client can do: `[{ "kind": "approve_token" \| "sign_typed_data" \| "send_transaction" \| "sign_transaction", "ecosystem": "evm" \| "solana" \| "tron" }, { "kind": "submit_execution" }]`. The server only offers funding methods the client can complete |
| `authorizationPolicy.allowedTypes` | no | Subset of `approval`, `permit2`, `erc3009`, `hypercore_action`, `spl_delegate` |
| `authorizationPolicy.approvalAmount` | no | `exact` or `infinite` (default). Prefer `exact` |
| `maxFee` | no | `{ bps, fixedUsd }`: the most USD value the user can lose to fees, app fee and slippage combined. Default allows 50% of input. Not a slippage value |
| `appFeeBps` | no | Your fee in basis points of the input value (`100` = 1%). Needs activation by Across; paid out to you monthly |
| `gasTopUp.amount` | no | Destination native-token amount (base units) to deliver alongside the transfer. See `capabilities.gasTopUp` for the maximum |
| `destinationActions` | no | Contract calls on the destination chain. See [embedded-crosschain-actions](../embedded-crosschain-actions/SKILL.md) |
| `externalId` | no | Your reconciliation id (1 to 256 chars). Filterable on `GET /v1/transfers` |
| `metadata` | no | Up to 16 string key/values, surfaced on the transfer |
| `attestation.verifier` | no | Opt in to an Across-signed EIP-712 attestation over the quote terms |
| `origin.minInputAmount` | no | Minimum-input quote for contracts that pre-swap on origin. Needs the `min-input-quote` permission |

Do not send `slippageBps`; it is a legacy compatibility field and cannot be
combined with `maxFee`.

### Response (key fields)

| Field | Meaning |
|---|---|
| `orderId` | The id to execute against and to track. Save it |
| `expiresAt` | Quote expiry (RFC 3339 UTC). Execute before it; re-quote after it |
| `originValidUntil` | Last instant the origin deposit may land on chain. Not a failure deadline for crosschain transfers |
| `expectedFillTimeMs` | Estimate only, not an SLA. Never mark a transfer failed when it elapses |
| `origin.amount`, `origin.maxAmount` | Expected and maximum spend |
| `destination.amount` | Expected delivery |
| `destination.minAmount` | Guaranteed minimum delivery. Show this as the floor |
| `fee.transferFee`, `fee.appFee`, `fee.totalFee`, `fee.maxTransferFee`, `fee.maxTotalFee` | Fee breakdown in the destination token (`amount` in base units, `amountUsd`). A missing `amount` means "not stateable", not zero |
| `gasEstimates[]` | Gas the wallet pays itself (`effect: "paid_by_wallet"`) vs gas already in the fee (`included_in_transfer_fee`). Empty does not mean free |
| `refundPolicy` | `supported: true` with refund token/address, or `supported: false` with a `reason` (for example `irreversible_rail`). Show it before the user signs |
| `refundTo[]` | Every token/chain/address a refund could arrive in (advisory) |
| `nextActions[]` | Ordered steps to complete |
| `attestation` | Present only if requested |

The auction sets the final price: delivery can land above
`destination.amount` and never below `destination.minAmount`.

## nextActions

Check the whole list before the first wallet prompt, so you never start a plan
the client cannot finish. Then complete it in order, dispatching on `kind`:

| `kind` | `id` values | What to do |
|---|---|---|
| `approve_token` | `approval`, `approval-reset` | ERC-20 `approve(spender, amount)` on `tokenAddress` on `chainId`. `approval-reset` sets the allowance to 0 first (tokens like USDT). Wait for the receipt |
| `sign_typed_data` | `funding`, `hypercore-transfer` | Sign `typedData` verbatim with `eth_signTypedData_v4` on `chainId`. Keep the signature keyed by `id`. `signerRequirement: "eoa"` means a smart-contract wallet cannot sign it |
| `send_transaction` (EVM / Tron) | `prefunded-store`, `swap-execute` | Send `{ to, data, value }` on `chainId` from `origin.depositor`. Wait for the receipt and keep the hash |
| `send_transaction` (Solana, `ecosystem: "solana"`) | `spl-delegate-approve`, `svm-prefunded-store` | Build a transaction from `instructions` with your own recent blockhash, `feePayer` as fee payer, sign and broadcast it **once** (a second copy deposits again). Keep the signature |
| `sign_transaction` (Solana) | `svm-prefunded-store` | Sponsored flow (`sponsoredBy: "across"`): add the depositor's signature to `transaction` (base64) without modifying it. Do not broadcast. Re-quote past `lastValidBlockHeight` |
| `submit_execution` | `submit-execution` | Always last. `inputs` lists the ids of earlier actions whose results to submit. Call the executions endpoint |

If `nextActions` has no `submit_execution`, there is nothing to POST. For
example, an approval-funded quote ends with a `prefunded-store` (or
`svm-prefunded-store`) transaction; once it lands, Across picks the deposit up
on chain and the transfer starts. Track `orderId`. A POST for such a quote can
answer `400`; that is not a failed deposit, so do not re-quote or deposit again.

## POST /v1/quotes/{orderId}/executions

Send exactly one `authorization` arm:

| Arm | When |
|---|---|
| `{ "signatures": { "<action id>": "0x..." } }` | The quote has `sign_typed_data` actions. Include an entry for every one of them, keyed by its `id` |
| `{ "txnRef": "0x..." }` | `submit_execution.inputs` names a transaction action, for example the Solana `spl-delegate-approve` signature (base58 or 0x-hex) |
| `{ "signedTransaction": "<base64>" }` | Sponsored Solana quotes (`sponsoredBy: "across"`) |

Response: `{ orderId, acceptedAt }`. A 2xx means accepted, not delivered.
Track it with `GET /v1/transfers/{orderId}`.

Retry rules:
- `conflict.quote_expired` or past `expiresAt`: request a new quote. Do not
  retry the execution.
- `409 conflict.submission_state_unknown`: retry the same request with
  backoff. Do not re-quote.
- `execution.invalid_signature`, `execution.insufficient_balance`,
  `execution.insufficient_allowance`: fix the cause; re-quote if `expiresAt`
  has passed.

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
      amount: "1000000", // 1 USDC (6 decimals)
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

// 2. Check every action before prompting the wallet: a kind this client supports,
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
const txHashes: Record<string, `0x${string}`> = {};
let submit: { inputs: string[] } | undefined;
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
    txHashes[a.id] = hash;
  } else if (a.kind === "submit_execution") {
    submit = a;
  }
}

// 4. Submit only when asked
if (submit) {
  const txInput = submit.inputs.find((id) => txHashes[id]);
  const authorization = txInput ? { txnRef: txHashes[txInput] } : { signatures };
  await across(`/v1/quotes/${quote.orderId}/executions`, {
    method: "POST",
    body: JSON.stringify({ authorization }),
  });
}

// 5. Track quote.orderId (see tracking-transactions)
```

## Variants

- **Gasless**: a signature-funded quote (`permit2` or `erc3009`) has the user
  sign instead of sending a transaction; origin gas is in the fee. Permit2 may
  need a one-time approval first.
- **Approval-only clients**: declare `approve_token` + `send_transaction` and
  `allowedTypes: ["approval"]`. Expect `approve_token` (if needed), then a
  `prefunded-store` transaction, and no `submit_execution`.
- **Same-chain swap**: same call with origin and destination on one chain. Its
  `orderId` is a hash of the swap calldata; `originValidUntil` is a hard
  deadline here.
- **Gas top-up**: set `gasTopUp.amount`; part of the input buys destination
  native gas, so the token output is lower.

## Errors

Two envelopes. Read `error.code`, fall back to the top-level `code`:

```json
{ "code": "unauthenticated", "message": "AUTH_INVALID", "details": null }
```
```json
{ "error": { "type": "request", "code": "request.unsupported_param", "message": "unsupported field: amountTyp", "param": "amountTyp", "requestId": "..." } }
```

| Situation | Result |
|---|---|
| Missing, wrong, revoked or expired API key | `401` flat `unauthenticated` |
| Unknown field or param | `400 request.unsupported_param` |
| Field not allowed for your key (for example `appFeeBps` above your entitlement) | `400 request.invalid_field` with `param` |
| Amount too small/large, or fee above `maxFee` | `400 policy.amount_out_of_range` (`details.amountLimits` when a bound is stateable) |
| Feature needs a permission | `403 permission.not_entitled` |
| No route | `not_found.route` |
| Rate limited | `429` flat |

Quote `requestId` in support requests.
