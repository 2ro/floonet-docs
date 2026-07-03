# Config keys

The keys an operator actually touches, across both packages. Package-specific pages: [floonet-strfry configuration](../floonet-strfry/config.md), [floonet-rs configuration](../floonet-rs/config.md).

## floonet-strfry policy environment

Read by the write-policy plugin and the bundled name authority (in compose, the `.env` file).

| Key | Default | Meaning |
| --- | --- | --- |
| `FLOONET_ALLOWED_KINDS` | the [Goblin + Magick Market set](allowed-kinds.md) (23 kinds) | The [whitelist](../concepts/whitelist.md). Default deny; everything not listed is dropped. `.env.example` pins the wallet-only core. |
| `FLOONET_REQUIRE_AUTH` | `false` | Reject writes unless the connection completed NIP-42 AUTH. |
| `FLOONET_PAY_MODE` | `off` | `off`, `name` (pay to claim a name), or `write` (pay to publish). |
| `FLOONET_NAME_PRICE_GRIN` | unset | Price of a name in GRIN. Required when `FLOONET_PAY_MODE=name`. |
| `GOBLINPAY_URL` | unset | The operator's GoblinPay server. Required for any paid mode. |
| `GOBLINPAY_TOKEN` | unset | GoblinPay API token. Secret: environment or 0400 file only. |
| `FLOONET_PAID_CACHE_SECS` | `60` | TTL for the plugin's cached paid-status lookups. |

## floonet-strfry (`strfry.conf`)

| Key | Floonet default | Meaning |
| --- | --- | --- |
| `relay.info.name` | `Floonet Relay` | NIP-11 name, [payment-neutral](../concepts/nip11.md). |
| `relay.info.description` | `A strfry Floonet relay for the Grin community Nostr network.` | NIP-11 description, same rule. |
| `relay.writePolicy.plugin` | plugin path | The [write-policy plugin](../floonet-strfry/write-policy.md). |
| `relay.auth.enabled` | `false` | NIP-42 authentication. |
| `events.maxEventSize` | shipped default | Keep large enough for gift-wrapped slatepacks. |

The read side may also set `filterValidation.allowedKinds` to mirror the whitelist on subscriptions.

Two compose-level `.env` keys control the [co-located mixnet exit](../concepts/nym.md):

| Key | Default | Meaning |
| --- | --- | --- |
| `COMPOSE_PROFILES` | unset | `exit` runs the bundled `mixexit` service beside the relay. |
| `FLOONET_EXIT_UPSTREAM` | `caddy:443` | Where the exit pipes streams; defaults to the stack's own TLS front. |

## floonet-rs (`config.toml`)

| Key | Floonet default | Meaning |
| --- | --- | --- |
| `info.name` | `floonet-rs-relay` | NIP-11 name, payment-neutral. |
| `info.description` | neutral Floonet wording | NIP-11 description, same rule. |
| `limits.event_kind_allowlist` | the same [23-kind set](allowed-kinds.md) | The whitelist, enforced in [admission](../floonet-rs/admission.md). |
| `limits.max_event_bytes` | shipped default | Keep large enough for gift-wrapped slatepacks. |
| `authorization.nip42_auth` | `false` | Require AUTH before writes. |
| `authorization.nip42_dms` | `false` | Require AUTH to read your gift wraps. |
| `authorization.pubkey_whitelist` | unset | Restrict writes to these pubkeys. |
| `goblinpay.pay_mode` | `off` | `off`, `name`, or `write`; drives the [GoblinPay processor](../floonet-rs/goblinpay.md). Env alias `FLOONET_PAY_MODE`. |
| `goblinpay.url` | unset | Your GoblinPay server. Env alias `FLOONET_GOBLINPAY_URL`. |
| `goblinpay.api_token` | unset | GoblinPay API token; prefer the `FLOONET_GOBLINPAY_TOKEN` env var over the file. |
| `goblinpay.name_price_grin` | `1.0` | Price of a name in GRIN when `pay_mode = "name"`. Env alias `FLOONET_NAME_PRICE_GRIN`. |
| `goblinpay.admission_price_grin` | `1.0` | Price of write admission in GRIN when `pay_mode = "write"`. |
| `name_authority.enabled` | `false` | Serve the [name authority](../floonet-rs/name-authority.md) in-process. |
| `name_authority.domain` | operator's domain | The NIP-05 domain names resolve under. |
| `exit.enabled` | `false` | Run the bundled [co-located mixnet exit](../floonet-rs/config.md#exit-the-co-located-mixnet-exit). |
| `exit.binary` | `/usr/local/bin/floonet-mixexit` | Path to the bundled exit binary. |
| `exit.data_dir` | `/var/lib/floonet-rs/mixexit` | Persistent mixnet identity; holds `nym_address.txt`. Back it up. |
| `exit.upstream` | unset (local listener) | Where the exit pipes streams; point it at your public TLS endpoint. |
