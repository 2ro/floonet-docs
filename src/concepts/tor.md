# Tor: how wallets reach a relay

> **Summary.** Goblin wallets reach every Floonet relay over [Tor](https://www.torproject.org). Tor has exactly one job here: hide the wallet's IP and network location from the relay and from anyone watching the network. The wallet embeds a Tor client (arti, compiled straight into the app) and dials each relay's ordinary clearnet host over a Tor exit, running the usual hostname-validated TLS for `wss://`. There is nothing to install or configure on the relay: a Floonet relay is a normal public Nostr relay that simply accepts connections arriving from Tor. Everything else a Grin wallet needs to hide is handled by the relay and the Nostr protocol: message content is end-to-end encrypted, the sender is a throwaway one-time key, and send/receive timing is decorrelated by the relay holding each message and releasing it on a short randomized delay.

## Why Tor between the wallet and the relay

TLS hides content but not the connection itself: an observer, or the relay operator, can still see which IP talks to which relay and when. Tor breaks that link, and that is the one thing we need it for. The relay is the thing a wallet connects to, so it is the single piece of the system that could otherwise see the wallet's real network identity. With the wallet dialing over Tor, a Floonet relay sees connections arriving from Tor exit nodes, never from a user's real IP.

That job is narrow on purpose, because the relay and Nostr already cover everything else:

- **Content** is end-to-end encrypted (NIP-44 inside NIP-59 [gift wraps](gift-wraps.md)); nobody but the recipient can read a payment, least of all the relay.
- **The sender** is a throwaway one-time key, so the relay never learns who actually sent a message.
- **Timing** is shuffled by the relay itself: it holds each incoming gift wrap and releases it to the recipient on a short, randomized delay, so nobody can match "you sent" to "they received." This timing decorrelation runs on the relay we already operate and fully control.

So no heavier privacy machinery, layered cover traffic or token-metered bandwidth, is needed for what a Grin wallet actually has to hide. Tor does the one narrow job it is perfect for; our own relay does the rest.

## What rides Tor, and what does not

Only the wallet's Nostr and identity traffic rides Tor: relay websockets, NIP-05 name lookups, the price feed, and the relay-pool fetch. The Grin node's own blockchain traffic is deliberately **not** routed through Tor. It stays on the clear internet, direct to the node, exactly as an ordinary Grin wallet talks to its node. Who pays whom lives entirely in the gift-wrapped Nostr layer, so the node has nothing to hide, and keeping node sync on the clear keeps it fast.

## What this means for a relay operator

Two things, both simple:

1. **Your relay must accept Tor.** Tor is the private path, and now a per-wallet setting: wallets updated from an older version keep it on, brand-new wallets choose at setup (off by default), and any user may turn it off and dial relays directly over clearnet (trading network-level anonymity for speed). The wallet's candidate pool reflects that. Most entries are chosen precisely because they accept Tor exits; a few large public relays that refuse Tor exit connections (for instance `relay.damus.io`, `nos.lol`, `relay.primal.net`) are carried as **clearnet-only** entries, reachable only by wallets that have Tor turned off. A relay that blocks or throttles Tor exit IPs is simply unreachable to the Goblin users who keep Tor on, which is most of them. If you run a Floonet relay for Grin users, do not put Tor behind an IP block.
2. **You cannot rate limit by IP alone, and you learn nothing from your logs.** Tor connections arrive from a handful of shared exit IPs, each carrying many users, so per-IP limits punish the wrong people; control abuse per connection instead. Combined with [gift wraps](gift-wraps.md), a relay operator sees only ciphertext arriving from anonymized connections. There is nothing to leak, subpoena, or sell. See [Rate limits](../operate/rate-limits.md).

## History: the onion experiment

A brief interim design (Goblin build 133) had each relay publish a co-located Tor **onion service**, so wallets could dial a pinned `.onion` with no public DNS on the path. Under load the shared onion hop flapped (WebSocket 1006 closes) and stalled payments, so build 134 dropped onion services entirely. Every relay is now reached over a plain Tor exit straight to its clearnet host, and the relay side carries no transport component at all.

The one honest thing a plain-Tor design gives up: resistance to an adversary who can watch the entire internet at once. Tor does not claim that, and neither do we. It is simply not the threat a low-value Grin payments wallet faces.

## References

- The Tor Project: <https://www.torproject.org>.
- Rate limiting under Tor traffic: [Rate limits](../operate/rate-limits.md).
