# Wire Protocol and Interoperability

Lustre clients and servers of different versions must interoperate. Any change
to an on-the-wire definition is a compatibility-critical change and must be
analyzed as a potential regression, even when the C code looks correct in
isolation.

## On-wire definitions

On-wire structures, enums and constants live mostly in the uapi idl header
(`include/uapi/linux/lustre/lustre_idl.h` and the other
`include/uapi/linux/lustre/*.h` headers). These describe the bytes exchanged
between hosts, so they are an ABI, not just an API.

Treat the following as wire definitions:
- structs sent in RPC request/reply buffers
- enums and `#define` flag values stored in those structs or negotiated at
  connect time (e.g. `OBD_CONNECT_*`, `OBD_CONNECT2_*`, `*_INCOMPAT`,
  `*_ROCOMPAT` feature flags)
- RPC opcodes, magic values, and version constants
- anything checked by `wirecheck.c` / `wiretest.c`

## Required companion updates (mandatory check)

When a wire definition is added, changed, removed, or renumbered, the patch
**must** keep the wire-test infrastructure in sync. Flag as a regression if a
wire change is missing any of:

- `lustre/utils/wirecheck.c` — generator: add/adjust the matching
  `CHECK_VALUE`, `CHECK_VALUE_X`, `CHECK_VALUE_O`, `CHECK_MEMBER`,
  `CHECK_STRUCT`, `CHECK_DEFINE_*`, `CHECK_BITFIELD` entry.
- `lustre/utils/wiretest.c` and `lustre/ptlrpc/wiretest.c` — generated
  assertions: the matching `LASSERTF(... == 0x....)` / `BUILD_BUG_ON` /
  offset and sizeof checks must be regenerated. Both copies must agree.
- `lustre/utils/wirehdr.c` — only if new headers are pulled in.

Worked example: commit 50aaabfc16b228b563bed2c6d96d1c32fa1a486d added one
`NODEMAP_RBAC_PROJID_SET = 0x00001000` value to an enum in `lustre_idl.h`. The
same patch added `CHECK_VALUE_X(NODEMAP_RBAC_PROJID_SET)` to `wirecheck.c` and a
new `LASSERTF(NODEMAP_RBAC_PROJID_SET == 0x00001000UL, ...)` to both
`wiretest.c` files, and it also had to update the dependent
`NODEMAP_RBAC_NONE == 0xffffe000UL` assertion because the mask value changed.
Watch for this kind of *dependent* constant (masks, "ALL"/"NONE" values,
totals) that shifts when a member is added.

If a wire member is added without the corresponding wiretest update, the build's
own consistency check will not catch the drift, and a mismatched peer can
misinterpret the buffer. Report the missing update as a regression.

## Interoperability logic (mandatory check)

Adding a field or behavior to the protocol is not enough — old and new peers
must still talk to each other. For any protocol change, look for the negotiation
that lets a peer detect whether the other side supports the new behavior:

- A new feature normally requires a new **connection flag**
  (`OBD_CONNECT_*` / `OBD_CONNECT2_*`) negotiated in the connect handshake, and
  code that only uses the new behavior when both sides set the flag.
- New on-disk or on-wire formats need an **incompat/rocompat** feature bit so
  that an older peer refuses or read-only-mounts rather than misreading data.
- New RPC opcodes or larger request/reply formats must degrade gracefully when
  talking to a peer that does not understand them.

Questions to raise as regressions when the interop story is missing:
- Does a new field get sent to (or expected from) a peer that predates it,
  with no flag guarding it?
- Is an existing wire struct **changed in place** (member reordered, resized,
  type changed, value renumbered) without a new flag/version? Changing the
  meaning of bytes an old peer already understands is almost always wrong —
  the existing layout is frozen; extend via a new flag-guarded field instead.
- Is a connect flag consumed (`exp_connect_flags`, `ocd_connect_flags`,
  `obd_connect_data`) but never advertised, or advertised but never honored?

Do not accept "the struct just grew one field" as safe by itself: confirm there
is explicit version/flag gating before the new field is read or written on the
wire.

