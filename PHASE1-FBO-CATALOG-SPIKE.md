# Phase 1 spike — read-only live FBO catalog + seed fallback

**Status:** spike on `main` for Brian Only testing (Build **42+**).  
**Do not** attach this binary to App Store Connect App Review 1.0 / Build 12. Parallel with 1.0 review only.

## Success criteria

1. `GET /v1/airports/{ident}/businesses` returns thin-seed-shaped rows for test airports (**KAPA**, **KTAD**).
2. iOS path: **live → disk cache → bundled seed**; failure/offline still opens Airport Info.
3. Weather cold-open unchanged — FBO fetch never gates METAR/TAF.
4. `etag` / `updated_at` present; stale badge stubs: soft **>7d**, stronger **>30d** or **seed-while-online**.
5. This stack note for Product / Catalog bot handoff.

## Chosen stack

| Layer | Choice | Why |
| --- | --- | --- |
| Source of truth | Thin seed CSV (`Resources/FBOLocations/FBO-dataset.csv`) | Already ships; field-aligned |
| Generator | `fbo-data/build-catalog-api.js` | One command; deterministic ETag hash |
| Hosting | **Static JSON on GitHub** via **jsDelivr CDN** (`cdn.jsdelivr.net/gh/...@main/docs`) | **$0**, no Worker/account, works right after push; HTTPS |
| Optional alt | GitHub Pages (`theberhart.github.io/WxStationPro`) | Also free; enable when Product wants a first-party host |
| Future | Cloudflare Worker + KV/R2 | Only if Catalog bot needs auth, filtering, or non-static merge |

**Rough cost:** ~$0/mo for spike traffic. jsDelivr/GitHub bandwidth is free at this scale. No client write keys.

**Ready for Catalog bot?** **Yes, with caveats** — read-only static export is ready to consume. Catalog bot can replace/augment generators later; keep `schema_version`, no reporter PII, Phase 0 GitHub reporting unchanged. Not ready for writes, claims moderation, or per-airport auth.

## API

Example (jsDelivr, after push to `main`):

```text
https://cdn.jsdelivr.net/gh/TheBerhart/WxStationPro@main/docs/v1/airports/KAPA/businesses.json
https://cdn.jsdelivr.net/gh/TheBerhart/WxStationPro@main/docs/v1/airports/KTAD/businesses.json
```

REST-shaped Pages paths (same payload):

```text
/docs/v1/airports/{IDENT}/businesses.json
/docs/v1/airports/{IDENT}/businesses/index.json
/docs/v1/airports.json
```

Response includes `schema_version`, `airport_ident`, `updated_at`, `etag`, and `businesses[]` with thin-seed fields (`name`, `phone`, `website`, `email`, `street_address`, `hours`, `freq`, `plus_code`, …).

Regenerate:

```bash
cd fbo-data && node build-catalog-api.js
# or: npm run build-catalog-api
```

## iOS wiring

- `FBOLiveCatalogClient` — feature-flagged (`settings.liveFBOCatalog`, default **on**); optional base URL override `settings.liveFBOCatalogBaseURL`.
- `FBODataAggregator` / `AviationBusinessService.discoverServicesFast` try live catalog first, then existing discovery, then catalog disk cache, then bundled seed.
- Airport Info still paints from bundled/cache immediately; online refresh is async (does not gate METAR/TAF).
- Disable live catalog: `UserDefaults` `settings.liveFBOCatalog = false` (or unreachable URL — degrades safely).

## How to test offline fallback

1. Open **KAPA** or **KTAD** Airport Info online once (live catalog populates disk + services cache).
2. Enable Airplane Mode / Low Data, force-quit, reopen Airport Info — listings still appear from disk cache / bundled seed.
3. Clear app data and open offline — bundled thin seed still shows (KAPA has 4 FBOs; KTAD has Pinnacle).
4. With live URL unreachable and no disk cache — seed path only; weather still loads.

## Product / hosting questions

1. Prefer **jsDelivr-from-main** vs enabling **GitHub Pages** as the canonical public base URL?
2. When Catalog bot lands, should static `docs/v1` stay as CDN fallback or become build artifact only?
3. Any need for signed URLs / edge auth before public listing of phone/email already in the shipped CSV?

## Non-goals (this spike)

- No App Review binary retarget / submit.
- No client write API keys.
- No reporter PII in catalog payloads.
- Phase 0 GitHub FBO reporting left as-is.
