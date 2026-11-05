# Security Checklist

Use this checklist when building or reviewing an Across API integration.

## Legacy APIs
- [ ] No calls to legacy hosts (`app.across.to/api`, `across.to/api`,
      `testnet.across.to/api`) or endpoints (`/swap/*`, `/suggested-fees`,
      `/limits`, `/available-routes`, `/deposit/status`, `/deposit`,
      `/deposits`, `/gasless/submit`).
- [ ] No `@across-protocol/app-sdk` and no hand-built SpokePool or
      `AcrossOriginSettler` calls.

## API keys
- [ ] The key is used only server-side. Browser, mobile and agent-facing
      clients call your backend, never `api.across.to` directly with the key.
- [ ] The key lives in a secrets manager or environment variable, never in
      source control, logs or error messages.
- [ ] Separate keys per service or environment, so one can be rotated or
      revoked in the integrator portal without touching the others.
- [ ] Handle `401` (missing, wrong, revoked or expired key) and `429` (rate
      limit) explicitly.

## Quote handling
- [ ] Quotes are never cached or reused; a new quote is requested after
      `expiresAt`.
- [ ] `maxFee` is set explicitly. The default tolerates up to 50% value loss.
- [ ] `slippageBps` is not used.
- [ ] Users see `destination.minAmount` as the guaranteed amount and
      `destination.amount` as the estimate.
- [ ] Token addresses and decimals come from `GET /v1/capabilities`, and
      amounts are base-unit strings.
- [ ] If `attestation` is requested, its `signature` is recovered over
      `typedData` and compared with the Across signer address pinned out of
      band (not the `signer` field in the response), and only attested values
      are acted on.

## Approvals and signing
- [ ] `authorizationPolicy.approvalAmount: "exact"` unless the user explicitly
      opts in to unlimited approval (the default).
- [ ] `capabilities.operations` declares only what the client can do, and the
      whole `nextActions` list is checked before the first wallet prompt.
- [ ] Before each wallet prompt: `chainId` is the expected chain,
      `approve_token.tokenAddress` is the quoted origin token, and with exact
      approvals `approve_token.amount` does not exceed `origin.maxAmount`.
- [ ] `typedData` is signed verbatim, never edited.
- [ ] `signerRequirement: "eoa"` actions are not sent to smart-contract
      wallets.
- [ ] `send_transaction` actions are sent from `origin.depositor`. A deposit
      sent from another wallet funds an order whose refund pays the quoted
      depositor.
- [ ] Solana store transactions are broadcast once; a second copy deposits
      again.

## Execution
- [ ] `POST /v1/quotes/{orderId}/executions` is called only when `nextActions`
      ends with `submit_execution`, and before `expiresAt`.
- [ ] `409 conflict.submission_state_unknown` is retried with the same request,
      never by re-quoting (avoids double-funding).
- [ ] An expired quote is re-quoted, not re-executed.

## Refunds
- [ ] `refundPolicy` is shown before the user signs. `supported: false`
      (for example `irreversible_rail`) means no automatic refund on that
      route.
- [ ] `destination.refundAddress` is set when the recipient is a contract
      that cannot handle the bridged token.

## Destination actions
- [ ] Every `target` and `spender` is validated; end users cannot inject
      arbitrary targets or calldata.
- [ ] Steps send the output to the intended owner explicitly.
- [ ] `onFailure` is chosen deliberately (`revert` refunds; `deliver-output-token`
      pays `onFailure.recipient`).

## Tracking and webhooks
- [ ] Irreversible actions (crediting users, releasing goods, retrying
      funds) wait for `status.finalized: true`, not just `isTerminal`.
- [ ] No transfer is marked failed because `expectedFillTimeMs` or
      `originValidUntil` passed.
- [ ] Webhook signatures are verified on the raw body with a Standard
      Webhooks library, deliveries are deduplicated on `webhook-id`, and
      out-of-order delivery is handled.
- [ ] The `whsec_...` secret is stored like an API key.

## Reporting vulnerabilities
- Bug bounty: https://docs.across.to/introduction/bug-bounty
- Audits: https://docs.across.to/introduction/audits
- Report bugs: bugs@across.to

## Risk Notes template

Include this in any change touching signing, fees, approvals, refunds,
destination actions or API keys:

```
## Risk Notes
- **API key**: [where it is stored, which service uses it, never client-side]
- **Signing**: [which nextActions are signed or sent, by which wallet]
- **Fees**: [maxFee setting, appFeeBps if any]
- **Approvals**: [token, amount (exact/infinite), spender from the quote]
- **Refunds**: [refundPolicy for the route, refund recipient]
- **Destination actions**: [targets, calldata, onFailure policy]
- **Tracking**: [what waits for status.finalized]
```
