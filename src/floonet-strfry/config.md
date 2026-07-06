# Configuration (floonet-strfry)

> **Summary.** Two files matter: `strfry.conf` (the relay itself) and the plugin/authority environment (the Floonet policy). The compose deployment reduces both to one `.env`.

## strfry.conf

The Floonet spec ships a `strfry.conf` with these keys set; everything else is stock strfry.

| Key | Floonet default | Meaning |
| --- | --- | --- |
| `relay.info.name` | `Floonet Relay` | NIP-11 name; keep it [payment-neutral](../concepts/nip11.md) |
| `relay.info.description` | `A strfry Floonet relay for the Grin community Nostr network.` | NIP-11 description; same rule |
| `relay.writePolicy.plugin` | path to the Floonet plugin | The policy engine; see [The write-policy plugin](write-policy.md) |
| `relay.auth.enabled` | `false` | NIP-42; enable for auth-gated modes |
| `events.maxEventSize` | large enough for gift-wrapped slatepacks | Do not shrink; see [Gift wraps](../concepts/gift-wraps.md) |

## Floonet policy environment

The plugin and the bundled name authority read one shared environment (in compose, the `.env` file):

| Key | Default | Meaning |
| --- | --- | --- |
| `FLOONET_ALLOWED_KINDS` | the [Goblin + Magick Market set](../reference/allowed-kinds.md) (24 kinds) | The [whitelist](../concepts/whitelist.md); default deny. `.env.example` pins the wallet-only core (`0,3,5,13,1059,10002,10050,27235`) as a conservative starting point |
| `FLOONET_REQUIRE_AUTH` | `false` | Reject events unless the connection completed NIP-42 AUTH (also enable `relay.auth` in `strfry.conf`) |
| `FLOONET_PAY_MODE` | `off` | `off`, `name` (pay to claim a name), or `write` (pay to write) |
| `FLOONET_NAME_PRICE_GRIN` | unset | Price of a name in GRIN when `FLOONET_PAY_MODE=name` |
| `FLOONET_PAID_CACHE_SECS` | `60` | TTL for the plugin's cached paid-status lookups |
| `GOBLINPAY_URL` | unset | Your GoblinPay server, required for any paid mode |
| `GOBLINPAY_TOKEN` | unset | GoblinPay API token; keep it out of the repo, mount it 0400 |

## Reaching the relay over Tor

There is nothing to configure. Wallets reach the relay [over Tor](../concepts/tor.md) by dialing the stack's ordinary clearnet host through a Tor exit; the Caddy TLS front and the relay behind it stay a normal public endpoint. Just make sure nothing upstream blocks Tor exit IPs. See [Tor: how wallets reach a relay](../concepts/tor.md).

The full key table for both packages lives in the [config keys reference](../reference/config-keys.md).

## The one-file experience

In the compose deployment, `.env.example` documents every key above with comments. Turning on paid names is three edits:

```bash
FLOONET_PAY_MODE=name
FLOONET_NAME_PRICE_GRIN=5
GOBLINPAY_URL=https://pay.yourdomain
```

See [Charge GRIN for your relay](../operate/charge-grin.md) for the walkthrough.
