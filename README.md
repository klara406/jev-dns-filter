# jev-dns-filter

Design notes for a **dynamic DNS-layer ad classifier**: take the query log of a self-hosted
AdGuard Home instance, enrich each not-yet-filtered domain with context, ask
[TypeSafe Jev](https://docs.typesafe.ai/introduction) for a typed verdict, and append the
high-confidence ones to a generated blocklist.

Status: **design notes and an evaluation plan. Nothing is implemented.** The point of this
document is to be honest about what such a system would buy and where it breaks, before
anyone writes the pipeline.

---

## 1. The idea

```
AdGuard Home query log
        │  (domains not matched by any current rule)
        ▼
   enrichment  ──►  one Jev request  ──►  typed verdict + confidence
 PTF/ASN/WHOIS/          (state = JSON)         │
 CNAME chain/TLS cert                           ├─ confidence ≥ θ  → generated blocklist
 HTTP HEAD/title                                └─ otherwise      → review queue
```

Two properties make the shape work:

* **Jev is a decision model, not a text generator.** You send a `state` (text, JSON or
  images) plus typed questions and get typed answers back — `Noul` (is this true, 0–1),
  `Choice` (pick one option), `Score` (rate on a rubric) — each with probabilities and,
  for most types, a `confidence`. There is no output to parse.
* **Questions run in parallel against one state.** Adding criteria barely costs latency, so
  the pipeline can ask five or ten independent questions about the same domain in one call.

It is not in the DNS resolution path. DNS has to answer in milliseconds; every LLM call is
asynchronous and happens after the fact. This is a *curation* system, not a filter.

---

## 2. What DNS filtering can and cannot reach

This boundary decides whether the project is worth doing, so it comes first.

| Ad kind | Example | DNS-blockable? |
|---|---|---|
| Third-party ad/tracker endpoint | `ads.example-net.com`, pixel hosts, telemetry | **Yes.** A domain of its own to block. |
| CNAME-cloaked tracker | `metrics.first-party.com` → `tracker.vendor.net` | **Partly.** The CNAME target is visible in resolution; the first-party alias is not blocked without breaking the site. |
| First-party ad slot | In-feed ads in social/media apps; ad units served from the site's own API | **No.** Same domain as content. Blocking it blocks the product. |

The last row is the important one: in-feed ads in apps like Pinterest are rendered from the
app's own API response, so there is no separate hostname for a blocklist — dynamic or static
— to catch. Getting at those needs client-side HTTPS interception, which is a different
project with different trade-offs (certificates, pinning, breakage).

**Consequence:** a Jev-classified DNS blocklist can only ever improve the *third-party* row,
where public lists are already very strong. That caps the expected gain, and it is why this
design is framed as a supplement with measurement attached.

---

## 3. Central question 1 — what would Jev classify on?

### 3.1 What a DNS query log actually contains

Per record: the queried name, qtype, the client that asked, the answer (IPs), whether it was
cached/blocked/allowed and which rule matched, and a timestamp.

Not present: the referring page, the URL path, request headers, cookies, or any payload. The
query log is a list of hostnames with metadata — nothing about *why* the client asked.

That single fact drives everything below. From the name alone, a classifier is doing what
blocklists already do, only slower and less reliably.

### 3.2 Signals you can attach, and what each costs

| Signal | How | Value to the classifier | Cost / risk |
|---|---|---|---|
| The name itself, labels, TLD, token shape | free (already in the log) | weak-ish; catches obvious patterns (`ads-`, long random labels, `-tracker`) | none |
| Query frequency and client spread | aggregate the log | moderate: a domain 40 devices hit once each is different from one device hitting it hourly | none |
| Resolved IPs | already in the log | the pivot for everything else | none |
| Reverse DNS (PTR) | resolver | moderate; ad nets often have uninformative PTR | fast, cheap |
| ASN + organisation | RDAP (no key) or IPinfo/Team Cymru | **strong**: "hosted at an ad-tech ASN/cloud range" is a real signal | external call, rate limits |
| Domain age / registrar | WHOIS or RDAP | **strong**: newly registered is the classic malicious/ad-rotation marker | rate limits, GDPR-redacted records |
| CNAME chain | explicit resolution of the name | **strong**: reveals cloaking that lists can't enumerate per-site | needs your own resolution |
| TLS certificate (issuer, SAN, reuse) | handshake or CT logs (crt.sh) | **strong**: cert reuse across ad domains is a good cluster signal | handshake may fail; CT logs are historical |
| HTTP HEAD / body title / `ads.txt` | fetch the domain | **strong when it works**, useless when not — see 3.3 | bandwidth, cloaking, legal/abuse optics |
| First-seen in CT logs | crt.sh | moderate: brand-new certs correlate with rotation | external call |

Everything beyond the first row is **state you must build**. The classifier is the cheap
part; the enrichment is the project.

### 3.3 Can content be pulled from the domain?

Sometimes, and the failure modes are systematic rather than random:

* **Ad endpoints are not websites.** Many answer `204`, `302`, or an empty body to anything
  that isn't a real browser with the right cookies and fingerprint.
* **Cloaking is normal.** Ad infrastructure commonly serves benign or blank content to
  crawlers and datacenter IPs, and the payload only to residential browser clients. A fetch
  from a server box sees a different (deliberately uninformative) site than a phone does.
* **JS-rendered walls.** Where content does exist, it is frequently assembled client-side, so
  a plain GET returns a shell. A headless browser gets closer, at much higher cost, and is
  itself detectable.
* **First-party ambiguity.** Without a referer you cannot tell whether a domain was loaded as
  a script on someone else's site or navigated to directly. This matters: the same hostname
  can be legitimate when top-level and suspicious when embedded.

So: content fetch is worth having as an *optional* signal, weighted as "evidence if
available", never as the primary basis. A pipeline that requires a successful body fetch will
silently degrade to name-only classification on exactly the domains that matter most.

### 3.4 Practical answer

Classify on a **JSON state combining name + network identity + registration + infrastructure
signal**, with content as an optional extra. The domain string alone is a weak feature; the
decisive ones are ASN/organisation, domain age, CNAME chain and certificate reuse. The
`state` fits comfortably in Jev's 32k-token window — a few hundred tokens of JSON per domain
— so a single request can carry the whole dossier.

---

## 4. Central question 2 — is this better than a static blocklist?

### 4.1 Where a dynamic classifier genuinely wins

* **The long tail of new domains.** Rotating ad and tracker domains exist for weeks. A list
  you subscribe to updates daily at best; a classifier can judge a domain the first time it
  appears on your network.
* **CNAME cloaking.** A per-site alias pointing at a known tracker vendor is nearly invisible
  to community lists (they would have to enumerate every site's alias). It *is* visible in
  your own resolution data.
* **Per-client policy.** "Analytics allowed on the work laptop, not on the phone" is trivial
  for a classifier with client context, and awkward for a global list.
* **Explainability.** Each decision carries a rationale and a confidence, so a review queue
  can show *why* something was flagged — lists give you a category, not a reason.
* **Your own traffic as the corpus.** The candidate set is what your devices actually queried,
  not a global list of 200k domains you will never see.

### 4.2 Where a static list wins outright

* **Latency.** A compiled list is a trie lookup in microseconds. Anything model-shaped has to
  be offline, which means it can never protect the request that revealed the domain.
* **Institutional memory.** Years of crowd curation, curated allowlists and breakage reports.
  A classifier has none of that, and its false positives are silent: an app just stops
  working and nobody knows why.
* **Determinism.** A list is a versioned, diffable, reproducible artifact. A model output can
  drift when the model or the prompt changes — in a system you rely on for network filtering,
  that is a serious property, not a nitpick.
* **Availability and cost.** Lists work offline and during outages. Enrichment means external
  APIs; classification means a hosted model.
* **Adversarial inertia.** Lists are compiled by many people over time; a single classifier
  can be probed and adapted to, and its verdicts are only as good as the features you feed it.
* **Coverage.** For mainstream third-party ad domains, the good public lists (Hagezi tiers,
  AdGuard DNS filter) are already close to saturated. The marginal domain is rare by
  construction.

### 4.3 Honest verdict

The advantage is real but **narrow**: newly rotated domains, CNAME cloaking, per-client
policy, and an auditable curation loop. It is not "a smarter blocklist" that replaces lists;
it is a low-yield, high-auditability supplement that must be measured before being trusted.
Any claim that a classifier beats curated lists on mainstream ad traffic should be treated as
unproven until the evaluation in [`docs/evaluation.md`](docs/evaluation.md) says otherwise.

---

## 5. Pros and cons at a glance

**Pros**

* Catches domains no list has seen yet — the first appearance is exactly when it matters.
* Reasons over infrastructure (ASN, cert, CNAME), which is where the durable signal lives.
* Typed verdicts with confidence make an automatic review queue possible, instead of binary
  hashes that are wrong without explanation.
* Cheap at household volume: on Cloudflare Workers AI, `typesafe/jev` is billed at
  $0.042 per 1M input tokens with free output, so a whole week of discovered domains costs
  fractions of a cent.
* Runs beside an existing AdGuard Home (it has an API for query log, rule set and filter URLs),
  so nothing has to be replaced to try it.

**Cons**

* Cannot touch first-party/in-feed ads — the motivating case in most homes.
* The engineering is in enrichment, not classification: RDAP/WHOIS, ASN, CNAME, certs,
  optional fetches, caching, rate limits, retries.
* False positives break apps quietly; every automated block needs an owner and a rollback.
* Model decisions are less auditable than a versioned list, and less stable across model
  versions.
* Requires shipping your query log (i.e. your household's browsing signals) to third-party
  APIs for enrichment and classification.
* Sustains a review queue, which is human work in perpetuity — the opposite of "set and
  forget".

---

## 6. Why it might not work

Ranked by how likely each is to kill the project:

1. **The motivating problem is out of scope.** In-app and in-page ads come from first-party
   domains. No DNS-layer system reaches them, dynamic or not. If that is the goal, stop here.
2. **Enrichment collapses on the domains that matter.** Ad endpoints return blanks and cloak
   against datacenter IPs; the pipeline degrades to name-only classification for exactly the
   unknown domains it was built for.
3. **Marginal yield is tiny.** Lists already cover the overwhelming majority of third-party ad
   queries. The measurable gain may be a handful of domains per month, against a permanent
   maintenance cost.
4. **False positives are worse than ads.** A wrongly blocked domain breaks a login flow, a
   payment, or a smart-home device — the user notices the breakage, not the missing ad.
5. **Non-determinism.** Two runs over the same domain should give the same answer; a hosted
   model with evolving versions does not guarantee that, and a filtering system that changes
   its mind is hard to debug.
6. **Privacy cost.** Query logs are browsing history. Sending them to enrichment and
   classification APIs is a real disclosure, even with zero-retention terms, and it must be a
   deliberate decision rather than a side effect.
7. **Operational drag.** Cron job, API keys, review queue, rollback path, monitoring. All of
   it must keep working for the benefit to persist.

Any of 1–3 alone is enough to make this a "measure first, build maybe" project.

---

## 7. If it is built anyway: the minimum credible version

1. **Shadow mode only.** Classify 30 days of query-log history, write to a log file, block
   nothing. Compare against the union of the current filter lists.
2. **Measure yield and precision** on the domains the lists miss — see
   [`docs/evaluation.md`](docs/evaluation.md). If the candidate set is small or precision is
   poor, the project ends there, cheaply.
3. **Only then** enable a generated filter list, at high confidence only, capped in size per
   day, with a one-line rollback and a named owner.
4. Treat every automated rule as probationary: re-check after 30 days, drop what nobody
   missed.

No part of this design requires blocking anything to be useful — the first two steps are
read-only, and they answer the only question that matters.

---

## 8. Documents

| File | Contents |
|---|---|
| [`docs/design.md`](docs/design.md) | Pipeline stages, the Jev request schema, AdGuard Home API surface, guardrails |
| [`docs/evaluation.md`](docs/evaluation.md) | Shadow-mode plan, ground truth, precision/yield/coverage metrics, kill criteria |
| [`docs/prior-art.md`](docs/prior-art.md) | Prior art: Jev-based and DNS-based projects that already exist |

## 9. References

* TypeSafe Jev — [introduction](https://docs.typesafe.ai/introduction) ·
  [primitives](https://docs.typesafe.ai/primitives/choice) ·
  [models](https://docs.typesafe.ai/models)
* Jev on Cloudflare Workers AI — [`typesafe/jev`](https://developers.cloudflare.com/ai/models/typesafe/jev/)
* AdGuard Home API — [`openapi.yaml`](https://github.com/AdguardTeam/AdGuardHome/blob/master/openapi/openapi.yaml)
* Hagezi DNS blocklists — <https://github.com/hagezi/dns-blocklists>
