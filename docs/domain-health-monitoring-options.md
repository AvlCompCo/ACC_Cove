# Client Domain & Email Health Monitoring — Options

Goal: run an automated scan (roughly daily) across all client domains covering DNS,
email authentication (SPF/DKIM/DMARC), MX/SMTP reachability and blacklist status, and
alert us when something **changes** — a record edited, an IP listed, a mail server down.

## Our parameters

- **Scale:** 10–20 client domains once fully onboarded
- **Retail:** ~$25/domain/month → **$250–500/mo revenue ($3,000–6,000/yr)**
- **PSA:** SuperOps
- **Ticketing:** SuperOps does inbound email parsing with multi-mailbox routing and
  automation rules on From / To / Description, so **any vendor that can send an alert email
  can create tickets for us**. Native PSA integration is a convenience, not a requirement —
  which is important, because no DMARC vendor integrates with SuperOps natively.

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

---

## Vendor economics at 10–20 domains

Cost modelled at 20 domains against $500/mo revenue. "Gross margin" is revenue minus
platform cost — before our labour.

| Vendor | Published model | Cost @ 20 domains | Per domain | Margin @ $25 | Blacklist | DNS change alerts |
|---|---|---|---|---|---|---|
| **DMARCeye** | Flat $4/domain/mo | **$80/mo** | $4.00 | **84%** | verify | verify |
| **DMARC Report** (MSP partner) | Framework $100/mo list, **50% partner** → $50; no per-domain fees on paid plans | **~$50–125/mo** | $2.50–6.25 | **75–90%** | verify | verify |
| **PowerDMARC** | Basic $8/mo (5 domains); alerts + reputation are premium/enterprise; partner pricing quoted | quoted | — | — | ✅ yes | ✅ yes |
| **EasyDMARC** | MSP per-domain, pay-as-you-grow, no minimum — **sales-gated** | quoted | — | — | ✅ yes (Enterprise tier on business plans — verify for MSP) | ✅ yes |
| **Red Sift OnDMARC** | Flat-rate MSP program; per-domain option for MSPs & orgs <250 users — amounts unpublished | quoted | — | — | verify | verify |
| **DMARCTrust** | Pro $49/mo (5 domains) + $12/extra domain | **$229/mo** | $11.45 | 54% | verify | verify |
| **dmarcian** | Enterprise $5,988/yr, up to 15 domains | **$499/mo** | ~$33 (at 15) | **negative** | verify | verify |

**dmarcian's published pricing costs more per domain than we plan to charge.** Only viable
if their partner discount is very deep — worth one email, not a pilot.

### The scope trap

Most of the cheap per-domain vendors are **DMARC-focused**. Our brief is broader — blacklist
status, DNS record change detection, MX/SMTP health. Only **PowerDMARC** and **EasyDMARC**
are confirmed to cover all three in one platform. Before shortlisting on price alone,
confirm for each vendor:

1. Does it monitor **blacklist/reputation**, or only DMARC aggregate reports?
2. Does it alert on **any DNS record change**, or only on DMARC/SPF/DKIM records?
3. Is either feature gated to a higher tier than the per-domain price quoted?

A $4/domain tool that only does DMARC leaves us still needing the MXToolbox API for
blacklists — which is fine (it is cheap at this scale), but it is two vendors and two alert
paths, not one.

---

## Recommendation

**Buy, don't build.** At 10–20 domains the entire revenue line is $3–6k/yr. Any meaningful
engineering time against that is a loss, and the ongoing maintenance — DNSBL zone churn,
false positives, alert plumbing — never stops. The custom scanner (Options B/C above) only
made sense at a much larger domain count. Shelve it.

**Shortlist to quote, in priority order:**

1. **PowerDMARC** — confirmed full scope (DMARC + blacklist/reputation + DNS change alerts),
   white-label, partner program with no contract commitment. Ask specifically what tier
   unlocks Alerts and Reputation Monitoring at partner pricing, since both are premium
   features on retail plans.
2. **EasyDMARC** — same confirmed scope, best-in-class UI and onboarding. Ask whether
   Reputation Monitoring is included in MSP per-domain pricing or an add-on. Note their PSA
   integration list (ConnectWise/Autotask/Halo/Syncro) does **not** include SuperOps, so we
   lose their headline MSP differentiator and fall back to email-to-ticket like everyone else.
3. **DMARC Report** — best published partner economics (50% off, no per-domain fees). Confirm
   the domain limit on the Framework tier and whether blacklist/DNS-change monitoring is in
   scope or DMARC-only.
4. **DMARCeye** — cheapest clean per-domain model at $4 flat with white-label client logins.
   Confirm scope beyond DMARC.
5. **Red Sift OnDMARC** — flat-rate MSP pricing is the most predictable structure as we grow
   past 20 domains. Worth a quote even if it loses today.

**Skip:** dmarcian on published pricing, and MXToolbox Delivery Center — 5-domain buckets at
$129/mo means 20 domains costs ~$516/mo, which erases the entire margin.

### Watch for

- **Minimums and floors.** Several partner programs are built for 50–200+ domain MSPs.
  At 10–20 we may be below the tier where partner pricing is offered at all — ask directly
  rather than assuming the published partner rate applies to us.
- **Per-domain vs per-plan.** DMARC Report's "no per-domain fees" is the most favourable
  structure at our size *if* the domain cap on the entry tier clears 20. Verify.
- **Email alert quality.** Since everything routes through SuperOps email parsing, the alert
  emails need consistent, parseable subjects/bodies to map cleanly to the right client.
  Ask each vendor for a sample alert email during the trial.

### If we build instead (not recommended at this scale)

**Option C as the core, Option B for blacklists, Option D on top for DMARC.**

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
