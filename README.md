# JEF embroidery tools

Browser-based tools for Janome `.JEF` embroidery files. No install needed: open the HTML file in Chrome, Edge or Firefox.

- **jef-editor.html** — stitch editor. Draw rectangles, ovals, lines and triangles (satin, running or triple-running stitch),
  move/resize/rotate them with draw.io-style handles and snap guides, edit individual stitch points, preview with realistic
  thread width, and save back to `.JEF`.
- **jef-viewer.html** — simple read-only viewer that renders a `.JEF` file and can save a PNG.
- **date.JEF** — sample design (single color, 77.4 × 8.0 mm).

## JEF format notes

Little-endian binary. Header: stitch-data offset (0x00), date string (0x08), thread count (0x18), stitch count (0x1C),
hoop code (0x20), design extents in 0.1 mm (0x24), thread palette indexes (0x74…). Stitch data: 2-byte signed dx/dy pairs
in 0.1 mm (+Y up); `0x80` starts a 4-byte command: `80 02 dx dy` jump/trim, `80 01 dx dy` color change, `80 10` end.
Some files append an `ENHANCE` block with real thread RGB values and names.
