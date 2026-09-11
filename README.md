# TSC Battleship 6800 (SL68-23) — Restoration Notes

## What this is

`BATTLESHIP 6800`, copyright 1977 by Technical Systems Consultants
(Box 2574, W. Lafayette, Indiana 47906), catalog number SL68-23. A
two-player-style Battleship implementation for the Motorola 6800,
where the computer plays the opposing fleet. As far as we've been able
to determine, no binary or source file of this program exists anywhere
online, in any archive, or in any vintage-computing collection. The
only surviving record was a scanned PDF of a printed assembly listing
and object-code dump — no working copy is known to have existed in
digital form before this restoration.

**As far as we can tell, this is the first time this program has run
in roughly 40 years.**

## Source material

The restoration was built entirely from a PDF listing containing:

- A full commented 6800 assembly source listing (mnemonics, operands,
  and per-instruction addresses/bytes), roughly pages 1–17.
- A compact "OBJECT CODE" hex dump in Motorola S-record-like form
  (pages 18–19), covering the same address range.
- A sample program transcript showing expected gameplay output.

No original `.asm` or `.s19`/`.hex` file was provided or is known to
exist — everything here was reconstructed from the scanned listing.

## Restoration method

The dense hex dump (pages 18–19) turned out to be the *less* reliable
of the two sources for OCR purposes — long runs of hex digits are easy
for OCR to misread, and hard for a human to sanity-check by eye. The
per-instruction disassembly listing was used as the primary source
instead, since:

- Every line states its own address explicitly.
- Each opcode/operand is unambiguous in context (a valid 6800 mnemonic
  next to its address, rather than a bare hex digit that could be
  misread).
- Errors are self-revealing: if a line's byte count doesn't exactly
  match the gap to the next line's stated address, something was
  mistranscribed.

That last property was used as an automated check: the full program
was reconstructed instruction-by-instruction, then every address from
`$0100` to `$07E4` was checked for gaps. This caught one real
transcription gap (see "Known reconstruction uncertainty" below) which
was otherwise invisible until checked programmatically.

The message/text tables were built from the plain English strings
printed in the listing (e.g. `FCC ;AIRCRAFT CARRIER;`) rather than
from hex, encoded with the high bit set on each character to match
this program's own convention (confirmed by decoding the finished
table back to text and checking it reads correctly — it does,
including the ship-name typo noted below). Every string's length was
also cross-checked against the address of the *next* entry in the
listing, catching any off-by-one transcription errors automatically.

The reconstructed binary was spot-checked against several rows of the
OBJECT CODE hex dump's own per-line checksums as an independent
sanity check.

## Known reconstruction uncertainty

One string — the column-header line printed above the ocean grid
(`"1 2 3 4 5 6 7 8"`) — computed two bytes shorter than the address
gap in the listing required. This is almost certainly extra leading
spaces to align the header under the row-letter column (rows are
printed as `"A "`-style two-character prefixes), and has been padded
accordingly. This is a **cosmetic-only** uncertainty (column
alignment), not a logic or gameplay difference — if the header doesn't
line up perfectly with the grid in your terminal, this is why.

Everything else in the reconstruction is either a direct, verified
transcription from the listing, or (in one case, described below) a
narrow, well-reasoned patch clearly separated from the original code.

## Files in this repository

- **`battleship_6800_original.s19`** — the restoration as transcribed,
  completely unmodified, including the original 1977 bugs/typos
  described below. This is the byte-for-byte historical artifact.
- **`battleship_6800_cruiser_fix.s19`** — the above, with only the
  `CRUISER` → `CRUISER` spelling fixed (see below). No other changes.
- **`battleship_6800_fixed.s19`** — the above two fixes, plus a patch
  for a genuine functional bug that prevented ship-type confirmation
  from ever succeeding on some systems (see below). This is the
  version worth using to actually *play* the game.
- **`galaxy_6800.asm`, `galaxy_swtbug.asm`, `galaxy_swtbug.s19`,
  `galaxy.s19`** — a separate SCELBI-derived Star Trek-style game
  ("GALAXY") restored and debugged during the same session, included
  for reference. Documented separately below.
