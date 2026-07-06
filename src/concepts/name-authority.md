# The name authority

> **Summary.** A name authority maps human names to Nostr keys via NIP-05, so people pay `alice` instead of a 64-character key. Both Floonet packages **bundle** one: registration is authenticated with NIP-98, names follow strict validation rules, each key holds at most one name, and operators may charge GRIN for names. The bundled service is selected in the relay's setup and can run alongside the relay, as a standalone service, or not at all.

## Motivation

Keys are unusable as addresses for humans. NIP-05 solves this with a well-known JSON file: `https://example.org/.well-known/nostr.json?name=alice` returns alice's pubkey. A Floonet name authority is the small service that maintains that file, plus an API to claim and release names. Anyone can run one, on any Floonet relay, under any domain.

## Enabling or disabling the bundled name service

The name service ships in the box; whether it runs is the operator's choice, made once at setup. There are three arrangements:

- **Alongside the relay (the default).** floonet-strfry brings the authority up as its own Compose `authority` service, wired to the relay's domain; floonet-rs serves it in-process from the relay binary when `[name_authority] enabled = true`.
- **Standalone.** Run the authority on its own: floonet-strfry's `name-authority/` crate builds and runs as an independent binary (with its own SQLite), and floonet-rs operators who want a separate process can run one as a sibling. This is also what the older goblin-nip05d edition was (see below).
- **Not at all.** A relay with no name service. In floonet-rs, leave `[name_authority] enabled = false` (the shipped default) and the relay runs pure event ingest with no NIP-05 surface. In floonet-strfry, drop the `authority` service from the Compose stack.

**First-run wizard.** floonet-strfry's authority binary has an interactive first-run setup: run it by hand with nothing configured, on a terminal, and it prompts for the essentials (names domain, HTTP bind address, data directory, pay mode and price, and whether to enable name transfers), writes a conventional env file, and starts up. It is skipped entirely when `FLOONET_DOMAIN` is already set or when stdin is not a TTY, so Docker Compose and systemd deploys stay fully headless. See [The bundled name authority](../floonet-strfry/name-authority.md).

## The rules

The rules originate in the standalone goblin-nip05d reference (the older minimal, single-purpose edition, now superseded by this bundled service) and are the same in both packages:

- **Validation.** Names are lowercase `[a-z0-9._-]`, must start and end alphanumeric, and are capped at **20 characters**.
- **One active name per key.** Enforced with a partial unique index in the database. Claiming a new name releases the old one.
- **Reserved names.** A reserved list plus domain-label reservation plus look-alike folding (so `a1ice` cannot impersonate `alice`).
- **Authenticated registration.** Claiming or releasing a name requires a [NIP-98](https://nips.nostr.com/98) HTTP auth event (kind `27235` with `u`, `method`, and `payload` tags), with a timestamp bound and a replay window, so a captured request cannot be replayed.
- **Cooldown.** A key that changes its name must wait out a cooldown before changing it again.

## Same subdomain as the relay

A name authority is only useful if `name@relay.yourdomain` actually resolves, so both packages serve it under the **relay's own subdomain** by default rather than a separate hostname. floonet-rs does this structurally: the authority is an in-process module answering on the same binary and port as the relay's HTTP surface, so there is nothing to configure. floonet-strfry's authority is its own process, but the shipped Docker Compose/Caddy stack routes NIP-05 and API paths to it on the same `FLOONET_DOMAIN` as the relay by default; operators splitting the two across separate subdomains can opt back into co-location with a small nginx snippet. See the package pages: [floonet-strfry](../floonet-strfry/name-authority.md), [floonet-rs](../floonet-rs/name-authority.md).

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

## Name transfers (the name marketplace)

The bundled strfry authority can also let one holder sell a name to another: a seller lodges a signed offer, a buyer pays in GRIN and claims it, and the name row is reassigned from the seller's key to the buyer's. This is **off by default** (`FLOONET_TRANSFERS`, disabled unless the operator turns it on) and, when on, is **strictly non-custodial**: the authority holds no funds and has zero GoblinPay involvement. It only verifies a seller-signed offer plus an on-chain Grin payment proof through a read-only Grin node foreign API, then moves the name. Keys never move; only which pubkey owns the name changes. See [The bundled name authority](../floonet-strfry/name-authority.md#name-transfers).

## A note for wallet users

One wallet can hold multiple Nostr identities (npubs). If you pay for a name and want to keep it, load the same wallet seed in Goblin and switch to (or add) the npub that owns the name; different npubs and identities share one wallet.

## References

- NIP-05: <https://nips.nostr.com/5>.
- Package specifics: [bundled authority (strfry)](../floonet-strfry/name-authority.md), [in-process module (rs)](../floonet-rs/name-authority.md).
