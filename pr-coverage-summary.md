## Test coverage summary

Ran with `cargo tarpaulin --lib --engine llvm` against the subpath changes, with all live-network tests skipped (no Reddit traffic): **68 tests passed, 0 failed, 18 filtered out.**

**Overall:** 31.02% line coverage (654/2108). Files with no coverage (`subreddit.rs`, `user.rs`, `post.rs`, `search.rs`, `settings.rs`, `duplicates.rs`, `instance_info.rs`) are HTTP route handlers that require a live `hyper::Request` — none of them are exercised by the existing unit suite, predating this branch.

**Coverage of new code added in this PR:**

| New code | Location | Coverage |
|---|---|---|
| `prefix()` | `utils.rs:1337` | ✅ fully covered (8 env permutations) |
| `cookie_path()` | `utils.rs:1355` | ✅ fully covered (both branches) |
| `with_prefix()` | `utils.rs:1366` | ✅ fully covered |
| `rewrite_urls` prefix-aware paths | `utils.rs:1100,1104,1129-1167` | ✅ covered |
| `format_url` prefix passthrough | `utils.rs:1072` | ✅ covered |
| `REDLIB_BASE_PATH` config plumbing | `config.rs:112,162,193` | ✅ covered |
| Request-prefix strip in `service_fn` | `server.rs:340-366` | ❌ not covered — closure isn't reachable from a unit test without spinning up a server or extracting a helper |

**Note on tooling:** must use `--engine llvm` (not the default ptrace). Tests use `sealed_test` for env-var isolation, which forks per test; ptrace doesn't merge coverage from forks and reports a misleading 15%.

**Reproduce:**
```bash
cargo tarpaulin --lib --engine llvm --exclude-files 'target/*' --out Stdout \
  -- --skip test_fetching --skip test_oauth_ --skip test_mobile_spoof \
     --skip test_generic_web --skip test_rate_limit --skip test_localization \
     --skip test_obfuscated --skip test_private_sub --skip test_banned_sub --skip test_gated
```
