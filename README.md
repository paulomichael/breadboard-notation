# breadboard-notation

**The Breadboard Notation Format (BBN)**

Document, version, and verify breadboard circuits as plain text.
Print an overlay, stick it on your board, and write the circuit down —
by hand or in any text editor. No apps, no proprietary format, no vendor lock-in.

[image: hero shot — a breadboard with the printed overlay applied, sitting next to a text editor open to a .bbn.md file. The whole pitch in one frame.](todo)

## Quick Start

### 1. Print the overlay for your board

Download the PDF for your breadboard model from `/boards/`.
Print at 100% scale (not "fit to page"). A calibration ruler is included.
Stick it on. Your board now matches a standard coordinate grid.

### 2. Write the circuit as text

`blinker.bbn.md`

```bbn
board: 830-ABCD-v1

R1   330   a5   a10    # current limiter
R2   10k   a15  a20
C1   10u   b20  b25    # polarized, long leg to b20
LED1 -     b10  b11
U1   555   f1   f8
wire j10 +rail         # red jumper
wire j10 -rail         # black jumper
```

Notes, schematics, and status go outside the block.
The block is the contract. Everything else is commentary.

### 3. Verify (optional)

Run the converter to get a netlist, and check it against your schematic.

```
$ bbn netlist blinker.bbn.md > blinker.sp
$ bbn verify blinker.bbn.md schematic.kicad_sch
```

That's it. The rest of this document explains the details.

## The Core Idea

### Breadboards are already a coordinate grid

Every hole has a position: row letter + column number (`b12`, `f27`).
The board is internally wired: each 5-hole column group is one net.
The grid is there. It's just inconsistently labeled.

[image: zoomed crop — the printed overlay showing "b12" next to a hole, with an arrow pointing to "b12" in a text file. The mapping made visual.](todo)

### We make it consistent

A printable overlay labels your board with a known scheme.
A plain-text file describes what you plugged in where.
A converter turns that into a netlist a simulator can read.

### Text as the source of truth

No screenshots. No Fritzing files. No proprietary format.
Your build is a text file. It diffs in git. It survives.
Coordinates are layout-specific. Nets are not.
The format keeps both: physical position for rebuilding,
net relationships for verification.

### Before / After

**Without BBN:**

A photo where half the jumpers are hidden under other jumpers.
A Fritzing file that only opens in one program.
"I think I used a 10k here but the label's gone."

**With BBN:**

`blinker.bbn.md` — 7 lines, complete, diffable, verifiable.

[image: before/after — left: a messy breadboard photo with tangled jumpers; right: the same circuit as a clean 7-line .bbn.md file.](todo)

## Directory Structure

```
/boards
  830-ABCD-v1/
    board.json        # internal hole groupings
    notes.md          # dimensions, print instructions
    overlay.pdf       # generated
    overlay.svg       # generated

/lib
  555.json            # pinout, DIP-8
  2n2222.json         # pinout, TO-92
  ...

/tools
  make_overlay.py     # board.json -> pdf/svg
  parse_build.py      # .bbn / .bbn.md -> netlist
  verify.py           # netlist vs schematic

/examples
  blinker_555.bbn.md
  transistor_switch.bbn.md
  power_supply.bbn.md

/docs
  FORMAT.md           # when README gets too long
  PRINTING.md
  BOARDS.md
```

## Basic Usage (No Terminal Required)

### Document a build

1. Print the overlay for your board model.
2. Build your circuit.
3. Open a text editor. Write one line per component or wire,
   using the coordinates printed on the overlay.
4. Save as `.bbn.md` (or plain `.bbn` for pure notation).

### Rebuild from a file

1. Open the `.bbn.md` file.
2. Place components at the listed coordinates.
3. Done. Exact reproduction, no guessing.

### Share a build

Send the text file. That's the whole thing.
The other person prints the same overlay, places parts,
and has your exact circuit.

## File Formats

Two forms, one notation:

```
.bbn      Pure notation. Starts with `board:`.
          For tools, piping, embedding, generating.

.bbn.md   Markdown with one or more ```bbn fenced blocks.
          For humans, git, sharing, teaching.
