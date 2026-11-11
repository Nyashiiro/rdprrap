# CI probe

Temporary branch with no functional changes.

This branch exists only to open a PR against `main` and trigger the `ci`
workflow on the `pull_request` event and captures the expected rust 1.99 lint output:

- `cargo fmt --check` — one import-wrapping diff in `service.rs`
- `cargo clippy --all-targets` with `-D warnings` — `chunks_exact_to_as_chunks`
  in `arm64.rs` (3) and `manual_div_ceil` in `service.rs` (1)

**Do not merge.** Close the PR once a workflow run has been recorded and its
log saved for the tracking issue.
