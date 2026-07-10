# The admission module

> **Summary.** All write policy in floonet-rs flows through one composable admission layer. Each policy is small and independent; the server calls a single entry point; the answer is accept or reject, fail-closed.

## Motivation

Upstream nostr-rs-relay makes its accept and reject decisions inline in a large `server.rs` (the event-acceptance region around `server.rs:1324-1361`, just before the event is handed to storage). That works for one policy, but Floonet has four (whitelist, auth, paid gate, name authority) and wants operators to add their own. So floonet-rs introduces `src/admission.rs`: one trait, many small policies, one call site.

## The shape

```rust
pub trait EventAdmissionPolicy {
    fn check(&self, event: &Event, authed_pubkey: Option<&Pubkey>, repo: &dyn Repo)
        -> Decision;   // Accept | Reject(reason) | ShadowReject
}
```

The configured policies run in order; the first rejection wins:

1. **Kind whitelist** (`event_kind_allowlist`): the keystone, always first, cannot be disabled.
2. **Gift-wrap retention and shape** (kind `1059` only, always on): rejects a NIP-40 `expiration` tag (so a payment gift wrap can never be reaped before the recipient reads it, the retention guarantee); requires exactly one `p` recipient tag in strict lowercase 32-byte hex (a mixed-case value would pass the read gate's lowercase comparison and become undeliverable); and rejects any other tag, so a relay-sized event cannot smuggle junk tags past admission. Every other kind is unaffected.
3. **Public-note lock** (`authorization.public_note_authors`): kinds `1` and `30023` are accepted only from the configured authors (hex or npub); closed by default, so with no authors set both kinds are rejected for everyone. Every other kind is unaffected.
4. **Auth policy**: NIP-42 requirement and `pubkey_whitelist` membership, when enabled.
5. **Paid gate**: confirmed GoblinPay payment for the authed pubkey, when `FLOONET_PAY_MODE=write`.
6. **Name authority policy**: checks that depend on name state.

`server.rs` calls exactly one function; every policy decision, log line, and metric hangs off that one seam.

## Fail-closed

A policy that errors (database unreachable, GoblinPay timeout, malformed event) returns a rejection, never an accept. A relay that cannot evaluate its policy does not guess.

## Adding a policy

Implement `EventAdmissionPolicy`, register it in the admission chain, add its config block. The composition is ordinary Rust; there is no plugin ABI to fight. For out-of-process extension, upstream's gRPC `nauthz` authorization hook remains available and can be exposed alongside the built-in chain.

## References

- The whitelist: [The whitelist: default deny](../concepts/whitelist.md).
- The paid gate: [The GoblinPay processor](goblinpay.md).
- Upstream write path: `nostr-rs-relay/src/server.rs:1324-1361`.
