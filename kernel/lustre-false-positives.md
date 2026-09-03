# Lustre false-positive guide

Load this in TASK 3 alongside `false-positive-guide.md`. It holds Lustre and
kernel facts that AI reviews have repeatedly got wrong — each one was corrected
by a maintainer in Gerrit — plus calibration rules for findings that get
rejected as overkill or out of scope. Check every candidate finding against it
before reporting.

## Facts the reviewer has got wrong before

- `CERROR()` **is** rate-limited: it is defined via `CDEBUG_LIMIT`. Don't say
  "CERROR is not rate-limited" or ask to switch to a limited variant.
- `return res ?: ERR_PTR(-EINVAL);` never returns NULL — the `?:` form yields
  `res` when it is non-NULL, otherwise the error pointer. Don't flag a NULL
  return there.
- Netlink message buffers: a large skb is allocated via `kvmalloc()` with a
  vmalloc fallback — don't flag "too large for kmalloc".
- `snprintf()` does not set `errno` on ordinary truncation; don't claim it does.
- `find_or_create_page()` already passes `FGP_ACCESSED`; don't ask to add it.
- An enum value of `0` is reserved for "not used" by Lustre coding style, so
  there is never a real LND type 0 — don't flag its absence from a table.
- Changelog user names returned by the server are always prefixed `clN-`;
  parsing that assumes the prefix is correct, not fragile.
- `sanity.sh` uses a single mount; only `sanityn.sh` uses a second mount
  (`$MOUNT2`). Don't assume two mounts in `sanity`.
- MDS-to-MDS version skew: `target_handle_connect()` allows a difference of 3
  in the patch (third) component between running MDTs — don't flag skew within
  that window.
- Interop is never tested against servers older than the previous LTS release
  (2.15) — don't ask for compatibility with anything older, and see tests.md
  for the 3-component version rule.
- Kernel-version attribution: don't assert from memory which kernel version
  introduced or changed an API; if it matters, state it as an assumption or
  verify with `git describe --contains` in a kernel tree.
- Placeholder `.\" Added in commit ...` hashes and dev-tag versions in man
  pages are expected before landing (see lustre-utils.md).

## Calibration: what not to report

- **Require a concrete, reachable path before `(defect)`.** "Could be NULL" /
  "could overflow" without a caller that actually produces the condition is not
  a defect; check the callers and say which one triggers it.
- **Don't demand hardening of ENOMEM-only paths, admin/test-only code, or
  fault-injection paths.** `-ENOMEM` on an MDT already means severe memory
  pressure; asking for recovery machinery or extra diagnostics there is
  overkill. Such items are `(suggestion)` at most, and usually not worth
  raising.
- **Pre-existing issues:** if the problem predates the patch (same pattern
  elsewhere, introduced by an earlier commit), say so explicitly, label it
  pre-existing, and don't block the patch on it — suggest a separate change.
- **Helper extraction / de-duplication:** don't suggest factoring out a helper
  unless the duplication is introduced by this patch.
- **Don't propose renames of uapi / on-wire / on-disk identifiers**; they are
  kept for compatibility even when a better name exists (see wire-protocol.md).
- **Series patches:** an unused symbol, parameter, tunable, or man-page
  reference is usually consumed by the next patch in the relation chain (see
  review-core.md TASK 2 step 3) — verify before flagging.
- **kernel-doc series:** for a kernel-doc cleanup patch, run
  `./contrib/scripts/kernel-doc -v -none <file.c>` and report only the warnings
  the tool actually emits.
- When relying on kernel internals (allocator behavior, page-cache flags, the
  version of an API), state the assumption ("if X still holds ...") instead of
  asserting it as fact.
