# Hosting: subdomain or zero-DNS path prefix

> **Summary.** A GoblinPay till needs one public HTTPS URL. You can give it a subdomain, or, if you would rather not add a DNS record, mount it under a path on a domain you already have. Either way, put a reverse proxy in front; the wizard prints the exact snippet.

GoblinPay binds locally (`GP_BIND`, default `127.0.0.1:8080`) and is reached at whatever public URL you tell it (`GP_PUBLIC_URL`). That public URL is the one value your store and your customers see, so pick how it is hosted before you run the wizard.

## Option A: a subdomain

Point a subdomain at your server and give the till its own host:

```
https://pay.myshop.com
```

Add the DNS record, terminate TLS at your reverse proxy (or let GoblinPay's built-in `rustls` do it), and forward to the local bind. This is the clean default when you can add a DNS record.

## Option B: zero-DNS path prefix

If you do not want to add a DNS record, mount the till under a path on a domain you already serve:

```
https://myshop.com/pay
```

GoblinPay supports a path prefix so every route, checkout page, and webhook URL it generates is prefixed correctly. Set the prefix and the public URL, then have your existing reverse proxy forward that path to the local bind. Nothing new in DNS.

## The reverse proxy

Whichever option you pick, run a reverse proxy in front (the wizard prints the exact snippet for your setup, and the repo ships a `deploy/Caddyfile`). The proxy terminates TLS and forwards to `GP_BIND`. The webhook URL GoblinPay hands your store is built from the public URL, so as long as the public URL is reachable over HTTPS, WooCommerce, Medusa, and direct API callers all resolve it correctly.

## Where this comes up

- [WooCommerce quick start](woocommerce.md): the till URL is one of the two answers that matter.
- [Medusa quick start](medusa.md): same till URL, different webhook route.
- [API integration](api.md): `pay_url` in every invoice response is built from the public URL.
