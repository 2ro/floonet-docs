# Allowed kinds

The shipped default whitelist, in full. Everything not listed is rejected; see [The whitelist: default deny](../concepts/whitelist.md).

| Kind | NIP | Name | Why Floonet carries it |
| --- | --- | --- | --- |
| `0` | [01](https://nips.nostr.com/1) | Profile metadata | Display names and avatars, so contacts render as people |
| `3` | [02](https://nips.nostr.com/2) | Contact list | Follow and contact lists |
| `5` | [09](https://nips.nostr.com/9) | Deletion request | Lets users retract their own events |
| `13` | [59](https://nips.nostr.com/59) | Seal | The inner encrypted layer of a gift wrap |
| `1059` | [59](https://nips.nostr.com/59) | Gift wrap | The opaque envelope everything private travels in |
| `10002` | [65](https://nips.nostr.com/65) | Relay list | Where a user can be found |
| `10050` | [17](https://nips.nostr.com/17) | DM relay list | Where to deliver a user's private messages |
| `27235` | [98](https://nips.nostr.com/98) | HTTP auth | Authenticates name-authority registration requests |

## Notes

- The canonical list is whatever the Goblin wallet publishes and reads (`goblin/src/nostr/` in the wallet source); the table above matches it.
- Kind `22242` (NIP-42 AUTH) is a **connection-level** event, not a stored one, so it does not appear in the storage whitelist; it is handled by the auth machinery when [authentication](../concepts/auth.md) is enabled.
- Operators upgrading an existing relay: the list may grow, but never narrow it below what live wallets already depend on.
- Kind reference: <https://nostrbook.dev/>.
