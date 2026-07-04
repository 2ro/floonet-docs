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

### Reaching the relay over Tor

Nothing to deploy. Wallets reach the relay [over Tor](../concepts/tor.md) by dialing its ordinary clearnet host through a Tor exit, so the compose stack needs no transport component of its own; the relay is simply a normal public endpoint behind Caddy's TLS. The only requirement is that nothing in front of the relay blocks Tor exit IPs. See [Tor: how wallets reach a relay](../concepts/tor.md).

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
