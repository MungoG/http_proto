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

---

# Follow-up Q&A

> **Question:** Can this be fixed on my side?

Short answer: **no, not really — this has to be fixed on Niall's side.** Here's why, specifically for your setup.

Your domain `hectorhattie.com` delivers to Google Workspace (`aspmx.l.google.com` at priority 1; there's an Outlook MX but only as a priority-99 backup that senders won't touch while Google is reachable). So the server rejecting Niall's mail is Google's inbound edge.

The `550-5.7.1 IPv6AuthError` is a **hard rejection at the SMTP transaction** (it happens "in reply to end of DATA", *before* the message is ever accepted into your mailbox). It's enforced by Google's edge MX as a baseline policy on all inbound IPv6 mail — it is not a spam-score decision tied to your tenant.

The two recipient-side levers people reach for don't apply here:

- **Email allowlist / approved-senders IP list (Admin console):** this only influences *spam classification of mail that has already been accepted*. It does not waive the IPv6 PTR/SPF-or-DKIM requirement — the message is bounced before that stage. There is no Workspace admin toggle to disable the IPv6 PTR requirement.
- **Inbound gateway:** governs how Google treats mail arriving *via your own configured gateway*, not arbitrary external senders like nedprod. Doesn't help.
- **Your backup Outlook MX (priority 99):** Niall's server only falls back to it if *every* Google MX (priorities 1/5/10) is unreachable, which they aren't. So it won't catch this mail.

So the fix genuinely belongs to nedprod — either pin their Postfix to IPv4 (`smtp_address_preference = ipv4`), or set up the missing PTR for `2a01:5e0:36:5093::1` and add `ip6:` to their SPF. All three are things only Niall (or his host, 9t4.net) can change.

The one practical thing you *can* do from your side: send Niall a precise description of what's broken so it gets fixed fast rather than bouncing around as "generic email problems."

---

> **Question:** Would Niall have issues sending to normal gmail addresses? I'm surprised he hasn't seen this before.

**Yes — a normal `@gmail.com` address would bounce exactly the same way.** Consumer Gmail's MX (`gmail-smtp-in.l.google.com`) publishes AAAA records (`2607:f8b0:4004:c1f::1a`) just like the Workspace MX his mail to you hit. It's the same Google edge enforcing the same IPv6 PTR + auth policy. There's no difference between Workspace and consumer Gmail here. So if his server reaches Gmail over IPv6, it bounces regardless of whether the recipient is `@gmail.com` or a Workspace domain.

So why hasn't he tripped over it before? The failure is **narrow and intermittent**, which is exactly the kind of thing that hides for a long time:

1. **It only bites over IPv6, and only against strict receivers.** Mail leaves his box over v6 *only* when (a) his server currently has a working outbound IPv6 route, and (b) the destination MX has AAAA records. The big receiver that *also strictly enforces* PTR+auth on v6 is Google. Plenty of other providers either have no AAAA on their MX (so his mail goes v4 and works) or are lenient. So the bulk of his mail flows fine over IPv4 — only Gmail-bound mail that happens to go v6 fails.

2. **He almost certainly doesn't think of his server as "doing IPv6."** `mail.nedprod.com` publishes only an A record, no AAAA — so inbound is pure IPv4 and the server *looks* v4-only. But the box has a global v6 address and the OS/Postfix silently prefers v6 outbound when the far end offers it. That mismatch (v4 identity, v6 egress) is the trap, and it's invisible unless you look at outbound connection logs.

3. **It's likely recent.** The smoking gun is the lame delegation: the `/64` reverse zone was delegated *to his own nameserver* (`46.167.244.58`) but that server refuses it. That pattern says the provider (9t4.net) handed nedprod the IPv6 block and pointed reverse DNS at his box expecting him to host it — and he never created the zone. Whenever that v6 connectivity/delegation got switched on (provider change, OS/Postfix update flipping address preference), outbound mail quietly started taking the v6 path. Before that, everything went v4 and there was nothing to notice.

4. **Intermittency masks it.** If v6 reachability is at all flaky, the same recipient succeeds one day (went v4) and bounces another (went v6). That reads as a random glitch, not a systematic config error — easy to shrug off.

Net: it's not that his setup was fine and your domain is special; it's that his server only recently/occasionally started emitting over a broken IPv6 identity, and Gmail is the receiver most likely to slam the door on it. As more receivers enable IPv6 and tighten enforcement, he'd see this spread. The IPv4-pin fix (`smtp_address_preference = ipv4`) makes it all go away immediately; the PTR+SPF fix makes IPv6 actually correct.
