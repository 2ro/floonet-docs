# GoblinPay: take Grin payments

> **Summary.** GoblinPay is the Grin payment backend behind Floonet's [paid names](../floonet-strfry/paid-names.md) and [paid writes](../floonet-rs/goblinpay.md), and it stands on its own as a merchant till: point a WooCommerce or Medusa store at it, or integrate its REST API directly the way [Magick Market](https://magick.market) does. It is receive-only, coins never pass through the API, and the recommended way to stand one up is the built-in setup wizard.

## What GoblinPay is

GoblinPay is a small, self-hosted till that receives Grin. A customer's [Goblin wallet](https://goblin.st) pays it as a gift-wrapped slatepack over Nostr (or, for non-Goblin wallets, over the optional grin1/Tor rail), and GoblinPay watches the chain and confirms the payment. Your service is only ever in the *authorization* path: it asks GoblinPay to create an invoice and reads back the status. No coins ever travel through your application.

An invoice moves through three states:

```
open ──▶ paid ──▶ confirmed
```

- `open`: created, not yet paid.
- `paid`: the payment landed and matched the invoice (in the mempool / low confirmations).
- `confirmed`: the paying kernel reached `GP_CONFIRMATIONS` on-chain confirmations (default 10). **Grant the goods on `confirmed`.**

The till has its own seed: run it hot but light, keep only a small working balance on it, and sweep to your main wallet regularly.

## Setup: the wizard (recommended path)

The fastest way to stand up a till is the built-in wizard. Install the binary and unit, which also offers to run the wizard for you:

```bash
sudo ./deploy/install.sh
# or, if GoblinPay is already installed:
sudo gp-server setup
```

It asks five questions, each with a default:

1. the public URL customers reach this till at,
2. your shop's website URL (used to build the webhook URL),
3. the Grin seed: press Enter to generate a fresh till seed (shown once, write it down) or paste an existing 24 words,
4. the currencies your shop prices in (default `usd`),
5. an advanced yes/no for the grin1/Tor rail (default no).

Everything else it does for you:

- generates the wallet password, the API token, the admin token, and the webhook secret, so you never invent or type a secret;
- creates the encrypted wallet on the spot from the seed, so the seed is **consumed once** and never lives in the service environment afterwards (it exists only encrypted at rest and in your written backup);
- probes a curated list of healthy mainnet Grin nodes and picks the first that answers, falling back automatically;
- writes `/etc/goblinpay.env` (mode 0640) and `/etc/goblinpay/secrets/wallet_password` (mode 0400), exactly where the shipped `gp-server.service` looks (`EnvironmentFile` + `LoadCredential`);
- prints the webhook URL and the three values to paste into a store (GoblinPay URL, API Token, Webhook Secret) plus the private admin token.

Re-running is safe: the wizard refuses to overwrite an existing wallet or config unless you pass `--reconfigure`, which keeps the existing seed and password and only rewrites the config and tokens. Other flags: `--prefix DIR` (write under a prefix instead of `/`), `--node URL` (skip the node probe), `--batch` (read scripted answers from a non-terminal stdin).

Then start the till:

```bash
sudo systemctl start gp-server
```

The env-var configuration is the advanced path for operators who want to configure GoblinPay by hand; the wizard hides all of it.

## The seed, once

GoblinPay is init-once. `GP_MNEMONIC` (or the seed you paste into the wizard) is used a single time to create the encrypted wallet; after the encrypted seed exists at rest under `GP_DATA_DIR` (mode 0600), the wallet opens with `GP_WALLET_PASSWORD` alone. Steady state is "password in, seed out": if `GP_MNEMONIC` is still set the server only checks it against the seed at rest and logs a notice to remove it. Prefer file-based delivery (`GP_MNEMONIC_FILE`, `GP_WALLET_PASSWORD_FILE`, mode 0400) with systemd `LoadCredential` or docker `/run/secrets` so secrets never sit in the environment. The Nostr identity (nsec) is a deliberately separate secret from the Grin mnemonic; a random one is generated on first start if unset and persisted NIP-49 encrypted under `GP_DATA_DIR/nostr/`.

## Where to go next

- **[WooCommerce quick start](woocommerce.md)**: take Grin on a WordPress/WooCommerce store.
- **[Medusa quick start](medusa.md)**: take Grin in a Medusa v2 store.
- **[API integration](api.md)**: integrate the REST API directly, the way Magick Market does.
- **[Hosting](hosting.md)**: subdomain vs zero-DNS path-prefix.
- **Relay operators**: to charge GRIN for names or writes on a Floonet relay, see [Charge GRIN for your relay](../operate/charge-grin.md) and [The GoblinPay processor](../floonet-rs/goblinpay.md).
