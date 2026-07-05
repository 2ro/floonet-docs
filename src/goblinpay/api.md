# Integrating with the GoblinPay API

> **Summary.** The direct-integration path: your service creates an invoice, the customer pays GoblinPay wallet-to-till, and you grant value once the payment reaches `confirmed`. This is the same path [Magick Market](https://magick.market) uses, so integrate like Magick.

The important property: **no coins ever pass through the API or through your service.** Your backend only ever calls create-invoice and reads status. The actual payment is a private, encrypted Grin transfer from the customer's Goblin wallet to the till over the Nostr gift-wrap rail (or, for non-Goblin wallets, over the optional grin1/Tor rail). Your server is never in the money path; it is only in the *authorization* path.

```
your service ──POST /invoice──▶ GoblinPay ──returns nprofile + pay_url──▶ you show the customer
customer's wallet ═══ encrypted Grin payment ═══▶ GoblinPay till   (never touches you)
GoblinPay ──payment.confirmed webhook──▶ your service        (or you poll GET /invoice/{id})
your service grants the goods on "confirmed"
```

## Authentication

Every API call carries a bearer token:

```
Authorization: Bearer <GP_API_TOKEN>
```

`GP_API_TOKEN` is generated for you by `gp-server setup` (shape `gp_live_…`). With no token configured the write API is closed (returns `503`), never open. A missing or wrong token returns `401`.

## Create an invoice

```
POST /invoice
Authorization: Bearer <GP_API_TOKEN>
Content-Type: application/json
```

Request body (provide **either** `amount_grin` **or** `amount_fiat` + `currency`):

| Field | Type | Notes |
|---|---|---|
| `amount_grin` | integer | Exact amount in **base units (nanogrin)**. 1 GRIN = 1_000_000_000 nanogrin. |
| `amount_fiat` | string | Decimal fiat amount (e.g. `"12.50"`). Priced to Grin at create time via the rate oracle. |
| `currency` | string | ISO code for `amount_fiat` (must be in `GP_RATE_CURRENCIES`). |
| `order_ref` | string | Your order id. Used as the memo/subject match key and echoed back. Optional but recommended. |
| `memo` | string | Human note shown on the checkout page. Optional. |
| `match_mode` | string | Per-invoice override: `memo`, `derived`, or `amount`. Optional; defaults to `GP_MATCH_MODE` (`derived` recommended). |

Example:

```bash
curl -sS https://pay.myshop.com/invoice \
  -H "Authorization: Bearer $GP_API_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"amount_grin": 2000000000, "order_ref": "order-1042", "memo": "Order #1042"}'
```

Response (`200`) carries `invoice_id`, `token`, `pay_url`, `recipient_pubkey`, `npub`, `nprofile`, `qr_svg`, `amount`, `status`, `confirmations`, `confirmations_required`, and your echoed `order_ref`/`memo`. To collect payment, either **redirect** the customer to `pay_url` (the hosted zero-JS checkout with QR, live status, and manual-paste fallback), or **render your own QR** from `nprofile` (a Goblin wallet scans `nostr:<nprofile>`) or embed the ready-made `qr_svg`.

### The amount field is base units (read this)

Base units and display strings both appear, so be careful:

- In the **request**, `amount_grin` is **base units (nanogrin)**, an integer.
- In the **response**, `amount` is a **human display string** (e.g. `"2 GRIN"`, or `"12.50 usd (~150 GRIN)"` for a fiat invoice). Display only; do not parse it for accounting.
- In the **webhook**, `payment.amount` is **base units (nanogrin)** as an integer, and `payment.amount_grin` is the human decimal string.

When you reconcile, trust the base-unit integers (`amount_grin` you sent and `payment.amount` in the webhook), not the display string.

## Read invoice status

```
GET /invoice/{invoice_id}
Authorization: Bearer <GP_API_TOKEN>
```

Returns the same JSON shape as create, with the current `status`, `confirmations`, and `confirmations_required`. Status advances `open ──▶ paid ──▶ confirmed`:

- `open`: created, not yet paid.
- `paid`: received and matched to this invoice (mempool / low confirmations).
- `confirmed`: the paying kernel reached `confirmations_required` (`GP_CONFIRMATIONS`, default 10). **Grant the goods on `confirmed`.**

Polling `GET /invoice/{id}` server-to-server is a complete integration on its own if you would rather not run a webhook endpoint.

## The `payment.confirmed` webhook

If you set `GP_WEBHOOK_URL` (and `GP_WEBHOOK_SECRET`, which the wizard generates), GoblinPay POSTs a signed JSON event to your endpoint on each payment event so you do not have to poll.

```json
{
  "event_id": "5f3c…",
  "event_type": "payment.confirmed",
  "payment": {
    "slate_id": "…",
    "amount": 2000000000,
    "amount_grin": "2",
    "status": "confirmed",
    "confirmations": 10
  },
  "invoice_id": "…",
  "order_ref": "order-1042"
}
```

- `event_type` is `payment.received` (first seen) or `payment.confirmed` (reached `GP_CONFIRMATIONS`). **Grant on `payment.confirmed`.**
- `payment.amount` is base units (nanogrin); `payment.amount_grin` is the human decimal string. `payment.confirmations` is present only on `payment.confirmed`.
- `invoice_id` / `order_ref` tie the event back to your order (`invoice_id` may be null for an unmatched payment).

### Verifying the signature

Every delivery carries two headers:

```
X-GoblinPay-Signature: sha256=<hex>
X-GoblinPay-Delivery: <event_id>
```

`<hex>` is `HMAC-SHA256(GP_WEBHOOK_SECRET, raw_request_body_bytes)`. Recompute it over the **raw bytes** you received (do not re-serialize the JSON) and compare in constant time. Reject on mismatch.

### Retry and idempotency semantics

- **At-least-once.** Deliveries are persisted; a mid-retry crash resumes.
- **Ack with 2xx.** Any `2xx` marks the delivery done; anything else, or a transport error, reschedules it.
- **Backoff.** `min(BASE * 2^(attempts-1), 3600s)`, doubling per failed attempt up to a 1-hour cap.
- **Give up after 12 attempts** (`MAX_ATTEMPTS`).
- **Deduplicate on `event_id`** (also the `X-GoblinPay-Delivery` header). A retried delivery repeats the same `event_id`, so make your handler idempotent: if you have already granted this order, just return `2xx`.

## Refunds

GoblinPay is receive-only: there is no refund API. Refunds are handled out-of-band by sending Grin back from your wallet.
