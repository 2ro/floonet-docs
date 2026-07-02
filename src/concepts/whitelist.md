# The whitelist: default deny

> **Summary.** A Floonet relay is default-deny: it accepts only the event kinds on its allow-list and drops everything else. The list is one editable config value in both packages, enforced fail-closed in the write path.

## Motivation

A general-purpose Nostr relay stores whatever anyone throws at it: notes, reactions, media metadata, bot spam. A payment relay does not need any of that, and storing it makes the relay bigger, slower, and a more attractive target. Floonet inverts the default: **nothing is accepted unless it is explicitly allowed**. Allow only what is needed now; expand later by editing config, not code.

## The allowed set

The initial list matches what the Goblin wallet actually publishes and reads (the canonical list is whatever the wallet uses, in `goblin/src/nostr/`):

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

See the [allowed kinds reference](../reference/allowed-kinds.md) for the full table.

## How each package enforces it

- **floonet-strfry** keeps policy in strfry's [write-policy plugin](../floonet-strfry/write-policy.md), strfry's intended extension point (`strfry.conf` key `relay.writePolicy.plugin`; see `strfry/docs/plugins.md` and `src/PluginEventSifter.h` upstream). The plugin checks `kind` against the allow-list first and rejects everything else, fail-closed: if the plugin cannot parse the event or reach its config, the answer is reject. The read side can additionally set `filterValidation.allowedKinds` so disallowed kinds cannot even be subscribed to.
- **floonet-rs** uses the upstream `event_kind_allowlist` limit (upstream `nostr-rs-relay/src/config.rs:54-79`, `config.toml:151-159`) and enforces it in the write path inside the [admission module](../floonet-rs/admission.md), before the event reaches storage. Wrapping the check in the admission layer means it composes cleanly with auth and paid checks.

## Behavior on rejection

Dropped events are **silently rejected** by default (a `shadowReject` in strfry terms). A spammer learns nothing about why their event vanished; a legitimate wallet never publishes disallowed kinds in the first place.

## Growing the list

`ALLOWED_KINDS` is a single config value: a comma-separated list of integers in the plugin config (strfry) or `event_kind_allowlist` in `config.toml` (rs). If you run a relay for a community that also wants, say, public chat, add the kind and restart. One rule for operators upgrading an existing relay: **never narrow the list below what live wallets already depend on**.

## References

- Enforcement: [The write-policy plugin](../floonet-strfry/write-policy.md), [The admission module](../floonet-rs/admission.md).
- The full table: [Allowed kinds](../reference/allowed-kinds.md).
- Kind and NIP details: <https://nostrbook.dev/>.
