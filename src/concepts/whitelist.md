# The whitelist: default deny

> **Summary.** A Floonet relay is default-deny: it accepts only the event kinds on its allow-list and drops everything else. The list is one editable config value in both packages, enforced fail-closed in the write path.

## Motivation

A general-purpose Nostr relay stores whatever anyone throws at it: notes, reactions, media metadata, bot spam. A payment relay does not need any of that, and storing it makes the relay bigger, slower, and a more attractive target. Floonet inverts the default: **nothing is accepted unless it is explicitly allowed**. Allow only what is needed now; expand later by editing config, not code.

## The allowed set

The core of the list matches what the Goblin wallet actually publishes and reads (the canonical list is whatever the wallet uses, in `goblin/src/nostr/`):

| Kind | NIP | Why a Floonet relay carries it |
| --- | --- | --- |
| `0` | 01 | Profiles: display names and avatars for contacts |
| `3` | 02 | Contact lists |
| `5` | 09 | Deletion requests, so users can retract events |
| `13` | 59 | Seals: the inner layer of a gift wrap |
| `1059` | 59 | Gift wraps: the opaque envelopes everything private travels in |
| `10002` | 65 | Relay lists: where a user can be found |
| `10050` | 17 | DM relay lists: where to deliver private messages |
| `27235` | 98 | HTTP auth events, used to register names with the name authority |

The shipped default in both packages is this wallet core **plus** the [Magick Market](https://magick.market) marketplace set (`1`, `7`, `14`, `16`, `17`, `1111`, `10000`, `30000`, `30003`, `30078`, `30402`, `30405`, `30406`, `31990`) and `24133` (Nostr Connect wallet login) — 23 kinds in total. See the [allowed kinds reference](../reference/allowed-kinds.md) for the full table with the reasoning per kind.

## The list in production: `relay.floonet.dev`

The flagship relay is a live example of exactly this policy. It runs floonet-strfry with the shipped default list — no override — and serves two applications at once: the Goblin wallet (private payments as gift wraps) and the Magick Market marketplace (listings, orders, receipts). Zap receipts (`9735`) are deliberately rejected: Lightning is dead in this GRIN-only ecosystem. Seals (`13`) are on the list for completeness but in practice only ever travel *inside* `1059` gift wraps.

## How each package enforces it

- **floonet-strfry** keeps policy in strfry's [write-policy plugin](../floonet-strfry/write-policy.md), strfry's intended extension point (`strfry.conf` key `relay.writePolicy.plugin`; see `strfry/docs/plugins.md` and `src/PluginEventSifter.h` upstream). The plugin checks `kind` against the allow-list first and rejects everything else, fail-closed: if the plugin cannot parse the event or reach its config, the answer is reject. The read side can additionally set `filterValidation.allowedKinds` so disallowed kinds cannot even be subscribed to.
- **floonet-rs** uses the upstream `event_kind_allowlist` limit (upstream `nostr-rs-relay/src/config.rs:54-79`, `config.toml:151-159`) and enforces it in the write path inside the [admission module](../floonet-rs/admission.md), before the event reaches storage. Wrapping the check in the admission layer means it composes cleanly with auth and paid checks.

## Behavior on rejection

Disallowed kinds are **rejected fail-closed**: the plugin answers `reject` with a terse `blocked: event kind not accepted by this relay`, and any malformed or unparseable input is rejected too, never accepted. A legitimate wallet never publishes disallowed kinds in the first place.

## Growing the list

The whitelist is a single config value: `FLOONET_ALLOWED_KINDS`, a comma-separated list of integers in the plugin environment (strfry), or `event_kind_allowlist` in `config.toml` (rs). If you run a relay for a community that also wants, say, public chat, add the kind and restart (floonet-strfry even reloads the plugin on file change, no restart needed). One rule for operators upgrading an existing relay: **never narrow the list below what live wallets already depend on**.

## References

- Enforcement: [The write-policy plugin](../floonet-strfry/write-policy.md), [The admission module](../floonet-rs/admission.md).
- The full table: [Allowed kinds](../reference/allowed-kinds.md).
- Kind and NIP details: <https://nostrbook.dev/>.
