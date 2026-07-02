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
# The keystone: default deny. Only these kinds are accepted.
event_kind_allowlist = [0, 3, 5, 13, 1059, 10002, 10050, 27235]
# Large enough for gift-wrapped slatepacks. Do not shrink.
max_event_bytes = 131072
```

`event_kind_allowlist` exists upstream (`nostr-rs-relay/src/config.rs:54-79`); floonet-rs enforces it in the write path through the [admission module](admission.md).

### `[authorization]`: NIP-42 and whitelists

```toml
[authorization]
nip42_auth = false          # require AUTH before writes
nip42_dms = false           # require AUTH to read gift wraps addressed to you
# pubkey_whitelist = ["<hex>", "<hex>"]
```

Upstream parsed `pubkey_whitelist` but did not gate writes on it; floonet-rs enforces it in admission. See [Authentication](../concepts/auth.md).

### `[pay_to_relay]` + GoblinPay

```toml
[pay_to_relay]
enabled = false             # flip on for paid modes
processor = "GoblinPay"
```

With the environment keys `FLOONET_PAY_MODE`, `FLOONET_NAME_PRICE_GRIN`, `GOBLINPAY_URL`, and `GOBLINPAY_TOKEN` (same names as floonet-strfry; see the [config keys reference](../reference/config-keys.md)). The [GoblinPay processor](goblinpay.md) implements the upstream `PaymentProcessor` trait.

### `[name_authority]`

```toml
[name_authority]
enabled = true
domain = "relay.yourdomain"
```

Serves the NIP-05 endpoints in-process. See [Name authority](name-authority.md).

### `[exit]`: the co-located mixnet exit

```toml
[exit]
enabled = true
# binary = "/usr/local/bin/floonet-mixexit"      # the bundled exit binary
# data_dir = "/var/lib/floonet-rs/mixexit"       # persistent mixnet identity
# upstream = "relay.yourdomain:443"              # your PUBLIC TLS endpoint
```

When enabled, the relay runs the bundled `floonet-mixexit` beside itself: an ordinary unbonded mixnet client that forwards every accepted stream to **one fixed upstream** (your relay), never a caller-chosen target, so it is structurally not an open proxy. Point `upstream` at your public TLS endpoint so wallets get your real certificate through the mixnet; empty means this relay's local listener (no TLS). The exit's stable mixnet address is printed at startup and written to `<data_dir>/nym_address.txt`; publish it (the [relay pool](https://gist.github.com/2ro/79cd885540c88d074fe52f8388a3e5b4) `exit` field) and back the directory up, since losing it rotates the address. See [The mixnet and the scoped exit](../concepts/nym.md).

## Environment

Secrets stay out of `config.toml`: the GoblinPay token and any keys are provided via the systemd environment file (mounted 0400) or container env.
