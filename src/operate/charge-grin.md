# Charge GRIN for your relay

> **Summary.** Turning on paid names is editing three config values: your GoblinPay server URL, the pay mode, and the price. No code, no payment processor account, no third party: your users pay you in GRIN directly.

## Prerequisites

- A running Floonet relay (either package).
- A running **GoblinPay** server you operate: the Grin payment backend that creates invoices, watches the chain, and confirms payments with payment proofs.

## The three edits

In your `.env` (compose) or environment file (systemd):

```bash
GOBLINPAY_URL=https://pay.yourdomain     # your GoblinPay server
FLOONET_PAY_MODE=name                    # charge for names
FLOONET_NAME_PRICE_GRIN=5                # the price, in GRIN
```

Restart, and name registration on your authority now quotes 5 GRIN and refuses to complete until GoblinPay confirms the payment on chain. Edit the price any time; the quote follows the config.

## The modes

| Mode | What is paid | Good for |
| --- | --- | --- |
| `off` | Nothing | Community relays, default |
| `name` | Claiming a name | The common case: free relay, paid vanity names |
| `write` | Publishing events | Invite-style relays where writing itself is the resource |

## What your users experience

A user claiming a name from a wallet gets sent to your GoblinPay pay page, where they pay from Goblin Wallet over Nostr. If you enable GoblinPay's optional grin1 rail (off by default), the page also lets them pay from any Grin wallet over Tor. Either way, a wallet that cannot hand its slatepack back automatically can paste it into the page instead. Once the payment confirms (GoblinPay's house standard of 10 on-chain confirmations), the name is theirs: one name per key, standard NIP-05, resolvable everywhere.

A note worth passing to your users: one wallet can hold multiple Nostr identities. If they pay for a name, the name belongs to the npub that registered it; loading the same wallet and switching to that npub keeps the name.

## Two rules to keep

1. **The relay's public metadata stays payment-free.** Charging for names does not change your [NIP-11 document](../concepts/nip11.md); the shipped defaults already comply.
2. **The GoblinPay token is a secret.** Environment or 0400-mounted file, never the repo.

## More things to charge for

Names are the first paid resource, not the last. The same gate can charge for media storage; see [Media for GRIN](media-for-grin.md).
