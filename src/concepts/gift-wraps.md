# Gift wraps: what a relay sees

> **Summary.** Everything private on Floonet travels as a NIP-59 gift wrap: a kind `1059` event encrypted with NIP-44 to a throwaway key, containing a kind `13` seal, containing the real message (NIP-17). A relay sees ciphertext, a random-looking recipient key, and a deliberately fuzzed timestamp. Nothing else.

## The three layers

1. **The rumor.** The actual message (for a wallet, a Grin slatepack riding a NIP-17 private direct message). It is never signed on its own, so it cannot be leaked and attributed.
2. **The seal (kind `13`).** The rumor is encrypted with [NIP-44](https://nips.nostr.com/44) to the recipient and signed by the real sender. The seal proves authorship to the recipient only.
3. **The gift wrap (kind `1059`).** The seal is encrypted again, this time signed by a **one-time throwaway key**, and addressed to the recipient's key in a `p` tag. This is the only layer a relay ever stores.

## What the relay can and cannot learn

A Floonet relay storing a gift wrap sees:

- A kind `1059` event, signed by a key that will never be used again.
- A `p` tag naming the recipient key (which wallets rotate independently of their funds).
- Ciphertext of unknowable content.
- A `created_at` that is **deliberately wrong**: NIP-59 tooling backdates both the seal and the wrap by a random amount up to two days, so the timestamp does not reveal real send time.

It cannot see the sender, the content, the amount, or whether the envelope is a payment, a message, or anything else. This is why [NIP-11 metadata stays payment-neutral](nip11.md): the relay genuinely does not know.

## One operational consequence: event size

Gift-wrapped slatepacks are much larger than typical Nostr notes. Both packages ship with a maximum event size large enough for wrapped slatepacks:

- floonet-strfry: `events.maxEventSize` in `strfry.conf`.
- floonet-rs: `max_event_bytes` in `config.toml`.

Do not tighten these below the shipped defaults or wrapped payloads will bounce.

## References

- NIP-17 (private DMs): <https://nips.nostr.com/17>.
- NIP-44 (encryption): <https://nips.nostr.com/44>.
- NIP-59 (gift wrap, kind 13 and 1059): <https://nips.nostr.com/59>.
- Kind pages: <https://nostrbook.dev/kinds/1059>.
