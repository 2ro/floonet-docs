# Authentication (NIP-42)

> **Summary.** Both packages support NIP-42 client authentication plus a pubkey whitelist. Auth is optional and configurable: run fully open, require auth to write, require auth to read, or restrict to a whitelist. Auth composes with the kind whitelist and the paid gate.

## The NIP-42 flow

NIP-42 lets a relay learn, cryptographically, which key is on the other end of a websocket:

1. The relay sends `["AUTH", "<challenge>"]`.
2. The client answers with a **kind `22242`** event carrying two tags: `relay` (the relay's URL) and `challenge` (the string from step 1), signed by the client's key.
3. The relay validates the signature, the challenge, the relay URL, and that `created_at` is recent (about a 10 minute window).

After that, the connection has an authenticated pubkey attached, and every policy decision can use it.

## What each package does with it

- **floonet-strfry**: NIP-42 is native to strfry (`relay.auth.enabled` and `relay.auth.serviceUrl` in `strfry.conf`; the kind `22242` challenge validation lives in upstream `RelayIngester.cpp`). The [write-policy plugin](../floonet-strfry/write-policy.md) receives the `authed` pubkey with every event, so auth checks, whitelist checks, and paid checks all live in one place: the plugin.
- **floonet-rs**: NIP-42 is fully implemented upstream (the auth state machine in `nostr-rs-relay/src/conn.rs:18-228`, and the `Authorization { pubkey_whitelist, nip42_auth, nip42_dms }` config in `config.rs:81-87`). floonet-rs enforces `pubkey_whitelist` in the [admission module](../floonet-rs/admission.md), which the upstream parsed but did not gate writes on.

## Modes

| Mode | Effect |
| --- | --- |
| off (default) | Anyone may read and write, subject to the kind whitelist |
| require auth to write | Unauthenticated publishes are rejected with `auth-required:` |
| require auth to read | Subscriptions require a completed AUTH first |
| whitelist only | Only the configured pubkeys may write, authenticated via NIP-42 |

All modes keep the [kind whitelist](whitelist.md) in force; auth never bypasses it.

## References

- NIP-42: <https://nips.nostr.com/42>.
- Config keys: [Config keys reference](../reference/config-keys.md).
