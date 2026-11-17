# Evaluation plan

The project may not be worth building. This file exists to find that out for the cost of an
evening, before any blocking is enabled.

## Principle: shadow mode first, and possibly only

Two read-only stages, both useful on their own:

1. **Replay.** Take 30 days of query log, run the candidate filter and the classifier over it,
   write every decision to a file. Block nothing.
2. **Measure.** Compare against the current filter lists. If the numbers in §3 are bad, the
   project ends here and the only artifact is this repo.

Only if the numbers justify it does anything get written to AdGuard Home.

## 1. Ground truth

There is no public label set for "domains my household should block", so build a small one:

* **Sample, don't label everything.** Draw a stratified sample: all candidates the classifier
  flagged with high confidence, plus a random sample of those it did not.
* **Label from evidence, not vibes.** For each sampled domain, record: ASN/organisation, domain
  age, CNAME chain, certificate SAN reuse, whether the name appears in any public list, and a
  human verdict (`ad`, `tracker`, `telemetry`, `legit`, `unknown`). Two independent passes on a
  50-domain overlap is enough to catch a sloppy rubric.
* **Reuse what is already labelled.** Membership in Hagezi Pro/Ultimate or the AdGuard DNS
  filter is not ground truth, but it is a strong prior and it is what the dynamic layer must
  beat to be interesting.

## 2. Metrics

| Metric | Definition | Why it matters |
|---|---|---|
| **Candidate yield** | distinct not-yet-filtered domains per day | if this is ~0, there is nothing to classify |
| **Precision** | of domains auto-blocked, share labelled `ad`/`tracker`/`malvertising` | the only number that determines whether blocking is safe |
| **Recall on labelled set** | of labelled ad domains the lists miss, share the classifier catches | the actual value over lists |
| **Coverage gain** | share of ad/tracker *queries* caught by dynamic layer that lists missed | yield × frequency; domains nobody queries are worthless |
| **False-positive rate** | wrong blocks per 100 blocks | target: < 1 |
| **Breakage incidents** | user-visible breakage attributable to a generated rule | the cost side of precision |
| **Review load** | queue items per week | if this exceeds ~15/week, the design is wrong |
| **Cost** | enrichment calls + model tokens per week | should be near zero; a surprise here means a bug |

Report precision with a confidence interval. A 95% precision claim from 20 blocks is
arithmetically meaningless.

## 3. Kill criteria

Decide these **before** looking at results, so the answer cannot be rationalised afterwards:

* candidate yield < 10 domains/day → **stop** (nothing to work with; lists are saturated)
* precision < 95% at a usable threshold → **stop** (a filtering layer that is wrong 1 in 10
  times is worse than ads)
* coverage gain < 1% of ad/tracker queries → **stop** (real but irrelevant)
* review load > 15 items/week sustained → **stop** (it is now a part-time job)
* enrichment succeeds on < 30% of candidates → **stop** (it has degraded to name-only
  classification, which lists already do better)

Any single stop is enough. Continuing past it needs a written reason.

## 4. Experiment list (cheap, ordered)

1. **Candidate census.** No classifier at all: how many distinct domains per day are neither
   blocked nor allowlisted, and how many of them have ≥ 5 sightings in 7 days? This alone
   might answer the whole question.
2. **List-diff.** Of those candidates, how many appear in Hagezi Pro? That is the true
   uncaptured set.
3. **Enrichment hit rate.** For 200 candidates: what fraction yield ASN, WHOIS age, CNAME
   chain, and a fetchable HTML body? Records which evidence is realistically available.
4. **Cloaking test.** Fetch the same 200 from a residential connection and from a datacenter
   host; diff status codes and body sizes. Establishes whether content fetch is worth having.
5. **Classifier pass.** Run Jev over the enriched 200, hand-label, compute precision. This is
   the first moment the model is involved, and by then most of the answer is already known.

Steps 1–4 need no model and no external AI cost — only local queries and public lookups.

## 5. What "good" would look like

A defensible positive result is narrow and specific, e.g.:

> Over 30 days, 34 domains not present in any subscribed list were flagged at confidence ≥ θ;
> hand review confirmed 33 as third-party trackers, including 11 CNAME-cloaked first-party
> aliases; zero user-visible breakage. Cost: $0.00 in model tokens, ~40 enrichment calls/day.

Anything vaguer than that ("it found lots of trackers") is not a result.
