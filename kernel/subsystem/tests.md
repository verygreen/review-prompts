# Lustre test scripts

Load for changes under `lustre/tests/` (the `sanity*.sh`, `conf-sanity.sh`,
`replay-*.sh`, `recovery-*.sh`, etc. suites) and for any patch that should add a
test. Whether a patch *needs* a test is decided in lustre-commit-message.md;
this file covers how Lustre tests are written and the conventions reviewers
enforce.

## Test case conventions

- Use a large/round new subtest number with gaps (e.g. `test_300`) so parallel
  patches adding tests to the same file don't conflict. Don't reuse a small
  number that's likely to clash.
- Keep skip-exception lists (e.g. `ALWAYS_EXCEPT`, per-test skip blocks) in
  numeric order so they're easy to find.
- A new feature/parameter needs a test that actually exercises it; name the
  suite explicitly when suggesting one (e.g. "add a case in sanity-sec.sh").
- Don't ask for the LU ticket number in a subtest's name or description — it
  is findable from git history and reviewers consider it noise.

## Version gating and interop

- Behavior that depends on peer version must be version-gated in the test:

      (( $MDS1_VERSION >= $(version_code 2.17.51) )) || skip "<why>"

  The skip message must explain *why* the version is needed, not just restate
  the version: `skip "need MDS >= 2.17.51 for projid in changelog"`, not
  `skip "need 2.17"`.
- **Version numbers are 3-component** (`major.minor.patch`, e.g. `2.17.57`),
  in both `version_code` gates and `Test-Parameters`. Never ask for, or
  suggest, a 4-component point-release value such as `2.17.51.23`, a
  `git describe` form (`v2_17_50-225-g...`), or "the next tag" — the version a
  patch will land in is unknowable before it lands, and interop is only ever
  tested against released majors, never between development tags, so that
  precision is meaningless. A gate at the current in-tree 3-component tag (or
  slightly below it) is **correct**; do not claim it "lets in builds without
  the fix" and do not nag for a newer or more precise value.
- Only flag a version gate that is *above* the current in-tree tag (it would
  skip permanently) or that names the wrong side (an MDS check for behavior
  that lives on the client, or vice versa). Never assert what version the tree
  "reports" from your checkout — it may be stale, and a wrong assertion has
  caused a correct gate to be broken. If a gate looks above-tree, phrase it as
  a question and point at `LUSTRE-VERSION-GEN`.
- Do not request a `CLIENT_VERSION` gate for a new test: an older client
  simply won't have the test, so there is nothing to gate.
- Interop coverage is requested via Test-Parameters in the commit message, but
  only against a **released** major (`serverversion=2.16`/`2.17`, or
  `clientversion=`), never a development tag — interop cannot be run against
  dev tags. Example:

      Test-Parameters: testlist=sanity serverversion=2.16.1

  A server-only change already runs in interop mode against old clients in the
  standard sessions, so an extra request there adds nothing.
