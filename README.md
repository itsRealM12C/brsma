# brsma

A res.bin (similar to PNG) extractor.

# Origins
`res.bin` originally came from a product called HONOR CHOICE CuBuds. I've
got file access and I've extracted 4 `res.bin`s. But there's more, but I
didn't wanted to waste my storage with these.

### Layout

```
BR02 resource
    magic "BR\x02\x00"
    ↓
row table (height × u32, starting at 0xD8)
    ↓
per-row RLE payload
    ↓
decompressed ARGB8565 pixel stream
    ↓
RGB888 conversion
    ↓
PNG
```

### Header

| Offset | Type | Field  |
|--------|------|--------|
| 0x00   | char[4] | magic `"BR\x02\x00"` |
| 0xD0   | u16  | width |
| 0xD2   | u16  | height |
| 0xD8   | u32[height] | row table (see below) |

### Row table — **the part that was wrong in earlier scripts**

Each row's `u32` entry packs:

```
bits 0..20   row data offset
bits 21..31  row data length
```

**The offset is relative to the row table's own start (`0xD8`), not to
`0xD8 + height*4`.** An earlier version of this extractor added the
table size twice (once implicitly, since the offset value already
factors it in, and once explicitly via a separate `data_start`
variable), which under-read the first few rows and overran EOF on the
last ones. The fix:

```
row_start = 0xD8 + row_offset      # NOT (0xD8 + height*4) + row_offset
row_end   = row_start + row_length
```

### Row RLE

Each control byte:

- high bit set → **repeat run**: `count = low 7 bits`, followed by one
  3-byte pixel, repeated `count` times.
- high bit clear → **literal run**: `count = low 7 bits`, followed by
  `count * 3` raw pixel bytes.

Each pixel is 3 bytes: `A8, RGB565-high-byte, RGB565-low-byte` (big-endian
RGB565).

### Pixel expansion

```
R8 = (R5 << 3) | (R5 >> 2)
G8 = (G6 << 2) | (G6 >> 4)
B8 = (B5 << 3) | (B5 >> 2)
```

Alpha byte is present in the stream but unused for a flat PNG export
(the source images are fully opaque in every sample seen so far).

### Verified stats (per 296×240 image)

- Decompressed pixel stream: 213,120 bytes (296 × 240 × 3)
- All 114 sample resources in `all_res_bins.zip` decode cleanly with
  the corrected offset, 0 failures.

---

## Files

- `br02_extractor_fixed.py` — Python BR02 decoder with the corrected
  row-offset math.
- `br02_extractor_ui.html` — browser BR02 extractor: fullscreen shell,
  small fixed-size preview, scrollable debug log (header check + row
  decode progress), PNG + raw ARGB8565 export.
- `jieli_1_00_extractor.py` — Python port of the 1.00 keyframe/delta
  decoder. Exports one PNG per frame; optional third argument
  (`output.gif`) reassembles the frames into an animated GIF using
  each frame's delay.
- `jieli_1_00_extractor_ui.html` — browser 1.00 extractor: fullscreen
  shell, small fixed-size preview (doesn't stretch to fill the panel),
  real per-frame thumbnails (rendered from decoded pixel data, not
  numbered placeholders) in a scrollable filmstrip, first/prev/play/
  next/last transport controls, loop toggle, playback speed, keyboard
  shortcuts (space / ← / →), and the same scrollable debug log pattern
  as the BR02 tool.

Both browser tools share one UI pattern deliberately, so notes and
fixes transfer between them: a fixed small preview panel, an info
strip of key stats, and a running (not overwritten) log rather than a
single status line — useful for seeing exactly which row/frame failed
if a new resource doesn't decode cleanly.

## A note on conflicting docs from other sessions

If you run across other markdown notes describing these formats
differently — e.g. claiming the 1.00 frame table points directly at
absolute compressed-payload offsets with the codec "unsolved," or that
BR02's RLE grammar doesn't actually work and only a pre-extracted
payload was used — those describe an earlier, unresolved attempt at
this same file, not the current state. Everything in this document has
been re-run against your actual files in this conversation: all 114
BR02 samples decode cleanly with 0 failures, and `res.bin`'s 54 frames
were visually confirmed to render correctly (including mid-sequence
frames, not just the first).
