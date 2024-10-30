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

| Name               | Description                                                                                                                                                                                                                                                   | Type      | Default                                                                        |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| `cache-group`      | The group of the cache, defaults to a unique identifier for the workflow job. If you want two jobs to share the same cache, give them the same group name.                                                                                                    | `string`  | `${{ hashFiles(env.workflow_path)-${{ github.job }}-${{ strategy.job-index }}` |
| `cargo-home`       | The location of the Cargo cache files. If you specify the `CARGO_HOME` env variable for your commands, you need to set it here too. This must NOT end with the trailing slash of the directory.                                                               | `string`  | `~/.cargo`                                                                     |
| `cargo-target-dir` | Location of where to place all generated artifacts, relative to the current working directory. If you specify the `CARGO_TARGET_DIR` env variable for your commands, you need to set it here too. This must NOT end with the trailing slash of the directory. | `string`  | `target`                                                                       |
|`manifest-path`|The path to `Cargo.toml`. This is used to determine where `Cargo.lock` is, which is used in the cache key.|`string`|`Cargo.toml`|
| `save-if`          | A condition which determines whether the cache should be saved. Otherwise, it's only restored                                                                                                                                                                 | `boolean` | `true`                                                                         |
| `save-always`      | Run the post step to save the cache even if another step before fails.                                                                                                                                                                                        | `boolean` | `true`                                                                         |
|`sweep-cache`|Use `cargo-sweep` to automatically delete files in the target folder that are not used between when this action is called and the end of the workflow. This can prevent the size of caches steadily increasing, since new caches are generated from fallback caches that may include stale artifacts.|`boolean`|`false`|
|`cache-cargo-sweep`|This input has been deprecated and will be removed in v3.0.0. It has no effect.|`boolean`||

[`cargo-sweep`]: https://crates.io/crates/cargo-sweep

## Outputs

| Name        | Description                                                          | Type      | Note                                            |
| ----------- | -------------------------------------------------------------------- | --------- | ----------------------------------------------- |
| `cache-hit` | A boolean value to indicate if an exact match was found for the key. | `boolean` | Passed through from the `actions/cache` action. |

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
