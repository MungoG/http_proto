I've debugged it end-to-end. Here's exactly what's wrong.

## What actually happened

Niall's server connected to Gmail **over IPv6 from `2a01:5e0:36:5093::1`** (that's the address in the brackets in the 550 message). Gmail's IPv6 rule is strict: the sending IPv6 address must (a) have a valid PTR record, and (b) the mail must pass SPF *or* DKIM with alignment. **Both halves are broken for that IPv6 address.** That's the literal meaning of "PTR records *and* authentication."

## Finding 1 — No PTR (reverse DNS) for the IPv6 address — *lame delegation*

```
PTR  2a01:5e0:36:5093::1  →  REFUSED / SERVFAIL  (no record)
```
The reverse zone for the `/48` (`6.3.0.0.0.e.5.0.1.0.a.2.ip6.arpa` = `2a01:5e0:36::/48`) is hosted by the provider's nameservers `ns1.9t4.net` / `ns2.9t4.net`. Within that, the `/64` block (`...5093...ip6.arpa`) has been **delegated to a nameserver running on `46.167.244.58` — i.e. nedprod's own mail host — but that server refuses the query**. It isn't actually serving the reverse zone, so the delegation is *lame* and the PTR simply doesn't exist. Gmail requires a forward-confirmed PTR on IPv6; this fails outright.

For contrast, the **IPv4** reverse DNS is perfectly fine:
```
PTR  46.167.244.58  →  europe7b.nedproductions.biz  →  46.167.244.58   (forward-confirmed ✓)
```

## Finding 2 — SPF does not authorize the IPv6 address (so it hard-fails)

```
nedprod.com TXT:
  v=spf1 ip4:46.167.244.57 ipv4:46.167.244.58 include:_spf.google.com include:_spf.mailhostbox.com -all
```
- There is **no `ip6:` mechanism at all**, and none of the includes (`_spf.google.com`, `_spf.mailhostbox.com` → titan/netblocks) cover `2a01:5e0:36::`. With `-all`, sending from the IPv6 address is an SPF **Fail**.
- Bonus bug: `ipv4:46.167.244.58` is **not a valid SPF token** — the keyword is `ip4`, not `ipv4`. That entry is junk/ignored, so even `.58` isn't actually SPF-authorized over IPv4 (Google's includes/DKIM are what save the v4 path; DMARC is only `p=none`).

## Why it's even using IPv6

`mail.nedprod.com` publishes only an **A record (46.167.244.58), no AAAA** — but the box clearly has working global IPv6 (`2a01:5e0:36:5093::1`). Since `aspmx.l.google.com` publishes AAAA records, Postfix prefers IPv6 for the outbound connection and sends from an address that has neither PTR nor SPF coverage. Mail to IPv4-only destinations works; mail to Gmail (and any IPv6-capable receiver enforcing the same rules) breaks.

## How to fix it — two real options

**Quickest (stop sending over IPv6):** on the Postfix host set
```
smtp_address_preference = ipv4
```
(`postconf -e 'smtp_address_preference = ipv4'` then reload). Delivery falls back to `46.167.244.58`, which has valid forward-confirmed PTR. This unblocks Gmail immediately.

**Proper (make IPv6 compliant), do both:**
1. **Create the PTR** for `2a01:5e0:36:5093::1`. The `/64` reverse was delegated to nedprod's own server at `46.167.244.58`, so either actually serve that `ip6.arpa` zone there (add the PTR → e.g. `mail.nedprod.com` or `europe7b.nedproductions.biz`) and ensure that name has a matching AAAA back to `…5093::1`, or ask the provider (9t4.net) to host the record. The name in the PTR must forward-resolve to the same IPv6 (FCrDNS).
2. **Add the IPv6 to SPF**, e.g. change the record to:
   `v=spf1 ip4:46.167.244.57 ip4:46.167.244.58 ip6:2a01:5e0:36:5093::1 include:_spf.google.com include:_spf.mailhostbox.com -all`
   (note `ip4:` not `ipv4:`).

While you're in the SPF record, fix the `ipv4:` typo regardless. DKIM would also satisfy Gmail's auth half, but the PTR is mandatory for IPv6 either way — so option 1 isn't optional if Niall wants to keep sending over v6.

The single root cause: the host prefers outbound IPv6, but only the IPv4 identity (PTR + SPF) was ever set up. Tell Niall either to pin Postfix to IPv4 or to finish the IPv6 setup (PTR + SPF) — both records, not one.
