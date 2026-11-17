# Prior art

What already exists. Verified as of November 2026 by reading the repositories and docs; star
counts and activity are what they were at that time and will drift.

## Jev-based, ad-adjacent

**[jev-vpn](https://github.com/newuser7171/jev-vpn)** — the closest thing to this idea.
A Windows-only "privacy tunnel & smart ad/tracker shield": a local HTTP CONNECT proxy that
pairs static lists (EasyList, AdGuard, StevenBlack) with a *"live AI Zero-Day Ad Classifier"*
that sends unknown domains to Jev (via `classifier.dev`) and gets back a category —
`advertisement_banner`, `behavioural_tracker`, `telemetry_analytics`, `malware_malvertising`.
This is precisely the "Jev classifies domains that lists missed" pattern, done at the proxy
layer instead of DNS.

Status: a sketch. Created September 2026, one star, no releases, no tests. Worth reading for
the category taxonomy and for how it answers unknown domains in an interactive path; not
worth depending on.

**[typesafe-adblock](https://github.com/realZachi/typesafe-adblock)** — 97★ Chrome extension
that asks Jev *"is this DOM element an ad?"* per element, turning ad blocking into a stream of
typed questions. Different layer (client-side DOM, not DNS) and therefore the complement
rather than the competitor: it is the only approach in this list that can reach first-party
ad slots, at the cost of living inside a browser.

## DNS-based, not Jev

**[dns-smart-block](https://github.com/LoganBarnett/dns-smart-block)** — the architectural
blueprint for this repo, built with a general LLM instead of Jev. Rust, ~128 commits.
It watches DNS logs, queues domains, fetches content from them, classifies with an LLM, stores
`domain_classification_events`, and serves the result as a blocklist with Prometheus metrics.
If you want to see the shape of the pipeline — including the boring parts (queues, retries,
metrics, migrations) — read this one first.

**[MazeDNS](https://github.com/amaze-labs/MazeDNS)** — the turnkey alternative. A self-hosted
DNS filtering resolver (a peer of AdGuard Home and Pi-hole, not an add-on) with domain
classification built in: a 5-million-domain category database, a 0–100 legitimacy score for
newly seen domains from threat feeds, WHOIS age, reputation services and an **optional AI
model**, and a review queue where risky domains wait for a human. "Nothing is sent to a third
party unless you turn it on." AGPL-3.0, actively developed (commits in November 2026).

Relevant caveat: adopting it means replacing the resolver, not extending it, and its AI
classification is a generic feature rather than something you can point at Jev.

**[ai-sinkhole](https://github.com/aimledu/ai-sinkhole)** — research framework: an
agent-augmented DNS sinkhole that dynamically discovers and classifies domains from exported
query logs. Academic in shape (scripts, papers, a pipeline you assemble), useful for the
method rather than for production.

**[AdVig](https://huggingface.co/saidutta69/AdVig)** — the opposite end of the spectrum: a
16 KB int8 model that blocks ads at the DNS layer offline, small enough for a NodeMCU. No
cloud, no runtime lists. Interesting as an existence proof that "tiny classifier at the DNS
layer" is a real design point — and as a reminder that a 16 KB model and a hosted System One
model are answering very different questions.

## Method precedents

**[DoltHub — "Working with Jev"](https://www.dolthub.com/blog/2026-11-12-working-with-jev/)**
(November 2026). Not ad blocking: they classify IP addresses in a sales-lead pipeline. But the
method is the one this repo proposes — enrich with IPinfo (ASN, organisation, location) and
reverse DNS, then let Jev make the typed call, keeping the state compact and asking one
question per criterion. The most useful thing to copy is the discipline, not the domain.

**[awesome-jev](https://github.com/Li-Evan/awesome-jev)** — community index of Jev use cases
(moderation, routing, guardrails, browsers). Worth scanning before inventing a prompt schema;
several entries are classification pipelines with published accuracy numbers.

## The baseline to beat

**[Hagezi DNS blocklists](https://github.com/hagezi/dns-blocklists)** — curated DNS blocklists
(ads, trackers, telemetry, phishing, malware) in tiers from "Light" to "Ultimate", plus
allowlists for the inevitable collateral damage. This is not prior art for the idea; it is the
incumbent. Any dynamic classifier has to demonstrate value *on the domains these lists do not
already cover*, which is a much smaller set than it first appears.

## Summary

| Project | Layer | Uses Jev | Maturity |
|---|---|---|---|
| jev-vpn | HTTP proxy (Windows) | yes | sketch |
| typesafe-adblock | browser DOM | yes | hobby, 97★ |
| dns-smart-block | DNS log → blocklist | no (LLM) | real pipeline, unmaintained? |
| MazeDNS | DNS resolver | generic AI | active product |
| ai-sinkhole | DNS sinkhole | no | research |
| AdVig | DNS, on-device | no | curiosity |
| Hagezi | DNS lists | no | the incumbent |

Nobody has published a Jev-based DNS classifier with measured results. That is the gap — and
also the warning: the absence of one after a year of Jev use elsewhere suggests the yield did
not justify the pipeline for anyone who tried.
