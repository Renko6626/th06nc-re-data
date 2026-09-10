# th06nc-re-data

Reverse-engineering symbols for **Touhou Koumakyou: New Classic** (the 2026 Steam
remake of Embodiment of Scarlet Devil), in the shape of ExpHP's
[`th-re-data`](https://github.com/exphp-share/th-re-data).

Nobody had published anything for this build: truth's `v0.anmm` stops at ANM
opcode 31, `th-re-data` has no NC entry, and `th06-re` targets the 2002 original.
This is a first pass.

**Sample**: `th06nc.exe`, MD5 `b0cc28ec904e9efce2439ad857c938b5` (x86-64, imagebase
`0x140000000`). Addresses are meaningless against any other build — check the
hash first.

## What's here

| Path | Contents |
| --- | --- |
| `data/th06nc/funcs.json` | 36 named functions |
| `data/th06nc/statics.json` | 18 named globals |
| `data/th06nc/labels.json` | 142 named code labels, incl. all 120 ECL opcode handlers |
| `data/th06nc/comments.json` | 183 annotations not attached to the above |
| `mapfiles/th06nc.anmm` | the ANM opcodes NC added (overlay) |
| `mapfiles/th06nc.eclm` | the ECL opcodes NC added (overlay) |

`funcs.json` / `statics.json` use `th-re-data`'s schema verbatim. `labels.json`
deviates: `th-re-data` stores `{group: [[addr, suffix], ...]}` and builds names
from the pair, while ours are standalone names, so it is a flat
`[{addr, name, comment}]` list instead.

## The headline finding: four new ANM opcodes

TH06's ANM VM dispatches opcodes 0..31. NC's dispatches **0..35** — the bound is
a literal `CMP DL,0x23 / JA` at `0x140006a50`, and the jump table at
`0x140007494` has 36 entries. Scanning all 125 shipped `.anm` files turns up no
opcode >= 36, so the range is exactly 0..35.

All four additions are zero-argument anchor setters. Each is a two-instruction
`AND`/`OR` pair on the VM flags word at `vm+0xc4`, writing a 3-bit field at bits
8..10. Every one of those bits is either force-cleared by the AND or force-set by
the OR, so they are absolute assignments, not toggles.

| opcode | provisional name | field | x align | y align |
| --- | --- | --- | --- | --- |
| 32 | `anchorTopCenter` | 3 | center | top |
| 33 | `anchorLeft` | 1 | left | center |
| 34 | `anchorTopRight` | 5 | right | top |
| 35 | `anchorRight` | 2 | right | center |

For reference, the pre-existing opcode 23 (`anchorTopLeft`) sets field 4, and the
default (no opcode) is field 0 = centered on both axes. **There is no bottom
row** — the grid is 3x2, not the 3x3 that modern ANM's parameterised `anchor`
instruction gives you.

### Don't carry TH06's bit patterns over

TH06 encodes the anchor as two independent switches: bit 8 = "x anchored left",
bit 9 = "y anchored top", and its opcode 23 is a plain `OR flags, 0x300`. NC
turned the same three bits into an enum compared by value, and rewrote opcode 23
as `AND ~0x300` + `OR 0x400`. So opcode 23 still means top-left in both, but the
bit pattern `0x300` means *top-left* in TH06 and *top-center* in NC. Opcode-level
semantics carry over; bit patterns do not.

Opcode 33 is implemented but unused in the shipped data — as are TH06's own
opcodes 6 (`nop`) and 8 (`flipY`), so that is not a reason to doubt it.

## The ECL side: one real addition, one dead alias

NC's ECL dispatch takes `opcode - 1`, bounds it with `CMP EAX,0xc8`, and goes
through a byte map at `0x140028f5c` into a jump table at `0x140028d78` — so it
accepts opcodes **1..201**, against TH06's **1..135** (a direct 135-entry table
at `0x40a9a4`, no byte map).

That extra range is mostly empty. Decoding both tables entry by entry:

- **136..199 all land on the dispatch default.** So do TH06's and NC's opcode
  127 (`DEBUGWATCH`) — the original leaves it unimplemented too.
- Within 1..135 the implemented/unimplemented sets are **identical in both
  directions** — the diff is empty each way.
- TH06 has 120 distinct handlers, NC has 121. The one extra is opcode 200's.

`labels.json` names **all 120 real handlers**, as `ECL_OP_<opcode>_<NAME>`, with a
comment on each listing every opcode that reaches it. The mapping is decoded from
the binary's own byte map and jump table rather than transcribed, so it cannot
drift from the sample. Sharing is informative in itself: opcodes 13/20, 14/21,
15/22, 16/23 and 17/24 pair the integer and float forms of add/sub/mul/div/mod
onto one handler each, 4/5 share, 85/86 share, and 67..75 share with 201.

**Opcode 200** is a genuine new instruction: a polar form of TH06's
`SHOOTOFFSET`, computing `shoot_offset = (cos(a)*r, sin(a)*r, 0)` at
`0x140026a41` and writing `enemy+0xc5c`.

**Opcode 201 is not a new bullet pattern.** It shares jump-table slot 60 with
opcodes 67..75, and that handler stores `aim_mode = opcode - 67` — so 134 for
opcode 201. The consumer, `0x14000f870`, switches on `bullet_props+0x30` with
cases 0..8 and **no default**, leaving the angle at its pre-switch `0.0`.
Searching the whole binary for a comparison against 134 turns up 18 hits, all
unrelated loop and array bounds. Nothing special-cases it.

Caveat: that is a static result. Opcode 201 appears 13 times in the shipped
scripts, which is a lot of uses for an instruction that looks degenerate, so it
has not been confirmed on a running game.

## Confidence

Names are ours and provisional where marked. The addresses, dispatch bound,
instruction bytes and file-side statistics are first-hand off the binary. The
*motivation* for the new opcodes is not established: they appear almost only in
the localized `text*.anm` files, whose entries are runtime-created blank text
slots, but neither TH06 nor NC resizes a sprite to the measured text width, so
the obvious "because translations have different widths" story does not hold as
stated.

## Where the names come from

The ECL opcode names (`JUMPDEC`, `BULLETFANAIMED`, `SHOOTOFFSET`, …) are taken
verbatim from the `EclRawInstrOpcode` enum in
[GensokyoClub/th06](https://github.com/GensokyoClub/th06), which is released
under **CC0** — so they can be reused and redistributed freely. Our copy is
checked against that header on every test run, so it cannot silently drift.

Everything else — the addresses, the opcode-to-handler mapping, the anchor
analysis, the annotations — is ours.

## What is NOT here

No game code, no game data, no assets — only addresses, names we invented, and
notes we wrote. Nothing is forwarded from upstream projects: ExpHP's
`th-re-data` and the `th06-re` XML both ship without a license, and truth's
mapfiles are marked personal-use, so none of their content appears here. The
mapfiles are deliberately overlays containing only what we derived ourselves.

## License

[CC0-1.0](LICENSE). ExpHP's repo has no license, which makes it awkward to build
on; we would rather not repeat that.
