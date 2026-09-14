# Phase 1 spike — read-only live FBO catalog + seed fallback

**Status:** spike on `main` for Brian Only testing (Build **43+**).  
**Do not** attach this binary to App Store Connect App Review 1.0 / Build 12 (or retarget App Review Build 46). Parallel with 1.0 review only.

## Product rulings (Phase 1 cut)

| # | Ruling | Status |
| --- | --- | --- |
| 1 | Keep **public mirror** + **jsDelivr** as CDN | Done |
| 2 | **Automate mirror sync** after `build-catalog-api` | Done — see [Mirror sync automation](#mirror-sync-automation) |
| 3 | Keep static **`v1`** as lasting CDN fallback | Done (committed under `docs/v1/` + mirror) |
| 4 | No edge auth | Done (public read-only JSON) |

**Out of scope now:** Catalog bot creation, Phase 2 writes, App Review binary retarget.

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
| Hosting | **Public mirror repo** `TheBerhart/wxstationpro-fbo-catalog` + **jsDelivr** | App repo is **private** (jsDelivr/Pages blocked). Mirror is **$0**, HTTPS, no Worker |
| Source in app repo | `docs/v1/` + `fbo-data/build-catalog-api.js` | Keep generating here; sync to public mirror after rebuild |
| Future | Cloudflare Worker + KV/R2, or GitHub Pages on a public host | Only if Catalog bot needs auth, filtering, or non-static merge |

**Rough cost:** ~$0/mo. No client write keys. No GitHub Pages on current private-repo plan.

**Ready for Catalog bot?** **Yes, with caveats** — read-only static export is ready to consume. Catalog bot can replace/augment generators later; keep `schema_version`, no reporter PII, Phase 0 GitHub reporting unchanged. Not ready for writes, claims moderation, or per-airport auth.

## API

Example (public, live):

```text
https://cdn.jsdelivr.net/gh/TheBerhart/wxstationpro-fbo-catalog@main/v1/airports/KAPA/businesses.json
https://cdn.jsdelivr.net/gh/TheBerhart/wxstationpro-fbo-catalog@main/v1/airports/KTAD/businesses.json
https://raw.githubusercontent.com/TheBerhart/wxstationpro-fbo-catalog/main/v1/airports/KAPA/businesses.json
```

Same payload also committed under private app `docs/v1/` for source control (lasting CDN fallback shape).

Response includes `schema_version`, `airport_ident`, `updated_at`, `etag`, and `businesses[]` with thin-seed fields (`name`, `phone`, `website`, `email`, `street_address`, `hours`, `freq`, `plus_code`, …).

Regenerate:

```bash
cd fbo-data && node build-catalog-api.js
# or: npm run build-catalog-api
```

## Mirror sync automation

### Trigger (preferred)

GitHub Action **`.github/workflows/sync-fbo-catalog-mirror.yml`** on private `WxStationPro`:

- Runs on **push to `main`** when any of these change:
  - `WxStationPro/Resources/FBOLocations/FBO-dataset.csv` (thin seed)
  - `fbo-data/build-catalog-api.js`
  - `fbo-data/sync-public-catalog-mirror.sh`
  - `docs/v1/**`, `docs/PHASE1-FBO-CATALOG-SPIKE.md`
  - the workflow file itself
- Also **`workflow_dispatch`** (manual re-sync).
- Steps: `node fbo-data/build-catalog-api.js` → `bash fbo-data/sync-public-catalog-mirror.sh` (push to mirror).

### Local / agent post-step

```bash
cd fbo-data && npm run sync-public-catalog-mirror
# optional dry-run:
FBO_CATALOG_MIRROR_DRY_RUN=1 npm run sync-public-catalog-mirror
```

Script builds `docs/v1/`, rsyncs into a clone of `TheBerhart/wxstationpro-fbo-catalog`, commits, and pushes.

### One-time secret (Brian)

Default `GITHUB_TOKEN` **cannot** push to another repo. Add repository secret on **`TheBerhart/WxStationPro`**:

| Secret name | Value |
| --- | --- |
| **`FBO_CATALOG_MIRROR_TOKEN`** | PAT or fine-grained token with **Contents: write** on `TheBerhart/wxstationpro-fbo-catalog` |

Until the secret exists, the Action fails with a clear error; local sync via `gh`/your login still works.

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

## Non-goals (this spike)

- No App Review binary retarget / submit (including Build 46).
- No Catalog bot creation / Phase 2 writes.
- No client write API keys.
- No reporter PII in catalog payloads.
- No edge auth / signed URLs.
- Phase 0 GitHub FBO reporting left as-is.
