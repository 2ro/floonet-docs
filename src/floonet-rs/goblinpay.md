# The GoblinPay processor

> **Summary.** nostr-rs-relay already has a pay-to-relay framework with a `PaymentProcessor` trait and `account` and `invoice` tables, built for Lightning. floonet-rs adds a GoblinPay implementation of the same trait, so relays can be paid in GRIN instead.

## What upstream provides

Upstream's `pay_to_relay` model (`nostr-rs-relay/src/payment/mod.rs`) defines a `PaymentProcessor` trait with LNBits and CLN implementations, plus `account` and `invoice` tables tracking who has paid. floonet-rs keeps that skeleton and swaps the money.

## What the GoblinPay processor does

The `src/payment/goblinpay.rs` implementation covers the trait's lifecycle against a GoblinPay server:

1. **Create invoice.** Ask GoblinPay for an invoice for the configured amount (`FLOONET_NAME_PRICE_GRIN` for names, or the write-access price). GoblinPay returns the payment details the client needs, including the pay-page URL.
2. **Confirm.** Poll GoblinPay's REST API for payment status. Confirmation is on-chain, with a Grin payment proof, so a confirmed invoice means real money moved.
3. **Record.** Mark the invoice paid; the [admission module](admission.md)'s paid gate and the [name authority](name-authority.md)'s registration path both read that record. Lookups are cached with a TTL.

## Enforcement points

- **Paid writes** (`FLOONET_PAY_MODE=write`): the admission chain rejects events from pubkeys without a confirmed payment.
- **Paid names** (`FLOONET_PAY_MODE=name`): `POST /api/v1/register` is refused until the quoted invoice confirms. Names get a dedicated `name_claims` table rather than overloading `account`, while reusing the upstream `invoice` table for the money side.

## Payment UX

Users can pay a GoblinPay invoice three ways, depending on what the operator enabled in GoblinPay: the generated pay page, a manual slatepack exchange, or a `grin1` address (Tor method). The relay does not care which; it only ever asks GoblinPay "is this invoice confirmed".

## Config

```toml
[goblinpay]
pay_mode = "name"             # or "write"
url = "https://pay.example.com"
```

Plus `FLOONET_GOBLINPAY_TOKEN` in the environment. Setting a paid `pay_mode` wires upstream's `[pay_to_relay]` to the GoblinPay processor automatically. Full table in the [config keys reference](../reference/config-keys.md), full walkthrough in [Configuration](config.md).
