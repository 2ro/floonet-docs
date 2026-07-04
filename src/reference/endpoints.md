# Endpoints

## The relay

| Endpoint | Protocol | Purpose |
| --- | --- | --- |
| `wss://relay.yourdomain/` | Nostr over websocket | The relay itself |
| `https://relay.yourdomain/` with `Accept: application/nostr+json` | HTTP | The [NIP-11 information document](../concepts/nip11.md) |
| `https://relay.yourdomain/` (browser) | HTTP | A neutral Floonet landing page with the Floonet logo |
| The relay's `.onion` address (the onion service `hostname` file) | Nostr over websocket, over Tor | The same relay, dialed over Tor via the [co-located onion service](../concepts/nym.md) when the onion toggle is on |

## The name authority

All endpoints are served under the relay's own domain by default (co-located, see [The name authority](../concepts/name-authority.md#same-subdomain-as-the-relay)) — floonet-rs always, since it's the same listener; floonet-strfry via its Compose/Caddy stack, or via an nginx opt-in for split relay/authority subdomains, in which case only the `GET /.well-known/nostr.json` read co-locates and the rest of `/api/*` stays on the authority's own domain. `(NIP-98)` means the request must carry a [NIP-98](https://nips.nostr.com/98) `Authorization` event (kind `27235`, `u` + `method` + `payload` tags, bounded timestamp, replay-protected).

| Endpoint | Auth | Purpose |
| --- | --- | --- |
| `GET /.well-known/nostr.json?name={name}` | none | NIP-05 resolution: returns `{"names": {"{name}": "<pubkey>"}}` |
| `POST /api/v1/register` | NIP-98 | Claim a name for the signing key. Refused until payment confirms when `FLOONET_PAY_MODE=name`; the refusal response carries the quote and invoice so a wallet can generate the pay page. |
| `DELETE /api/v1/register/{name}` | NIP-98 | Release a name (only by its owner) |
| `GET /api/v1/by-pubkey/{pubkey}` | none | Reverse lookup: the name currently held by a key |
| `GET /api/v1/profile/{name}` | none | Profile data for a name |
| `GET /api/v1/name/{name}` | none | Availability: is this name free, reserved, or taken |
| `GET /api/v1/health` | none | Health probe for monitoring |

## Example

```bash
$ curl 'https://relay.yourdomain/.well-known/nostr.json?name=alice'
{
  "names": {
    "alice": "7d2f19c0...a4c41a"
  }
}
```

## Rules enforced behind the endpoints

Name validation (lowercase `[a-z0-9._-]`, alphanumeric ends, cap 20), one active name per key, reserved list and look-alike folding, replay windows, and the name-change cooldown. Details: [The name authority](../concepts/name-authority.md).
