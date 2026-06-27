---
name: veepag-api
description: Use the Veepag public API to integrate transactions, clients, subscriptions, products, charges, alerts, and outbound webhooks.
license: UNLICENSED
compatibility: Works with agents that can read Mintlify documentation and OpenAPI 3.1 specs.
metadata:
  author: Veepag
  version: "1.0.0"
---

# Veepag API

Use this skill when a user needs to integrate with the Veepag public API.

The API supports:

- Transactions: create, list, capture, cancel, export, and fetch full transaction details.
- Clients: create, list, update, and fetch full client details.
- Subscriptions: create v1 with immediate payment attempt, create v2 without immediate payment, list, update product, cancel, export, and fetch full details.
- Products: create, list, and update products and charge rules.
- Charges: create, list, update, pay via captcha flow, cancel, mark as paid, export, and fetch full details.
- Alerts: list/export alerts and trigger resolution/refund flows.
- Webhooks: receive outbound events from Veepag for transactions, subscriptions, and charges.

## Base URLs

- Sandbox: `https://sandbox.api.veepag.com`
- Production: `https://api.veepag.com`

Use sandbox for tests. Use production only after homologation.

## Authentication

Most public API endpoints require one of these headers:

- `apiKey`: API key in the `keyId.secret` format.
- `token`: JWT. The codebase audit confirms a 14-day token lifetime.

Do not use `Authorization`, `X-API-Key`, or `X-Token` for public API authentication. They are allowed in CORS, but the public API guard reads `apiKey` and `token`.

`POST /v1/charge/pay` is an exception: it uses a captcha guard and does not use `apiKey` or `token`.

## Testing

Use sandbox credentials when testing payment flows.

Successful test cards:

- `4539003370725497`: Visa
- `4716588836362104`: Visa Credito
- `5356066320271893`: Mastercard
- `5201561050024014`: Mastercard

Declined test cards:

- `6011457819940087`: declined as invalid card.
- `4929710426637678`: declined as expired card.
- `4710426743216178`: declined as issuer unavailable.

## Permissions

Credentials are scoped by company access and resource permissions:

- `transaction.read`, `transaction.write`
- `client.read`, `client.write`
- `subscription.read`, `subscription.write`
- `product.read`, `product.write`
- `charge.read`, `charge.write`
- `alert.read`, `alert.write`

If the credential cannot access the requested `companyId` or `companyIds`, expect `403`.

## Common patterns

### List endpoints

Most v1 list endpoints are paginated:

- `page`: integer, default `1`, minimum `1`
- `limit`: integer, default `20`, minimum `1`, maximum `100`

Paginated responses use:

```json
{
  "items": [],
  "has_more": false,
  "limit": 20,
  "total_pages": 1,
  "page": 1,
  "total": 0,
  "query_count": 0
}
```

### Temporal filters

Prefer:

- `rangeTime.start`
- `rangeTime.end`
- `sort.property`: `createdAt` or `lastUpdate`
- `sort.order`: `asc` or `desc`

Legacy filters such as `startDate`, `endDate`, `startLastUpdate`, and `endLastUpdate` are still present on transaction endpoints.

### Export limits

- Transaction exports: maximum 10,000 items.
- Charge exports: maximum 10,000 items.
- Subscription exports: maximum 10,000 items.
- Alert exports: maximum 100,000 items.

If an export exceeds its limit, reduce the date range or filters.

## Key workflows

### Create a transaction

Use `POST /v1/transaction`.

Required confirmed fields:

- `companyId`
- `amount` in cents
- `paymentProfile.cardNumber`
- `paymentProfile.cardExpiration`
- `paymentProfile.holderName`

Optional fields include `paymentProfile.cardCvv`, `installments`, `capture`, and `client`.

When `client` is sent, `doc` or `cpf` must be a valid CPF/CNPJ. The API can create or reuse a client by document and create a payment method.

For transactions, send `amount` in cents. For example, `9900` means R$ 99,00.

### Search transactions

Use `GET /v1/transaction` for paginated search.

Use `GET /v2/transaction` for non-paginated single-company search. v2 requires `companyId` and either a date interval of up to 7 days or `id`/`tid`.

