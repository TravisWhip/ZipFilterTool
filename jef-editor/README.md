# JEF embroidery tools

Browser-based tools for Janome `.JEF` embroidery files. No install needed: open the HTML file in Chrome, Edge or Firefox.

- **jef-editor.html** — stitch editor. Draw rectangles, ovals, lines and triangles (satin, running or triple-running stitch),
  move/resize/rotate them with draw.io-style handles and snap guides, edit individual stitch points, preview with realistic
  thread width, and export to `.JEF`. **Align** (A) lines up two or more selected elements TinkerCad-style: one stays put
  (the biggest, or click another; or right-click one → **Align to this**), and clicking a left/center/right or bottom/middle/top dot moves the others.
  **Text** (X) writes with Embrilliance-style `.bx` embroidery fonts: load them with Open…, drop them on the canvas, or click
  one in the Files panel (they're remembered in the browser), click where the text goes and type; letter size (the font's own size is shown), letter and
  line spacing, line alignment and font stay editable (hold Shift and point at a font, in the list or the Files panel, to try it on the text), and the text keeps its stitches even where the font isn't loaded. The **Files** panel lists any number of folders from anywhere on your PC (**Add folder…**, or right-click its empty space; right-click a folder to remove it from the list). **Save** writes a `.jefproj` project (JSON) that keeps live shapes, groups,
  measurements, sew order, hoop and photo alignment; reopen it with **Open…** and keep editing, then **Export JEF**.
  Upload a photo of your hooped fabric and line it up: mark the hoop's inside edge and press Apply to straighten and size
  it automatically, or move, rotate, resize and stretch it by hand onto the ghost hoop, to design directly on it.
- **date.JEF** — sample design (single color, 77.4 × 8.0 mm).

## JEF format notes

Little-endian binary. Header: stitch-data offset (0x00), date string (0x08), thread count (0x18), stitch count (0x1C),
hoop code (0x20), design extents in 0.1 mm (0x24), thread palette indexes (0x74…). Stitch data: 2-byte signed dx/dy pairs
in 0.1 mm (+Y up); `0x80` starts a 4-byte command: `80 02 dx dy` jump/trim, `80 01 dx dy` color change, `80 10` end.
Some files append an `ENHANCE` block with real thread RGB values and names.
