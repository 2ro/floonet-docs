# The write-policy plugin

> **Summary.** All Floonet policy in floonet-strfry lives in one small, documented write-policy plugin: kind whitelist first, then auth, then the paid gate, then the name-authority consult. strfry core stays stock. Rejections are fail-closed.

## How strfry plugins work

strfry's intended extension point is the write policy (`relay.writePolicy.plugin` in `strfry.conf`; upstream docs in `strfry/docs/plugins.md`, implementation in `src/PluginEventSifter.h`). For every incoming event, strfry writes a JSON object to the plugin's stdin, including the event (with its `kind`) and, when NIP-42 is enabled, the `authed` pubkey of the connection. The plugin answers with one of:

- `accept`: store the event.
- `reject`: refuse, with a message the client sees.
- `shadowReject`: refuse silently; the client thinks it succeeded.

## The Floonet plugin

The Floonet plugin is a small program structured as a chain of pluggable checks, each with its own config:

1. **Kind whitelist** (the keystone). `kind` must be in `ALLOWED_KINDS`, or the event is shadow-rejected. This check runs first and cannot be disabled.
2. **Auth**, when enabled: require an `authed` pubkey, or require membership in a pubkey whitelist.
3. **Paid gate**, when `FLOONET_PAY_MODE=write`: the authed pubkey must have a confirmed GoblinPay payment on record. Results are cached with a TTL so the plugin does not call GoblinPay on every event.
4. **Name authority consult**, for policies that depend on name state.

**Fail-closed** is the invariant across all checks: malformed input, an unreachable config, or an errored check means reject, never accept.

## Extending it

The plugin is meant to be edited by operators:

- **Add a kind:** edit `ALLOWED_KINDS` and restart. No code.
- **Add a policy:** add a check function to the chain; each check receives the event and the `authed` pubkey and returns accept, reject, or pass-to-next.
- **Replace it entirely:** point `relay.writePolicy.plugin` at your own program; the stdin/stdout contract is all there is.

## References

- The whitelist rationale: [The whitelist: default deny](../concepts/whitelist.md).
- The paid gate: [Paid names via GoblinPay](paid-names.md).
- strfry plugin docs: <https://github.com/hoytech/strfry/blob/master/docs/plugins.md>.
