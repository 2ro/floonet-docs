# Media for GRIN (NIP-96 / Blossom)

> **Summary.** The paid gate is one mechanism applied to many resources. Names are the first implementation; media storage is the documented second: an operator charges GRIN for hosting files, over the same GoblinPay flow, via NIP-96 or Blossom.

## The pattern

Everything paid in Floonet goes through one small interface over GoblinPay:

```
PaidResource {
    quote()    -> price + invoice for this resource
    is_paid()  -> has a confirmed payment for it
}
```

`name` is the first implementation ([paid names](../floonet-strfry/paid-names.md)). `blob/media` is the designed-for second, so a chat app or community can enable it by config without reworking payments.

## The media case

A community running a chat app on a Floonet relay needs somewhere for images and files. The events referencing media are tiny; the bytes themselves need an HTTP host. Two standard shapes exist:

- **[NIP-96](https://github.com/nostr-protocol/nips/blob/master/96.md)**: HTTP file storage with a well-known discovery document and NIP-98-authenticated uploads.
- **[Blossom](https://github.com/hzrd149/blossom)**: content-addressed blobs, stored and fetched by sha256, advertised with a kind `10063` server list.

Either way, the paid flow is identical to names:

1. Client asks to upload; the server quotes a price (per upload, per MB, or per month, admin's choice).
2. The server creates a GoblinPay invoice; the user pays in GRIN.
3. On confirmed payment, the upload is accepted and served.

## Configuration sketch

```bash
FLOONET_PAY_MODE=name                    # names stay paid (or off)
FLOONET_MEDIA_ENABLED=true
FLOONET_MEDIA_PRICE_GRIN_PER_MB=0.1     # admin-set, like the name price
```

## Status

Media-for-GRIN is documented as the extensibility example and kept modular as a later add-on, in line with the Floonet principle of shipping only what is needed now. The `PaidResource` seam and the GoblinPay flow it needs are already in both packages; enabling it is an add-on module, not a rework.