Gate new wire behavior on a **negotiated connect flag**, not on a hardcoded peer
version number. Version-number checks (`peer >= 2.15`) are fragile and have been
reverted — backports and rolling upgrades break them. A new feature should add
an `OBD_CONNECT*` flag and only use the new behavior when both peers set it.
When the peer lacks the capability, **degrade gracefully** (fall back to the old
path / return 0 to retry buffered) rather than hard-failing the operation.

## ABI and UAPI struct rules — `(defect)`

These apply to anything in `include/uapi/linux/lustre/` or otherwise crossing the
kernel/userspace or on-wire boundary:

- **Explicit values for wire/ABI enums and flags.** Every enum entry and flag in
  a wire/UAPI definition must have an explicit value (`FOO = 1`, flag `0x...`) so
  inserting or removing an entry can't silently renumber the others. An unvalued
  wire enum is a defect. (And the new value must be added to wirecheck.c /
  wiretest.c per above.)
- **`__packed` only on wire/UAPI structs.** Don't mark internal in-memory structs
  `__packed`; it pessimizes access and signals confusion about what is on the
  wire. Flag `__packed` on a non-wire struct, and a missing `__packed` on a true
  wire struct that needs a fixed layout.
- **No userspace pointers into the kernel.** A UAPI/ioctl struct must not contain
  a pointer passed from userspace to the kernel. Put fixed fields first, store a
  length for each variable-sized buffer, and pack the buffers at the end. Flag a
  pointer-typed member in a UAPI struct.
- **Don't propose renames of uapi / on-wire / on-disk identifiers** (struct,
  field, enum, or command names) even when a clearer name exists — they are
  kept for compatibility with existing implementations.
- **Reserved fields and alignment.** Add reserved/padding fields and align 64-bit
  members on 8-byte boundaries in wire/UAPI structs, to allow future expansion
  without breaking layout. Suggest a `__u32 ..._reserved[]` where a struct
  grows to an odd size or has unaligned 64-bit fields.
- **Never `LASSERT()` on data received over the network** — a malicious or
  mismatched peer must not be able to crash the node. Validate and return an
  error (see lustre-style.md, LASSERT discipline).
- **Never `LASSERT()` on data read from persistent storage** - data corruption
  can happen on persistent storage and may present arbitrarily bad data.  Data
  read from disk or over the network should be sanity-checked first before the
  data is used for anything.  Fields that are used for memory allocation or
  array bounds checking must always be validated, using the reply buffer size
  or upper limits based on specified constants.

## Connection-flag allocation

`OBD_CONNECT_*` / `OBD_CONNECT2_*` bits are a shared, hand-managed resource and
get double-allocated when people grab one unilaterally.

- A newly added connect flag must use a bit that is confirmed free (reserved in
  advance with the maintainers). Flag a new `OBD_CONNECT*` value and ask whether
  the bit has been reserved, and check it does not collide with an existing
  definition in the same word.
- List a new flag alongside the other `OBD_CONNECT2_*` definitions.
- The connect flags are kernel-internal negotiation; the userspace UAPI cannot
  see them, so don't reference a connect flag from a userspace-only header.
- Access to fields in `struct obd_connect_data` are controlled by the presence
  of `OBD_CONNECT*` flags.  They should never be accessed if the flag is not
  set, since the size of the struct itself depends on the peer's code version.
  If the flag is set then it means the peer has prepared a large enough reply
  buffer for the field.

## Endianness / swabbing

On-wire integers are little-endian and are byte-swapped by `lustre_swab_*`
routines. A new or resized wire member usually needs its swab routine updated
too. Flag a new multi-byte wire member whose `lustre_swab_*` handler was not
updated.

## Request testing
A patch that changes anything related to wire protocol - either the structures
or any related processing logic - must add Test-Parameters tags to request
interop testing using serverversion (and/or clientversion) against a
**released** major version (e.g. `serverversion=2.16` / `2.17`). Interop
cannot be run against development tags, so never suggest a dev tag or a
4-component point release there. A server-only change already runs in interop
mode against old clients in the standard sessions; an explicit older-server
run is what to ask for when the client side changed.
Architecture interop testing could be requested with clientarch/serverarch
parameters (e.g `clientdistro=rocky9.5 clientarch=aarch64`).
