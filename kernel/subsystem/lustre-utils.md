# Lustre and LNet utils, liblustre library documentation info

- When updating any of the user visible functionality in utilities, make sure relevant manual pages are correspondingly updated/added/removed in Documentation/man* directories.
- When updating lustre/utils/lib* for liblustre library, make sure the API manual pages in Documentation/man3 are kept up to date.
- When new tunable parameter is added with LDEBUGFS_SEQ_FOPS* or LPROC_SEQ_FOPS* or LUSTRE_{RW,RO,WO}_ATTR or modified, the corresponding manual page in Documentation/man4 should be added or modified as appropriate.
- wirecheck.c, wiretest.c and wirehdr.c are special test files that don't need to be documented.

## Man pages belong in the same patch — `(minor)`

- A new user-visible tunable, command, or option should add/update its man page
  (man4 for `lctl` parameters, man8 for tools) in the **same** patch that adds
  the functionality, so the documentation is reviewed against the implementation
  and not forgotten. Flag a new parameter/command/option with no man page change.
- Add the `.so` man-page reference/symlink files for each new parameter
  (e.g. a `man4/<module>.<param>.4` that does `.so man8/<tool>.8`).
- This applies to module parameters added anywhere in the tree, not only to code
  under `{lnet,lustre}/utils/` — wherever a user-visible knob is introduced.
- Do **not** flag man-page metadata: placeholder or `.\" Added in commit ...`
  hashes (they can only be filled in after landing, and that line is emitted by
  checkpatch itself — removing it is an error), the `.TH` date, section
  ordering, or an AVAILABILITY/"since" version. The correct availability
  version is the next *release* (e.g. `2.18.0`), never a development tag such
  as `2.17.5x`.
- Man-page EXAMPLES are illustrative and need not compile verbatim. Generic
  problems in an example — a leaked handle, an unchecked return, a small compile
  error, a missing include — are worth pointing out but only as a `(nit)`
  (below `(minor)`): "if the patch is refreshed, ...", never a reason to
  re-spin.
- That is different from an example that **contradicts what the code actually
  does** — a wrong option name, wrong semantics, output the command doesn't
  produce, a call sequence that would not work. That is a real documentation
  bug that misleads users; report it at the severity the error warrants
  (`(minor)` normally, `(defect)` if following the example would do the wrong
  thing).
- Referencing an llapi man page that doesn't exist yet is not an error.

## Userspace tool checkpoints — `(defect)` unless noted

Distilled from two years of utils `Fixes:` commits:

- **getopt / command tables must be NULL-terminated.** `struct option
  long_opts[]` must end with `{ .name = NULL }`, and `command_t` arrays passed to
  `cfs_parser()` must end with a `{ .pc_name = NULL }` sentinel — a missing
  sentinel segfaults on an unknown option/command.
- **Use `ssize_t`, not `size_t`, for `read()`/`write()` results**, and handle the
  negative error before any unsigned comparison (an error return compared as
  unsigned reads as a huge positive).
- **Clean up on every error path**: each `malloc`/`calloc` needs a matching
  `free()` on all returns (use `goto out_<var>` labels). Only `close(fd)` when
  `fd >= 0`.
- **errno sign discipline**: store `rc = -errno`, and pass the matching sign to
  `strerror()` (`strerror(-rc)`); a negative argument to `strerror()`/an
  unsigned compare is a real bug.
- **Parse NIDs with the `cfs_nidstr_*` helpers**, not `strchr(':')` — IPv6 and
  large/multi-rail NIDs contain colons; reserve `MAXNIDSTR`/`LNET_NIDSTR_SIZE`.
- **Verify each dispatch-table entry calls its intended `jt_` handler** — a
  copy-pasted row that points at the wrong function silently runs the wrong
  subcommand.
- `(style)` **Output consumed by scripts must keep stable delimiters**: don't drop
  the field separator for a single value, and print a header only when there is
  content (`count > 0`, not `>= 0`). Tools and tests parse this output.
- `(minor)` Handle `-V`/`--version` before option parsing so it works without the
  otherwise-mandatory arguments.
