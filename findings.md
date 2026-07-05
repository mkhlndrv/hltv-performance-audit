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
across several of them. The photos come back from the image CDN as WebP, so the
format is fine — they just aren't sized for their display box, and the banner is
a raw animated GIF.

**Solution:** replace the GIF with a video or a static image, serve the player
photos at display size with responsive `srcset`/`sizes`. The images are already
lazy-loaded and already WebP, so this is about dimensions, not format or timing.

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

---

## Networking

Numbers from the DevTools Network panel (desktop, fresh load then soft refresh —
see [baseline](baseline.md) → Network Activity). HLTV is light for a media site
(208 requests, 5.3 MB), and two things are already done well: text compresses
~78 % (js/css 8.0 MB → 1.7 MB via brotli/gzip/zstd), and caching cuts a soft
refresh to 1.3 MB (~75 % less). Images are already WebP. So the problems here are
a couple of oversized payloads, not request sprawl.

## 6. One stylesheet is enormous

**Users:** the browser downloads and parses ~2.3 MB of CSS (uncompressed) before
the first render can complete — dead time in front of a blank screen on a slow
connection.

**Metrics:** First Contentful Paint (this CSS is render-blocking); part of the
5.3 MB transfer.

**Cause:** one global stylesheet (`EverythingDay.css`) ships ~2.3 MB uncompressed
(~311 KB over the wire as brotli) instead of page-specific CSS — the same file
flagged as render-blocking in finding 1.

**Solution:** split the CSS and load only what the homepage needs up front; defer
the rest.

## 7. The JavaScript payload is heavy

**Users:** 5.6 MB of uncompressed JavaScript across 43 files to download, parse
and run — the main reason the page freezes (finding 2).

**Metrics:** Total Blocking Time; JS transfer 1.4 MB (5.6 MB uncompressed).

**Cause:** a large first-party bundle plus the ad and tracking scripts. PSI
already flagged 458 KiB of unused JS and 43 KiB of legacy polyfills inside it.

**Solution:** code-split and defer the non-critical JS, drop the legacy
transpilation, and cut the third-party scripts that don't need to run at load.

## 8. Images are the biggest download, and some are oversized

**Users:** images are 2.4 MB of the 5.3 MB — the largest single content type —
and a few are far bigger than the box they're shown in, so mobile users pay for
pixels they never see.

**Metrics:** Speed Index and LCP; image transfer 2.4 MB.

**Cause:** the format is already right — the CDN returns WebP — so the waste is
dimensions: the ranking photos come back at 400×417 for a 70×73 display, and the
top banner is still a 624 KB animated GIF (finding 4).

**Solution:** request the photos at display size with responsive `srcset`/`sizes`,
and replace the GIF banner with a video.
