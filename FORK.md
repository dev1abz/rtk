# dev1abz/rtk — fork notes

Branch `raileurope-v1` is upstream `rtk-ai/rtk` **v1.0.0** (`master`, `01f819c4`)
plus the patches below. Everything else is upstream, unmodified. Re-forked on
2026-10-14 from the 0.45-based `raileurope` branch (kept on GitHub as history),
carrying only what still reproduced against an untouched v1.0.0 build.

## Patches

| Commit subject | Why it is still needed on v1.0.0 |
|---|---|
| `fix(rspec): keep expected/got in failure summaries` | RSpec's `ExpectationNotMetError` message starts with a newline; v1.0.0 takes its first line and prints a bare `ExpectationNotMetError:`. Reproduced with a stub `rspec` emitting a real failure payload. No upstream issue or PR. |
| `fix(ruby): resolve the Gemfile the way bundler does` | `ruby_exec` checks only the working directory, so `rtk rspec` from `spec/models` spawns bare `rspec`, not `bundle exec rspec`. The hook rewrites an explicit `bundle exec rspec` to `rtk rspec`, so rtk itself drops the bundler. |
| `fix(diff): claim no savings for identical files` | `rtk gain` accounting only. v1.0.0 books an identical pair as fully saved (39 KB pair: `saved=19441`). Upstream chose this on purpose (test "identical files should still measure against the dump"), so this stays a fork patch. |

## Configuration instead of code

`mise exec -- <cmd>` is handled by config, not by the 0.45 fork's builtin
wrapper patch. In `~/.config/rtk/config.toml`:

```toml
[hooks]
transparent_prefixes = ["mise exec --", "mise x --"]
```

With it, `mise exec -- bundle exec rspec` becomes `mise exec -- rtk rspec`, as the
0.45 patch did, and it also reaches inside pipelines
(`cd x && mise exec -- bundle exec rspec 2>&1 | tail -20`), which the 0.45 patch did
not. Without it, v1.0.0 routes every `mise exec …` through the `mise` TOML filter,
which caps output at 50 lines and cuts lines at 150 characters, so
`mise exec -- bin/rails runner …` output was silently truncated. With the prefix,
commands rtk has no filter for (`bin/rails runner`, `bin/rake`) run unrewritten
and print in full.

Not covered: tool specs before the separator (`mise exec ruby@4 -- …`). That form
appeared 0 times in 4,027 agent Bash commands on this machine, against 311 for
`mise exec -- …`.

## Dropped from the 0.45 fork

- The two `mise exec` rewrite commits: replaced by the config above.
- `fix(diff): measure savings against diff(1) output`: upstream 8495176c measures
  changed files against the classic diff; only the identical-file case remains
  (patch above).
- The 20 upstream PRs that had been merged into `raileurope`: out of scope for
  this re-fork. Ten have landed upstream in some form; the rest were left to
  upstream.

## Updating

```bash
git fetch upstream --tags
git rebase --onto <new-tag> v1.0.0 raileurope-v1   # then re-run the checks below
cargo fmt --all -- --check && cargo clippy --all-targets && cargo test --all
cargo test --no-default-features
```

Re-test each patch against the new tag before carrying it forward. The reason to
drop one is upstream fixing the problem, not the rebase conflicting.
