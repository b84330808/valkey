# Valkey contribution report

## Summary

| | |
|---|---|
| Upstream | https://github.com/valkey-io/valkey (default branch `unstable`, base commit `b3b5ab9` "Feature pathhash prefixes (#5070)") |
| Fork | https://github.com/b84330808/valkey |
| Fix branch | `fix-cli-bigkeys-pathhash` |
| Fix commit | `f1818acaed881200de69cd044240054da2105a5a` |
| Files changed | `src/valkey-cli.c` (+2), `tests/integration/valkey-cli.tcl` (+10) |
| Issue | None. I found the bug myself (see "How it was found"). |
| Second fix | Not done. I looked, but did not find a second bug that is unfixed and has no competing PR (see "Second fix search"). |

## The bug

`unstable` adds a native data type, the path hash (`TYPE` returns `pathhash`, commands `PH*`).
`valkey-cli --bigkeys` has a size command only for the original types
(`STRLEN`, `LLEN`, `SCARD`, `HLEN`, `ZCARD`, `XLEN`). Every other type gets the fallback for
unknown module types, which has no size command. So every path hash key is counted with size 0
and the unit `?`, and no biggest path hash key is ever reported:

```
1 pathhashs with 0 ? (100.00% of keys, avg size 0.00)
```

The fix registers `pathhash` with `PHCARD` (the number of paths) as its size command and the unit `paths`,
the same way `XLEN`/`entries` is used for streams. After the fix:

```
Biggest pathhash found '"ph"' has 2 paths
...
2 pathhashs with 3 paths (100.00% of keys, avg size 1.50)
```

`--memkeys` already worked, because it uses `MEMORY USAGE` for every type.

I checked the rest of the tree for the same gap (per-type tables that have no `pathhash` entry):
- `src/valkey-cli.c`: the `typeinfo` table in `findBigKeys()` is the only per-type table. Fixed.
- These already handle `pathhash`: `obj_type_name` in `db.c` (TYPE, SCAN TYPE), `type_name` in `valkey-check-rdb.c`,
  notify flags (`p`) in `notify.c` and `valkey.conf`, AOF rewrite, RDB save/load, DEBUG DIGEST,
  MEMORY USAGE, lazyfree effort, `COPY` (dup), the module `KeyType`/`ValueLength` APIs,
  and `valkey-cli HELP @pathhash`.
- Not changed, because it is test tooling and not a user-facing bug: `src/fuzzer_command_generator.c`
  has a type list without `pathhash`.

## How it was found

Every open upstream issue labelled `bug` that I checked already had a linked or competing PR
(I searched for PRs that mention each issue number). So I probed new code by hand. On a server
built from `unstable` I created path hash keys and ran the generic tools against them.
`valkey-cli --bigkeys` printed `1 pathhashs with 0 ?`.

## Proposed PR

**Title:** `Report path hash sizes in valkey-cli --bigkeys`

Upstream has no PR template (no `.github/pull_request_template.md` or `PULL_REQUEST_TEMPLATE` directory).
This draft uses a plain description:

```
`valkey-cli --bigkeys` has a size command only for the original data types.
A native path hash key gets the fallback for unknown module types, which has
no size command. So every path hash key is counted with size 0 and the unit
"?", and no biggest path hash key is ever reported:

    1 pathhashs with 0 ? (100.00% of keys, avg size 0.00)

This registers the pathhash type with PHCARD as its size command, so path hash
keys are sized by their number of paths, like XLEN for streams:

    Biggest pathhash found '"ph"' has 3 paths
    1 pathhashs with 3 paths (100.00% of keys, avg size 3.00)

`--memkeys` was not affected, because it uses MEMORY USAGE for every type.

A test in tests/integration/valkey-cli.tcl creates a path hash with three paths
and checks both the biggest key line and the summary line of `--bigkeys`.

I used Claude Code to write this change.
```

