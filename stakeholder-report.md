# Stakeholder report — hltv.org performance

This is the decision layer of the audit, written for the people who fund and
schedule work rather than the people who implement it. It says what the
performance problems cost, what fixing them buys, and in what order. The
implementer layer — mechanisms, exact fixes, effort basis — is
[implementer-notes.md](implementer-notes.md), with the findings catalog in
[findings.md](findings.md); every claim here traces to them by finding ID
(F-01…F-15) with matching numbers, and the capture conditions are in
[baseline.md](baseline.md). Measurements: July 2026.

Intended use: a ranked improvement plan. One thing up front, because it
frames every number below: nothing in this report is an emergency. The site
passes Google's user-experience bars for real users today. This is about who
it still fails, what that pass is costing, and what quietly threatens it.

## The two-minute version

hltv.org is healthy at the median and heavy at the tail. The typical reader
gets the main content in well under two seconds; a reader on a mid-range
phone and a slow connection waits five times longer — and that reader is not
an edge case, it's the arena crowd on congested mobile networks during a
major, exactly when traffic and ad impressions peak.

Three numbers carry the situation:

- **1.8 s vs 10.7 s** — main content for the median mobile reader (Google's
  field data; the bar for "good" is 2.0 s) versus the same page in Google's
  standard mobile stress test (slow 4G, mid-range phone, 39/100). The site
  passes at the median with 0.2 s of headroom and collapses at the tail.
- **99%** — the share of the 2.3 MB stylesheet that the homepage downloads
  and never uses. The 3 MB main script is 42% unused, and rendering waits
  for it. This waste rides along on every visit and grows with every
  feature, because everything lands in the same two files.
- **624 KiB** — one animated GIF banner at the top of the homepage, the
  single heaviest asset swap available; two-thirds of it comes off with a
  format change.

The ask, in order:

| # | Purchase | What it buys | Price |
|---|----------|--------------|-------|
| 1 | Image delivery package (F-01–F-04) | The hero image loads first, the GIF becomes a video, phones stop downloading desktop-sized photos | Days; one front-end engineer |
| 2 | Edge caching for finished content (F-08) | News articles and finished-match pages served from the CDN edge instead of a round-trip to origin | Config-scale, plus one freshness decision |
| 3 | Front-end build rework (F-06, F-10, F-11) | The 2.3 MB stylesheet and 3 MB script cut down to what each page uses; the mobile stress numbers move; the field pass is protected as the site grows | Multi-week; next quarter's engineering investment |

If this is all you read: fund 1 now — it is days of work and covers the
entire mechanically-cut priority tier. Approve the freshness decision behind
2. Schedule 3 before growth turns today's stress-test numbers into real
readers' numbers. And note what none of this touches: revenue partners. The
weight here is our own code, which means it is entirely ours to fix.

## What is already strong

We measured before judging, and the list of strengths is unusually long:

- **The site passes Core Web Vitals in the field, on mobile and desktop.**
  Median mobile: main content at 1.8 s, input response 96 ms, zero layout
  shift. Most sites this content-heavy don't pass. Everything below is
  about protecting and extending that, not rescuing it.
- **The layout never moves.** Layout shift is 0.00 in field and lab — no
  jumping content, despite ads, banners and live widgets.
- **Delivery is modern where it counts**: images come as WebP from a
  resizing CDN, text ships brotli/gzip (one script already uses zstd,
  ahead of the curve), and static assets are fingerprinted for caching.
- **The rendering machinery is clean.** Pages are server-rendered (content
  shows even with scripts disabled), there's no framework hydration tax,
  and the compositing layer is lean — scrolling isn't blocked by handlers.

## The four priorities

All fifteen corrective findings are scored with WSJF — Cost of Delay ÷ Job
Size; scales documented in [findings.md](findings.md). Because the site
passes in the field, time-criticality is deliberately scored low across the
board: this ranking sorts improvements, not fires. The top quarter by score
— four of fifteen, cut mechanically — are the priorities, and all four turn
out to be the same kind of work: image delivery.

| ID | Priority | Score |
|----|----------|:-----:|
| F-01 | The main image has no loading priority | 18.0 |
| F-02 | The top banner is a 624 KiB animated GIF | 5.0 |
| F-03 | Phones download desktop-sized photos | 4.3 |
| F-04 | Ranking photos arrive ~30× larger than their box | 3.7 |

**F-01 — The main image has no loading priority (score 18.0).** The hero
image is in the page from the start but carries no signal telling the
browser to fetch it first, so it queues behind scripts and styles; waiting
in that queue is the largest measured component of its load time. *Risk:*
the field pass on this metric has 0.2 s of headroom, and Google tightened
the threshold this spring — this is the metric to protect. *Cost:* the
cheapest fix in the report — an attribute and a preload; hours to days.
*Opportunity:* the stress-test main-content time (10.7 s) takes its single
biggest available step down, and the median reader gets faster too.

**F-02 — The top banner is a 624 KiB animated GIF (score 5.0).** One
decorative asset is an eighth of the mobile page weight. *Risk:* pure cost —
every visit, every reader, metered plans included. *Cost:* a contained
format swap (video or animated AVIF), a day. *Opportunity:* the heaviest
single-asset saving available, with no visible change.

