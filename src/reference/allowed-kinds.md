# Allowed kinds

The shipped default whitelist, in full. It is identical in both packages (`DEFAULT_ALLOWED_KINDS` in `floonet-strfry/plugin/floonet_writepolicy.py` and in `floonet-rs/src/admission.rs`), and it is exactly what runs in production on the flagship `relay.floonet.dev`, which serves two applications: the [Goblin](https://goblin.st) wallet and the [Magick Market](https://magick.market) marketplace. Everything not listed is rejected; see [The whitelist: default deny](../concepts/whitelist.md).

## Goblin wallet kinds

| Kind | NIP | Name | Why Floonet carries it |
| --- | --- | --- | --- |
| `0` | [01](https://nips.nostr.com/1) | Profile metadata | Display names and avatars, so contacts render as people |
| `3` | [02](https://nips.nostr.com/2) | Contact list | Follow and contact lists |
| `5` | [09](https://nips.nostr.com/9) | Deletion request | Lets users retract their own events |
| `13` | [59](https://nips.nostr.com/59) | Seal | The inner encrypted layer of a gift wrap |
| `1059` | [59](https://nips.nostr.com/59) | Gift wrap | The opaque envelope everything private travels in |
| `10002` | [65](https://nips.nostr.com/65) | Relay list | Where a user can be found |
| `10050` | [17](https://nips.nostr.com/17) | DM relay list | Where to deliver a user's private messages |
| `24133` | [46](https://nips.nostr.com/46) | Nostr Connect | Remote signing (ephemeral); wallet login |
| `27235` | [98](https://nips.nostr.com/98) | HTTP auth | Authenticates name-authority registration requests |

## Magick Market marketplace kinds

The marketplace also reuses `0`, `5`, `1059`, and `10002` above.

| Kind | NIP | Name | Why Floonet carries it |
| --- | --- | --- | --- |
| `1` | [01](https://nips.nostr.com/1) | Text note | Bug reports, shared listings |
| `7` | [25](https://nips.nostr.com/25) | Reaction | Likes on listings and posts |
| `14` | Gamma | Order chat | Plaintext order messages in the market's order flow |
| `16` | Gamma | Order status | Order processing and status updates |
| `17` | Gamma | Payment receipt | Payment confirmation in the order flow |
| `1111` | [22](https://nips.nostr.com/22) | Comment | Threaded comments |
| `10000` | [51](https://nips.nostr.com/51) | Mute list | Merchant/product blacklist |
| `30000` | [51](https://nips.nostr.com/51) | People set | Admins, editors, featured users |
| `30003` | [51](https://nips.nostr.com/51) | Bookmark set | Featured collections |
| `30078` | [78](https://nips.nostr.com/78) | App-specific data | Cart, relay preferences, registries |
| `30402` | [99](https://nips.nostr.com/99) | Product listing | Classified listing / product |
| `30405` | Gamma | Product collection | Collections and featured products |
| `30406` | Gamma | Shipping option | Shipping options in the order flow |
| `31990` | [89](https://nips.nostr.com/89) | Handler information | Marketplace instance settings |

"Gamma" marks kinds from the Gamma Markets spec the marketplace implements; they have no merged NIP yet.

## Excluded on purpose

| Kind | Why it is rejected |
| --- | --- |
| `9735` | Lightning zap receipts; dead in this GRIN-only ecosystem |
| `25910` | ContextVM; only ever rides *inside* a `1059` gift wrap, never raw |
| `30017`/`30018` | Legacy NIP-15 market events; read from sellers' own relays during migration, never written here |

## Notes

- The canonical list is the union of what the two live applications publish and read; the tables above match `DEFAULT_ALLOWED_KINDS` in both packages and the policy running on `relay.floonet.dev`.
- Kind `22242` (NIP-42 AUTH) is a **connection-level** event, not a stored one, so it does not appear in the storage whitelist; it is handled by the auth machinery when [authentication](../concepts/auth.md) is enabled.
- Operators upgrading an existing relay: the list may grow, but never narrow it below what live wallets already depend on.
- Kind reference: <https://nostrbook.dev/>.
