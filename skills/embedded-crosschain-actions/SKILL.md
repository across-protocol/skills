# Destination Actions

Run contract calls on the destination chain with the bridged funds (for
example supply USDC to Aave on Base, or wrap USDC.e into pUSD on Polygon) by
adding `destinationActions` to `POST /v1/quotes`. Everything else about the
quote, its execution and tracking is unchanged
([quote-and-execute](../quote-and-execute/SKILL.md)).

**EVM to EVM only.** Destination actions work on transfers from an EVM chain
to an EVM chain. Destination actions involving Solana are not available yet.

To simply deliver to another address, you don't need actions: set
`destination.recipient`.

## Example: bridge USDC from Arbitrum, supply it to Aave on Base

```json
{
  "origin": {
    "token": { "chainId": 42161, "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831" },
    "amount": "5000000",
    "depositor": "0xYourWalletAddress"
  },
  "destination": {
    "token": { "chainId": 8453, "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" },
    "recipient": "0xYourWalletAddress"
  },
  "amountType": "exact_input",
  "maxFee": { "bps": 100 },
  "destinationActions": {
    "steps": [
      {
        "kind": "approve",
        "token": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "spender": "0xA238Dd80C259a72e81d7e4664a9801593F98d1c5",
        "populateDynamically": true
      },
      {
        "kind": "function-call",
        "target": "0xA238Dd80C259a72e81d7e4664a9801593F98d1c5",
        "functionSignature": "supply(address,uint256,address,uint16)",
        "args": [
          { "value": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" },
          { "populateDynamically": true, "balanceSourceToken": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" },
          { "value": "0xYourWalletAddress" },
          { "value": 0 }
        ]
      }
    ],
    "onFailure": { "action": "revert" }
  }
}
```

The aTokens are minted to `0xYourWalletAddress` because it is the `onBehalfOf`
argument of `supply`. `maxFee` caps the user's value loss at 1%; set it on
every quote.

## Steps

Steps run in order, in the same transaction as the fill, against the output
after any app fee.

| `kind` | Fields | What it does |
|---|---|---|
| `approve` | `token`, `spender`, and `amount` or `populateDynamically: true` | Lets `spender` (usually the next step's `target`) pull `token` |
| `function-call` | `target`, `functionSignature`, `args[]`, optional `value` or `populateCallValueDynamically: true` | Calls `target`; one `args` entry per input of `functionSignature`, in order |
| `raw-call` | `target`, `calldata` (selector included), optional `value` | Calls `target` with calldata you encoded |
| `native-transfer` | `target`, and `value` or `populateCallValueDynamically: true` | Sends the chain's native token to `target` |

`args[]` entries are `{ "value": ... }` for a fixed value, or
`{ "populateDynamically": true, "balanceSourceToken": "0xToken" }` to use the
amount of that token that arrived. `value` and `amount` are wei/base-unit
strings.

## onFailure

| `action` | If a step fails |
|---|---|
| `revert` (default) | The fill doesn't happen and the transfer is refunded |
| `deliver-output-token` | The fill still happens and the bridged token goes to `onFailure.recipient` (an EVM address on the destination chain). Anything left over after successful steps goes there too |

With `attestation` or `gasTopUp`, `onFailure.recipient` must equal
`destination.recipient`. A gas top-up is delivered before the steps run.

## Tracking

A completed transfer with destination actions reports `status.reason`
`filled_destination_action_successful` or `filled_destination_action_failed`
on `GET /v1/transfers/{orderId}`.

## Rules
- **Your steps decide where the funds end up.** With steps present, the
  output is not sent to `destination.recipient`; the last step has to deliver
  the final asset. Pass the user's address explicitly (for example as
  `onBehalfOf` or the recipient argument).
- **Use `populateDynamically`** on the step that spends the funds, so it uses
  the full amount that arrived.
- **Validate every `target` and `spender`.** Never let end users supply
  arbitrary targets or calldata.
- **A failed action refunds the bridged token**, not the token the user
  wanted. Set `destination.refundAddress` if the recipient cannot receive it.
