## Why

`Swatinem/rust-cache` hard-codes GitHub's Azure-backed cache storage, making it unusable on self-hosted runners with S3-compatible object storage (RustFS/MinIO). Adding an S3 provider enables Rust CI caching on self-hosted firecracker runners with MMDS-injected credentials, without requiring a separate caching action.

## What Changes

- Add `src/custom/` S3 module (copied from `runs-on/cache@v4`) implementing `saveCache()` and `restoreCache()` against S3-compatible backends
- Extend `getCacheProvider()` in `src/utils.ts` to recognize `"s3"` as a valid provider
- Auto-detect S3 mode when `RUNS_ON_S3_BUCKET_CACHE` env var is present, overriding the default `"github"` provider
- Add AWS SDK dependencies (`@aws-sdk/client-s3`, `@aws-sdk/lib-storage`, `@aws-sdk/s3-request-presigner`)
- Update `action.yml` description to document S3 auto-detection behavior
- Rebuild `dist/` with bundled S3 support

## Capabilities

### New Capabilities
- `s3-cache-provider`: S3-compatible cache backend that plugs into the existing `CacheProvider` interface; auto-selected when `RUNS_ON_S3_BUCKET_CACHE` env var is set, otherwise the action behaves identically to upstream

### Modified Capabilities
<!-- None — all changes are additive. The existing github/buildjet/warpbuild providers are untouched. -->

## Impact

- **`src/utils.ts`**: new import + new `case "s3"` + auto-detection logic in `getCacheProvider()`
- **`src/custom/`**: three new files copied from `runs-on/cache@v4` (`cache.ts`, `backend.ts`, `downloadUtils.ts`)
- **`package.json`**: three new AWS SDK prod dependencies
- **`action.yml`**: updated `cache-provider` description
- **`dist/`**: must be rebuilt and committed; bundle size will increase due to AWS SDK
- No changes to `src/restore.ts`, `src/save.ts`, `src/config.ts`, or `src/cleanup.ts`
