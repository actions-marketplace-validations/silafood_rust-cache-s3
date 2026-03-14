## Context

`Swatinem/rust-cache` uses a `CacheProvider` abstraction in `src/utils.ts` that wraps `@actions/cache` (Azure), `@actions/buildjet-cache`, or `@actions/warpbuild-cache`. All providers expose the same interface: `isFeatureAvailable()`, `restoreCache()`, `saveCache()`. Selection is driven by the `cache-provider` input, defaulting to `"github"`.

`runs-on/cache@v4` provides a battle-tested S3 module (`src/custom/`) that replicates the `@actions/cache` API surface over S3-compatible backends using AWS SDK v3. It supports presigned uploads, concurrent chunked downloads, and arbitrary endpoints.

Self-hosted firecracker runners inject `RUNS_ON_S3_BUCKET_CACHE` (and related env vars) via MMDS. No workflow changes should be required for projects already using this pattern.

## Goals / Non-Goals

**Goals:**
- Drop-in S3 backend satisfying the existing `CacheProvider` interface
- Auto-detect S3 mode via `RUNS_ON_S3_BUCKET_CACHE` env var — zero workflow changes for self-hosted runners
- Allow explicit opt-in via `cache-provider: s3` input
- Preserve identical behavior when env var is absent (no regression for GitHub-hosted runners)
- Ship a self-contained `dist/` that works without `npm install` in CI

**Non-Goals:**
- Modifying the Rust-specific cache key logic (`src/config.ts`, `src/cleanup.ts`)
- Supporting additional S3 operations beyond save/restore
- Adding new action inputs for S3 configuration (env vars are sufficient)
- Supporting `@actions/cache` API versions older than what upstream uses

## Decisions

### D1: Copy `src/custom/` verbatim from `runs-on/cache@v4`

Copy the three files (`cache.ts`, `backend.ts`, `downloadUtils.ts`) rather than writing a new adapter from scratch. The `runs-on/cache` module is production-tested and correctly handles multipart upload, presigned URLs, and chunked downloads. Only import-path fixes are expected.

**Alternative considered**: Write a thin npm-package adapter. Rejected — no published package exists; would require git submodule or manual copy anyway.

### D2: Auto-detect via env var, fallback gracefully

In `getCacheProvider()`, check `process.env["RUNS_ON_S3_BUCKET_CACHE"]` before processing the `cache-provider` input. If set and provider is still `"github"` (default), upgrade to `"s3"` and log a message.

**Alternative considered**: Require explicit `cache-provider: s3` in workflow YAML. Rejected — defeats the goal of zero-config for self-hosted runners.

### D3: S3 config via environment variables only

Do not add new action inputs for bucket, endpoint, region, etc. All S3 config comes from env vars (`RUNS_ON_S3_BUCKET_CACHE`, `RUNS_ON_S3_BUCKET_ENDPOINT`, `AWS_ACCESS_KEY_ID`, etc.) already injected by the runner infrastructure.

### D4: Commit `dist/` to the repository

The built `dist/restore/index.js` and `dist/save/index.js` must be committed. GitHub Actions resolves actions from the repository at the referenced tag/SHA. `ncc` bundles all dependencies including the AWS SDK into single files.

## Risks / Trade-offs

- **Bundle size increase** → AWS SDK v3 adds ~2–4 MB to `dist/`. `ncc` tree-shakes unused modules; acceptable for self-hosted runners.
- **Type mismatch between `@actions/cache` versions** → `runs-on/cache@v4` may have been written against an older `@actions/cache` API. The `GhCache` interface requires `isFeatureAvailable`, `restoreCache`, `saveCache`. Fix: inspect and align types; add stubs if needed.
- **Import path breakage in copied files** → `src/custom/` may reference internal helpers (e.g., `../utils/actionUtils`). Fix: audit imports at copy time; inline missing helpers.
- **Upstream divergence** → Merging upstream updates may conflict in `src/utils.ts`. Mitigated by keeping changes minimal (one import + one `case` + detection block).

## Migration Plan

1. Copy `src/custom/` from `runs-on/cache@v4`
2. Fix any import paths in copied files
3. Install AWS SDK dependencies
4. Modify `src/utils.ts` (import + auto-detect + new case)
5. Update `action.yml` description
6. `npm run prepare` — fix any TypeScript errors
7. `git add -A && git commit`
8. Push and tag `v2`

**Rollback**: Force-move the `v2` tag back to any prior commit.