Before opening the PR, check:
- Open PR valkey-io/valkey#5071 ("Replace PHCARD with prefix-aware PHPATHCOUNT") removes `PHCARD`.
  If #5071 merges first, change `"PHCARD"` to `"PHPATHCOUNT"` in this fix. Without a `PREFIX`
  argument, `PHPATHCOUNT` returns the same count, so the test needs no change.
  The PR could also mention #5071 so that whichever PR merges second is adjusted.
- Path hash, including `PHCARD`, is also on the upstream `9.2` branch and in tag `9.2.0-rc1`,
  so the bug ships in the 9.2 release candidate. Check `9.2` before the final release. A local
  `git cherry-pick f1818acae` onto `upstream/9.2` applies the `src/valkey-cli.c` change cleanly.
  The only conflict is in `tests/integration/valkey-cli.tcl`, because the `--hotkeys-count` test
  next to the new test is not on that branch. The maintainers' backport sweep would resolve it.
  The PR could ask for the `backport 9.2` label (or whatever the maintainers use).

## Commands and output

Setup:

```
git remote add upstream https://github.com/valkey-io/valkey.git
git fetch upstream unstable
git checkout -b fix-cli-bigkeys-pathhash upstream/unstable
make -j16
sudo apt-get install -y tcl tcl-tls   # the container had no tclsh
```

Manual reproduction on `unstable`:

```
./src/valkey-server --port 7777 --daemonize yes
./src/valkey-cli -p 7777 phset ph /a FIELDS 1 f v
./src/valkey-cli -p 7777 phset ph /a/b FIELDS 2 f v g w
./src/valkey-cli -p 7777 --bigkeys | tail -8
```

```
0 strings with 0 bytes (00.00% of keys, avg size 0.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
1 pathhashs with 0 ? (100.00% of keys, avg size 0.00)
```

### Fail before (test kept, fix reverted)

```
git checkout HEAD~1 -- src/valkey-cli.c
make -j16 -C src valkey-cli
./runtest --single integration/valkey-cli --only "--bigkeys reports the number of paths in path hash keys"
```

```
Test Summary: 0 passed, 1 failed
!!! WARNING The following tests failed:
*** [err]: --bigkeys reports the number of paths in path hash keys in tests/integration/valkey-cli.tcl
Expected '
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).
-------- Summary --------
Sampled 1 keys in the keyspace!
Total key length in bytes is 2 (avg len 2.00)
0 strings with 0 bytes (00.00% of keys, avg size 0.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
1 pathhashs with 0 ? (100.00% of keys, avg size 0.00)' to match '*Biggest pathhash found '"ph"' has 3 paths*' (context: type eval line 7 cmd {assert_match "*Biggest pathhash found '\"ph\"' has 3 paths*" $output} proc ::test)
```

### Pass after (fix restored)

```
git checkout HEAD -- src/valkey-cli.c
make -j16 -C src valkey-cli
./runtest --single integration/valkey-cli --only "--bigkeys reports the number of paths in path hash keys"
```

```
Test Summary: 1 passed, 0 failed
\o/ All tests passed without errors!
```

Whole file, with the fix:

```
./runtest --single integration/valkey-cli
```

```
[ok]: --bigkeys reports the number of paths in path hash keys (23 ms)
...
Test Summary: 54 passed, 0 failed
\o/ All tests passed without errors!
```

Formatting: `clang-format-18 -i src/valkey-cli.c` left the diff unchanged.

## Competing PR search (all states)

I searched with `gh api -X GET search/issues -f q='repo:valkey-io/valkey is:pr <terms>'` (open and closed)
and also searched issues:

