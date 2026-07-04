# Rate limits

> **Summary.** Floonet rate limiting is designed for Tor reality: relay connections all arrive from the co-located Tor process (localhost), and clearnet HTTP to the name authority arrives from a few shared Tor exit IPs, so naive per-IP limits either tell you nothing or punish the wrong people. Limit per connection where possible, keep per-IP windows loose, and lean on the whitelist and NIP-98 replay protection to do the real work.

## The Tor caveat first

Wallet traffic arrives over [Tor](../concepts/nym.md). Relay websocket connections come in through the co-located system-Tor onion service, so they all originate from `127.0.0.1` — the source IP is always localhost and identifies no one. The name authority's clearnet HTTP lookups arrive instead from a handful of shared Tor exit IPs, each carrying many users. Either way, any control keyed only on IP address will eventually rate limit, or ban, a source that dozens of honest wallets sit behind. The rules that follow all account for this:

- **Relay websockets: limit per connection, not per IP.** Event-rate and subscription limits apply to each websocket independently — which is the only thing that means anything when every connection reads as localhost.
- **Do not put Floonet services behind IP-reputation banning** (fail2ban and friends) without exempting Tor — relay connections come from localhost, and HTTP lookups come from shared Tor exit IPs — or you will ban your own users in bulk.

## What actually protects the relay

1. **The kind whitelist.** Most abuse is simply not storable on a Floonet relay; disallowed kinds are dropped before any quota is touched.
2. **Per-connection event and subscription limits.** Both upstreams provide these; the shipped configs set sane values.
3. **Per-IP HTTP windows on the name authority.** Registration and lookup endpoints keep per-IP windows (reads generous, writes tight), fed by `X-Real-IP` from the proxy, with limits loose enough that a shared Tor exit IP does not starve. The proxy forwarding `X-Real-IP` is load-bearing; without it every request appears to come from the proxy.
4. **NIP-98 replay protection.** Registration requests are single-use: the auth event's timestamp bound and replay window mean a captured request cannot be replayed to burn a name.
5. **Name-change cooldown.** A key that changes its name waits out a cooldown, which caps name-churn abuse at negligible cost to honest users.

## Tuning

The shipped defaults are deliberately generous on reads and conservative on writes. If you tighten them, watch for the Tor signature in your logs first: many distinct pubkeys behind one IP — localhost for relay connections, a shared exit IP for HTTP lookups — is normal Floonet traffic, not an attack.
