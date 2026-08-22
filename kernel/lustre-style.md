# Lustre code-style rules

Lustre-specific C coding-style conventions that maintainers (notably Andreas
Dilger) raise repeatedly in Gerrit review. Distilled from ~600 reviewed changes
/ ~4,500 inline comments. These apply to Lustre C code anywhere in the tree, so
this file is always loaded.

Mark each finding with the severity marker defined in gerrit-review.md
(`(defect)`/`(style)`/`(minor)`/`(typo)`/`(suggestion)`) and follow its phrasing
rules.

## Logging / error messages — `(style)`

- Start console error messages with a device name `device: error message`.
  If a specific `struct obd_device` is involved, then this uses `obd->obd_name`
  to print the name.  For llite this uses `sbi->ll_fsname` to print the name.
- End console/error messages with `: rc = %d\n` (use `%lld` and cast for 64-bit
  `rc`). The form is `"dev: message: rc = %d\n"`.
- Set `rc = -EXXX;` first, then use `rc` in both the message and the `return`/
  `GOTO`, so they can never drift apart.
- Include the relevant value/limit in the message, not just static text
  (e.g. print the limit so an admin sees it).
- Don't split an error message string across source lines (even if checkpatch
  suggests it); keep it readable and under 80 columns whenever possible.
- Cast 64-bit values to `%lld`/`%llu` in any format string.
- Remove a trailing space before `\n`.

## Error handling / cleanup — `(style)` / `(minor)` / `(defect)`

- Handle errors out-of-line at the end of the function with
  `GOTO(out_xxx, rc = -ERR);` rather than duplicating cleanup at each error site.
- Use a separate label per resource acquired (`out_free`, `out_put`, ...) instead
  of one label guarded by extra NULL checks.
- When there's nothing to clean up, `RETURN(-ENOMEM)` directly.
- Don't NULL-check before `OBD_FREE()` / `kfree()` / a freer that already handles
  NULL; remove the redundant check (and from helpers that free internally too).
- `(defect)` Every allocation must be freed on **every** error return path. The
  most common leak: a validation that fails *after* the allocation (a later
  `snprintf` overflow, a size check) returns directly instead of routing through
  the cleanup label. When you add an error path below an allocation, send it to
  the existing label.
- `(defect)` Free only what this function allocated. If a buffer/`op_data` may be
  either caller-supplied or allocated here, track an "allocated here" bool and
  free only in that case — otherwise you free a borrowed pointer or double-free.

## Copy-paste and parallel branches — `(defect)`

Near-duplicate code blocks are where the second copy silently keeps the first
copy's field or constant. When two branches differ only by one token, verify
every token that should differ actually does. Real recurring bugs:
- wrong field in the parallel branch (`nm_offset_start_uid` used in the GID
  branch; rx buffer size assigned to `sk_sndbuf`);
- wrong constant copied (`CONNECTION_SWITCH_MAX` where `MIN` was meant;
  `HRTIMER_MODE_REL` vs `HRTIMER_MODE_ABS`);
- transposed symmetric counters (PUT vs GET, send vs recv);
- `memcpy(&ptr, src, n)` (copies the pointer bytes) where `memcpy(ptr, src, n)`
  was meant;
- a dispatch-table entry wired to the wrong `jt_*` handler.

## Buffers and strings — `(defect)`

- Size a fixed buffer to include the NUL terminator (`char name[NAME_MAX + 1]`),
  and bound writes with `>= sizeof(buf)`, not `> sizeof(buf)`.
- Use `strscpy()` rather than `strncpy()` so a truncated copy is still
  NUL-terminated; use `memmove(dst, src, strlen(src) + 1)` to carry the
  terminator when shifting a string.
- Assign an object's "actual size" only **after** its bounds/validation check
  passes; assigning the raw/minimum size first and validating later corrupts the
  tracked size.

## LASSERT / LBUG discipline — `(minor)` / `(defect)`

- Don't add a `LASSERT` for a condition that would OOPS on the next line anyway
  (e.g. a NULL guard right before the deref) — it doesn't improve the code.
- Prefer a compile-time `BUILD_BUG_ON()` / `CLASSERT()` or returning an error over
  a runtime `LBUG()`.
- If an assert-fail is truly needed, use `LASSERTF(0, "opc = %u\n", opc);` so the
  triggering value is captured.
- Never `LASSERT()` on data received over the network — validate and return an
  error instead.

## Naming — `(style)`

- Struct fields need a per-struct lowercase prefix derived from the struct name
  (e.g. `cep_proj_id`, `gis_*`, `gicc_*`). Flag unprefixed fields.
- Functions need unique, non-generic names to avoid clashing across subsystems;
  prefer `noun_verb` so related functions sort together (`xattr_get`,
  `xattr_set`, `xattr_clear`).
- Spell names out instead of cryptic abbreviations; module parameter names should
  include the module name.
- Don't name things after version numbers; name by purpose
  (`lst_force_large_nid`, not `..._v2`).

## Formatting — `(style)`

- Wrap at 80 columns.
- Use tabs for indentation and alignment in new/changed code (kernel style).
  Lustre is legacy code much of which still uses spaces for alignment; the rule
  of thumb is to convert some surrounding lines to tabs when you touch nearby
  code. This is **not enforced** — do not warn when a patch leaves surrounding
  space-aligned code unconverted. And when a patch *does* convert nearby lines to
  tabs, that is welcome, not an unrelated change (see lustre-commit-message.md).
- Indent continuation lines one space after the opening parenthesis on the previous line which the wrapped line is nested inside of.  If the alignment to he parenthesis is so deep that it causes the indented line(s) to exceed 80 columns, or this is not a nested statement, then it should be indented one extra tab beyond the parent statement.
- Remove extra/trailing blank lines and double or trailing spaces.
- Keep option lists, enum entries, and `#include`s in alphabetical order.
  Includes are grouped kernel, then lustre, then local, alphabetical within
  each group.
- Use designated initializers in struct/option tables
  (`{ .val = 'c', .name = "cache", .has_arg = required_argument }`).
- Local variable declarations should have only a single space between the variable type and the variable name. Older code used tabs for aligning the variable names, but this style is deprecated and patches modifying such a local variable declaration block may remove this alignment.
- variable declarations in struct definitions should be tab aligned.

## Constants / magic numbers — `(style)`

- Don't hard-code pathnames or magic values; expose them via a module parameter
  or a named constant.