- **Don't request test runs autotest already performs.** `Test-Parameters:
  trivial` already runs sanity-lnet, so `testlist=sanity-lnet` adds nothing;
  every new or modified subtest is automatically looped (the
  review-dne-subtest-change session runs it for ~60 minutes, hundreds of
  times); ZFS is always run for non-trivial patches, so don't ask for
  `fstype=zfs`. Only suggest a Test-Parameters line that adds coverage the
  default sessions don't already provide.
- Version gated tests should only cover the **test** functionality.  The code
  itself **must** be able to handle interoperation with newer/older peers in
  a robust manner, see `wire-protocol.md`.

## Skip / precondition form — `(style)`

A precondition for a test to run must be written as the **positive** condition
that must hold, followed by `||` and the skip/abort action — not as the negated
condition with `&&`:

      (( OSTCOUNT >= 5 )) || skip_env "needs >= 5 OSTs"     # correct

not

      (( OSTCOUNT < 5 )) && skip_env "needs >= 5 OSTs"      # flag this

The `check || action` form reads as "the test requires <check>", matches the
rest of the suite, and avoids the negation mistakes that `&&` invites. Flag any
run-condition / skip / `error` guard expressed as `negated-condition && action`
and suggest the inverted `condition || action` form. (This is the same shape as
the version gate above, `(( ... >= ... )) || skip "<why>"`.)

## Shell style

- Prefer `$(...)` over backticks for command substitution.
- Use bash arithmetic `$(( ... ))` rather than external tools.
- Call `$LFS quota` directly instead of the deprecated `getquota` wrapper; don't
  add new uses of wrappers that are being phased out.
- Quote variable expansions; follow the existing style of the suite being
  edited.
- Full Lustre test script coding style is documented at:
  https://wiki.lustre.org/Lustre_Script_Coding_Style

## Recurring test-script pitfalls

Distilled from two years of test-script `Fixes:` commits — flag these in new or
changed tests:

- **Version-gate first.** Put the `version_code` skip at the very top of the test
  body, before any `mkdir`/`touch`/parameter setup, so old-server runs skip
  without wasting time or leaving state behind.
- **Wait for async state explicitly.** Use `wait_update*` / `wait_delete_completed*`
  instead of a bare immediate check, and don't assume a backgrounded process has
  progressed — missing waits are the top source of flaky tests.
- **Respect topology variables.** Honour `local_mode` (single-node / `0@lo`),
  `$MOUNT`/`$MOUNT2`, `$OSTCOUNT`/`$MDSCOUNT`; don't hard-code `/mnt/lustre`, a
  NID, or a node/stripe count. Use `skip_env` when a test genuinely needs a
  topology it doesn't have.
- **Make cleanup robust.** Register restores with `stack_trap`, append `|| true`
  to teardown commands that can fail on an already-stopped target, and always
  restore any tunable the test changed (`fail_loc` is the exception as it is
  automatically reset on subtest exit).
- **Clean up excessive files.** Subtests that create large files (over 1MB)
  or many files (over 100) should register a `stack_trap` to delete these
  files at the end of the subtest.
- **Scale limits to the backend.** Thresholds (counts, timeouts, sizes) must
  account for `$FSTYPE` (ZFS is slower) and `SLOW`; avoid magic numbers tuned to
  one setup.  Subtests that create 10000+ files should cap this by the number
  of free inodes on the MDT or OST, if creating only on a specific target,
  or by the free inode count of the filesystem.
- **No vacuous passes.** Ensure every helper is actually called and the assertion
  runs; `init_test_env` must run before sourcing test-specific framework files;
  use double quotes where a variable must expand in `awk`/`sed`.
- **Run server-side commands on the right facet.** Test-script commands run on
  the client node. Anything that must execute on a server (e.g. `mkfs`, `mount`,
  a server-side `lctl set_param`), or a check that reads server state (e.g. a
  server-side `lctl get_param`), must be wrapped as `do_facet FACET command` so
  it runs on that server node — and `FACET` must be the target the check is
  actually about (`mds1` vs `mds2`, `ost1` vs the OST holding the object). A
  command run bare on the client, or on the wrong facet, silently checks the
  wrong node.
- **Use the right facet variable.**  Some pre-defined environment variables
  are facet specific (e.g. `facet_FSTYPE`, `facet_VERSION`), and the right one
  must be used, such as `$mds1_FSTYPE` not undefined `$mgs_FSTYPE`; a typo'd
  facet variable silently uses an empty string.

## fail_loc / OBD_FAIL injection — `(style)` + `(defect)`

When a test sets a fault-injection point (`fail_loc`, `fail2_loc`, `fail_val`
via `lctl set_param`), it should name the symbol so a reader doesn't have to go
look up what the bare hex id means:

- `(style)` Put the matching `#define OBD_FAIL_xxx 0xNNN` (or `CFS_FAIL_*`) on its
  own comment line immediately above the `set_param fail_loc=` line, mirroring
  the C definition. Flag a `fail_loc=0x...` with no symbolic `#define` comment
  above it.
- `(defect)` **Verify the value matches the real define.** Look the symbol up in
  `lustre/include/obd_support.h` and confirm the hex in the test's comment equals
  the actual `#define`. The `fail_loc` value itself is the base value optionally
  OR'd with `CFS_FAIL_*` modifier bits in the high nibble (`CFS_FAIL_ONCE
  0x80000000`, skip/timeout/etc.), so mask those off before comparing — e.g.
  `fail_loc=0x80000141` with `#define OBD_FAIL_MDS_LOV_PREP_CREATE 0x141` is
  correct. A comment whose value (or the low bits of `fail_loc`) does not match
  the real define is almost certainly a copy-paste error or typo, and the test is
  triggering a different fault than it claims — flag it.

## Reporting

These are mostly `(style)`/`(minor)` findings — phrase them softly per
gerrit-review.md ("if the patch is refreshed, ..."), except a missing
version-gate that would break an interop run, which is a real `(defect)`.
