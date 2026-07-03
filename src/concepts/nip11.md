# Neutral relay metadata (NIP-11)

> **Summary.** A relay's public NIP-11 information document (name, description, supported NIPs, software) never mentions payments, transactions, slatepacks, or money. Floonet relays only ever see opaque ciphertext, so payment wording would be both inaccurate and a liability. The shipped defaults are neutral Floonet branding.

## Motivation

NIP-11 is the JSON document a relay serves when an HTTP client asks for `application/nostr+json`. It is the relay's public face: crawlers index it, relay browsers display it, and anyone can fetch it.

A Floonet relay stores gift wraps (kind `1059`), which are opaque, encrypted envelopes. The relay cannot know what is inside them. Describing the relay as handling "payments" would therefore be a claim it cannot verify about content it cannot read, and it would paint a target on the operator. So the rule is simple: **the relay's own metadata says nothing about payments**.

## Shipped defaults

| Package | NIP-11 `name` | NIP-11 `description` | Where it is set |
| --- | --- | --- | --- |
| floonet-strfry | `Floonet Relay` | `A strfry Floonet relay for the Grin community Nostr network.` | `relay.info.name` / `relay.info.description` in `strfry.conf` |
| floonet-rs | `floonet-rs-relay` | `A Floonet relay for the Grin community Nostr network.` | the `[info]` block in `config.toml` |

Operators can customize both freely. The point is that the *defaults* carry zero payment language, so nobody ships a payment-labelled relay by accident.

## The audit rule

The neutrality rule covers every relay-facing surface, not just NIP-11:

- The NIP-11 `name` and `description`.
- The HTML landing page the relay serves to browsers (both packages serve a neutral Floonet page with the Floonet logo).
- Any other served JSON.
- The example configs in the READMEs.

Operator documentation (like this book) may of course explain paid names and paid access. The relay's own public metadata may not.

## References

- NIP-11: <https://nips.nostr.com/11>.
- What the relay actually sees: [Gift wraps](gift-wraps.md).
- Charging for resources without advertising it on the relay: [Charge GRIN for your relay](../operate/charge-grin.md).
