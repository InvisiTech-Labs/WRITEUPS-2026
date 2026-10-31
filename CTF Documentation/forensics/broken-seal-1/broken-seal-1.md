# Broken Seal — Solution

**Category:** Forensics  **Difficulty:** Easy  
**Flag:** `flag{inm_png_crc_corruption_solve_using_pngcheck}`

## Challenge Summary
You get a file called `postcard` with no file extension, and it won't open. It's a PNG whose **IHDR chunk CRC** has been corrupted. The image data is intact, so fixing those 4 bytes makes the image open and shows the flag.

## Step 1 — Identify the file type
```bash
file postcard
```
```
postcard: PNG image data, 900 x 120, 8-bit/color RGB, non-interlaced
```
`file` checks the first bytes of the file (the "magic bytes"), not the extension. Every PNG starts with `89 50 4E 47 0D 0A 1A 0A`.

## Step 2 — Confirm it won't open, and try the obvious checks
```bash
cp postcard postcard.png      # try opening it in an image viewer: it fails
strings postcard | grep -i flag   # nothing
exiftool postcard                 # nothing useful
```
The flag is drawn as pixels, and PNG pixel data is compressed, so `strings` won't find it.

## Step 3 — Find what's broken with pngcheck
```bash
sudo apt install pngcheck
pngcheck -v postcard
```
```
File: postcard
  chunk IHDR at offset 0x0000c, length 13
  CRC error in chunk IHDR (computed 2594ab93, expected deadbeef)
ERRORS DETECTED in postcard
```
**What this means:** a PNG is built from chunks. Each chunk is laid out as:

| Length (4 bytes) | Type (4 bytes) | Data | CRC (4 bytes) |
|---|---|---|---|

The CRC is a checksum calculated over the chunk's type and data. Image viewers recalculate it, and if it doesn't match the stored value they reject the file. Here the file stores `deadbeef`, but the correct value is `2594ab93`.

## Step 4 — Locate the CRC bytes
IHDR is always the first chunk and always holds 13 bytes of data:

| Bytes | Content |
|---|---|
| 0–7 | PNG signature |
| 8–11 | Length (`00 00 00 0D`) |
| 12–15 | `IHDR` |
| 16–28 | Width, height, bit depth, colour type, compression, filter, interlace |
| **29–32 (0x1D–0x20)** | **CRC**, the bytes to fix |

```bash
xxd postcard | head -3
```
```
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
00000010: 0000 0384 0000 0078 0802 0000 00de adbe  .......x........
00000020: ef00 ...
```
`de ad be ef` starts at offset `0x1D`.

## Step 5 — Fix the CRC
**Option A: Hex editor (HxD, Bless, hexed.it)**
1. Open `postcard` and go to offset `0x1D`.
2. Overwrite `DE AD BE EF` with `25 94 AB 93`.
3. Save the file as `fixed.png`.


**Option B: Python (works out the CRC itself)**
```python
import zlib, struct
d = bytearray(open("postcard", "rb").read())
d[29:33] = struct.pack(">I", zlib.crc32(d[12:29]) & 0xffffffff)
open("fixed.png", "wb").write(d)
```

## Step 6 — Open the image
Open `fixed.png` to see the flag:

```
flag{inm_png_crc_corruption_solve_using_pngcheck}
```

## Key Takeaways
- File extensions can be misleading. Use `file` and the magic bytes to identify a file.
- PNG chunks each carry a CRC, and one wrong CRC is enough for a viewer to reject a file whose image data is fine.
- `pngcheck` is the go-to tool for diagnosing damaged PNGs.

## Resources
- PNG specification: http://www.libpng.org/pub/png/spec/1.2/PNG-Contents.html
- PNG chunk specification (IHDR and CRC): https://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html
- pngcheck: https://man.cx/pngcheck(1)
- HxD: https://mh-nexus.de/en/hxd/
- hexed.it (browser hex editor): https://hexed.it/