```

Parsing rule:

1. File starts with `board:` — parse whole file.
2. Otherwise — concatenate every ` ```bbn ` block.
3. Plain ` ``` ` fences are ignored by tools — use them for
   prose examples that should not be machine-read.

Tools never read prose. The fence is the only contract.

## Verification (Optional)

This is where the format earns its keep.

The pipeline:

```
.bbn(.md)  +  board.json  ->  netlist  ->  compare to schematic
```

[image: pipeline diagram — boxes: ".bbn file" + "board.json" → "netlist" → "schematic" with a checkmark. Four boxes, three arrows.](todo)

The board model knows which holes are internally connected.
The build file knows which pins you inserted where.
Together they produce a complete netlist.
That netlist is diffed against your schematic's netlist.
Mismatches mean a miswired jumper — caught before power-on.

```
$ bbn netlist blinker.bbn.md
$ bbn verify blinker.bbn.md schematic.net
```

## Advanced Usage (Optional)

The following sections are entirely optional.
The core system works with a text editor and a printer.

### Version control with git

```
cd my-circuits/
git init
git add .
git commit -m "Blinker 555 working"
```

Every change is tracked. Every build is reproducible.

### Custom board models

If your board isn't in `/boards/`, copy an existing `board.json`,
edit the hole groupings and dimensions, and regenerate the overlay.

```
$ bbn overlay my-board.json -o my-board.pdf
```

### Round-trip with existing tools

Import from Fritzing, Wokwi, or a KiCad netlist.
Export to SPICE, KiCad, or a plain netlist.
The format is a bridge, not a walled garden.

## File Manager Tips

```
In Thunar:      View -> Side Pane -> Tree
In Nautilus:    View -> Sidebar -> Tree
In Finder:      View -> Show Sidebar -> Column View
In Explorer:    View -> Navigation Pane -> Expand folders
```

Terminal:

```
tree -L 2 ~/my-circuits/
```

## Reference

### The .bbn format

```
board: <model-id>              # required, first line of the block
<ref> <value> <coord> <coord>          # 2-pin component
<ref> <value> <coord> <coord> <coord>  # 3-pin component (transistor, pot)
<ref> <value> <pin>=<coord> ...        # explicit pin mapping
wire <coord> <coord>           # jumper wire
# comment                      # end-of-line or whole-line
```

### Coordinates

```
<row><column>   e.g. a5, b12, j27
+rail, -rail    power rails
Rows a-e and f-j are the two halves of the main strip.
```

### Components with more than two pins

**Three-pin parts** take three coordinates in pin order:

```bbn
Q1   2N2222 b5   b6   b7      # B C E (BJT)
Q1   BS170  b5   b6   b7      # G D S (FET)
RV1  10k    a5   a6   a7      # pot: 1 wiper 3
```

**ICs** can be written two ways:

*Short form* — if the tool knows the part's pinout:

```bbn
U1   555   f1   f8            # DIP-8, pins 1..8 left-to-right
```

*Long form* — explicit pin mapping, always unambiguous:

```bbn
U1   555   f1=1  f2=2  f3=3  f4=4  f5=5  f6=6  f7=7  f8=8
```

The short form is what people will actually type for common parts.
The long form is the escape hatch for anything else.

### Reserved words

```
board    declares the board model
wire     a jumper wire
```

Use descriptive names for anything else:
`rail_wire`, `wire1`, `wire_rail`, etc.

### Values are freeform

Tools verify connectivity, not values.
`330`, `330R`, `330ohm`, `10k`, `0.1u` — all pass through unchanged.

### Rules

- Line order is not significant.
- Duplicate coordinates are an error, not a warning.
- Comments use `#` and may follow any line.

### Naming conventions

```
R1, R2...    resistors
C1, C2...    capacitors
U1, U2...    ICs
LED1...      LEDs
D1...        diodes
Q1...        transistors
RV1...       potentiometers
SW1...       switches
```

### Status markers (in comments, not filenames)

```
# status: verified
# status: untested
# note: swapped R2 for 4.7k on second build
```

