# Design sketch

Nothing here is implemented. This is the shape a pipeline would take if the evaluation in
[`evaluation.md`](evaluation.md) justifies building it.

## Constraints that shape the design

1. **Never in the resolution path.** DNS answers in milliseconds; a model call is 10s–1000s of
   milliseconds plus network. Classification is asynchronous, post-hoc, and only ever affects
   *future* queries.
2. **Idempotent and cheap to re-run.** Enrichment results are cached by domain; a second run
   over the same day costs almost nothing.
3. **Fail open.** If enrichment or classification fails, the domain is *not* blocked. Silence
   must never mean "block".
4. **Every automated block is traceable** to a decision record: state, questions, verdicts,
   confidence, model version, timestamp. Without that, a false positive is unfixable.

## Stages

```
 ┌────────────────┐   ┌──────────────┐   ┌────────────┐   ┌───────────┐   ┌──────────┐
 │ 1 collect      │──►│ 2 filter      │──►│ 3 enrich   │──►│ 4 decide  │──►│ 5 apply  │
 │ query log      │   │ candidates    │   │ (cached)   │   │ via Jev   │   │ + review │
 └────────────────┘   └──────────────┘   └────────────┘   └───────────┘   └──────────┘
```

### 1. Collect

`GET /control/querylog?limit=…&offset=…` (paginate with `older_than` or `offset`), or
`GET /control/stats` for aggregates. Persist to local SQLite: `(domain, client, qtype, ips,
status, time)`.

The query log is the only source of truth about what actually happened on the network. It
contains no referer and no path — see the README, §3.1.

### 2. Filter candidates

Drop a domain early if any of these hold, in this order:

* it is already blocked or allowed by a current rule — `GET /control/filtering/check_host?name=…`
* it appears in the manual allowlist, or is a parent/child of a manually allowlisted name
* it is a known-infrastructure name (captive portal checks, NTP, OCSP, OS update hosts, CDNs
  used for content)
* it has fewer than *N* sightings in *M* days (default: 5 in 7) — weak signal, not worth a call
* it was classified in the last *K* days with unchanged enrichment (cache hit)

### 3. Enrich

Build the `state` object. Each field is optional; missing values are omitted rather than
guessed, so the classifier can see what evidence exists.

| Field | Source | Notes |
|---|---|---|
| `domain`, `labels`, `tld` | log | free |
| `parent_domain` | log | lets the model reason about first-party vs third-party |
| `sightings_7d`, `clients_7d`, `qtypes` | log | free |
| `answers` | log | IPv4/IPv6 of last answer |
| `asn`, `asn_org`, `country` | RDAP / Team Cymru DNS | cache 7 days |
| `ptr` | resolver | often uninformative; include only if present |
| `whois_created`, `whois_registrar` | RDAP | cache 30 days |
| `cname_chain` | explicit resolution | **most valuable single signal for cloaking** |
| `cert_issuer`, `cert_sans`, `cert_first_seen` | TLS handshake / crt.sh | cluster signal across domains |
| `http_status`, `page_title` | optional HEAD/GET | only if the response is HTML and small |
| `ads_txt_present` | `/.well-known/` or `/ads.txt` | a real ad seller publishes one |

Cache all enrichment in SQLite keyed by `(domain, field)` with per-field TTLs. Enrichment is
the expensive part; never repeat it inside its TTL.

### 4. Decide

One request per domain, several questions in parallel. Shape as accepted by the Workers AI
binding for `typesafe/jev`:

