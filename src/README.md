# Floonet

**Floonet is a network of Nostr relays for the [Grin](https://grin.mw) community. Anyone can run one, and anyone can run a name authority on one so people can claim (and optionally pay for) a name.**

A Floonet relay is an ordinary Nostr relay with strong opinions. It stores only the handful of event kinds the Grin ecosystem actually uses, it says nothing about payments in its public metadata, it welcomes connections arriving through the [Nym](https://nym.com) mixnet, and it ships hardened by default. Wallets like [Goblin](https://goblin.st) use Floonet relays to deliver gift-wrapped Grin payments and to resolve names like `alice`.

The flagship relay, **`relay.floonet.dev`**, runs floonet-strfry with the [co-located mixnet exit](concepts/nym.md) enabled and is the Goblin wallet's default money-path relay: wallets dial it straight over the mixnet, with no public DNS on the payment path. The same relay also hosts the [Magick Market](https://magick.market) marketplace, so it runs the shipped default whitelist unmodified — one relay, two applications.

## The two packages

Floonet ships as two relay packages. Both carry the same conventions; pick the one that fits how you like to operate.

| Package | Base | Shape |
| --- | --- | --- |
| **[floonet-strfry](floonet-strfry/deploy.md)** | [strfry](https://github.com/hoytech/strfry) (C++) | Stock strfry at a pinned ref plus a spec: a modular write-policy plugin, a bundled name authority, and a TLS proxy, deployed as one Docker Compose unit. |
| **[floonet-rs](floonet-rs/deploy.md)** | [nostr-rs-relay](https://github.com/scsibug/nostr-rs-relay) (Rust) | A single binary with an installer and a hardened systemd unit. Policy lives in a composable admission module; the name authority and the GoblinPay payment processor are built in. |

Both add the same five features, each configurable, optional, and modular:

1. **An event-kind whitelist** (the keystone: default deny, see below).
2. **Authentication**: NIP-42 plus pubkey whitelists.
3. **Paid access and paid names** via [GoblinPay](floonet-strfry/paid-names.md) (Grin).
4. **A name authority**: the NIP-05 service that maps names to keys.
5. **A co-located mixnet exit**: a scoped forwarder wallets dial straight over the mixnet, so reaching your relay needs no public DNS. See [The mixnet and the scoped exit](concepts/nym.md).

## The whitelist keystone

The single most important design decision in Floonet is **default deny**. A Floonet relay accepts *only* the event kinds it has been explicitly told to allow, and drops everything else. The core of the allowed set is exactly what a Grin payment wallet needs:

| Kind | What it is |
| --- | --- |
| `0` | Profile metadata |
| `3` | Contact list |
| `5` | Deletion request (NIP-09) |
| `13` | Seal (NIP-59) |
| `1059` | Gift wrap (NIP-59): the sealed envelope payments travel in |
| `10002` | Relay list (NIP-65) |
| `10050` | DM relay list (NIP-17) |
| `27235` | HTTP auth (NIP-98): used by the name authority |

The shipped default in both packages is this wallet core plus the [Magick Market](https://magick.market) marketplace kinds (listings, orders, receipts) and Nostr Connect login — 23 kinds in total, and exactly the list running in production on `relay.floonet.dev`; the [allowed kinds reference](reference/allowed-kinds.md) has the full table. Everything else, long-form content, zaps, bot spam, is rejected. This keeps a Floonet relay lean, cheap to run, and uninteresting to abuse. The list is one editable config value in both packages, so it can grow (or shrink to the wallet core) without code changes. See [The whitelist: default deny](concepts/whitelist.md).

## How to read these docs

1. **[Concepts](concepts/whitelist.md)**: the ideas every operator should know, whichever package they run.
2. **[floonet-strfry](floonet-strfry/deploy.md)** and **[floonet-rs](floonet-rs/deploy.md)**: deploy, configure, and extend each package.
3. **[Operate](operate/hardening.md)**: hardening, rate limits, and charging GRIN for relay resources.
4. **[Reference](reference/config-keys.md)**: config keys, endpoints, and the allowed-kinds table.

Where these docs cite code, they use `file:line` references into the package source or the pinned upstream so you can read along.
