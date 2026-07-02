# Deploy floonet-strfry

> **Summary.** floonet-strfry is stock [strfry](https://github.com/hoytech/strfry) at a pinned upstream ref plus a Floonet spec laid on top: the write-policy plugin, the bundled name authority, and a TLS proxy, shipped as one unit. Three deploy paths, easiest first.

## 1. Docker Compose (recommended)

One command brings up the whole unit: relay, name authority, and reverse proxy with automatic TLS.

```bash
git clone https://github.com/2ro/floonet-strfry.git
cd floonet-strfry
cp .env.example .env      # edit: your domain, contact, and (optionally) prices
docker compose up -d
```

The `.env` file is the only thing you edit. Containers run non-root with a read-only filesystem except the data volume, and the upstream strfry ref is pinned so you always build a known tree.

Verify it is up:

```bash
curl -H 'Accept: application/nostr+json' https://relay.yourdomain/   # NIP-11
curl https://relay.yourdomain/api/v1/health                          # name authority
```

### The mixnet exit

The compose unit also carries the [co-located scoped mixnet exit](../concepts/nym.md), off by default behind the `exit` compose profile. To turn it on:

```bash
# in .env
COMPOSE_PROFILES=exit
# optional: where the exit pipes streams. Defaults to this stack's own
# TLS front (caddy:443), so wallets get your real certificate.
#FLOONET_EXIT_UPSTREAM=caddy:443
```

then `docker compose up -d` again. The exit's stable mixnet address is printed at startup (`docker compose logs mixexit`) and written to `nym_address.txt` in the `mixexit-data` volume. Publish that address (the [relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4) `exit` field) so wallets can dial your relay straight over the mixnet; they fall back to the public mixnet route automatically whenever the exit is down. Back the volume up: losing it rotates the address.

## 2. apply-spec (build strfry yourself, add the Floonet layer)

If you already run strfry or want it on bare metal, `deploy/strfry/apply-spec.sh` builds stock strfry at the pinned ref and lays the Floonet conf, plugin, and name authority on top:

```bash
./deploy/strfry/apply-spec.sh
```

strfry core stays stock; the spec only adds config and the plugin. This is the "stock + spec" pattern: upgrades track upstream strfry directly.

## 3. From source

Build strfry per its upstream docs (`make setup-golpe && make`), then:

1. Install `strfry.conf` from `deploy/strfry/strfry.conf` (see [Configuration](config.md)).
2. Install the write-policy plugin and point `relay.writePolicy.plugin` at it.
3. Run the bundled name authority (its own small service with its own SQLite; see [The bundled name authority](name-authority.md)).
4. Front both with a TLS reverse proxy that forwards `X-Real-IP` (see [Hardening](../operate/hardening.md)).

## After deploying

- Confirm the whitelist: publish an allowed kind (it persists) and a kind `1` note (it is dropped). This is the primary acceptance test.
- Check the NIP-11 document reads as a neutral Floonet relay with no payment wording.
- Add the relay to your wallet and send yourself a payment end to end.
