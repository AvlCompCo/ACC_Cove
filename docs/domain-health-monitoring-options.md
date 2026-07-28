# Client Domain & Email Health Monitoring — Options

Goal: run an automated scan (roughly daily) across all client domains covering DNS,
email authentication (SPF/DKIM/DMARC), MX/SMTP reachability and blacklist status, and
alert us when something **changes** — a record edited, an IP listed, a mail server down.

> **Note on sources:** `mxtoolbox.com` is blocked by the network policy in the environment
> this was researched from, so API details below come from third-party integrations and
> working client code, and pricing from 2026 review-site listings. Verify endpoint shapes
> and quotas against the live API reference and your account's plan page before building.

---

## What MXToolbox actually offers

Two separate products, often confused:

### 1. Hosted monitors (the Delivery Center product)

MXToolbox runs the checks on their schedule and notifies us. Covers blacklist listing
events, DNS record change detection, SMTP connectivity, HTTP/website checks, and
SPF/DKIM/DMARC record change tracking. Alerts via email; monitor state is also readable
through the API.

Pricing (per review-site listings, 2026):

| Plan | Cost | Domains |
|---|---|---|
| Free account | $0 | 1 domain, weekly blacklist check |
| Delivery Center | ~$129/mo | 5 domains, 500k messages/mo |
| Delivery Center Plus | ~$399/mo | 5 domains, 5M messages/mo, adds API/bulk ops, SPF flattening |
| Managed / Enterprise | quoted | contact sales |

**The problem for us:** it is priced per-domain in small buckets. Five domains at $129/mo
does not scale to an MSP client base — 50 client domains lands in enterprise-quote
territory. There is also no real multi-tenant model: no per-client separation, no
white-labelled client reporting, no clean way to hand a client "their" dashboard.

### 2. REST API (usable on free and paid accounts)

This is the more interesting piece for us — we drive the schedule, they do the lookups.

- **Base URL:** `https://api.mxtoolbox.com/api/v1/`
- **Auth:** `Authorization: <your-api-key-uuid>` header — plain UUID, *no* `Bearer` prefix.
  Also accepted as an `?authorization=` query parameter.
- **Lookups:** `GET /api/v1/Lookup/{command}/{argument}`
  Commands cover `mx`, `spf`, `dkim`, `dmarc`, `blacklist`, `dns`, `a`, `ptr`, `smtp`,
  `https`, `soa`, `txt`. DKIM needs a selector: `/Lookup/DKIM/?argument={domain}:{selector}`.
- **Response shape:** JSON with `Failed`, `Warnings`, `Passed`, `Information` arrays, each
  entry `{Name, Info}`. That structure is stable enough to diff between runs.
- **Monitors:** `GET /api/v1/monitor` returns current state of all monitors on the account,
  filterable by command, domain, or tag. Useful if we buy hosted monitors and want their
  results in our own dashboard.
- **Quota:** `GET /api/v1/Usage` returns `DnsRequests`/`DnsMax` and
  `NetworkRequests`/`NetworkMax`. **DNS and Network requests are metered separately** —
  blacklist and SMTP checks burn the Network bucket, which is the tighter one. Limits reset
  daily at 00:00 UTC. Free accounts are reported at ~10,000 API credits/month (and a much
  smaller starter DNS allotment); paid plans raise both.

**Volume math to check before committing:** 50 domains × 8 checks/day = 400 requests/day
≈ 12,000/month, and roughly a quarter of those hit the Network bucket. Pull `/Usage`
first to see what the account actually allows, then size the check list to fit.

---

## The options

### Option A — Buy MXToolbox hosted monitoring
Least engineering: their monitors, their alerts, done in an afternoon.
Rejected as the primary approach on cost and tenancy — the per-domain pricing does not
survive contact with a full client list, and there is no client-facing reporting story.

### Option B — MXToolbox API + our own scheduler
We run a nightly job, call `/Lookup/*` per client domain, persist the JSON, diff against
the previous run, and alert only on deltas. Keeps MXToolbox's aggregated blacklist
coverage (100+ lists in one call) without paying per-domain monitor pricing.
Constraint: the daily request quota is the ceiling on how many domains × checks we can run,
and blacklist checks consume the scarcer Network bucket.

### Option C — DIY the cheap parts, buy only the hard parts
Most of what MXToolbox reports is *just DNS queries* and free to run ourselves:

- SPF, DKIM, DMARC, MX, A, TXT, SOA, PTR, DNSSEC → direct DNS lookups (`dnspython`,
  Node `dns/promises`). Unlimited, no vendor, no quota.
- SMTP reachability → open a socket to port 25/587, read the banner, check STARTTLS.
- TLS/cert expiry on mail and web hosts → same idea, free.
- Blacklist → DNSBL queries against the zone list directly. Free at low volume, **but**
  Spamhaus blocks queries from public resolvers and requires a Data Query Service key above
  free thresholds; several other lists have similar terms. This is the piece worth buying.

So: build the record/diff engine ourselves, and use the MXToolbox API only for blacklist
aggregation (and optionally SMTP diagnostics). Cheapest at scale, most code to write.

### Option D — A multi-tenant MSP platform instead