## FAQ

**Q: Why not just use Fritzing / Wokwi / a photo?**

A: Those are fine for a single build. This is for builds you want to
keep, share as text, diff in git, or verify against a schematic.
It's a complement, not a replacement.

**Q: My board doesn't have letters or numbers printed on it.**

A: That's exactly the problem this solves. Print the overlay.

**Q: My board's markings are different from the standard.**

A: Print the overlay anyway. The overlay defines the coordinate system,
not the board.

**Q: Why letter-based coordinates instead of numeric?**

A: Chessboard-style rows (a-j) plus columns (1-30) read more naturally
and are less ambiguous than two numbers. "b12" is a position.
"12-5" is a question.

**Q: How do I write a transistor or an IC?**

A: Transistors take three coordinates in pin order
(`Q1 2N2222 b5 b6 b7` — B C E for a BJT, G D S for a FET).
ICs can take either endpoints (`U1 555 f1 f8`) if the tool knows
the part, or an explicit pin map (`U1 555 f1=1 f2=2 ...`) if it
doesn't. See the Reference section for details.

**Q: Won't the label peel off / get torn?**

A: Sticker paper + a strip of clear tape over the top handles most wear.
The sliver design avoids covering the power rails.

**Q: Why Markdown? Why not plain text only?**

A: Pure notation is available as `.bbn`. Markdown is for builds you want
to annotate — notes, schematics, status. The block is the contract;
the prose is commentary.

**Q: What if I move a component?**

A: Edit one line. The netlist regenerates. Nothing else changes.

**Q: Do I need to verify? I just want to document.**

A: Verification is optional. The format is useful without it.

**Q: Can I use this without printing anything?**

A: Yes, if your board already has clear, standard markings. The overlay
exists to make the markings match the format — not the other way around.

**Q: Is this compatible with simulation?**

A: The converter outputs SPICE netlists, which most simulators read.
The format is simulator-agnostic.

**Q: What about stripboard / perfboard?**

A: Same principles, different board models. A stripboard model describes
the copper strips instead of the 5-hole groups. Planned.

**Q: What about SMD / PCB?**

A: Out of scope. The format is for through-hole breadboard work.

**Q: Is my data really mine?**

A: Yes. Plain text files, no proprietary format, public domain.
Same answer as always.

## Context

A breadboard is a grid pretending not to be one.
Every hole has a coordinate. Every column group is a net.
The information is all there — it's just not written down anywhere
in a form a computer can read or a person can share.

Meanwhile, the tools that could use this information —
simulators, verifiers, version control, diffing —
have no common input format for the physical build.
Screenshots are opaque. Proprietary files are locked in.
Netlists describe connectivity but lose position.

This project fills that gap with the smallest possible thing:
a printable label and a plain-text file.

The trade-off is explicit: you give up one-click GUI export
and drag-and-drop convenience in exchange for text, portability,
and verifiability. That trade-off is intentional.

## Prior Art

Pieces of this exist:

```
Fritzing            has an internal breadboard model, but the
                    format is proprietary XML and the coordinates
                    don't correspond to real board markings.
Wokwi / Tinkercad   simulate breadboards, but each reinvents the
                    model privately, with no portable format.
DIY Layout Creator  coordinate-based, but stripboard-specific and
VeeCAD / Lochmaster not breadboard-marking-aware.
SPICE netlists      the verification target, but not positional.
Patch cable charts  conceptually similar (from A3 to B7), used in
                    modular synths and telephone exchanges.
```

What's been missing is the connective tissue:
an open, human-writable notation that maps to a real physical board,
plus a printable overlay so the mapping is the same for everyone.

## Contributing

This project is in the public domain. Fork it, modify it, use it, share it.
If you make improvements, consider sharing them back.
Open an issue or pull request if you'd like.

## License

Public domain under the Unlicense. Use, modify, share freely.

## Acknowledgments

Inspired by the Unix philosophy: "Do one thing and do it well."
Built on the simplicity of the filesystem and the breadboard.
Made possible by free and open source software.

*A grid pretending not to be one — now written down.*
