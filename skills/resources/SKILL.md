# Resources and Links

## Across API
- API keys: **https://dashboard.across-protocol.xyz**. Get one before starting an
  integration or migration.
- Base URL: `https://api.across-protocol.xyz` (every path starts with `/v1`)
- Testnet: none. Test on mainnet with small amounts.

| Endpoint | Reference |
|---|---|
| `GET /v1/capabilities` | https://docs.across-protocol.xyz/api-reference/v1/capabilities/get |
| `POST /v1/quotes` | https://docs.across-protocol.xyz/api-reference/v1/quotes/post |
| `POST /v1/quotes/{orderId}/executions` | https://docs.across-protocol.xyz/api-reference/v1/quotes/orderid/executions/post |
| `GET /v1/transfers/{orderId}` | https://docs.across-protocol.xyz/api-reference/v1/transfers/orderid/get |
| `GET /v1/transfers` | https://docs.across-protocol.xyz/api-reference/v1/transfers/get |

## Guides
- Get an API key: https://docs.across-protocol.xyz/introduction/api-keys
- Quickstart: https://docs.across-protocol.xyz/introduction/quickstart
- Migration guide (legacy APIs → Across API): https://docs.across-protocol.xyz/introduction/migration-guide
- Destination actions: https://docs.across-protocol.xyz/introduction/quickstart/destination-actions
- Fees and app fees: https://docs.across-protocol.xyz/introduction/fees
- Refunds: https://docs.across-protocol.xyz/introduction/refunds
- Docs home: https://docs.across-protocol.xyz

## Machine-readable
- OpenAPI spec: https://docs.across-protocol.xyz/openapi.yaml
- Docs index for agents: https://docs.across-protocol.xyz/llms.txt
- Any docs page as markdown: append `.mdx` to its URL

If a docs page tells you to call `/suggested-fees`, `/swap/*`,
`/deposit/status` or any other legacy endpoint, it is the old version of the
page: ignore it and use only the `/v1` endpoints above. The legacy reference
at https://legacy-docs.across-protocol.xyz is only for reading existing code during a
[migration](../migration/SKILL.md); never build anything new from it.

## Support
- Discord: https://discord.across-protocol.xyz
