# azure-custom-provider-fixes (fork of openai/codex)

This fork applies two small, unmerged fixes on top of `openai/codex@main` for
users running Codex against a **custom `model_provider`** (e.g. Azure OpenAI
with `wire_api = "responses"`) instead of OpenAI's own hosted backend.

Both fixes have been reported upstream but have not been merged yet, since
`openai/codex` only accepts pull requests by invitation
([contributing.md](https://github.com/openai/codex/blob/main/docs/contributing.md)).
This fork exists purely as a stopgap until either fix lands upstream — track
the linked issues and drop this fork (switch back to an official release) once
they're resolved.

## What's patched

### 1. Retry transient "Item with id ... not found" errors
**File:** `codex-rs/codex-client/src/retry.rs`
**Upstream issue:** [openai/codex#31718](https://github.com/openai/codex/issues/31718)

The Responses API intermittently returns `HTTP 400 invalid_request_error:
Item with id '<id>' not found` when a follow-up turn references a
reasoning/function-call item from the immediately preceding response. This is
a transient server-side lookup race (most reliably observed against Azure
OpenAI), not a malformed request — retrying the exact same request succeeds.
Upstream `codex-client` does not retry on HTTP 400 at all, so this failure
was always surfaced to the user. This patch narrowly retries only this one
error shape (status 400 + `invalid_request_error` + `not found` + `Item with
id`/`"param":"input"` in the body), reusing the existing retry/backoff budget.

### 2. Guardian auto-review model fallback for custom providers
**File:** `codex-rs/core/src/guardian/review.rs`
**Upstream issue:** [openai/codex#31732](https://github.com/openai/codex/issues/31732)

When `approvals_reviewer = "auto_review"` ("Approve for me") is used against
a custom provider, the guardian reviewer can pick the default preferred
review model id (`codex-auto-review`) even though it isn't a real deployment
on that provider. The request then 404s, and because guardian fails **closed**
on any review error, the user's original action (e.g. a sandbox escalation)
gets denied as "unacceptable risk" — even though the real cause is an
unrelated backend 404. This patch only trusts the `codex-auto-review` catalog
match when the provider is actually OpenAI-hosted (or an explicit
`auto_review_model_override` is set); otherwise it falls through to the
active session's real model, matching the pattern already used for
`image_generation_runtime_enabled` in `codex-rs/core/src/tools/spec_plan.rs`.

## Building

```bash
git clone https://github.com/<your-fork>/codex.git
cd codex/codex-rs
cargo build --release -p codex-cli
```

Building the full workspace (`cargo build --release` with no `-p`) may hit an
unrelated rustc recursion-limit error in the `codex-thread-manager-sample`
example crate — building only `-p codex-cli` avoids it and is all that's
needed to produce `target/release/codex`.

## Using the build with Codex Desktop

Point the official Codex Desktop app at this binary instead of its bundled
one by setting, in `~/.codex/config.toml`:

```toml
[mcp_servers.node_repl.env]
CODEX_CLI_PATH = "/absolute/path/to/codex/codex-rs/target/release/codex"
```

(or export `CODEX_CLI_PATH` in your shell profile before launching Codex
Desktop). The Desktop app UI/shell is unchanged — only the underlying
`codex` engine binary is swapped.

## Staying in sync with upstream

This fork should be rebased on `openai/codex@main` periodically. Both patched
files are actively developed upstream, so expect occasional merge conflicts
near (but historically not exactly on) the patched lines — resolve by keeping
the patch logic described above and taking upstream's surrounding changes.

## License

Unchanged from upstream: Apache License 2.0. See `LICENSE` and `NOTICE`. Per
Apache-2.0 §4(b), this file documents that the two source files above have
been modified from the original.
