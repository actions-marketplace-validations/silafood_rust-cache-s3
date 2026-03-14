# Rust Cache S3 Action

A fork of [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) that adds S3-compatible storage support (RustFS, MinIO, AWS S3) alongside the original GitHub, BuildJet, and WarpBuild cache providers.

## S3 Support

When `RUNS_ON_S3_BUCKET_CACHE` is set in the environment, cache operations automatically route to S3 instead of GitHub's storage — no workflow changes needed. You can also opt in explicitly with `cache-provider: s3`.

### S3 Environment Variables

| Variable | Purpose |
|---|---|
| `RUNS_ON_S3_BUCKET_CACHE` | Bucket name (presence triggers S3 mode) |
| `RUNS_ON_S3_BUCKET_ENDPOINT` | Custom endpoint URL for self-hosted S3 (RustFS/MinIO) |
| `RUNS_ON_S3_FORCE_PATH_STYLE` | Set `"true"` for self-hosted S3 |
| `AWS_ACCESS_KEY_ID` | S3 access key |
| `AWS_SECRET_ACCESS_KEY` | S3 secret key |
| `AWS_REGION` | S3 region (default: `us-east-1`) |

### Example — Self-hosted runner with S3

```yaml
jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: rustup toolchain install stable --profile minimal
      # S3 env vars injected by runner — action auto-detects S3 mode
      - uses: silafood/rust-cache-s3@v2.0.0
      - run: cargo build --release
```

### Example — Explicit S3 provider

```yaml
- uses: silafood/rust-cache-s3@v2.0.0
  with:
    cache-provider: s3
```

---

## Example usage

```yaml
- uses: actions/checkout@v5

# selecting a toolchain either by action or manual `rustup` calls should happen
# before the plugin, as the cache uses the current rustc version as its cache key
- run: rustup toolchain install stable --profile minimal

- uses: silafood/rust-cache-s3@v2.0.0
  with:
    # The prefix cache key, this can be changed to start a new cache manually.
    # default: "v0-rust"
    prefix-key: ""

    # A cache key that is used instead of the automatic `job`-based key,
    # and is stable over multiple jobs.
    # default: empty
    shared-key: ""

    # An additional cache key that is added alongside the automatic `job`-based
    # cache key and can be used to further differentiate jobs.
    # default: empty
    key: ""

    # If the automatic `job`-based cache key should include the job id.
    # default: "true"
    add-job-id-key: ""

    # Whether the a hash of the rust environment should be included in the cache key.
    # default: "true"
    add-rust-environment-hash-key: ""

    # A whitespace separated list of env-var *prefixes* who's value contributes
    # to the environment cache key.
    # default: "CARGO CC CFLAGS CXX CMAKE RUST"
    env-vars: ""

    # The cargo workspaces and target directory configuration.
    # default: ". -> target"
    workspaces: ""

    # Additional non workspace directories to be cached, separated by newlines.
    cache-directories: ""

    # Determines whether workspace `target` directories are cached.
    # default: "true"
    cache-targets: ""

    # Determines if the cache should be saved even when the workflow has failed.
    # default: "false"
    cache-on-failure: ""

    # If `true` all crates will be cached, otherwise only dependent crates will be cached.
    # default: "false"
    cache-all-crates: ""

    # If `true` the workspace crates will be cached.
    # default: "false"
    cache-workspace-crates: ""

    # Determines whether the cache should be saved.
    # default: "true"
    save-if: ""
    # To only cache runs from `master`:
    save-if: ${{ github.ref == 'refs/heads/master' }}

    # If `true` the cache key will be checked but the cache won't be restored.
    # default: "false"
    lookup-only: ""

    # Specifies the cache backend. Options: "github", "buildjet", "warpbuild", "s3"
    # Auto-selects "s3" when RUNS_ON_S3_BUCKET_CACHE env var is set.
    # default: "github"
    cache-provider: ""

    # Determines whether to cache the ~/.cargo/bin directory.
    # default: "true"
    cache-bin: ""

    # A format string used to format commands to be run, i.e. `rustc` and `cargo`.
    # Must contain exactly one occurance of `{0}`.
    # default: "{0}"
    cmd-format: ""
    # To run within a Nix shell:
    cmd-format: nix develop -c {0}
```

## Outputs

**`cache-hit`**

A boolean flag set to `true` when there was an exact cache hit.

## Cache Effectiveness

This action only caches the _dependencies_ of a crate, so it is more effective if
the dependency / own code ratio is higher.

It is also more effective for repositories with a `Cargo.lock` file. Library
repositories with only a `Cargo.toml` file have limited benefits, as cargo will
_always_ use the most up-to-date dependency versions, which may not be cached.

Usage with Stable Rust is the most effective, as a cache is tied to the Rust version.
Using it with Nightly Rust is less effective as it will throw away the cache every day,
unless a specific nightly build is being pinned.

## Cache Details

This action currently caches the following files/directories:

- `~/.cargo` (installed binaries, the cargo registry, cache, and git dependencies)
- `./target` (build artifacts of dependencies)

This cache is automatically keyed by:

- the github [`job_id`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_id)
(if `add-job-id-key` is `"true"`),
- the rustc release / host / hash (for all installed toolchains when available),
- the following values, if `add-rust-environment-hash-key` is `"true"`:
  - the value of some compiler-specific environment variables (eg. RUSTFLAGS, etc), and
  - a hash of all `Cargo.lock` / `Cargo.toml` files found anywhere in the repository (if present).
  - a hash of all `rust-toolchain` / `rust-toolchain.toml` files in the root of the repository (if present).
  - a hash of all `.cargo/config.toml` files in the root of the repository (if present).

An additional input `key` can be provided if the builtin keys are not sufficient.

Before being persisted, the cache is cleaned of:

- Any files in `~/.cargo/bin` that were present before the action ran (for example `rustc`).
- Dependencies that are no longer used.
- Anything that is not a dependency.
- Incremental build artifacts.
- Any build artifacts with an `mtime` older than one week.

## Cache Limits and Control

For GitHub-backed caches, the same restrictions apply as the upstream cache action:
[Caching dependencies to speed up workflows](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)

Caches are currently limited to 10 GB in total. For S3-backed caches, limits are determined by your bucket configuration.

## Debugging

The action prints detailed information about which information it considers for its cache key.
You can [enable debug logging](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) to see additional details.

## Known issues

- The cache cleaning process currently removes all the files from `~/.cargo/bin`
  that were present before the action ran (for example `rustc`), by default.
  This can be an issue on long-running self-hosted runners, where such state
  is expected to be preserved across runs. You can work around this by setting
  `cache-bin: "false"`.

## Upstream

This action is a fork of [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache).
All Rust-specific caching logic is inherited from upstream. Only the cache provider
layer has been extended to support S3.
