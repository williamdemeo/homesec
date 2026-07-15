<!--
File: docs/adr/ADR-001-remote-access.md

First Architecture Decision Record for the homesec project (issue M1-2).
The ADR format here is provisional; the ADR index and a shared template
are formalized in M1-5 (repository scaffolding).  If that work changes the
house style, this record should be reformatted to match.
-->

# ADR-001: Remote access via a Tailscale overlay network

- **Status**: Accepted
- **Date**: 2026-07-14
- **Deciders**: @williamdemeo
- **Issue**: M1-2 (#3)
- **Related**: M3-4 (Tailscale + secrets on the server), M5-1 (HTTPS + family devices), M5-2 (push notifications), M6-4 (ACL review), M7-4 (own-router / bypass-mode expansion)

## Context

The NVR (Frigate, on the X1 Yoga) must be reachable from household phones and
laptops when they are away from home, while staying **invisible to the public
internet**.

Two facts fix the shape of the solution:

1. **Starlink puts us behind CGNAT.**  Standard residential service assigns a
   private address from `100.64.0.0/10` and blocks all inbound IPv4; a public
   IP is offered only on Priority/Mobile-Priority/Maritime plans.  Starlink does
   provide a native IPv6 `/56`, but the stock router firewalls inbound IPv6 and
   exposes no setting to change that.  **Conventional port forwarding is
   therefore impossible** — and, since it would mean exposing an NVR to the
   internet, undesirable anyway.
2. **The access path must serve non-technical family members** on their own
   phones, and must present **valid TLS**, because browser web-push
   notifications and PWA install (M5) require a secure context with a
   trusted certificate.

So we need an **outbound-only overlay** that traverses CGNAT, has trivial phone
clients, and gives us a real HTTPS certificate — without opening a single
inbound port.

## Decision

**Use [Tailscale](https://tailscale.com) (a WireGuard-based mesh VPN) as the
remote-access layer.**  The server joins the tailnet (M3-4); family devices join
with a single login (M5-1); the Frigate UI is served over HTTPS on the server's
MagicDNS `*.ts.net` name using a Tailscale-provisioned Let's Encrypt certificate
(M5-1).  No inbound ports are opened on the Starlink network.

## Alternatives considered

| Criterion | **Tailscale (chosen)** | WireGuard + rented VPS | ZeroTier | Cloudflare Tunnel |
|---|---|---|---|---|
| Works behind Starlink CGNAT | ✅ direct WireGuard when possible; **DERP relay fallback over TCP :443** when symmetric NAT blocks hole-punching | ✅ client dials out to the VPS's public IP (you operate the relay) | ✅ via ZeroTier's relays | ✅ `cloudflared` makes an outbound tunnel to Cloudflare's edge |
| Phone UX (non-technical family) | ✅ first-class apps, one-tap connect | ⚠️ WireGuard app + hand-managed keys/config per device | ✅ apps, join by network ID | ✅ no app (browser) — but each user authenticates through Cloudflare Access |
| TLS for web push (M5) | ✅ built-in: `tailscale cert` / `tailscale serve`, valid LE cert on the `ts.net` name | ❌ DIY: own domain + reverse proxy + certbot | ❌ DIY | ✅ TLS terminated at Cloudflare's edge |
| Operational cost | ✅ free Personal plan: 6 users, unlimited devices, MagicDNS, HTTPS certs | 💲 VPS ~$4–6/mo + ongoing maintenance | ✅ free tier | ✅ free tier |
| Data path / privacy | ✅ end-to-end WireGuard; relays **cannot** decrypt; video stays peer-to-peer whenever a direct path exists | ✅ end-to-end; but relayed video transits *your* VPS | ✅ end-to-end; relays can't decrypt | ❌ **video traverses Cloudflare's edge** (they terminate TLS); CF TOS §2.8 restricts serving video through the CDN |
| Public exposure | ✅ none (overlay only) | ✅ only the WireGuard UDP port on the VPS | ✅ none (overlay only) | ⚠️ the service is published to the internet (guarded by CF Access) |
| Self-host / lock-in escape | ✅ [Headscale](https://github.com/juanfont/headscale) is a drop-in self-hosted control plane | ✅ already fully self-hosted | ⚠️ self-hostable controller, less common | ❌ Cloudflare-only |

**Why not the others.**

- **Cloudflare Tunnel** is the strongest "no app to install" contender, but it
  reintroduces a public entry point and routes camera video through a
  third-party edge that terminates TLS — the opposite of this project's
  local-first, private stance — and Cloudflare's TOS discourages streaming video
  over the CDN.  Wrong tool for a private NVR.
- **WireGuard + a rented VPS** is the purest self-hosted design and a genuine
  fallback (see revisit conditions), but it adds a paid always-on VPS, manual
  per-device key management, and a DIY certificate/reverse-proxy stack for the
  TLS that Tailscale gives us for free.
- **ZeroTier** is closest to Tailscale in shape, but has a less mature
  access-control (ACL) story and no built-in convenience for provisioning a
  trusted TLS certificate — which we need for web push.

Tailscale is the only option that clears **all** of: CGNAT traversal with zero
inbound config, one-tap family clients, free at household scale, end-to-end
encrypted video kept off third-party edges, **and** turnkey valid TLS.

### Note: Cloudflare as DNS registrar is orthogonal to this decision

We register domains and manage DNS through Cloudflare.  That is unrelated to the
rejection of *Cloudflare Tunnel* above: the objection there is to tunnelling
camera video through Cloudflare's edge, not to Cloudflare-the-registrar.  Using
Cloudflare for DNS neither argues for Cloudflare Tunnel nor against Tailscale.

It is, however, a useful **asset** to keep in mind for later work.  A
Cloudflare-managed domain makes DNS-01 ACME certificate issuance trivial, which
matters if **M5-1** later opts for a custom hostname (e.g. `nvr.<domain>`) for
the Frigate UI instead of the default Tailscale `ts.net` certificate — and it
would similarly simplify TLS for the WireGuard-+-VPS fallback named in the
revisit conditions.  Recorded here so M5-1 can take advantage of it; it does not
change this decision.

## Consequences

**Positive**

- No inbound ports; the NVR is unreachable from the public internet by design.
- Each family device joins with one login and reaches the server by a stable
  MagicDNS name; revocation is per-device (M5-3).
- Valid HTTPS certificate out of the box → unblocks web push / PWA (M5).
- Zero recurring cost at our scale.
- Video stays end-to-end encrypted; peer-to-peer whenever a direct path exists.

**Negative / to watch**

- **Coordination-server dependency.**  Tailscale's control plane brokers
  connections and identity.  Mitigations: existing connections keep working if
  the control plane is briefly unreachable; **Headscale** is a self-hosted
  escape hatch if lock-in ever becomes a concern; and LAN access to the server
  does not depend on Tailscale at all (documented as the disaster path in M3-4).
- **CGNAT may force a relayed (DERP) path.**  Starlink's CGNAT is often
  symmetric, which can defeat UDP hole-punching and pin the connection to a DERP
  relay.  That still works (encrypted, over :443) but adds latency and consumes
  Starlink bandwidth in both directions.  **Measure direct-vs-relayed and RTT in
  M3-4**, and set remote-viewing expectations accordingly (M5-1 bandwidth note).
- **Identity is now an access dependency.**  Whichever identity provider backs
  the tailnet governs who can reach the NVR; choose a household-appropriate
  setup and document it (M5-3), and review ACLs so devices get least privilege
  (M6-4).

## Revisit conditions

Reopen this decision if any of the following hold:

- Tailscale changes its free-plan terms so that household use no longer fits.
- We complete **M7-4** (Starlink bypass mode + our own router): a stable
  self-hosted WireGuard endpoint then becomes attractive and would remove the
  coordination-server dependency.
- A direct connection proves unattainable and DERP-relayed latency makes remote
  live view unusable — at which point a VPS-hosted WireGuard endpoint with a
  public IP (a direct, non-relayed target) is the leading alternative.

## References

- [Starlink IP Address / CGNAT policy](https://www.starlink.com/support/article/ac09301b-cef6-a125-c251-856196a77f92)
- [Tailscale — How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works) and [DERP servers](https://tailscale.com/docs/reference/derp-servers)
- [Tailscale — Enabling HTTPS certificates](https://tailscale.com/docs/how-to/set-up-https-certificates)
- [Tailscale free plans](https://tailscale.com/docs/account/manage-plans/free-plans-discounts)
- [Headscale (self-hosted control plane)](https://github.com/juanfont/headscale)
