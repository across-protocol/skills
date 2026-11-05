# Destination Actions

Run contract calls on the destination chain with the bridged funds (deposit
into a lending pool, wrap, stake, send to another address) by adding
`destinationActions` to `POST /v1/quotes`. Never use the legacy
`POST /swap/approval` with an `actions` body.

For a plain "send to a different address", you do not need actions: set
`destination.recipient`.

## How it works
1. Add `destinationActions` to the quote request. Everything else is a normal
   quote.
2. Complete `nextActions` and execute as usual
   ([quote-and-execute](../quote-and-execute/SKILL.md)).
3. On the destination chain the steps run in order, in the same transaction as
   the fill, against the delivered output (after any app fee).

## Request shape

```json
{
  "origin": {
    "token": { "chainId": 42161, "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831" },
    "amount": "5000000",
    "depositor": "0xUser"
  },
  "destination": {
    "token": { "chainId": 8453, "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913" },
    "recipient": "0xUser"
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
          { "value": "0xUser" },
          { "value": 0 }
        ]
      }
    ],
    "onFailure": { "action": "revert" }
  }
}
```

This bridges 5 USDC from Arbitrum, approves the Aave v3 Pool on Base and calls
`supply` with the delivered balance. The aTokens go to `0xUser` because it is
the `onBehalfOf` argument. `maxFee` caps the user's total value loss at 1%;
set it explicitly on every quote, with a bound the user agreed to (omitting it
allows up to 50%).

## Step kinds

| `kind` | Fields | What it does |
|---|---|---|
| `approve` | `token`, `spender`, `amount` or `populateDynamically: true` | Lets `spender` pull `token` |
| `function-call` | `target`, `functionSignature`, `args[]`, optional `value`, `populateCallValueDynamically` | Calls `target`; arguments are ABI-encoded from the signature |
| `raw-call` | `target`, `calldata`, optional `value` | Calls `target` with calldata you encoded |
| `native-transfer` | `target`, `value` or `populateCallValueDynamically: true` | Sends the chain's native token to `target` |

`args[]` entries are either `{ "value": <static value> }` or
`{ "populateDynamically": true, "balanceSourceToken": "0xToken" }`, which uses
the balance that actually arrived. `value` and `amount` are wei/base-unit
decimal strings.

## onFailure

| `action` | Result when any step fails |
|---|---|
| `revert` (default) | The fill does not happen and the transfer is refunded (in the bridged token) |
| `deliver-output-token` | The bridged token goes to `onFailure.recipient` instead |

When the quote also uses `attestation` or `gasTopUp`, `onFailure.recipient`
must equal `destination.recipient`.

## Tracking

`GET /v1/transfers/{orderId}` reports `status.reason`
`filled_destination_action_successful` or `filled_destination_action_failed`.
See [tracking-transactions](../tracking-transactions/SKILL.md).

## Rules
- **Funds go wherever your steps send them**, not automatically to
  `destination.recipient`. Pass the user's address explicitly (as the
  `onBehalfOf`, recipient or `to` argument).
- **Use `populateDynamically`** for amounts; the delivered amount is only known
  at fill time. Static amounts risk reverting or leaving dust.
- **Validate every `target` and `spender`.** Never let end users supply
  arbitrary targets or calldata.
- **Order matters.** Steps run in array order; approve before the call that
  pulls.
- **Set `destination.refundAddress`** if the recipient cannot handle the
  bridged token that a failed action would refund.
