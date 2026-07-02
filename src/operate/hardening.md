# Hardening

> **Summary.** Floonet relays ship hardened by default: fail-closed policy, sandboxed systemd units, non-root read-only containers, a reverse proxy with `X-Real-IP`, and no secrets in the repo. These defaults are inherited from the goblin-nip05d deployment and are non-negotiable in the shipped packages.

## Fail-closed everywhere

Every policy surface treats errors as rejection: a malformed event, an unreadable config, an unreachable database or GoblinPay server all produce a reject, never an accept. A relay that cannot evaluate its policy does not guess.

## systemd sandboxing

The shipped units (installer path for floonet-rs; available for bare-metal strfry) carry the full sandbox set:

```ini
DynamicUser=yes
ProtectSystem=strict
ProtectHome=yes
NoNewPrivileges=yes
MemoryDenyWriteExecute=yes
PrivateTmp=yes
PrivateDevices=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
CapabilityBoundingSet=
```

The process owns nothing but its state directory. A compromise of the relay process is a compromise of a throwaway user with a read-only view of the system.

## Containers

The compose deployments run every service non-root, with a read-only filesystem except the data volume, and build upstream at a **pinned ref** (the "stock + spec" pattern), so what you run is a known tree, not whatever upstream's default branch says today.

## The reverse proxy

Both packages expect a TLS-terminating reverse proxy (Caddy and nginx examples ship in each repo) in front of the relay and the name authority. The proxy must forward **`X-Real-IP`**: it is load-bearing for [rate limiting](rate-limits.md). The compose units include the proxy already wired.

## Secrets

No secrets in the repo, ever. The GoblinPay token and any keys arrive via environment files or mounted files with `0400` permissions.

## Event size

Keep the maximum event size large enough for gift-wrapped slatepacks (`events.maxEventSize` in strfry, `max_event_bytes` in floonet-rs). Shrinking it below the shipped default silently breaks payments; see [Gift wraps](../concepts/gift-wraps.md).
