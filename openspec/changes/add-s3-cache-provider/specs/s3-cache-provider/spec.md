## ADDED Requirements

### Requirement: S3 provider satisfies CacheProvider interface
The `s3` cache provider SHALL export `isFeatureAvailable()`, `restoreCache()`, and `saveCache()` with signatures compatible with the `GhCache` interface defined in `src/utils.ts`.

#### Scenario: Interface compatibility
- **WHEN** `getCacheProvider()` returns the `s3` provider
- **THEN** calling `isFeatureAvailable()`, `restoreCache()`, and `saveCache()` on it SHALL NOT throw type errors at compile time

### Requirement: Auto-detection via environment variable
When `RUNS_ON_S3_BUCKET_CACHE` is set in the environment and `cache-provider` input is at its default value of `"github"`, the action SHALL automatically select the `s3` provider and log an informational message.

#### Scenario: Auto-selection when env var present
- **WHEN** `RUNS_ON_S3_BUCKET_CACHE` is set AND `cache-provider` input is `"github"`
- **THEN** `getCacheProvider()` SHALL return a provider with `name === "s3"` and log `"S3 cache bucket detected, using S3 cache provider"`

#### Scenario: No auto-selection when env var absent
- **WHEN** `RUNS_ON_S3_BUCKET_CACHE` is not set AND `cache-provider` input is `"github"`
- **THEN** `getCacheProvider()` SHALL return a provider with `name === "github"`

### Requirement: Explicit opt-in via input
The action SHALL accept `cache-provider: s3` as an explicit input value to select the S3 provider regardless of environment variables.

#### Scenario: Explicit S3 selection
- **WHEN** `cache-provider` input is set to `"s3"`
- **THEN** `getCacheProvider()` SHALL return a provider with `name === "s3"`

### Requirement: Existing providers unaffected
When S3 auto-detection does not trigger and `cache-provider` is `"buildjet"` or `"warpbuild"`, behavior SHALL be identical to the upstream action.

#### Scenario: Non-default provider unchanged
- **WHEN** `cache-provider` input is `"buildjet"` and `RUNS_ON_S3_BUCKET_CACHE` is set
- **THEN** `getCacheProvider()` SHALL return a provider with `name === "buildjet"` (auto-detection only overrides the `"github"` default)

### Requirement: S3 config via environment variables
The S3 provider SHALL read its configuration exclusively from environment variables: `RUNS_ON_S3_BUCKET_CACHE` (bucket), `RUNS_ON_S3_BUCKET_ENDPOINT` (endpoint URL), `RUNS_ON_S3_FORCE_PATH_STYLE`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`.

#### Scenario: Bucket name from env var
- **WHEN** `RUNS_ON_S3_BUCKET_CACHE=my-bucket` is set
- **THEN** the S3 backend SHALL target the bucket named `my-bucket`

#### Scenario: Custom endpoint for self-hosted S3
- **WHEN** `RUNS_ON_S3_BUCKET_ENDPOINT=https://rustfs.example.com` is set
- **THEN** the S3 client SHALL send requests to `https://rustfs.example.com` instead of AWS

### Requirement: Built dist committed to repository
After modifying source files, `npm run prepare` SHALL be executed and the resulting `dist/restore/index.js` and `dist/save/index.js` SHALL be committed to the repository so the action can be referenced by tag/SHA in workflows.

#### Scenario: dist files present after build
- **WHEN** `npm run prepare` completes without error
- **THEN** `dist/restore/index.js` and `dist/save/index.js` SHALL exist and include the bundled AWS SDK
