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
2. **Gift-wrap retention and shape** (kind `1059` only, always on, no config). A gift wrap carries a payment, so the relay guards its retention and shape at admission rather than trusting the publishing client. Three rules, each rejecting fail-closed:
   - **No expiration.** A NIP-40 `expiration` tag is refused. It is the only automatic deletion trigger strfry has (its reaper runs about every 9 seconds), so forbidding it means a payment gift wrap can never be early-deleted out from under a wallet that has not yet fetched it. This is the retention guarantee: **an accepted payment gift wrap stays until the recipient reads it.**
   - **One well-formed recipient.** Exactly one `p` tag is required, and its value must be strict lowercase 32-byte hex (`^[0-9a-f]{64}$`). A gift wrap with zero, several, or a mixed-case `p` tag is rejected. The recipient-only read gate matches `#p` against the authed pubkey as a case-sensitive lowercase string, so an uppercase or mixed-case recipient would otherwise be admitted and then be permanently undeliverable.
   - **No extraneous tags.** A well-formed NIP-59 gift wrap legitimately carries just that one `p` tag, so any other tag is refused. This stops a relay-sized (up to 128 KiB) event from smuggling thousands of junk tags in alongside one valid `p`.

   Every other kind is unaffected by all three.
3. **Public-note lock.** The two public-note kinds, `1` (text notes) and `30023` (long-form articles), are accepted **only** when the event's author is in `FLOONET_AUTHORIZED_AUTHORS` (hex or npub, comma-separated). This list is closed by default: with no authors configured, both kinds are rejected for everyone, so a payment relay never fills with public-note spam. Every other kind is unaffected. The author list can also live in a `floonet.env` KEY=VALUE file next to the plugin (`FLOONET_ENV_FILE` overrides the path); real environment variables win, and strfry reloads the plugin on mtime change, so `touch`ing it after an edit applies new authors with no restart.
4. **Auth**, when `FLOONET_REQUIRE_AUTH=true`: the connection must have completed NIP-42 AUTH (also enable `relay.auth` in `strfry.conf`).
5. **Paid gate**, when `FLOONET_PAY_MODE=write`: the authed pubkey must hold a confirmed payment grant, checked against the bundled name authority (`FLOONET_AUTHORITY_URL`, which talks to GoblinPay). Results are cached for `FLOONET_PAID_CACHE_SECS` (default 60) so the plugin does not call out on every event.

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