These are built for MSPs with per-client tenancy, white-labelling, and PSA ticketing. They
add **DMARC aggregate (RUA) reporting** — which MXToolbox is weak at — on top of the
monitoring scope above. RUA parsing tells us *who is sending as the client and whether it
passes*, which is different information from "is the record present," and it is the thing
clients actually get breached over.

| Vendor | MSP economics | Notable |
|---|---|---|
| **EasyDMARC** | Per-domain "pay-as-you-grow", monthly, no minimum commit or setup fee. **Not published — sales-gated.** | Cleanest UI, fastest onboarding, PSA integrations for ConnectWise / Autotask / HaloPSA / SyncroMSP |
| **DMARC Report** | 50% off list for partners (published) | White-label multi-tenant, dedicated onboarding |
| **Red Sift OnDMARC** | Flat-rate MSP pricing (predictable at scale) | API-first, Dynamic SPF |
| **PowerDMARC** | Channel partner program | White-label, deep API, 1,000+ partners |
| **Sendmarc** | Partner-first | Certified ConnectWise PSA integration, 90-day compliance guarantee |
| **dmarcian** | Lower price point | Strong source classification, good educational material |
| **Postmark DMARC** | Free | Weekly DMARC digest only, no other monitoring |

**Important scope correction:** EasyDMARC is *not* DMARC-only. It also does blacklist /
reputation monitoring and DNS change alerts, which means it covers most of the check table
below on its own — it is a plausible full replacement for the MXToolbox scope, not just a
complement to it. **Caveat to verify:** on the published *business* tiers, Reputation
(blacklist) Monitoring appears to be an Enterprise-tier feature. Confirm whether it is
included in MSP per-domain pricing or billed as an add-on — that single answer moves the
build-vs-buy decision.

Published EasyDMARC business pricing, for reference only (MSP pricing is separate and
quoted): Free tier; Plus ~$35.99/mo annual (~$44.99 monthly) for 2 domains; Premium
~$71.99/mo for 4 domains; Enterprise quoted. Third-party sources put full MSP client
packages at roughly $5,000–$15,000/yr depending on domain count — treat that as a very
loose signal, not a quote.

---

## Recommendation

**The decision hinges on one number: how many client domains, and what per-domain price we
can get quoted.** Get quotes from EasyDMARC, Sendmarc, and DMARC Report before writing code.
If a multi-tenant platform lands under roughly $1–2/domain/month with PSA ticketing
included, buying beats building — the platform covers the check table, feeds tickets into
our PSA automatically, and gives us a white-labelled client report we can actually sell.
Building only wins if per-domain pricing is prohibitive at our client count, or if we want
the checks MXToolbox and the DMARC vendors both skip (nameserver delegation, domain expiry).

**If we build: Option C as the core, Option B for blacklists, Option D on top for DMARC.**

1. Build a scheduled scanner in this repo. One row per client domain, one JSON snapshot per
   scan, diff-on-write.
2. Free DNS/SMTP/TLS checks run ourselves — no quota, no per-domain cost, scales to every
   client we have.
3. MXToolbox API for blacklist aggregation only, sized to the free/low tier quota. Start
   with a free key and read `/Usage` to confirm headroom before adding domains.
4. Alert on **change**, not on state — a domain that has been fine for 200 days should
   generate zero noise; an edited MX record at 2am should generate a ticket.
5. Route alerts into ClickUp as tasks (we already have that integration) plus email for
   anything urgent — blacklist listing, MX record change, mail server unreachable.
6. Add a DMARC aggregator later if we want to sell deliverability as a service rather than
   just monitor for breakage.

## Proposed scan set, per domain, per day

| Check | Source | Alert when |
|---|---|---|
| MX records | DNS | changed, or resolves to nothing |
| SPF (TXT) | DNS | changed, missing, >10 lookups, syntax invalid |
| DKIM (per known selector) | DNS | changed, missing, key length dropped |
| DMARC (TXT `_dmarc`) | DNS | changed, missing, policy weakened (reject→none) |
| A / AAAA / NS / SOA | DNS | changed |
| Nameserver delegation | DNS | registrar/NS change — often the first sign of a hijack |
| DNSSEC status | DNS | enabled→disabled |
| Mail host reachability | SMTP socket | banner fails, STARTTLS unavailable |
| TLS cert expiry (mail + web) | TLS handshake | <30 days to expiry |
| Blacklist (domain + sending IPs) | MXToolbox API | newly listed on any list |
| Domain expiry | RDAP/WHOIS | <60 days to expiry |

The nameserver-delegation and domain-expiry checks are not things MXToolbox monitors well
and are cheap for us to add — both are high-value catches for an MSP.

## Open questions before building

- How many client domains are in scope, and does that list live somewhere queryable
  (ClickUp, a CRM, a spreadsheet) or does it need to be maintained by hand?
- Do we know DKIM selectors per client, or do we need to probe common ones
  (`google`, `selector1`/`selector2`, `k1`, `default`, `s1`)?
- Do clients see this — white-labelled monthly report — or is it internal alerting only?
- Where does it run: a small always-on host, a cloud scheduler, or an existing server?
