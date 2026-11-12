# Security Checklist

Use this checklist when building or reviewing an Across API integration.

## Endpoints
- [ ] Only `https://api.across.to` with `/v1/capabilities`, `/v1/quotes`,
      `/v1/quotes/{orderId}/executions` and `/v1/transfers`. No legacy APIs
      (the search in [migration](../migration/SKILL.md) returns no hits).
- [ ] No hand-built Across contract calls; only the quote's `nextActions`.

## API keys
- [ ] The key was issued by Across, after a request through
      https://docs.across.to/introduction#get-your-api-key, before the
      integration or migration starts.
- [ ] The key is used only server-side. Browser and mobile clients call your
      backend, never the Across API with the key.
- [ ] The key is in a secrets manager, never in source control, logs or error
      messages.
- [ ] Separate keys per service or environment, so one can be rotated or
      revoked without touching the others.
- [ ] `401` (missing, wrong, revoked or expired key) and `429` are handled.

## Quotes
- [ ] Quotes are never cached; a new one is requested after `expiresAt`.
- [ ] `maxFee` is set explicitly (the default can allow up to 50% value loss).
- [ ] `slippageBps` is not used.
- [ ] Users see `destination.minAmount` as the guaranteed amount and
      `destination.amount` as the estimate.
- [ ] Token addresses and decimals come from `GET /v1/capabilities`; amounts
      are base-unit strings.
- [ ] With `attestation`: `signature` is recovered over `typedData` and
      compared with the Across signer address received during onboarding (not
      the response's `signer` field), and only values in `typedData.message`
      are relied on.

## Approvals and signing
- [ ] `authorizationPolicy.approvalAmount: "exact"` unless the user opts in to
      an unlimited approval (the default).
- [ ] `capabilities.operations` declares only what the client can do, and the
      whole `nextActions` list is checked before the first wallet prompt.
- [ ] Each action's `chainId` and `tokenAddress` match the quoted route before
      the wallet is prompted.
- [ ] `typedData` is signed verbatim.
- [ ] `signerRequirement: "eoa"` actions are not sent to contract accounts.

## Execution
- [ ] The executions endpoint is called only when `nextActions` ends with
      `submit_execution`, and before `expiresAt`.
- [ ] An expired quote is re-quoted, not re-executed.
- [ ] `409 conflict.submission_state_unknown` is retried with the same request,
      never by re-quoting.

## Refunds
- [ ] `refundPolicy` is shown before the user signs; `supported: false` (for
      example `irreversible_rail`) means no automatic refund on that route.
- [ ] `destination.refundAddress` is set when the recipient cannot handle the
      bridged token.

## Destination actions
- [ ] Every `target` and `spender` is validated; end users cannot inject
      targets or calldata.
- [ ] Steps send the output to the intended owner explicitly.
- [ ] `onFailure` is chosen deliberately.

## Tracking
- [ ] Tracking is by `orderId`, polling stops at `status.isTerminal`, and
      logic branches on `status.state`, never on `status.description`.
- [ ] No transfer is marked failed because `expectedFillTimeMs` or
      `originValidUntil` passed.

## Risk Notes template

Include in any change touching keys, signing, fees, approvals, refunds or
destination actions:

```
## Risk Notes
- **API key**: [where it is stored, which service uses it]
- **Signing**: [which nextActions are signed or sent, by which wallet]
- **Fees**: [maxFee, appFeeBps]
- **Approvals**: [token, exact or infinite, spender from the quote]
- **Refunds**: [refundPolicy for the route, refund recipient]
- **Destination actions**: [targets, calldata, onFailure]
- **Tracking**: [orderId storage, polling or webhooks, isTerminal handling]
```