```jsonc
{
  "state": {
    "domain": "metrics.some-site.example",
    "parent_domain": "some-site.example",
    "sightings_7d": 214,
    "clients_7d": 3,
    "asn": 64500, "asn_org": "EXAMPLE-ADTECH", "country": "DE",
    "cname_chain": ["metrics.some-site.example", "elb.tracker-vendor.net"],
    "cert_issuer": "Let's Encrypt", "cert_sans": ["*.tracker-vendor.net"],
    "whois_created": "2026-10-02", "http_status": 204
  },
  "questions": {
    "is_ad_or_tracker": {
      "type": "noul",
      "instructions": "Is this endpoint third-party advertising or behavioural tracking infrastructure?",
      "criteria": {
        "true":  "CNAME to a known tracker vendor, ad-tech ASN, or ad/tracker naming, with no plausible end-user service",
        "false": "Consistent with a genuine service, CDN, or the site's own analytics it needs to function"
      }
    },
    "category": {
      "type": "choice",
      "instructions": "Which category best describes it, if it is not a legitimate service?",
      "criteria": {
        "display_ad": "Ad delivery or ad auction endpoints",
        "tracker": "Cross-site behavioural tracking, pixels, fingerprinting",
        "telemetry": "First-party product analytics or crash telemetry",
        "malvertising": "Malicious or scam redirect infrastructure",
        "legitimate": "A service the user or application actually needs"
      }
    },
    "confidence_note": {
      "type": "score",
      "instructions": "How strong is the evidence in the state?",
      "criteria": ["No evidence", "Weak", "Circumstantial", "Strong", "Conclusive"]
    }
  }
}
```

Decision rule, deliberately conservative:

```
block  iff  is_ad_or_tracker ≈ 1
        AND category ∈ {display_ad, tracker, malvertising}
        AND score ≥ "Circumstantial"
        AND no allowlist conflict
        AND daily_added < cap
```

Everything else goes to the review queue with its rationale. `telemetry` is never
auto-blocked: it is the category most likely to break an app for no user-visible benefit.

### 5. Apply

Two write paths in AdGuard Home:

* **Preferred — generated filter list.** Write rules to a file, serve or reference it, and
  register it once with `POST /control/filtering/add_url` (a URL or an absolute file path),
  then `POST /control/filtering/refresh` when it changes. The list can be regenerated
  wholesale, diffed in git, and disabled with a single toggle.
* **Avoid — `POST /control/filtering/set_rules`.** This *replaces* the entire set of custom
  rules, so every write must be a read-modify-write of `GET /control/filtering/status`. One
  bug here wipes a hand-written rule set.

Generated rules carry a marker comment group so they are trivially separable from manual
rules:

```
! jev-dns-filter: generated <timestamp> — do not edit by hand
||tracker-vendor.net^
```

## Guardrails

* **Cap** new blocks per day (start with 5). A sudden burst is a bug, not a discovery.
* **Probation.** Newly added rules expire after 30 days unless re-confirmed.
* **Kill switch.** One config flag stops classification and empties the generated list; the
  resolver keeps working.
* **No blocking of a parent domain** when only a subdomain was seen — collapsing to
  `||parent^` is the fastest way to break a site.
* **Breakage feedback.** Any client-side "site is broken" report should be able to name the
  rule that caused it and disable it in one step.

## Cost and load

At household volume (say 20–60k queries/day, of which tens are new candidates), one Jev call
per candidate with a few hundred tokens of JSON input is a rounding error: `typesafe/jev` on
Workers AI is billed at $0.042 per 1M input tokens with free output and zero data retention.
The real cost is enrichment API calls (RDAP is free, IPinfo's free tier is rate-limited) and
the time to build it.

## Open technical questions

1. Which enrichment field is actually decisive? Start with `cname_chain`, `asn_org`,
   `whois_created` and prove the rest add value — or cut them.
2. Do ad endpoints cloak against datacenter IPs in practice? Test by fetching the same
   candidate set from a residential connection and from a server, and diff the responses. If
   they differ, `http_*` is worthless as designed.
3. Can Jev separate `tracker` from `telemetry` reliably enough to act on? If not, drop
   `telemetry` from the choice set entirely and just never auto-block it.
4. What confidence threshold gives < 1 false positive per 100 blocks on a labelled sample?
5. How do you keep decisions reproducible across model versions — pin the model id
   (`typesafe/jev` 1.13) and store it with every decision record?
