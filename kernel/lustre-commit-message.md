# Lustre commit message and change-completeness verification

Load this for every Lustre patch. The full house rules are at
https://wiki.whamcloud.com/display/PUB/Commit+Comments — this file captures the
parts an automated reviewer can check. Each check below produces a regression
(a Gerrit comment) when it fails.

## Subject line

The subject line has three parts:

    LU-XXXXX subsystem: short imperative summary

1. **Ticket: `LU-XXXXX`** must be present as the first token.
   - Verify the ticket actually exists and is the right one. The LU project on
     jira.whamcloud.com is public and queryable without auth:

         curl -s "https://jira.whamcloud.com/rest/api/2/issue/LU-XXXXX?fields=summary,status,issuetype"

     - HTTP 404 / `errorMessages` => the ticket does not exist. Flag it.
     - Compare the JIRA `summary` and the patch's actual change. If the ticket
       is about something unrelated to the diff (e.g. ticket is a quota bug but
       the patch changes the LNet socklnd), question whether the ticket number
       is wrong. Minor wording differences are fine; a topic mismatch is not.
     - A `Resolved`/`Closed` ticket getting a brand-new feature patch (though
       not a bugfix on a patch for that ticket) is worth a
       gentle question (should this be a new ticket?). Do not hard-fail on it.
   - Some legitimate non-LU prefixes exist (e.g. `LUDOC-` for documentation
     issues, `EX-` for patches on `b_es_` vendor branches).
     If the prefix is not `LU-` on the `fs/lustre-release` repository, flag it
     and suggest that the `PREFIX-bug-id:` label should be used for this
     reference, but do not assume it is wrong for other repositories.

2. **Subsystem** — a lowercase component tag after the ticket, ending
   with `:`. It should match the code the patch touches. Pick the most specific
   accurate tag:
   - A localized patch uses a single component directory name, e.g. `osc`,
     `mdt`, `mdc`, `lov`, `lmv`, `llite`, `ldlm`, `ptlrpc`, `obdclass`, `ofd`,
     `osd-ldiskfs`, `osd-zfs`, `osp`, `mgs`, `mgc`, `quota`, `lfsck`, `fid`,
     `fld`, `mdd`, `lod`, `target`.
   - LNet code uses `lnet`, or a specific LND like `o2iblnd`, `socklnd`,
     `ksocklnd`.
   - Utilities under `{lnet,lustre}/utils/` use `utils` (but the liblustreapi
     library is a separate concern — see lustre-utils.md).
   - Tests use `tests`; build/packaging use `build`; documentation uses `doc`.
   - Cross-cutting meta-components are allowed when the change genuinely spans
     layers, e.g. `clio` for client IO spanning llite/vvp..osc, or a feature
     name like `pcc`, `sec`, `nodemap`, `hsm`, `dne`, `wbc`.
   - Flag a prefix that does not correspond to where the bulk of the change is
     (e.g. subject says `osc:` but the diff is entirely/mostly in `mdt/`).

3. **Summary** — one concise imperative phrase describing what the patch does.
   Keep the whole subject within ~62 characters where practical. Flag an empty,
   vague ("fix bug", "update code"), or duplicated-from-body summary.
   - The subject and body should name the actual symbols changed — the new
     parameter, struct, function, or command name — so the patch is findable
     later via `git log` grep. Flag a message that describes the change only in
     the abstract ("add a new parameter") without naming it.

## Body completeness (accidental / unrelated changes)

The body must explain the change such that nothing in the diff is a surprise.

- The body should open with an introductory paragraph saying what the patch
  accomplishes and why, before describing how. Flag a body that jumps straight
  into implementation detail with no statement of intent.
- Reference functions with `()` in prose, arrays with `[]`,
  and quote command/parameter names with
  backticks (`` `lfs pool pin` ``, `` `osc.*.max_pages_per_rpc` ``).
- Build a mental list of every distinct change in the diff (reuse the CHANGE
  CATEGORIES from review-core.md).
- For each one, confirm the commit message accounts for it.
- **Any behavior-changing or user-visible change in the diff not explained by
  the commit message must be flagged** and questioned as possibly
  unrelated/accidental. It is common for a dirty working tree to get an
  unrelated hunk committed by accident (a debug print left in, an unrelated
  file, a reverted-then-reapplied line, a bumped version). Ask whether that hunk
  belongs in this patch or should be split out.
