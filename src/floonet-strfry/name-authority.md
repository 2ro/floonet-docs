# The bundled name authority

> **Summary.** floonet-strfry ships the name authority inside the package: one compose unit contains strfry, the authority, and the proxy. The authority is a lean port of goblin-nip05d, swept down to what strfry deployments actually need, keeping its own small SQLite database.

## Why bundled

The name authority is what makes a relay *useful* to a community: it is where `alice` comes from. Telling operators to "also go run this other service" is how services never get run. So floonet-strfry ships the authority in the box: the default compose brings it up alongside the relay, already wired to the same domain and proxy.

## Shape

- **Its own service, its own storage.** The authority is a separate small process with its own SQLite database. It is deliberately not bolted into strfry's LMDB; the relay stores events, the authority stores names, and neither can corrupt the other.
- **Swept lean.** The code is a port of the goblin-nip05d reference, reduced to what a strfry deployment needs. Everything not load-bearing was removed.
- **Consulted by the plugin.** The [write-policy plugin](write-policy.md) can consult the authority for policies that depend on name state.
- **Same rules as everywhere.** Validation, cap 20, one name per key, reserved list, look-alike folding, NIP-98 auth, replay protection, cooldown: see [The name authority](../concepts/name-authority.md).

## Co-located on the relay's domain

`FLOONET_AUTHORITY_COLOCATED` is on by default: the shipped `deploy/Caddyfile` routes `/.well-known/nostr.json` and `/api/*` to the authority and everything else to the relay, both on the single `FLOONET_DOMAIN` the compose stack brings up — so `name@FLOONET_DOMAIN` resolves with nothing to configure. Operators who split the relay and the authority across separate subdomains behind nginx (the `deploy/us-east/` pattern: relay on `relay.example`, the authority's own vhost on `nm.example`) can opt back into co-location by including the shipped `deploy/us-east/colocated-authority.conf` snippet in the relay vhost, ahead of the WebSocket catch-all. That snippet only proxies the exact-match NIP-05 read (`GET /.well-known/nostr.json`); registration and the rest of `/api/*` stay on the authority's own domain. See the README's "Co-locating names on the relay domain" section for the full nginx include.

## Endpoints

The authority serves the standard set under the relay's domain, via the shared proxy:

```
GET  /.well-known/nostr.json?name=alice
POST /api/v1/register                 (NIP-98)
DELETE /api/v1/register/{name}        (NIP-98)
GET  /api/v1/by-pubkey/{pubkey}
GET  /api/v1/profile/{name}
GET  /api/v1/name/{name}
GET  /api/v1/health
```

Full request and response shapes are in the [endpoints reference](../reference/endpoints.md).

## Paid names

When `FLOONET_PAY_MODE=name`, the authority requires a confirmed GoblinPay payment before `POST /api/v1/register` succeeds. See [Paid names via GoblinPay](paid-names.md).

## Backups

Back up one file: the authority's SQLite database. Every registered name on your domain lives there, and NIP-05 resolution for your users depends on it.
