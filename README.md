# Handoff: Pipeline Navigator Congress – iOS offline data loss

## Context
- App: **Pipeline Navigator Congress**, Angular + NgRx / @ngrx/data, Nx monorepo, iOS via Cordova (WKWebView, `scheme=app`, `hostname=localhost`).
- Repo: `pipeline-navigator-congress-fork` (Bitbucket). iOS app: `apps/pipeline-navigator-cordova-mobile`. Bundle ID `com.novartis.pipeline-navigator-congress-ios`.
- **Already resolved, do not re-investigate:** network/VPN/middleware reachability and MSAL login/redirect URI config.

## Current problem
Offline data package is lost or partial on iOS:
- Download loads for a while, then times out / spins.
- After closing and reopening the app, data is missing or partial.
- After the first kill the app re-downloads everything; later it works.
- "Cache invalidation not working correctly."
- Requirement: data must survive app restart **and logout**.

## Findings from reviewing the pre-loader code
Files: `libs/pipeline-navigator/offline-mode/data-access-pre-load/src/lib/state/effects/` (`pre-loader.effects.ts`, `pre-loader.effects.base.ts`, `oncology.pre-loader.effects.ts`, `pharmacology.pre-loader.effects.ts`, `rolling-download.helpers.ts`) and `libs/pipeline-navigator/auth/data-access-cross-divisional-access/src/lib/state/effects/package-expiration.effects.ts`.

1. **Any download error deletes the saved package.** `clearCacheOnDownloadError$` invalidates the disk cache, calls `entityCacheDispatcher.clearCollections()`, and resets `lastDownloaded`. `handleDownloadError` dispatches that error for every failing request, including each image download. A re-download also overwrites the cache key early with shared data only (`fetchSharedDataAttempted$`). So there's no safe fallback.
2. **A partial package is loaded on cold start.** `initDataFromCache$` loads the cache key without checking `lastDownloaded`. Between the shared-data write (`fetchSharedDataAttempted$`) and the full write (`cacheDispatchers`), the key holds shared data only.
3. **Top-level `catchError` completes effects for the session:** `loadDataFromCache$`, `dataFetchSuccess$`, `oncologyDataFetchAttempted$` (outer `handleDownloadError` + `catchError`), `pharmaDataFetchAttempted$` (same). After one error, retries do nothing until the app restarts.
4. **A failed image makes the download hang.** `downloadMoaForOfflineView(..., this.handleDownloadError(NEVER))` returns `NEVER`, and `recursiveFetch` waits for completion, so `dataLoadedToEntityCacheSuccess` never fires.
5. **Invalidation is never subscribed.** `forceDownloadPackage$` calls `diskCacheService.invalidateCacheEntry(...)` inside `tap` without subscribing. If `DiskCacheService` returns a lazy observable, nothing is invalidated.
6. Smaller issues:
   - `cacheDispatchers`: `[...(cachedDispatch as EntityAction[]), ...dispatch]` throws when `cachedDispatch` is null.
   - `updatePackageDownloadInformation` (marks the package downloaded) is dispatched before images/MOA are fetched.
   - `fetchDataAttempted$` uses `withLatestFrom(forceDownloadPackage)`, so it's silently dropped until a `forceDownloadPackage` action has been dispatched in this session.
   - Package expiry deletes the package and logs out before new data is downloaded. Check `isExpired` (device clock?).

## Tasks
1. **Confirm first.** Read and report on:
   - `DiskCacheService`: storage backend (localforage driver? IndexedDB / localStorage / SQLite), size limits, whether `invalidateCacheEntry` / `cacheItem` are lazy or eager
   - `isExpired` (`util-package-tools`)
   - `PackageCacheUtilsService.updatePackageInformation`, and when `lastDownloaded` is actually persisted
   - logout effects: does logout clear the disk cache?
   - whether `ContentUpdateAPIActions.forceDownloadPackage` and `CrossDivisionalActions.forceDownloadPackage` are the same action (line ~477 dispatches the latter after every load)
   - what dispatches `CrossDivisionalActions.forceDownloadPackage` on startup
2. **Then fix, with small focused diffs:**
   - Download into a temp key (e.g. `key + '__tmp'`). After everything (incl. images) succeeds: write the final key, set `lastDownloaded`, delete the temp key. On error, delete only the temp key and keep the old package.
   - On cold start, load only packages marked complete.
   - Move `catchError` / `handleDownloadError` inside the `switchMap` / `exhaustMap` so effects survive errors.
   - Replace `NEVER` for image failures with a completing value (e.g. `of(null)`), and collect failures for a later retry.
   - Subscribe to `invalidateCacheEntry` (`concatMap`), and guard `cachedDispatch ?? []`.
   - If storage is localStorage-based, propose moving the package to IndexedDB or SQLite (`cordova-sqlite-storage`) or files in `cordova.file.dataDirectory`.
3. Add unit tests for: an error during an update keeps the old package; a kill mid-download doesn't load a partial package; retry works after an error.

## Second problem: app startup takes 4–7 minutes
First establish whether this happens on **every** launch (means the cache isn't being used and the full package is re-downloaded) or only on first launch / after an update (means the download design is slow).

Why the download is slow, from the code:
- **One HTTP request per item.** The download makes one request per compound card and per trial card (`getCompoundRequests` / `getTrialRequests`), plus per tumor type / modality / pair at levels 2–3, per combination partner and cohort, **4 MOA requests per card** (MOA Visual + Partner Logo, for compounds and trials), then every image separately. That's likely hundreds to thousands of requests over VPN.
- **Low, uneven parallelism.** `convertRequestsToRecursiveMultiThreadRequests` runs 3–5 sequential chains (`recursiveFetch`). Total time is roughly (requests ÷ threads) × latency.
- **Retries and slow failures.** `retry(2)` on every request, with no explicit `timeout`, so one slow or failing call can cost minutes.
- **Possibly a token request per API call.** Web Inspector showed `GetToken` several times. Check whether the token interceptor caches the token until `exp`.
- **Images block startup.** The app isn't usable until all images are downloaded.
- **Loading from cache dispatches each cached action separately** (`loadDataFromCache$` `forEach(store.dispatch)` + `setLoadingStage` per action).

Tasks:
1. Measure a cold start and a warm start: number of requests, total time, slowest endpoints, how many `GetToken` calls.
2. Make warm starts load from cache only, and check for updates in the background.
3. Propose a bulk package endpoint in the middleware (one gzip JSON per division + version/ETag), or a prebuilt package file.
4. Until then: use `mergeMap` with concurrency 6–10 instead of `recursiveFetch` chains; add `timeout()` + `retry({ count: 2, delay })` with backoff; cache the token; download images in the background after the app is usable.
5. Load the cache with one action (`EntityCacheDispatcher.loadCollections` / `mergeQuerySet`) instead of hundreds of dispatches.
6. Check that gzip, keep-alive and HTTP/2 are enabled on the middleware / IIS.

## Constraints
- Don't change expiry/logout semantics without confirming. Expiry may be a compliance rule.
- Don't commit or push without asking.
- Don't put tokens or keys in code, commits or chat.
