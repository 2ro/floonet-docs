# The bundled name authority

> **Summary.** floonet-strfry ships the name authority inside the package: one compose unit contains strfry, the authority, and the proxy. The authority is the successor to goblin-nip05d (the older minimal standalone edition), keeping its own small SQLite database. The operator chooses in setup whether to run it alongside the relay, standalone, or not at all.

## Why bundled

The name authority is what makes a relay *useful* to a community: it is where `alice` comes from. Telling operators to "also go run this other service" is how services never get run. So floonet-strfry ships the authority in the box: the default compose brings it up alongside the relay, already wired to the same domain and proxy.

The standalone [goblin-nip05d](https://code.gri.mw/GRIN/goblin-nip05d) project is the older minimal, single-purpose edition of this same name service, now superseded by the bundled one here (which carries the ongoing work: paid names, name transfers, co-location on the relay's domain). Reach for the standalone repo only if you specifically want a name service with no relay.

## Running it, or not

The service is optional and the choice is made at setup:

- **Alongside the relay (default).** Leave the `authority` service in the Compose file; `docker compose up -d` brings up strfry, the authority, and the proxy together on one domain.
- **Standalone.** The `name-authority/` crate is a plain `cargo build` binary you can run on its own host, with its own SQLite and its own vhost.
- **Not at all.** Remove (or do not start) the `authority` service and the relay runs pure event ingest with no NIP-05 surface.

### First-run setup wizard

Run the authority binary by hand with nothing configured, on an interactive terminal, and it walks a short first-run wizard: it prompts for the names domain, the HTTP bind address, the data directory, the pay mode and price, and whether to enable name transfers, then writes a conventional env file (`/etc/floonet-authority.env` for a root install, else `./.env`) and starts. The wizard is skipped entirely when `FLOONET_DOMAIN` is already set, or when stdin/stdout is not a TTY, so the Docker Compose and systemd deployments (which inject the environment) never see it and stay headless.

## Shape

- **Its own service, its own storage.** The authority is a separate small process with its own SQLite database. It is deliberately not bolted into strfry's LMDB; the relay stores events, the authority stores names, and neither can corrupt the other.
- **The successor to goblin-nip05d.** It grew from the goblin-nip05d reference and is now where the name service is maintained.
- **Consulted by the plugin.** The [write-policy plugin](write-policy.md) can consult the authority for policies that depend on name state.
- **Same rules as everywhere.** Validation, cap 20, one name per key, reserved list, look-alike folding, NIP-98 auth, replay protection, cooldown: see [The name authority](../concepts/name-authority.md).

## Co-located on the relay's domain

`FLOONET_AUTHORITY_COLOCATED` is on by default: the shipped `deploy/Caddyfile` routes `/.well-known/nostr.json` and `/api/*` to the authority and everything else to the relay, both on the single `FLOONET_DOMAIN` the compose stack brings up, so `name@FLOONET_DOMAIN` resolves with nothing to configure. Operators who split the relay and the authority across separate subdomains behind nginx (the `deploy/us-east/` pattern: relay on `relay.example`, the authority's own vhost on `nm.example`) can opt back into co-location by including the shipped `deploy/us-east/colocated-authority.conf` snippet in the relay vhost, ahead of the WebSocket catch-all. That snippet only proxies the exact-match NIP-05 read (`GET /.well-known/nostr.json`); registration and the rest of `/api/*` stay on the authority's own domain. See the README's "Co-locating names on the relay domain" section for the full nginx include.

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

## Name transfers

The authority can optionally run a **non-custodial name marketplace**: one holder sells a name to another. It is **off by default** and mounts no routes unless the operator sets `FLOONET_TRANSFERS=true` (which also requires `FLOONET_GRIN_NODE_URL`, a read-only Grin node foreign API used to confirm payment). When on, four routes appear under `/api/v1/transfer/`:

```
POST   /api/v1/transfer/offer          (NIP-98)  seller lodges a signed kind-3402 offer
GET    /api/v1/transfer/offer/{id}               public: read an offer and its status
DELETE /api/v1/transfer/offer/{id}     (NIP-98)  seller revokes a live offer
POST   /api/v1/transfer/claim          (NIP-98)  buyer claims the name with a Grin payment proof
```

A transfer reassigns one name row from the seller's pubkey to the buyer's after the authority verifies a seller-signed offer and an on-chain Grin payment proof (default `FLOONET_TRANSFER_MIN_CONF=10` confirmations). It is **strictly non-custodial with zero GoblinPay involvement**: the authority holds no funds, it only checks crypto and a read-only node. Keys never move; only the name's owning pubkey changes. This is independent of the pay mode: paid names and transfers can each be on or off separately.

## Backups

Back up one file: the authority's SQLite database. Every registered name on your domain lives there, and NIP-05 resolution for your users depends on it.
