# Name authority (floonet-rs)

> **Summary.** floonet-rs builds the name authority in as a module: the same endpoints, the same rules as the bundled strfry authority, served in-process from the relay binary, with its tables in the relay database via a normal migration.

## Shape

- **A module, not a sidecar.** `src/name_authority.rs` serves the NIP-05 and registration endpoints from the same binary and the same port as the relay's HTTP surface. One process to run, one unit to monitor. (Operators who prefer a separate authority process can run one as a sibling instead; both arrangements are supported.)
- **Storage via migration.** The authority's tables (`name_claims`, and `paid_pubkeys` for the paid gate) are added with a standard repo migration and a `DB_VERSION` bump, so they upgrade and back up with the rest of the relay database.
- **Same rules as everywhere.** Validation (lowercase `[a-z0-9._-]`, alphanumeric ends, cap 20), one active name per key via a partial unique index, reserved list, look-alike folding, NIP-98 registration with replay protection, cooldown. See [The name authority](../concepts/name-authority.md).

## Endpoints

```
GET  /.well-known/nostr.json?name=alice
POST /api/v1/register                 (NIP-98)
DELETE /api/v1/register/{name}        (NIP-98)
GET  /api/v1/by-pubkey/{pubkey}
GET  /api/v1/profile/{name}
GET  /api/v1/name/{name}
GET  /api/v1/health
```

Shapes in the [endpoints reference](../reference/endpoints.md).

## Paid names

With `FLOONET_PAY_MODE=name`, registration is gated on a confirmed GoblinPay payment of `FLOONET_NAME_PRICE_GRIN`, using the [GoblinPay processor](goblinpay.md) and the `invoice` tables. The quote endpoint responds with enough for a wallet to generate a pay page automatically.

## Config

```toml
[name_authority]
enabled = true
domain = "relay.yourdomain"
# reserved = ["admin", "root", ...]   # extends the built-in list
# cooldown_seconds = 86400
```
