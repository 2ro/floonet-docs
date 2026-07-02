# Nym privacy and the Floonet exit

> **Summary.** Wallet traffic reaches Floonet relays through the Nym mixnet: five hops of cover traffic that hide who is connecting to whom, even from the relay. The Floonet stack can also operate its own Nym exit gateway, which wallets prefer as a reliable anchor while always keeping the public exit pool as fallback.

## Why a mixnet in front of a relay

TLS hides content but not the connection itself: an observer (or the relay operator) can still see which IP talks to which relay and when. The [Nym](https://nym.com) mixnet breaks that link. Wallet traffic is sphinx-packeted through five hops with cover traffic, so a Floonet relay sees connections arriving from **mixnet exit IPs**, not from users.

Two consequences for operators:

1. **You cannot rate limit by IP alone.** Many users legitimately share one Nym exit IP. Floonet controls abuse per connection, not per IP. See [Rate limits](../operate/rate-limits.md).
2. **You learn nothing from your own logs.** Combined with [gift wraps](gift-wraps.md), a relay operator sees ciphertext arriving from anonymized connections. There is nothing to leak, subpoena, or sell.

## The anchor + fallback exit

Public Nym exits vary in reliability. The Floonet stack therefore supports running **its own Nym exit gateway** (a `nym-node` in `exit-gateway` mode, bonded into the Nym directory) as part of the infrastructure, with a firm design rule:

- The wallet treats the Floonet exit as a **preferred anchor**: it is tried first, once per selection cycle, because it is known-good and monitored.
- If the anchor is down or slow, the wallet **falls back to public auto-select** immediately and retries the anchor on the next reselect.
- **Pin-only is forbidden.** A wallet must never be configured to use only one exit. Pinning a single exit recreates the single point of failure the design exists to kill, and it would concentrate all exit traffic at one party.

This gives reliability without centralization: the anchor makes the common case fast and dependable, the pool keeps the network alive when the anchor is not.

## What the exit sees

Even the operator of the Floonet exit sees only what any public exit operator sees: opaque ciphertext (traffic is TLS end-to-end inside the tunnel, wrapping NIP-44 gift wrap underneath) plus destination IPs. DNS rides DoT and DoH through the tunnel, so not even lookups are visible.

One binding operational rule: **run the exit on infrastructure separate from the relay hosts.** If the same operator vantage point hosts both the exit and the relay, exit-side timing could be lined up with relay-side arrivals. Separation keeps the two vantage points honest.

## References

- Nym operator docs: <https://nym.com/docs/operators/nodes/nym-node/setup>.
- Rate limiting under mixnet traffic: [Rate limits](../operate/rate-limits.md).
