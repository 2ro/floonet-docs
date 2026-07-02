# Paid names via GoblinPay

> **Summary.** An operator can charge GRIN for names (or for write access). The price is a config value, payments are handled by a GoblinPay server the operator runs, and the relay's public metadata stays payment-free throughout.

## The model

[GoblinPay](../operate/charge-grin.md) is the Grin payment backend. The name authority (or the write-policy plugin, for paid writes) talks to it over REST:

1. A user asks to register `alice`.
2. With `FLOONET_PAY_MODE=name`, the authority quotes `FLOONET_NAME_PRICE_GRIN` and creates a GoblinPay invoice.
3. The user pays in GRIN: via the GoblinPay pay page, a manual slatepack exchange, or a `grin1` address if the operator enabled GoblinPay's Tor method.
4. GoblinPay confirms the payment on chain (payment proof included).
5. The authority sees the confirmed payment and completes the registration. Results are cached with a TTL, so checks are cheap.

Until step 5, registration is refused. Change the price in config and the quote changes; no code involved.

## Modes

| `FLOONET_PAY_MODE` | Behavior |
| --- | --- |
| `off` | Everything free (default) |
| `name` | Claiming a name requires a confirmed payment of `FLOONET_NAME_PRICE_GRIN` |
| `write` | Writing events requires a confirmed payment (the plugin enforces it per authed pubkey) |

## What the client sees

The name-authority response for an unpaid registration carries enough information for a wallet to generate the payment page automatically: the future wallet claim flow is "type a name server and a name, press claim, get sent to a pay page". The server side supports that flow today.

## The neutrality rule still applies

Charging for names changes nothing about the relay's public face: [NIP-11 metadata stays neutral](../concepts/nip11.md). Payments are a matter between the operator's authority and the user's wallet; the relay itself neither sees nor mentions them.

## Beyond names

The paid gate is one mechanism applied to many resources. Names are the first; media storage is the documented second. See [Media for GRIN](../operate/media-for-grin.md).
