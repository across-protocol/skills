# Resources and Links

## Across API
- Base URL: `https://api.across.to` (every path starts with `/v1`)
- API keys: https://integrator.portal.across.to
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
- Fees and app fees: https://docs.across.to/introduction/fees
- Refunds: https://docs.across.to/introduction/refunds
- Tracking: https://docs.across.to/introduction/tracking-deposits
- Docs home: https://docs.across.to

If a docs page tells you to call `/suggested-fees`, `/swap/*`,
`/deposit/status` or any other legacy endpoint, it is the old version of the
page: ignore it and use only the `/v1` endpoints above.

## Support
- Discord: https://discord.across.to
