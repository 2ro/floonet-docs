# Config keys

The keys an operator actually touches, across both packages. Package-specific pages: [floonet-strfry configuration](../floonet-strfry/config.md), [floonet-rs configuration](../floonet-rs/config.md).

## Shared Floonet keys (environment)

| Key | Default | Meaning |
| --- | --- | --- |
| `ALLOWED_KINDS` | `0,3,5,13,1059,10002,10050,27235` | The [whitelist](../concepts/whitelist.md). Default deny; everything not listed is dropped. |
| `FLOONET_PAY_MODE` | `off` | `off`, `name` (pay to claim a name), or `write` (pay to publish). |
| `FLOONET_NAME_PRICE_GRIN` | unset | Price of a name in GRIN. Required when `FLOONET_PAY_MODE=name`. |
| `GOBLINPAY_URL` | unset | The operator's GoblinPay server. Required for any paid mode. |
| `GOBLINPAY_TOKEN` | unset | GoblinPay API token. Secret: environment or 0400 file only. |

## floonet-strfry (`strfry.conf`)

| Key | Floonet default | Meaning |
| --- | --- | --- |
| `relay.info.name` | `Floonet Relay` | NIP-11 name, [payment-neutral](../concepts/nip11.md). |
| `relay.info.description` | `A strfry Floonet relay.` | NIP-11 description, same rule. |
| `relay.writePolicy.plugin` | plugin path | The [write-policy plugin](../floonet-strfry/write-policy.md). |
| `relay.auth.enabled` | `false` | NIP-42 authentication. |
| `events.maxEventSize` | shipped default | Keep large enough for gift-wrapped slatepacks. |

The read side may also set `filterValidation.allowedKinds` to mirror the whitelist on subscriptions.

## floonet-rs (`config.toml`)

| Key | Floonet default | Meaning |
| --- | --- | --- |
| `info.name` | `floonet-rs-relay` | NIP-11 name, payment-neutral. |
| `info.description` | neutral Floonet wording | NIP-11 description, same rule. |
| `limits.event_kind_allowlist` | `[0, 3, 5, 13, 1059, 10002, 10050, 27235]` | The whitelist, enforced in [admission](../floonet-rs/admission.md). |
| `limits.max_event_bytes` | shipped default | Keep large enough for gift-wrapped slatepacks. |
| `authorization.nip42_auth` | `false` | Require AUTH before writes. |
| `authorization.nip42_dms` | `false` | Require AUTH to read your gift wraps. |
| `authorization.pubkey_whitelist` | unset | Restrict writes to these pubkeys. |
| `pay_to_relay.enabled` | `false` | Master switch for the [GoblinPay processor](../floonet-rs/goblinpay.md). |
| `pay_to_relay.processor` | `GoblinPay` | Payment backend selection. |
| `name_authority.enabled` | `true` | Serve the [name authority](../floonet-rs/name-authority.md) in-process. |
| `name_authority.domain` | operator's domain | The NIP-05 domain names resolve under. |
