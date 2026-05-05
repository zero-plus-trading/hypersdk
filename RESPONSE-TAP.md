# Response-tap fork notes

This branch carries a single additive patch on top of upstream
`infinitefield/hypersdk`: a `tracing::trace!` event emitted at every
`/exchange` HTTP response chokepoint, so consumers can spool raw exchange
responses to disk for an audit trail.

## What the patch does

Adds:

- `Action::kind() -> &'static str` in `src/hypercore/types/api.rs` — a
  `&'static str` label per `Action` variant, used as a structured field on
  the trace event so subscribers can route or filter without parsing the
  body.
- `tracing = "0.1"` in `Cargo.toml`.
- A `tracing::trace!` call on the target `hypersdk::http::response` inside
  both response chokepoints in `src/hypercore/http.rs`:
  - `Client::sign_and_send_sync` (covers `place`, `cancel`,
    `cancel_by_cloid`, `modify`, `schedule_cancel`, `gossip_priority_bid`,
    transfers, `update_leverage`, etc.)
  - `Client::send` (covers all multisig paths via `MultiSig::*`)

  Each event carries `url`, `http_status`, `nonce`, `action`, `body`.

The patch is purely additive — no public types, traits, or function
signatures change. With no subscriber installed, the macro is a no-op
(filtered before field formatting; no allocation).

## How consumers use it

Install a `tracing-subscriber` filtered to `hypersdk::http::response`
at TRACE level. See `eris/src/oms/raw_response_log.rs` in the eris
repo for the canonical setup (rolling daily JSON-Lines via
`tracing-appender::non_blocking`).

## Rebasing onto a new upstream version

The fork tracks tagged upstream releases. Re-base, don't merge — keeps
the patch as a single cherry-pickable commit.

```bash
cd /root/build/hypersdk

# 1. Pull latest upstream tags
git fetch upstream

# 2. Rebase response-tap onto the new tag (or upstream/main)
git rebase v0.2.12 response-tap   # adjust the tag

# 3. Resolve conflicts if any. The patch only touches three files:
#       Cargo.toml
#       src/hypercore/types/api.rs    (Action::kind only)
#       src/hypercore/http.rs         (sign_and_send_sync, send)
#    so conflicts are bounded and mechanical to fix. The intent is:
#    emit `tracing::trace!` between `let text = res.text().await?` and
#    the `serde_json::from_str(&text)` parse — re-apply on whatever
#    structure upstream has.

# 4. Verify it builds
cargo check

# 5. Force-push (rebase rewrites history)
git push origin response-tap --force-with-lease
```

After pushing, in eris:

```bash
cd /root/build/eris
cargo update -p hypersdk
```

## When upstream adds a new `Action` variant

`Action::kind()` is exhaustive — the compiler will fail with a
non-exhaustive match. Add an arm to the match, picking a snake_case
label that matches upstream's naming convention.

## When upstream refactors `sign_and_send_sync` or `send`

The patch's only structural assumption is that the function reads the
response body via `let text = res.text().await?` (or equivalent) before
parsing. If upstream switches to a streaming parse, the tap needs to be
relocated — probably to wherever the body is materialized for error
reporting (since upstream's own error messages embed the body, that
materialization will continue to exist somewhere).

## Upstreaming

This patch is generic enough that it has a reasonable chance of being
accepted upstream (`infinitefield/hypersdk`). If/when that happens, the
fork can be retired and eris can revert to the crates.io version.
