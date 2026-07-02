# The name authority

> **Summary.** A name authority maps human names to Nostr keys via NIP-05, so people pay `alice` instead of a 64-character key. Both Floonet packages ship one: registration is authenticated with NIP-98, names follow strict validation rules, each key holds at most one name, and operators may charge GRIN for names.

## Motivation

Keys are unusable as addresses for humans. NIP-05 solves this with a well-known JSON file: `https://example.org/.well-known/nostr.json?name=alice` returns alice's pubkey. A Floonet name authority is the small service that maintains that file, plus an API to claim and release names. Anyone can run one, on any Floonet relay, under any domain.

## The rules

The rules are inherited from the goblin-nip05d reference implementation and are the same in both packages:

- **Validation.** Names are lowercase `[a-z0-9._-]`, must start and end alphanumeric, and are capped at **20 characters**.
- **One active name per key.** Enforced with a partial unique index in the database. Claiming a new name releases the old one.
- **Reserved names.** A reserved list plus domain-label reservation plus look-alike folding (so `a1ice` cannot impersonate `alice`).
- **Authenticated registration.** Claiming or releasing a name requires a [NIP-98](https://nips.nostr.com/98) HTTP auth event (kind `27235` with `u`, `method`, and `payload` tags), with a timestamp bound and a replay window, so a captured request cannot be replayed.
- **Cooldown.** A key that changes its name must wait out a cooldown before changing it again.

## The endpoints

| Endpoint | Purpose |
| --- | --- |
| `GET /.well-known/nostr.json?name=` | NIP-05 resolution |
| `POST /api/v1/register` | Claim a name (NIP-98 auth) |
| `DELETE /api/v1/register/{name}` | Release a name (NIP-98 auth) |
| `GET /api/v1/by-pubkey/{pubkey}` | Reverse lookup: which name does this key hold |
| `GET /api/v1/profile/{name}` | Profile data for a name |
| `GET /api/v1/name/{name}` | Availability check |
| `GET /api/v1/health` | Health probe |

See the [endpoints reference](../reference/endpoints.md).

## Free or paid

By default names are free. An operator can instead require a confirmed GoblinPay payment of `FLOONET_NAME_PRICE_GRIN` before a registration succeeds. The price is plain config; see [Charge GRIN for your relay](../operate/charge-grin.md).

## A note for wallet users

One wallet can hold multiple Nostr identities (npubs). If you pay for a name and want to keep it, load the same wallet seed in Goblin and switch to (or add) the npub that owns the name; different npubs and identities share one wallet.

## References

- NIP-05: <https://nips.nostr.com/5>.
- Package specifics: [bundled authority (strfry)](../floonet-strfry/name-authority.md), [in-process module (rs)](../floonet-rs/name-authority.md).
