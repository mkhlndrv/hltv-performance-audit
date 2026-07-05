# Findings — HLTV homepage

Five findings from the [baseline](baseline.md), ordered by how much they hold the
page back. The numbers come from the PageSpeed Insights mobile report's Insights
and Diagnostics for https://www.hltv.org/ (3 July 2026). The red metrics all
describe the same slow load, but each finding is a distinct cause with its own
fix.

## 1. Render-blocking scripts and CSS delay the whole page

**Users:** on mobile the screen stays blank for about three seconds, and nothing
— including the main image — can paint until these files finish loading.

**Metrics:** First Contentful Paint (3.2 s) and Largest Contentful Paint
(10.7 s). PSI's "render-blocking requests" insight estimates up to 10,180 ms of
savings here — the single biggest lever on the page.

**Cause (measured):** the initial render waits on a 563.5 KiB first-party script
(`hltv.js`, ~7,530 ms on the throttled profile), a 303.5 KiB stylesheet
(`EverythingDay.css`), and `font-awesome.min.css`, plus two third-party scripts
sitting in the critical path — cadmus `script.ac` (81 KiB, 2,660 ms) and the
Cookiebot consent script (37 KiB, 1,800 ms). No origins are preconnected.

**Solution:** defer or `async` the render-blocking scripts, inline only the CSS
the top of the page needs and load the rest non-blocking, and load the consent
script without blocking render. Adding `preconnect` to the key origins is worth
~530 ms of LCP on its own per PSI.

## 2. The first-party JavaScript bundle is large and partly legacy

**Users:** the page freezes for close to a second on a mid-range phone while
scripts run, right when it looks ready to use.

**Metrics:** Total Blocking Time (970 ms mobile). PSI puts main-thread work at
2.8 s — script evaluation alone is 1,439 ms — and JavaScript execution at 1.8 s.
Third-party scripts are much of it: Cookiebot 284 ms, cadmus 265 ms, Google Tag
Manager 186 ms, Facebook 135 ms.

**Cause (measured):** the first-party bundle is heavy and wasteful — PSI flags
458 KiB of unused JavaScript, ~230 KiB of it in `hltv.js` alone — and it still
ships 43 KiB of unnecessary legacy polyfills and transforms (babel
transform-classes/spread/regenerator, `Array.prototype` polyfills). Two scripts
also force synchronous reflows — cadmus `script.ac` (99 ms) and liftdsp's
`admtracker` (171 ms).

**Solution:** stop transpiling to legacy JavaScript and ship modern ES,
code-split and defer the non-critical parts of the bundle, and move third-party
tags behind first paint. Fix the forced reflows by not reading layout
(`offsetWidth` and similar) right after writing to the DOM.

## 3. The LCP image isn't prioritized

**Users:** the featured-article hero image is what defines LCP — it appears
around 1.8 s for median users but up to 10.7 s on the slow, cold-cache profile.

**Metrics:** Largest Contentful Paint.

**Cause (measured):** PSI identifies the LCP element as
`<img class="hero-image">` served from `img-cdn.hltv.org`. It is already in the
HTML and not lazy-loaded (both good), but `fetchpriority="high"` is not applied,
so it competes for bandwidth instead of loading first — and it sits behind the
render-blocking chain in finding 1. The CrUX field breakdown agrees: load delay
(745 ms) is the largest part of real-user LCP.

**Solution:** add `fetchpriority="high"` to the hero image and preconnect to
`img-cdn.hltv.org` so it starts early. Clearing the render-blocking from finding
1 then lets it paint as soon as it arrives.

## 4. Images are oversized and one banner is a 624 KiB GIF

**Users:** on Slow 4G the images saturate the connection, so text and pictures
fill in slowly.

**Metrics:** Speed Index (10.2 s) and LCP. PSI's "improve image delivery"
estimates 1,157 KiB of savings.

**Cause (measured):** the top "day-only" banner is a 624.8 KiB animated GIF
(~425 KiB saveable by using a video format). The ranking-widget player photos are
served at 400×417 but displayed at 70×73 — roughly 100 KiB wasted per player
across several of them. The images are PNG/GIF rather than WebP/AVIF and aren't
sized for their display box.

**Solution:** replace the GIF with a video or a static image, serve the player
photos at display size with responsive `srcset`/`sizes`, and switch to WebP/AVIF.
The images are already lazy-loaded, so this is about size and format, not timing.

## 5. A stack of third-party tag, consent and ad scripts

**Users:** tag-manager, consent, analytics and ad/tracking scripts all load
during the initial page load, competing for the CPU and the connection and adding
to the freeze.

**Metrics:** Total Blocking Time and Speed Index; the consent script also hits
FCP and LCP because it is render-blocking.

**Cause (measured):** the largest third parties by transfer are Google Tag
Manager 490 KiB (186 ms main thread), Cookiebot consent 316 KiB (284 ms,
render-blocking), Facebook 164 KiB (135 ms) and cadmus `script.ac` 163 KiB
(265 ms), followed by Outbrain, Yahoo, Doubleclick, Google Analytics and several
ad/tracking pixels.

**Solution:** load Tag Manager and analytics after first paint, load the consent
script without blocking render, and drop or defer the ad/tracking pixels that
aren't needed at load. Fewer tag managers means fewer bytes and less main-thread
time.
