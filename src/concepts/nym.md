# Tor and the relay's onion service

> **Summary.** Wallet traffic reaches Floonet relays over [Tor](https://www.torproject.org). Tor has exactly one job here: hide the wallet's IP and network location from the relay and from anyone watching the network. That is the only thing a mixnet was ever needed for — everything else is handled by the relay and the Nostr protocol (message content is end-to-end encrypted, the sender is a throwaway one-time key, and send/receive timing is decorrelated by the relay holding each message and releasing it on a short randomized delay). Both Floonet packages run a **co-located Tor onion service**: plain system Tor, configured with a couple of `torrc` directives, bundled with the relay and enabled by one toggle, fronting the relay's existing TLS/websocket port. Wallets dial the relay's pinned `.onion` over Tor, with no public DNS on the path. It is the fast money path.

## Why Tor in front of a relay

TLS hides content but not the connection itself: an observer, or the relay operator, can still see which IP talks to which relay and when. Tor breaks that link, and that is the one thing we need it for. The relay is the thing a wallet connects to, so it is the single piece of the system that structurally cannot hide the wallet's network identity by itself — and hiding your network location from the thing you are connecting to is exactly what Tor is the best tool in the world for. With Tor in front, a Floonet relay sees connections arriving from Tor, never from a user's real IP.

That job is narrow on purpose, because the relay and Nostr already cover everything else:

- **Content** is end-to-end encrypted (NIP-44 inside NIP-59 [gift wraps](gift-wraps.md)); nobody but the recipient can read a payment, least of all the relay.
- **The sender** is a throwaway one-time key, so the relay never learns who actually sent a message.
- **Timing** is shuffled by the relay itself: it holds each incoming gift wrap and releases it to the recipient on a short, randomized delay, so nobody can match "you sent" to "they received." This is the one property a mixnet used to provide, rebuilt on the relay we already run and fully control.

So a full mixnet's heavy machinery — the layered cover traffic, the token-metered bandwidth — was never needed for what a Grin wallet actually has to hide. Tor does the one narrow job it is perfect for; our own relay does the rest.

Two consequences for operators:

1. **You cannot rate limit by IP alone.** Every onion connection arrives from the co-located Tor process on `127.0.0.1`, so the source IP is always localhost and tells you nothing about users. Floonet controls abuse per connection, not per IP. See [Rate limits](../operate/rate-limits.md).
2. **You learn nothing from your own logs.** Combined with [gift wraps](gift-wraps.md), a relay operator sees ciphertext arriving from anonymized Tor connections. There is nothing to leak, subpoena, or sell.

## Why Tor, not a mixnet

Floonet's wallet traffic used to ride the Nym mixnet. We moved to Tor for a concrete, unhappy reason: Nym's free bandwidth tier is being removed. It is testnet scaffolding, hard-coded to expire at the next UTC-midnight rollover, and Nym's public gateways are switching to a paid model that requires holding the NYM token to buy bandwidth. A money wallet cannot stand on bandwidth that expires on a schedule or has to be rented with a speculative token — it went dark on us more than once, and "your payments work unless it's the wrong time of day" is not something to ship.

Tor has none of those problems. It is free, unmetered, battle-tested, has no token to hold and nothing to expire, runs in-process on a phone, is lighter on the battery, and is faster for the moments a user actually waits on — sending a payment, opening the app — because it skips the mixnet's built-in per-hop delays. The one honest thing we give up: a full mixnet resists an adversary who can watch the entire internet at once. Tor does not claim to, and neither do we — that is simply not the threat a low-value Grin payments wallet faces.

## The onion service

A wallet can reach your relay over Tor by dialing its public hostname over an ordinary circuit (a public Tor exit, then a public DNS lookup, then your TLS endpoint). That works, but it leans on a shared public exit and a public DNS lookup — both usually fine, both the flakiest links in the chain. An onion service removes both.

Each Floonet package can run a co-located **Tor onion service**: plain, mature **system Tor** — C-Tor, the most-audited part of the Tor codebase, doing the "hosting half" of Tor — fronting your relay's existing TLS/websocket port. Concretely it is a `torrc` onion service:

```
HiddenServiceDir /var/lib/tor/floonet-relay/
HiddenServicePort 443 127.0.0.1:443
```

The relay's `.onion` address is the `hostname` file inside `HiddenServiceDir`. Enable **Vanguards** on the service side. The design is deliberately narrow:

- **It is not an open proxy.** The onion service forwards to exactly one target, the relay's own websocket port, and nothing else.
- **Wallets dial it by `.onion` alone.** The address is published in the wallet-side [relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4)'s `onion` field (the Goblin pool is public: the live relays, each with its `.onion`); a wallet that has it dials the relay's onion straight over embedded Tor, so the money path needs **no public DNS** and no shared public exit.
- **Forward-safe.** The `.onion` sits beside each relay entry in the pool, and builds that predate the field simply ignore it. Publishing an onion adds the fast onion path without breaking older wallets.

### The fast money path

Dialing the relay's `.onion` directly — skipping the public Tor exit, the public DNS lookup, and the old mixnet's per-hop mixing delays entirely — is not just tidier, it is dramatically faster. Against `relay.floonet.dev` (the Floonet flagship, and the Goblin wallet's default money-path relay: floonet-strfry with its co-located onion service), a cold-started wallet connects in about **0–2 seconds**, and a funded Grin payment finalizes in about **6 seconds** end to end. On the wallet side, the money path dials the relay's `.onion` directly over Tor — the pinned onion *is* the money path, so a payment is never quietly routed around it.

## What the operator sees, and why co-location is fine

Co-locating an onion service with the relay it fronts is deliberate, and safe:

- **TLS is end to end.** The onion carries the wallet's ordinary hostname-validated TLS handshake and websocket upgrade against your relay's certificate. System Tor pipes ciphertext; the onion layer cannot read or tamper with the connection.
- **Connections arrive anonymized.** Tor strips the client's network identity before the connection reaches your relay, so there is no source to log — every onion connection looks like it came from the local Tor process.
- **Nothing new to correlate.** The onion forwards only to the relay beside it, so the operator's combined vantage point is exactly what running the relay alone already gives them: ciphertext arriving from anonymous connections.

## Turning it on

One toggle per package brings up the bundled system-Tor onion service (via `torrc`) in front of the relay's websocket port. The relay's stable `.onion` address lands in the `hostname` file inside `HiddenServiceDir`. Publish that address (for example in the [Floonet relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4)'s `onion` field) so wallets can dial your relay over Tor, and back the `HiddenServiceDir` up: losing it rotates the `.onion`.

- **floonet-strfry**: set `COMPOSE_PROFILES=onion` in `.env`. See [Deploy floonet-strfry](../floonet-strfry/deploy.md#the-tor-onion-service).
- **floonet-rs**: set `enabled = true` in the `[onion]` section of `config.toml`. See [Configuration (floonet-rs)](../floonet-rs/config.md#onion-the-co-located-tor-onion-service).

## References

- The Tor Project: <https://www.torproject.org>.
- Tor onion services (what the relay runs): <https://community.torproject.org/onion-services/>.
- Rate limiting under Tor traffic: [Rate limits](../operate/rate-limits.md).
- The onion config keys: [Config keys](../reference/config-keys.md).