| Query | Result |
|---|---|
| `bigkeys` (PRs) | #2819 (open, real-time bigkey scan feature), #1151 (open, KEYINFO command), #4830 (open, per-key traffic tracking), #2397 (closed, summary hyphens), #2472 (closed, module shared memory). None touch the `--bigkeys` type table. |
| `bigkeys pathhash`, `PHCARD bigkeys`, `type_pathhash` | none |
| `PHCARD` | #5071 (open, renames PHCARD to PHPATHCOUNT; does not touch valkey-cli), #4506 (closed, original feature) |
| `valkey-cli pathhash`, `valkey-cli path hash` | only #4506 (closed, original feature) and unrelated PRs |
| `pathhash` / `path hash` (open PRs) | #5067, #5068, #5071, #5112, #4878, #4883, #4879. I fetched each head (`git fetch upstream pull/N/head`) and diffed it. None change `src/valkey-cli.c`. |
| `bigkeys`, `memkeys` (issues) | feature requests only (#1827, #1880, #4189, #1969, #1167). No report of this bug. |

## Policy notes

- **CONTRIBUTING.md, "Use of AI tools":** "You own the quality of the code that you submit to the project. The Valkey
  project doesn't impose any limitations on the use of AI tooling as long as the contribution meets the quality bar
  of the project." AI-assisted PRs are allowed, and the policy does not require a human to write the PR text.
  The PR body still ends with "I used Claude Code to write this change." as asked.
- **DCO:** every commit needs `Signed-off-by` with a known identity (no pseudonyms). The commit was made with
  `git commit -s`. Author and sign-off: `Wei-Lun Li <b84330808@gmail.com>`. The message also has
  `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **AGENTS.md:** keep changes minimal and easy to backport; match surrounding style; run `clang-format-18` on
  touched C files; add tests (Tcl for end-to-end behavior); push only to the user's fork, never to upstream or `unstable`.
- **CONTRIBUTING.md, patches:** minor fixes go straight to a PR. Use "Fixes #xyz" only when there is an issue
  (there is none here). Formatting is checked by CI with `clang-format-18`.
- **DEVELOPMENT_GUIDE.md:** keep PRs small for backporting; every contribution needs a test; use `/* */` comments.
- **PR template:** upstream has none.
- **Upstream interaction:** none. I only fetched `unstable` and the heads of open PRs with git, and ran read-only searches.
  Nothing was pushed, opened or commented upstream.

## Second fix search

I did not find a second bug that is unfixed and has no competing PR. The upstream tree is being swept heavily,
with dozens of small fix PRs merged each day. These are the areas I checked and found correct or already covered:

- Open `bug` issues: for each of #5223, #5222, #5202, #5094, #5054, #4992, #4979, #4938, #4936, #4935, #4934,
  #4866, #4847, #4842, #4804, #4801, #4727, #4705, #4630, #4592, #4538, #4530, #4274, #4203, #4014, #3457,
  #3238, #3065, #2952, #1846, a PR search for the issue number returned at least one PR (open or closed).
  #5240 (CLOCK_MONOTONIC fallback) and #3809 (8.1.4 crash) returned none. #5240 can't be reproduced in this
  container or covered by a Tcl test, and #3809 was reported on 8.1.4. I did not check the older
  environment- and deployment-specific issues (pod restarts, packaging, sentinel or cluster incidents).
- Path hash commands (PHSET FNX/FXX, PHMSET/PHMGET group parsing, PHDEL, PHDELPREFIX, PHPREFIXES
  COUNT/MAXPATHLEN/EXCLUDESELF/LENGTHS/FIELDS, PHPATHSCAN cursors and prefixes, PHSCAN), RDB load
  validation and AOF rewrite: no bugs found.
- Hash field expiration (HEXPIRE/HSETEX/HGETEX/HGETDEL limits and keyspace events; logically expired
  fields across HGET/HSCAN/HRANDFIELD/HINCRBY/HSETNX/COPY; volatile-key tracking across COPY/RENAME/MOVE): correct.
- DEBUG subcommand argc guards: already fixed (#5147).
- A sample of argument edge cases (MSETEX, DELIFEQ, LPOS, LMPOP, ZRANGESTORE, ZRANDMEMBER, SRANDMEMBER,
  GETRANGE, SETRANGE, BITPOS BIT): correct.
