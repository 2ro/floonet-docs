# The write-policy plugin

> **Summary.** All Floonet policy in floonet-strfry lives in one small, documented write-policy plugin: kind whitelist first, then auth, then the paid gate. strfry core stays stock. Rejections are fail-closed.

## How strfry plugins work

strfry's intended extension point is the write policy (`relay.writePolicy.plugin` in `strfry.conf`; upstream docs in `strfry/docs/plugins.md`, implementation in `src/PluginEventSifter.h`). For every incoming event, strfry writes a JSON object to the plugin's stdin, including the event (with its `kind`) and, when NIP-42 is enabled, the `authed` pubkey of the connection. The plugin answers with one of:

- `accept`: store the event.
- `reject`: refuse, with a message the client sees.
- `shadowReject`: refuse silently; the client thinks it succeeded.

## The Floonet plugin

The Floonet plugin is a small program structured as a chain of pluggable checks, each with its own config:

1. **Kind whitelist** (the keystone). `kind` must be in `FLOONET_ALLOWED_KINDS` (default: the [Goblin + Magick Market set](../reference/allowed-kinds.md)), or the event is rejected. This check runs first and cannot be disabled. It applies to every ingest path, including events pulled in via negentropy sync.
2. **Auth**, when `FLOONET_REQUIRE_AUTH=true`: the connection must have completed NIP-42 AUTH (also enable `relay.auth` in `strfry.conf`).
3. **Paid gate**, when `FLOONET_PAY_MODE=write`: the authed pubkey must hold a confirmed payment grant, checked against the bundled name authority (`FLOONET_AUTHORITY_URL`, which talks to GoblinPay). Results are cached for `FLOONET_PAID_CACHE_SECS` (default 60) so the plugin does not call out on every event.

**Fail-closed** is the invariant across all checks: malformed input, an unreachable config, or an errored check means reject, never accept. The first rejection wins.

## Extending it

The plugin is meant to be edited by operators:

- **Add a kind:** edit `FLOONET_ALLOWED_KINDS` and restart, or just touch the plugin file; strfry reloads it on mtime change. No code.
- **Add a policy:** write `def check_foo(req, cfg): return None or "reject reason"` and append it to `CHECKS`; each check receives the request (event plus the `authed` pubkey) and the config, and the first rejection wins.
- **Replace it entirely:** point `relay.writePolicy.plugin` at your own program; the stdin/stdout contract is all there is.

## References

- The whitelist rationale: [The whitelist: default deny](../concepts/whitelist.md).
- The paid gate: [Paid names via GoblinPay](paid-names.md).
- strfry plugin docs: <https://github.com/hoytech/strfry/blob/master/docs/plugins.md>.
