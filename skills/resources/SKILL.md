# Resources and Links

## Across API
- API keys: **https://dashboard.across.to**. Get one before starting an
  integration or migration.
- Base URL: `https://api.across.to` (every path starts with `/v1`)
- Testnet: none. Test on mainnet with small amounts.

| Endpoint | Reference |
|---|---|
| `GET /v1/capabilities` | https://docs.across.to/api-reference/v1/capabilities/get |
| `POST /v1/quotes` | https://docs.across.to/api-reference/v1/quotes/post |
| `POST /v1/quotes/{orderId}/executions` | https://docs.across.to/api-reference/v1/quotes/orderid/executions/post |
| `GET /v1/transfers/{orderId}` | https://docs.across.to/api-reference/v1/transfers/orderid/get |
| `GET /v1/transfers` | https://docs.across.to/api-reference/v1/transfers/get |

## Guides
- Get an API key: https://docs.across.to/introduction/api-keys
- Quickstart: https://docs.across.to/introduction/quickstart
- Migration guide (legacy APIs → Across API): https://docs.across.to/introduction/migration-guide
- Destination actions: https://docs.across.to/introduction/quickstart/destination-actions
- Fees and app fees: https://docs.across.to/introduction/fees
- Refunds: https://docs.across.to/introduction/refunds
- Docs home: https://docs.across.to

## Machine-readable
- OpenAPI spec: https://docs.across.to/openapi.yaml
- Docs index for agents: https://docs.across.to/llms.txt
- Any docs page as markdown: append `.mdx` to its URL

If a docs page tells you to call `/suggested-fees`, `/swap/*`,
`/deposit/status` or any other legacy endpoint, it is the old version of the
page: ignore it and use only the `/v1` endpoints above. The legacy reference
at https://legacy.docs.across.to is only for reading existing code during a
[migration](../migration/SKILL.md); never build anything new from it.

## Support
- Discord: https://discord.across.to
