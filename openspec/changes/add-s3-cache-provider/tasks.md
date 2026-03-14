## 1. Copy S3 module from runs-on/cache

- [x] 1.1 Clone `runs-on/cache@v4` to a temp dir and copy `src/custom/` into this repo
- [x] 1.2 Audit imports in `src/custom/cache.ts`, `src/custom/backend.ts`, `src/custom/downloadUtils.ts` and fix any broken paths

## 2. Add AWS SDK dependencies

- [x] 2.1 Install `@aws-sdk/client-s3`, `@aws-sdk/lib-storage`, `@aws-sdk/s3-request-presigner`

## 3. Modify src/utils.ts

- [x] 3.1 Add `import * as s3Cache from "./custom/cache"` at the top
- [x] 3.2 Add auto-detection logic: when `RUNS_ON_S3_BUCKET_CACHE` is set and provider is `"github"`, switch to `"s3"` and log
- [x] 3.3 Add `case "s3": cache = s3Cache; break;` to the switch statement

## 4. Update action.yml

- [x] 4.1 Update `cache-provider` input description to mention S3 auto-detection

## 5. Build and verify

- [x] 5.1 Run `npm run prepare` and fix any TypeScript errors
- [x] 5.2 Verify `dist/restore/index.js` and `dist/save/index.js` exist

## 6. Commit

- [ ] 6.1 Commit all changes including `dist/` with message `feat: add S3 cache provider support (runs-on/cache compatible)`