**F-03 — Phones download desktop-sized photos (score 4.3).** The photo
pipeline can resize on demand, but pages don't tell it to — a phone gets
the same files a desktop does; images are 2.2 MB of the 5.2 MB mobile load.
*Risk:* wasted bytes exactly where connections are worst. *Cost:* a
template pass wiring sizes into the existing image CDN; days.
*Opportunity:* the mobile payload drops with zero design impact.

**F-04 — Ranking photos arrive ~30× larger than their box (score 3.7).**
Player photos are delivered at 400×417 pixels for a 70×73 slot — roughly
100 KiB of waste per photo, several per page. Same fix family as F-03,
same template pass, immediate byte savings.

Two findings sit just under the mechanical line at 3.6: the third-party
loading pass (F-05 — the tag/consent scripts load earlier than they need
to; a small policy fix, not a vendor cut) and the theme stylesheet (F-06 —
funded inside purchase 3 rather than as a separate ask).

## The structural problem, priced separately

**The front-end build ships everything to everyone — and rendering waits
for it** (F-06, F-10, F-11; the render-path findings F-09, F-13, F-14 are
the same problem measured three ways).

The evidence, stated so it can be checked: one 2.3 MB stylesheet of which
the homepage uses about 1%; one 3 MB script, 42% unused, that the browser
must download and run before rendering; the whole blocking chain costs an
estimated 10.2 seconds in the mobile stress test. Mobile and desktop
download almost identical bytes (5.2 vs 5.3 MB) — the mobile collapse is
this chain meeting a 4× slower processor, which is why it is fixable in
code rather than in infrastructure.

*Risk while it stands:* every new feature lands in the same two files, so
the stress-test numbers (39/100, 10.7 s) drift toward the median reader
over time, eating the 0.2 s of field headroom. Today's pass is not
automatically tomorrow's.

*Cost:* honest answer — a multi-week build rework: splitting the
stylesheet and script so each page ships what it uses, and extracting the
above-the-fold CSS so rendering stops waiting. Entirely in-house; no
vendor contract or revenue conversation involved. A gradient exists:
defer-and-extract first (weeks, most of the render-path win), full
per-route splitting after (F-15's long-term shape).

*Opportunity:* the mobile stress score is where competitors' sites are
beatable; more practically, it is the insurance premium on the field pass
— paid once, while the codebase is still one team's size.

## What we are not asking for

- **No rendering-architecture change.** Server-rendering is verified per
  page type and it is the right choice — it's what earns the field pass.
- **No vendor cuts.** The third-party stack is light by industry standards
  (a tag manager, consent, analytics pixels, one ad script); F-05 asks to
  load it later, not to remove it.
- **No urgency theater.** Real users pass today. This is maintenance
  investment and is priced as such — the ranking's time-criticality
  scores say so in writing.

## The three objections we expect

- **"We pass Google — why spend anything?"** The pass has 0.2 s of headroom
  on a threshold Google just tightened, the failing tail is the live-event
  mobile audience at peak commercial moments, and the structural waste
  grows monotonically. Cheap now beats forced later.
- **"Esports fans will wait."** At home on fiber, yes. During a major
  they're on congested venue and mobile networks — the stress-test profile
  — and a 10.7 s wait for scores is when second-screen readers give up and
  check a competitor or Twitter instead.
- **"Surely it's the ads."** Measured: it isn't. The third-party stack is
  a few hundred KiB; the weight is our own stylesheet and script. That's
  good news — no partner negotiation stands between us and the fix.

## Check the numbers yourself

- Type `hltv.org` into **pagespeed.web.dev**. One screen shows the whole
  thesis: the field panel green (real users pass), the mobile lab score
  39/100 (the tail doesn't). Both are Google's numbers, not ours.
- On the homepage, open the top banner image in its own tab — the browser
  will show it's an animated GIF weighing ~0.6 MB.
- In this repo, `evidence/05-bundles/` holds the coverage capture: the
  2.3 MB stylesheet with ~99% of it unused on the page that loaded it.

## What a yes looks like

An **owner** for the image package; the **freshness decision** for edge
caching minuted ("what actually breaks if a finished-match page is five
minutes stale?" — in numbers, per route); the **build rework** on next
quarter's plan. Verification is fast here: lab numbers move the day a fix
ships and can be re-run publicly; the field panel just has to stay green
month over month.

## Method note

Scores are WSJF: Cost of Delay (user value + time criticality +
risk/opportunity, each 1–10) divided by Job Size (1–10), documented with
scales in [findings.md](findings.md). Soft numbers are visible in the
table itself — time-criticality sits low everywhere because the field
passes, and the third-party finding's high risk/opportunity score marks a
risk reduction, not a measured saving. Lab numbers are Google's standard
mobile stress test (slow 4G, 4× CPU); the device-lab run is WebPageTest on
an iPhone 15 over 4G; field numbers are Google CrUX at the 75th
percentile. One capture constraint is documented rather than hidden: the
site's bot protection blocks automated local Lighthouse runs, so the
scored lab bench is PSI (Google's infrastructure), which it admits. Full
conditions: [baseline.md](baseline.md).
