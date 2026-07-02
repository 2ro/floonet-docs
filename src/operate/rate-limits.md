# Rate limits

> **Summary.** Floonet rate limiting is designed for mixnet reality: many honest users share a few Nym exit IPs, so naive per-IP limits punish the wrong people. Limit per connection where possible, keep per-IP windows loose, and lean on the whitelist and NIP-98 replay protection to do the real work.

## The mixnet caveat first

Wallet traffic arrives through the [Nym mixnet](../concepts/nym.md), which means a handful of exit-gateway IPs carry many users. Any control keyed only on IP address will eventually rate limit, or ban, an exit that dozens of honest wallets are using. The rules that follow all account for this:

- **Relay websockets: limit per connection, not per IP.** Event-rate and subscription limits apply to each websocket independently.
- **Do not put Floonet services behind IP-reputation banning** (fail2ban and friends) without exempting known mixnet exits, or you will ban your own users in bulk.

## What actually protects the relay

1. **The kind whitelist.** Most abuse is simply not storable on a Floonet relay; disallowed kinds are dropped before any quota is touched.
2. **Per-connection event and subscription limits.** Both upstreams provide these; the shipped configs set sane values.
3. **Per-IP HTTP windows on the name authority.** Registration and lookup endpoints keep per-IP windows (reads generous, writes tight), fed by `X-Real-IP` from the proxy, with limits loose enough that a shared mixnet exit does not starve. The proxy forwarding `X-Real-IP` is load-bearing; without it every request appears to come from the proxy.
4. **NIP-98 replay protection.** Registration requests are single-use: the auth event's timestamp bound and replay window mean a captured request cannot be replayed to burn a name.
5. **Name-change cooldown.** A key that changes its name waits out a cooldown, which caps name-churn abuse at negligible cost to honest users.

## Tuning

The shipped defaults are deliberately generous on reads and conservative on writes. If you tighten them, watch for the mixnet signature in your logs first: many distinct pubkeys behind one IP is normal Floonet traffic, not an attack.
