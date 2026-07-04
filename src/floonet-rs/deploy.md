# Deploy floonet-rs

> **Summary.** floonet-rs is a fork of [nostr-rs-relay](https://github.com/scsibug/nostr-rs-relay): one Rust binary containing the relay, the admission policies, the name authority, and the GoblinPay processor. Three deploy paths, easiest first.

## 1. Installer + systemd (recommended)

The repo ships an installer that drops the binary, the default config, and a hardened systemd unit. Build once, then let it lay everything out:

```bash
git clone https://github.com/2ro/floonet-rs.git
cd floonet-rs
cargo build --release
sudo sh deploy/install.sh
sudo systemctl enable --now floonet-rs
```

The installer is idempotent: re-running it upgrades the binary and unit but never overwrites an existing `/etc/floonet-rs/config.toml`. Run from an unpacked release archive it needs no toolchain at all; the same script finds the prebuilt binary next to itself.

The unit ships with the full sandbox set (`DynamicUser`, `ProtectSystem=strict`, `NoNewPrivileges`, and friends; see [Hardening](../operate/hardening.md)). Edit `/etc/floonet-rs/config.toml` and the environment file, then restart.

Verify:

```bash
curl -H 'Accept: application/nostr+json' https://relay.yourdomain/   # NIP-11
curl https://relay.yourdomain/api/v1/health                          # name authority
```

## 2. Docker Compose

The repo ships a compose file with the relay and a TLS proxy, mirroring the floonet-strfry unit:

```bash
git clone https://github.com/2ro/floonet-rs.git
cd floonet-rs
cp .env.example .env      # edit: domain, contact, prices if any
docker compose up -d
```

Containers are non-root with a read-only filesystem except the data volume.

## 3. From source

```bash
git clone https://github.com/2ro/floonet-rs.git
cd floonet-rs
cargo build --release
./target/release/floonet-rs --config config.toml
```

Front it with a TLS reverse proxy that forwards `X-Real-IP` (load-bearing for rate limiting), then follow [Configuration](config.md).

## After deploying

- Confirm the whitelist: publish an allowed kind (persists), then a kind `1` note (dropped). The primary acceptance test.
- Fetch the NIP-11 document and confirm it reads as `floonet-rs-relay` with a neutral description and no payment wording.
- Add the relay to your wallet and complete a payment end to end.
- Confirm nothing in front of the relay blocks Tor: wallets reach it [over Tor](../concepts/tor.md), dialing its clearnet host through a Tor exit.
