# Configuration (floonet-rs)

> **Summary.** One `config.toml`, inherited from nostr-rs-relay and extended with the Floonet sections. The four Floonet features (whitelist, auth, paid access, name authority) are each a small block of keys.

## The blocks that matter

### `[info]`: neutral metadata

```toml
[info]
name = "floonet-rs-relay"
description = "A Floonet relay for the Grin community Nostr network."
```

Keep it [payment-neutral](../concepts/nip11.md). Operators may customize; the defaults carry zero payment language.

### `[limits]`: the whitelist and event size

```toml
[limits]
# The keystone: default deny. Only these kinds are accepted. The shipped
# list is the Goblin wallet + Magick Market union; see the allowed kinds
# reference for what each one is.
event_kind_allowlist = [
    0, 1, 3, 5, 7, 13, 14, 16, 17, 1059, 1111, 10000, 10002, 10050, 24133,
    27235, 30000, 30003, 30078, 30402, 30405, 30406, 31990,
]
# Default 262144: large enough for gift-wrapped slatepacks. Do not shrink.
#max_event_bytes = 262144
```

`event_kind_allowlist` exists upstream (`nostr-rs-relay/src/config.rs:54-79`); floonet-rs enforces it in the write path through the [admission module](admission.md).

### `[authorization]`: NIP-42 and whitelists

```toml
[authorization]
nip42_auth = false            # enable NIP-42 (the relay sends AUTH challenges)
nip42_dms = false             # send gift wraps only to their authenticated recipients
# require_auth_to_write = false   # with nip42_auth: only AUTHed clients may publish
# pubkey_whitelist = ["<hex>", "<hex>"]
```

Upstream parsed `pubkey_whitelist` but did not gate writes on it; floonet-rs enforces it in admission. See [Authentication](../concepts/auth.md).

### `[goblinpay]`: paid names and paid writes

```toml
[goblinpay]
pay_mode = "off"              # off | name | write
#url = "https://pay.example.com"
#api_token = ""               # prefer the FLOONET_GOBLINPAY_TOKEN env var
#name_price_grin = 1.0
#admission_price_grin = 1.0
```

Every key is also readable from the environment (`FLOONET_PAY_MODE`, `FLOONET_GOBLINPAY_URL`, `FLOONET_GOBLINPAY_TOKEN`, `FLOONET_NAME_PRICE_GRIN`; see the [config keys reference](../reference/config-keys.md)). Setting `pay_mode = "write"` configures upstream's `[pay_to_relay]` section for GoblinPay automatically — you normally never edit that section yourself. The [GoblinPay processor](goblinpay.md) implements the upstream `PaymentProcessor` trait.

### `[name_authority]`

```toml
[name_authority]
enabled = true                          # off in the shipped config
domain = "relay.yourdomain"             # the @domain names live under
base_url = "https://relay.yourdomain"   # LOAD-BEARING: NIP-98 auth is verified against it
```

Serves the NIP-05 endpoints in-process. Name length, change cooldown, rate limits, and a reserved-names file are further keys in the same section. See [Name authority](name-authority.md).

### `[onion]`: the co-located Tor onion service

```toml
[onion]
enabled = true
# hidden_service_dir = "/var/lib/tor/floonet-relay"   # maps to torrc HiddenServiceDir: persistent onion key, holds `hostname`
# upstream = "127.0.0.1:443"                           # maps to the torrc HiddenServicePort target: your relay's TLS endpoint
```

When enabled, the package runs a co-located **Tor onion service** in front of the relay's websocket port — plain, mature system Tor (C-Tor) doing the hosting half, not a bundled custom binary. The real configuration is the `torrc` it brings up:

```
HiddenServiceDir /var/lib/tor/floonet-relay/
HiddenServicePort 443 127.0.0.1:443
```

`HiddenServicePort` forwards to your relay's TLS endpoint (for example `127.0.0.1:443`) so wallets get your real certificate through the onion; the hostname-validated TLS handshake stays end to end. Enable **Vanguards** on the service side. The relay's stable `.onion` address is the `hostname` file inside `HiddenServiceDir`; publish it (the [relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4) `onion` field) and back the directory up, since losing it rotates the `.onion`. See [Tor and the relay's onion service](../concepts/nym.md).

## Environment

Secrets stay out of `config.toml`: the GoblinPay token and any keys are provided via the systemd environment file (mounted 0400) or container env.