- **`hexdump.asm`** — a small memory-dump utility written from scratch
  for the Altair 680b's ACIA-based Turnkey Monitor, used as a
  diagnostic tool throughout this project.

## Bugs found

### 1. "CRUSIER" — original 1977 typo (preserved, optionally fixed)

The ship name is spelled "CRUSIER" instead of "CRUISER" throughout the
program's text table. This matches the sample output shown in the
*original manual* exactly, confirming it's an authentic typo in the
shipped 1977 product, not a transcription error introduced during
restoration. It's preserved as-is in `battleship_6800_original.s19`
for historical accuracy, and corrected in the other two builds (a
same-length two-letter swap, so no addresses shift).

### 2. Ship-type confirmation always rejected as cheating (functional bug)

**Symptom:** After the computer scores a hit and you correctly answer
`HIT` (`H`), the game asks `SHIP TYPE?`. Answering with the actual
ship's initial (e.g. `A` for a genuine Aircraft Carrier hit) is
rejected with `NO CHEATING!`, every time, regardless of the correct
answer given.

**Root cause:** Every text string in this program (ship names,
prompts, messages) is stored with the high bit (bit 7) set on each
character — a common convention on 1970s serial terminals/monitors
that used "mark parity" or simply expected 8-bit-clean links. Most of
the program's input checks (hit/miss, yes/no, beginner/master rating)
compare a keystroke against a plain 7-bit literal baked directly into
the code, so they're unaffected. The ship-type check is the one
exception: it compares your keystroke directly against **the first
letter of the ship's name as stored in the text table** — which has
the high bit set. On a monitor that delivers keystrokes as clean 7-bit
ASCII (confirmed behavior of SWTBUG's `INEEE`/`$E1AC`, which explicitly
strips the high bit before returning a character), that comparison can
never succeed, for any ship, regardless of what's typed.

This is very likely a genuine period compatibility issue rather than a
bug the original author would have seen: the program's own manual says
it needs four patch points (input, output, stack location, monitor
entry point) to run on anything other than the specific MIKBUG-family
monitor it was developed against, and the original test environment
plausibly delivered characters with the high bit already set.

**Fix:** Rather than forcing the high bit onto every keystroke (which
would break the dozens of *other* comparisons in the program that
correctly expect plain 7-bit ASCII), the fix is narrowly targeted: the
three bytes at `$0424` (the broken comparison) are replaced with a
jump to a small routine placed in free memory just past the end of the
original program (`$07E5`–`$07F0`), which strips the high bit from the
*table* byte before doing the same comparison and branch the original
code always did. No other game logic, table, or address is touched.

Confirmed working end-to-end: ship placement, firing, hit/miss
detection and reporting, and ship-type confirmation for at least two
ship types (Battleship, Submarine) on real hardware.

## Hardware/software this was tested on

- Altair 680b, ACIA-based Turnkey Monitor (TURMON)
- SWTPC 6800 running SWTBUG, real MC6850 ACIA at `$8004`/`$8005`

Along the way, restoring and debugging GALAXY on the same two machines
surfaced several unrelated hardware/firmware quirks worth documenting
for anyone doing similar work on this class of vintage 6800 system:

- The original 6800's `CPX` instruction only reliably sets flags for
  equality (`BEQ`/`BNE`); using it with `BHI`/`BLS`/etc. gives
  unreliable results (fixed in later parts like the 6801, not present
  here).
- VTL-2's `/` operator on this hardware does not handle negative
  (two's-complement) values correctly, requiring an explicit
  sign-handling routine for any fixed-point arithmetic.
- SWTBUG performs two *separate* PIA-vs-ACIA hardware detection checks
  — one at power-up, one on every `INEEE`/`OUTEEE` call — and they can
  disagree with each other depending on the specific board's address
  decoding, silently routing character I/O through an entirely wrong,
  software-timed code path.

## A note on authenticity

Where this restoration required a judgment call, the aim throughout
was to preserve what TSC actually shipped — typo and all — rather than
a cleaned-up modern reinterpretation. The `_original` build exists so
that anyone who wants the unaltered 1977 artifact, quirks included,
has it. The `_fixed` build exists so the game is actually playable on
common vintage hardware without hitting a bug that makes ship-type
confirmation permanently impossible.
