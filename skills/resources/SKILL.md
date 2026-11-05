# Resources and Links

## Across API
- Base URL: `https://api.across.to` (every path under `/v1`)
- Integrator portal (API keys, webhooks, app fees): https://integrator.portal.across.to
- Testnet: none. Test on mainnet with small amounts.

| Endpoint | Purpose |
|---|---|
| `GET /v1/capabilities` | Chains, tokens, routes, funding options, limits |
| `POST /v1/quotes` | Price a transfer and get its `nextActions` |
| `POST /v1/quotes/{orderId}/executions` | Submit signatures or transaction refs when the quote asks |
| `GET /v1/transfers/{orderId}` | Track one transfer |
| `GET /v1/transfers` | List and reconcile transfers |

## Documentation
- Docs home: https://docs.across.to
- Get an API key: https://docs.across.to/introduction/api-keys
- Quickstart: https://docs.across.to/introduction/quickstart
- Migration guide (legacy → Across API): https://docs.across.to/introduction/migration-guide
- Fees and app fees: https://docs.across.to/introduction/fees
- Refunds: https://docs.across.to/introduction/refunds
- Tracking: https://docs.across.to/introduction/tracking-deposits
- Webhooks: https://docs.across.to/introduction/webhooks
- API reference: https://docs.across.to/api-reference (use only the `/v1` operations; everything under "Deprecated" is legacy)
- Machine-readable docs: https://docs.across.to/llms.txt

These pages are being rewritten for the Across API, and some (fees, refunds,
tracking, API keys) still show the legacy version until that release ships;
the quickstart, migration guide and webhooks pages 404 until then. If a page
tells you to call `/suggested-fees`, `/swap/*`, `/deposit/status` or any other
legacy endpoint, it is the old version: ignore it and follow these skills.

## Do not use
Anything for the legacy APIs: `app.across.to/api`, `testnet.across.to`, the
Swap API, `/suggested-fees`, `/deposit/status`, `@across-protocol/app-sdk`,
`sdk-v2`, the integrator ID registration form, and ERC-7683
`AcrossOriginSettler` contracts. See
[SKILL.md](../SKILL.md#rule-zero-never-use-the-legacy-apis).

## Community and support
- Discord: https://discord.across.to
- Other channels: https://docs.across.to/resources/support-links

## Security
- Audits: https://docs.across.to/introduction/audits
- Bug bounty: https://docs.across.to/introduction/bug-bounty
- Report bugs: bugs@across.to
