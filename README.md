# `cargo-cache` Action

This GitHub Action caches Rust Cargo build files to speed up your CI workflows.

## Example workflow

```yaml
on:
  pull_request:

jobs:
  cargo-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # Install Rust and Cargo
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: stable

      # Cache the Cargo build files
      # This action must be used *after* `dtolnay/rust-toolchain`!
      - uses: Leafwing-Studios/cargo-cache@v2

      # Do stuff with Cargo
      - name: Run cargo check
        run: cargo check
```

## Inputs

```yaml
- uses: Leafwing-Studios/cargo-cache@v2
  with:
    # The group of the cache, defaulting to a unique identifier for the workflow job.
    #
    # If you want two jobs to share the same cache, give them the same group name.
    cache-group: ${{ hashFiles(env.workflow_path)-${{ github.job }}-${{ strategy.job-index }}

    # The location of the Cargo home directory.
    #
    # If you specify the `CARGO_HOME` env variable for your commands, you need to set it here too.
    # This must *not* end with the trailing slash of the directory.
    cargo-home: ~/.cargo

    # The location where Cargo places all generated artifacts, relative to the current working
    # directory.
    #
    # If you specify the `CARGO_TARGET_DIR` environmental variable or `--target-dir` for your
    # commands, you need to set it here as well. This must *not* end with the trailing slash of the directory.
    cargo-target-dir: target

    # The path to `Cargo.toml`.
    #
    # This is used to determine where `Cargo.lock` and is, which is used in the cache key.
    manifest-path: Cargo.toml

    # This input has been deprecated and will be removed in v3.0.0. It has no effect.
    #
    # This input used to specify the `save-always` input for `actions/cache`, but has been
    # deprecated due to its unintended behavior. If you still require this input, you will need
    # to manually use `actions/cache`. For more information, please see
    # <https://github.com/actions/cache/tree/v4/save#always-save-cache>.
    save-always: ''

    # Determines if the cache should be saved, or only loaded.
    #
    # Setting this to `false` will prevent new caches from being created.
    save-if: true

    # Automatically delete files in the target folder that are not used between when this action is
    # called and the end of the job.
    #
    # This can prevent the size of caches snowballing. Since old caches are used to create new
    # caches, unused files can slowly pile up over time, causing larger caches are longer runtimes.
    sweep-cache: false

    # This input has been deprecated and will be removed in v3.0.0. It has no effect.
    cache-cargo-sweep: ''
```

## Outputs

- `cache-hit`: A string value that indicates if an exact match was found for the key.
  - This is passed from `actions/cache`, so please see [its output documentation](https://github.com/actions/cache/tree/v4#outputs) for more information.

## How It Works

This action caches the `target` directory, as well the Cargo registry.
Under the hood, this action uses the `actions/cache` action to do the actual caching.
The cache key is generated from the `Cargo.toml` and `Cargo.lock` files, to ensure that the cache is always up-to-date.
When the `Cargo.toml` or `Cargo.lock` files change, the cache falls back to previously cached versions to not start from scratch.

When the `Cargo.lock` file was not committed to the repository (this is the case for libraries), it is first generated via `cargo update`.
This is why the `dtolnay/rust-toolchain` action must be used _before_ this action.
For more information on why the `Cargo.toml` files don't specify the dependencies completely, please consult [the Cargo Book](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html).

Additionally, the Rust version reported by Cargo is included in the cache key.
This prevents the cache from being outdated when a new Rust version is released, as this will result in a re-compile of all dependencies.
