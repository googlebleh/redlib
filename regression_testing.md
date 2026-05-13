# Regression testing in this fork

A short guide to verifying that a change hasn't introduced regressions, given the quirks of redlib's test suite. Written for both humans and future Claude sessions.

## Context

The test suite mixes two kinds of tests:

- **Unit tests** — pure functions, env-driven config, in-memory compression. Reliable.
- **Live-network tests** — outbound calls to `oauth.reddit.com` / `www.reddit.com`. **Unreliable by design**: they depend on Reddit's response, which varies with rate-limit state, OAuth backend health, and per-IP throttling.

This makes "does the test suite pass?" a poor proxy for "did I introduce a regression?" — many failures come from outside the codebase.

## Known upstream failure modes

Two well-known failure categories on `redlib-org/redlib`, both present on `main` and on any branch:

| Failure mode | Tests usually affected | Upstream tracking |
|---|---|---|
| `assertion failed: response.is_ok()` at `oauth.rs` (no `SendRequest` prefix) — Reddit returned non-JSON. The `genericweb` OAuth flow is broken. | `oauth::test_generic_web_backend` (deterministic) | [redlib-org/redlib#485](https://github.com/redlib-org/redlib/issues/485) |
| `client error (SendRequest)` — Reddit dropped the TLS handshake. Throttling. | Any `oauth.reddit.com`-touching test, especially `test_rate_limit_check`, `test_fetching_*` | [redlib-org/redlib#446](https://github.com/redlib-org/redlib/issues/446), [#511](https://github.com/redlib-org/redlib/issues/511) |

The `genericweb` backend failure also exists on the `default` (mobile-spoof) backend in some windows — see the commented-out `REDLIB_OAUTH_BACKEND=genericweb` in `/etc/redlib.conf`.

## What confuses regression analysis

Pitfalls observed in this repo's environment:

1. **A test failing 2/2 on branch A and 0/2 on branch B is not a regression** if the runs happened in different time windows. Reddit's rate-limit state varies independently of code.
2. **Failure count tracks position in the test sequence**, not branch. The second back-to-back full-suite run usually has more failures than the first, because Reddit's per-IP bucket isn't fully refilled after only 10 min idle.
3. **Different selectors give different signals.** A 3-test subset run hits Reddit far less than a 57-test full-suite run; comparing the two is apples-to-oranges.
4. **Tests passing in isolation isn't sufficient** to prove no load-coupled regression. They must pass under matched load.

## Recommended regression check

When adding a new feature, run this sequence to verify no regressions:

### Step 1 — Pure unit tests must pass

```bash
cargo test --lib config::tests utils::tests server::tests
# also: oauth::tests::test_creating_device oauth::tests::test_creating_backends
# also: client::tests::test_default_subscriptions
```

These are deterministic. Any failure here is a real regression — investigate it.

### Step 2 — Check that any newly-failing live-network test also fails on `origin/main`

If the suite shows a live-network test failure that wasn't there before, **do not assume regression** until you've confirmed it doesn't also fail on `main`. Use a worktree to avoid disturbing local state:

```bash
git fetch origin main
git worktree add /tmp/redlib-main origin/main
(cd /tmp/redlib-main && cargo test --lib <the_failing_test>)
```

If `main` also fails: not a regression. Likely upstream — check the issues table above.

### Step 3 — If still ambiguous, run the ABBA experiment

Used when one live-network test fails consistently on the feature branch but passes on `main`. Reddit's rate-limit state can produce false asymmetries; ABBA controls for both order and time-window.

```
T+0:00  cooldown 20 min   (let Reddit recover from any prior testing)
T+0:20  full suite on main
T+0:30  idle 10 min
T+0:40  full suite on feature
T+0:50  idle 10 min
T+1:00  full suite on feature   (mirror — same branch, different position)
T+1:10  idle 10 min
T+1:20  full suite on main      (mirror)
```

Total: ~90 min wall time. A reference orchestration script lived in `/tmp/redlib-abba.sh` during the run that produced this doc. Compare per-test fail rates **across positions**:

- If a test fails ≥3/4 on feature and ≤1/4 on main (or vice versa): possible regression — read the diff between branches for files that test touches.
- If fail rates are similar (e.g. feature 2/4 vs main 2/4, or mean-fails-per-run within ~1 of each other): not a regression, it's network coupling.

### Step 4 — Diff-narrowing for "regression candidates"

If Step 3 shows a real branch-correlated failure, look at the source files the failing test touches and `git diff origin/main..HEAD -- <those files>`. If the diff is empty, the failure is environmental, not code.

In this branch's case: the subpath commits touch zero lines in `src/oauth.rs` or `src/client.rs`, so an OAuth/client regression from this branch is implausible by construction.

## What "regression-clean" means here

In this environment, "no regression introduced" is established when:

1. All deterministic unit tests pass.
2. Any live-network failures on the feature branch also occur on `origin/main` under matched load (Step 2 or Step 3).
3. The feature branch's diff doesn't touch source files relevant to the failing tests' codepath.

Failing to meet (2) or (3) is the only signal worth chasing. A red suite is not, on its own.