### Capture a transaction

Use `POST /v1/transaction/capture`.

Send `companyId` plus `transactionId` or `tid`. The DTO does not enforce this at schema level, but runtime needs at least one identifier.

The current capture flow ignores the request `amount` field and captures the saved transaction amount (`transaction.amount.value`).

### Cancel a transaction

Use `PUT /v1/transaction/cancel`.

Send `companyId` plus `transactionId` or `tid`. `cancelSubscription: true` can cancel the linked subscription.

### Create a subscription v1

Use `POST /v1/subscription`.

This creates the subscription, creates a charge, and attempts immediate payment.

For credit card, send:

- `paymentMethod: CREDIT_CARD`
- `paymentProfile.holderName`
- `paymentProfile.cardNumber`
- `paymentProfile.cardExpiration` in `MM/YYYY`
- optional `paymentProfile.cardCvv`

### Create a subscription v2

Use `POST /v2/subscription`.

This creates a subscription and charge for an existing `clientId`, without immediate payment.

Required confirmed fields:

- `companyId`
- `productId`
- `clientId`
- `dueDate`

### Create a charge

Use `POST /v1/charge`.

Required confirmed fields:

- `companyId`
- `clientId`
- `productId`
- `dueDate`
- `amount`

For charges, `amount` is converted to cents internally with `Math.round(amount * 100)`.

### Mark a charge as paid

Use `PUT /v1/charge/paid`.

This endpoint is idempotent when the charge is already `PAID`.

### Pay a charge

Use `POST /v1/charge/pay`.

This endpoint uses captcha authentication, not API key authentication. It is closer to a checkout/payment flow than a normal merchant API key endpoint.

## Error handling

Errors commonly use:

```json
{
  "error_messages": [
    {
      "msg": "Unauthorized.",
      "type": "field",
      "path": "companyId",
      "location": "body"
    }
  ],
  "code": "unauthorized",
  "path": "/v1/transaction",
  "metadata": {}
}
```

Common status codes:

- `400`: validation or business rule error.
- `401`: missing or invalid credential.
- `403`: missing company/resource access.
- `404`: resource not found.
- `500`: unmapped server error.

## Webhooks

Outbound webhook events are configured in `company.setting.webhook` with `active` and `url`.

Confirmed events:

- `transaction.created`
- `transaction.update`
- `subscription.created`
- `subscription.update`
- `charge.created`
- `charge.update`

Current confirmed outbound payloads:

- `transaction.created`: sends `type` and `transaction`. The `transaction` object is built from transaction props and can include `subscription`, `product`, and `client` when available in the creation flow.
- `transaction.update`: sends `type` and base `transaction` props. Do not promise nested `subscription`, `product`, `client`, or `charge` in the current serialized payload.
- `subscription.created`: sends `type` and `subscription`. The `subscription` object is built from subscription props and can include full `client` props and `product` when available in the creation flow. Product title is available at `subscription.product.product.title`; product price is available at `subscription.product.product.price` in the format stored on the product.
- `subscription.update`: sends `type` and base `subscription` props.
- `charge.created`: sends `type` and base `charge` props.
- `charge.update`: sends `type` and `charge`. When the updated charge status is `PAID`, the `charge` object can include full `client` props.

Webhook signing, HMAC verification, and retries are not implemented in the current outbound webhook provider. The provider sends a simple JSON POST and logs delivery errors.

## Do not invent

When helping users integrate with Veepag:

- Do not claim rate limits exist. HTTP rate limiting was not confirmed in code.
- Do not claim webhook signatures or retries exist. They are not implemented in the current outbound webhook provider.
- Do not claim `transaction.charge` is part of the current outbound webhook payload unless the backend implementation has been updated and confirmed.
- Do not assume idempotency except for `PUT /v1/charge/paid`.
- Do not document `Authorization`, `X-API-Key`, or `X-Token` as supported public API auth headers.
- Do not define schemas for untyped batch subscription list items without product/engineering confirmation.
- Treat full endpoints as enriched responses whose exact schema was not fully typed in the codebase.