- **Do not flag trivial drive-by cleanups as undescribed, and do not suggest
  splitting them out.** Maintainers explicitly tolerate small style/whitespace
  cleanups to nearby code ("to avoid the overhead of testing/reviewing separate
  patches") and consider them not worth a commit-message mention: whitespace or
  reflow, dropping a redundant `!= NULL`, blank-line changes, reordering
  declarations, an obvious one-character fix or guard next to the real change.
  A commit message need not document *everything*; only what changes behavior.
- For a large mechanical series (checkpatch cleanups, kernel-doc fixes, and the
  like) a generic templated commit message shared across the series is
  acceptable; don't demand a bespoke description per patch.
- Flag a body written as a **delta from a previous patchset** ("compared to v3
  this now also ...", "reworked per review"): the message must describe the
  change against master, not against an earlier revision of itself.
- Even when an extra change is deliberate, an unrelated bug fix, code move, or
  independently-landable piece should usually go in its own patch with a new
  `Change-Id:` so it can be reviewed and land separately. The exception is a bug
  in the very code being modified that is low-complexity. Suggest splitting when
  the diff mixes clearly independent concerns.
- Conversely, claims in the message with no corresponding code (a described
  behavior the diff does not implement) are also regressions.
- **Exception — space-to-tab whitespace conversion.** Lustre is legacy code that
  historically used spaces for alignment and is being converted to tabs (kernel
  style). It is accepted and encouraged that a patch touching space-aligned code
  also converts some surrounding lines to tabs. So do **not** flag nearby
  whitespace hunks that change space alignment/indentation to tabs as unrelated
  or accidental, even if the commit message doesn't mention them. (This is not
  required either — see lustre-style.md — so don't demand it when it's absent.)
- **Exception - changes to better align code with `lustre-style.md`** Minor
  fixups to code style (e.g. alignment, error message formatting, variable
  naming and declaration ordering) that do not change code functionality do
  **not** need to be explicitly referenced in the commit summary.

## Required trailers

- `Signed-off-by:` must be present, and should use a real name, not an email
  address. Preserve any upstream `Signed-off-by:` lines on ported patches.
- `Change-Id: I...` is required by Gerrit; note if absent (a human can add it,
  but flag it).
- Order: `Signed-off-by:` comes before `Change-Id:`. A `Change-Id:` appearing
  first signals the Lustre commit hooks are not installed — flag it as `(style)`.
- `Test-Parameters:` is optional but encouraged for changes that need specific
  test coverage; do not require it.
- Do not flag the absence of `Reviewed-by:`/`Tested-by:` — Gerrit adds those.

## Fixes: tag enforcement

A patch that fixes a real bug should point at the commit that introduced it; a
pure feature/cleanup/refactor does not need a `Fixes:` tag. The mechanics
(format, 10+ char sha, existence/reachability, subject match, bug relationship)
live in `fixes-tag.md` and `missing-fixes-tag.md`, which review-core.md TASK 2.1
loads after determining bug-fix status. Don't restate them here.

## Tests

- A patch that adds new functionality should add tests exercising it
  (typically a new `test_NNN` in the relevant `lustre/tests/*.sh` suite, or a
  unit test under `lustre/kunit/`). Flag new user-visible functionality with no
  accompanying test as a question ("should this include a regression/coverage
  test in sanity*/conf-sanity/etc.?").
- A bug fix should ideally add a regression test that fails before and passes
  after. Strongly encourage it; raise it as a suggestion, not a hard failure.
- Worked example: commit 50aaabfc... added the `projid_set` rbac role and
  introduced `sanity-sec test_64j` in the same patch to exercise it.
- For how Lustre tests are written (subtest numbering, version-gating,
  Test-Parameters interop, shell style), see `subsystem/tests.md`, which loads
  for changes under `lustre/tests/`.

## Output

Emit, in your running analysis (not the gerrit-review.json artifact unless a
problem is found):

    COMMIT MESSAGE CHECK:
      Ticket: LU-XXXXX (exists: yes/no, matches diff: yes/no)
      Subsystem prefix: <tag> (matches diff: yes/no)
      Unexplained diff hunks: <count> (list them)
      Fixes: tag: present/absent/not-needed
      Tests added: yes/no/not-needed
