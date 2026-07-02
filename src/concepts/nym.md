# The mixnet and the scoped exit

> **Summary.** Wallet traffic reaches Floonet relays through the [Nym](https://nym.com) mixnet: five hops of cover traffic that hide who is connecting to whom, even from the relay. Both Floonet packages can also run a **co-located scoped mixnet exit**: a tiny forwarder, bundled with the relay and enabled by one config toggle, that pipes mixnet streams to that one relay and nowhere else. Wallets that know the exit's mixnet address dial the relay straight through it, with no public DNS and no shared public exit on the path. It is the fast money path.

## Why a mixnet in front of a relay

TLS hides content but not the connection itself: an observer (or the relay operator) can still see which IP talks to which relay and when. The mixnet breaks that link. Wallet traffic is sphinx-packeted through five hops with cover traffic, so a Floonet relay sees connections arriving from **mixnet IPs**, not from users.

Two consequences for operators:

1. **You cannot rate limit by IP alone.** Many users legitimately share one mixnet exit IP. Floonet controls abuse per connection, not per IP. See [Rate limits](../operate/rate-limits.md).
2. **You learn nothing from your own logs.** Combined with [gift wraps](gift-wraps.md), a relay operator sees ciphertext arriving from anonymized connections. There is nothing to leak, subpoena, or sell.

## The scoped exit

A wallet on the mixnet still has to *leave* it somewhere to reach your relay. By default that egress is a shared public exit, plus a public DNS lookup for your hostname: both usually fine, both the flakiest links in the chain. The scoped exit removes both.

Each Floonet package bundles `floonet-mixexit`: an ordinary, **unbonded** mixnet client (no bonding, no NYM tokens, no `nym-node`) that runs next to your relay and forwards every accepted mixnet stream to **one fixed upstream, your own relay**, never a caller-chosen target. That scoping is the whole design:

- **It is structurally not an open proxy.** There is no exit policy to manage and no abuse surface: the exit can reach nothing but the relay it fronts.
- **Wallets dial it by mixnet address alone.** The address is published in the wallet-side [relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4)'s `exit` field (the Goblin pool is public: the live relays, each with its exit address); a wallet that has it opens a mixnet stream straight to your machine, so the money path needs **no public DNS** and no shared public exit.
- **Anchor + fallback, never pin-only.** Wallets prefer the exit and fall back to the public mixnet route the moment it fails, so a dead exit costs seconds, never a lockout. Pinning a single route is deliberately impossible: it would recreate the single point of failure this design exists to kill.

### The fast money path

Skipping the public-exit hop and the in-mixnet DNS lookup is not just tidier, it is dramatically faster. Against `relay.floonet.dev` (the Floonet flagship, and the Goblin wallet's default money-path relay: floonet-strfry with the co-located exit), a cold-started wallet connects in about **0–2 seconds**, and a funded Grin payment finalizes in about **6 seconds** end to end. Over the public route the first connect used to take anywhere from 15 seconds to 3 minutes. The public route remains the automatic fallback, so an exit outage degrades to the old speed, never to a dead end.

## What the exit sees, and why co-location is fine

Co-locating an exit with the relay it fronts is deliberate, and safe, *because it is scoped*:

- **TLS is end to end.** The mixnet stream carries the wallet's ordinary hostname-validated TLS handshake against your relay's certificate. The exit pipes ciphertext; a hostile or compromised exit cannot read or tamper with the connection.
- **Streams arrive anonymized.** The mixnet strips client identity before the exit ever sees a byte, so there is no source to log.
- **Nothing new to correlate.** The exit forwards only to the relay beside it, so the operator's combined vantage point is exactly what running the relay alone already gives them: ciphertext arriving from anonymous connections.

## Turning it on

One toggle per package; the exit's stable mixnet address is printed at startup and written to `nym_address.txt` in its data volume or directory. Publish that address (for example in the [Floonet relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4)'s `exit` field) so wallets can prefer your exit, and back the data directory up: losing it rotates the address.

- **floonet-strfry**: set `COMPOSE_PROFILES=exit` in `.env`. See [Deploy floonet-strfry](../floonet-strfry/deploy.md#the-mixnet-exit).
- **floonet-rs**: set `enabled = true` in the `[exit]` section of `config.toml`. See [Configuration (floonet-rs)](../floonet-rs/config.md#exit-the-co-located-mixnet-exit).

## References

- Nym stream API (what `floonet-mixexit` is built on): <https://nym.com/docs/developers/rust>.
- Rate limiting under mixnet traffic: [Rate limits](../operate/rate-limits.md).
- The exit config keys: [Config keys](../reference/config-keys.md).
