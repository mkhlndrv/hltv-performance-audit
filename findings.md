# Findings — HLTV homepage

Grouped by area, corrective first within each. Each finding is a distinct,
independently observable problem — where several causes feed one symptom, they
sit under that finding. Numbers come from the PSI mobile Insights/Diagnostics, the
DevTools Network panel, and the throttled-mobile PSI column (Slow 4G + 4× CPU) —
see [baseline](baseline.md).

Context: at the 75th percentile HLTV already passes Core Web Vitals (see the
"Good" findings). The corrective findings are the low-end, slow-network,
cold-cache experience the lab profile exposes — the long tail, not the median
visit.

## Rendering

- The page is blank for about three seconds before anything renders.
  - **Baseline**: FCP (3.2 s mobile lab).
  - **Cause**:
    - Render-blocking resources in the critical path: a 2.3 MB stylesheet (`EverythingDay.css`, ~311 KiB brotli), the 563 KiB `hltv.js`, and `font-awesome.min.css`.
    - Two third-party scripts also block render: cadmus `script.ac` (2,660 ms) and the Cookiebot consent script (1,800 ms). No origins are preconnected.
    - PSI estimates ~10,180 ms of render-blocking savings.
  - **Solution**:
    - Inline the critical CSS and load the rest non-blocking; split `EverythingDay.css` so a page only loads what it needs.
    - Defer/`async` the scripts, load the consent script without blocking render, and add `preconnect` to the key origins.

- The main article image appears late.
  - **Baseline**: LCP (10.7 s mobile lab / 1.8 s field — field passes).
  - **Cause**:
    - The LCP hero image (`img-cdn.hltv.org`) is discoverable and not lazy-loaded, but has no `fetchpriority="high"`, so it competes for bandwidth instead of loading first.
    - It also sits behind the render-blocking chain above. The field LCP breakdown shows load-delay (745 ms) as the biggest component.
  - **Solution**:
    - Add `fetchpriority="high"` and a preload to the hero image, and preconnect to `img-cdn.hltv.org`.

- The page freezes when you scroll or tap while it's still loading.
  - **Baseline**: TBT (970 ms mobile lab); main-thread work 2.8 s.
  - **Cause**:
    - 5.6 MB of first-party JavaScript to parse and run — PSI flags 458 KiB of it unused and 43 KiB of unnecessary legacy polyfills.
    - Two scripts force synchronous reflows: cadmus `script.ac` (99 ms) and liftdsp's `admtracker` (171 ms).
  - **Solution**:
    - Code-split and defer non-critical JS, ship modern ES (drop the legacy transpilation), and remove the unused code.
    - Stop reading layout (`offsetWidth` and similar) right after writing to the DOM.

- A stack of third-party tag, consent and ad scripts loads on every visit.
  - **Baseline**: TBT and Speed Index (plus bytes and privacy).
  - **Cause**:
    - Google Tag Manager 490 KiB (186 ms), Cookiebot 316 KiB (284 ms), Facebook 164 KiB (135 ms) and cadmus 163 KiB (265 ms), plus Outbrain, Yahoo, Doubleclick and Google Analytics.
  - **Solution**:
    - Load tag-manager and analytics after first paint, load the consent script without blocking render, and drop or defer the ad/tracking calls that aren't needed at load.

## Networking

### Corrective

- Ranking images are downloaded far bigger than they're shown.
  - **Baseline**: image bytes (2.4 MB of the load); Speed Index / LCP.
  - **Cause**:
    - The image CDN returns WebP (good), but the ranking player photos come back at 400×417 for a 70×73 display — roughly 100 KiB wasted per photo across several.
  - **Solution**:
    - Request the photos at display size with responsive `srcset`/`sizes`.

- The top banner is a 624 KiB animated GIF.
  - **Baseline**: image bytes; Speed Index. PSI estimates ~425 KiB saveable.
  - **Cause**:
    - A single animated GIF is used for the top "day-only" banner instead of a video.
  - **Solution**:
    - Replace the GIF with a video (or a static image with the animation on interaction).

### Good

- Text compression is effective.
  - **Baseline**: network compression.
  - JS and CSS ship brotli/gzip (one script even zstd): 8.0 MB uncompressed comes down as 1.7 MB (~78 %). Images aren't gzipped, which is correct — they're already compressed.

- First-party caching is effective.
  - **Baseline**: network caching.
  - Static assets are content-hashed with a long TTL, so a soft refresh drops transfer from 5.3 MB to 1.3 MB (~75 % less).

- Images already use a modern format.
  - **Baseline**: image format.
  - The CDN serves WebP, so the remaining image issue is dimensions, not format.

## Mobile

Mobile is Google's primary ranking signal, and Slow 4G + 4× CPU is the condition
the audit weights first. HLTV's field data passes on mobile, so these are about
the throttled low-end — but that's the tail the lab exposes and the profile mobile
ranking is scored on.

### Corrective

- The same page is far slower on mobile than on desktop.
  - **Baseline**: throttled-mobile lab (Slow 4G + 4× CPU) vs desktop lab — Performance 39 vs 69, LCP 10.7 s vs 1.8 s (~6×), FCP 3.2 s vs 0.8 s, Speed Index 10.2 s vs 2.4 s, TBT 970 ms vs 410 ms.
  - **Cause**:
    - Mobile and desktop download almost the same bytes (5.2 MB vs 5.3 MB), so the gap isn't payload — it's the render-blocking chain (est. 10,180 ms; `hltv.js` 563 KiB / ~7,530 ms, `EverythingDay.css` 2.3 MB) and the JS main-thread work meeting a 4× slower CPU and a slow radio.
  - **Solution**:
    - The rendering fixes elsewhere in this report (inline critical CSS, split and defer the CSS/JS, `fetchpriority` on the hero) barely move the desktop score but are what rescue mobile — budget and test against Slow 4G + 4× CPU, not a desktop link.

- Mobile downloads desktop-sized images.
  - **Baseline**: images are 2.2 MB of the 5.2 MB mobile load; Speed Index / LCP.
  - **Cause**:
    - There's no responsive `srcset`, so a phone fetches the same full-size assets desktop does — the ranking photos at 400×417 (shown ~70×73) and the 624 KiB animated GIF banner — and on Slow 4G those image bytes dominate the load. (The oversized-image root is the networking finding above; the mobile-specific point is that nothing is downscaled for the small screen or the slow link.)
  - **Solution**:
    - Serve responsive `srcset`/`sizes` at display size and replace the GIF with a video or static image — the saving is largest on mobile.

### Good

- The layout is rock-stable on mobile.
  - **Baseline**: CLS 0 (field) / 0 (PSI lab) — well under the 0.1 threshold.
  - Despite the ad slots, ranking widgets and the banner, nothing shifts as the page loads on a phone. Layout stability is the one thing HLTV nails on mobile; the problem is load speed, not jumpiness.

## Overall

### Good

- Real users already have a good experience.
  - **Baseline**: Core Web Vitals (field / CrUX).
  - HLTV passes Core Web Vitals at the 75th percentile on both mobile and desktop (mobile LCP 1.8 s, INP 96 ms, CLS 0). The corrective findings above are the low-end and slow-network tail, not the median visit.
