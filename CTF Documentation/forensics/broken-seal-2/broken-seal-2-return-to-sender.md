# Broken Seal 2 — Solution

**Category:** Forensics  **Difficulty:** Hard  

## Challenge Summary
`postcard_v2` is a damaged PNG with the flag split into **four parts**, each hidden a different way:

| Part | Where it's hidden | Technique to recover it |
|---|---|---|
| 1/4 | `tEXt` metadata chunk, Base64-encoded | `exiftool` / `strings`, then Base64 decode |
| 2/4 | Top of the image | Make the image open |
| 3/4 | Rows hidden by a reduced IHDR height | Brute-force the real height against the **stored** CRC |
| 4/4 | A PNG appended after `IEND`, with its signature removed | Restore the 8-byte PNG signature and carve it out |

---

## Step 1 — Identify the file 
```bash
file postcard_v2
```
```
postcard_v2: PNG image data, 900 x 160, 8-bit/color RGB, non-interlaced
```
It's a PNG, but it won't open in an image viewer.

## Step 2 — Check the metadata (Flag Part 1)
```bash
exiftool postcard_v2
# or
strings postcard_v2 | head
```
A `Comment` field holds a Base64 string:
```
Comment : UGFydCAxLzQ6IGZsYWd7aW5tX3Awc3RjNHJkXw==
```
```bash
echo 'UGFydCAxLzQ6IGZsYWd7aW5tX3Awc3RjNHJkXw==' | base64 -d
```
```
Part 1/4: flag{inm_p0stc4rd_
```
PNG `tEXt` chunks store plain text without compression, which is why `strings` finds this one as well.

## Step 3 — Find the structural error
```bash
pngcheck -v postcard_v2
```
`pngcheck` reports a **CRC error in chunk IHDR**:

- **computed:** `dbf935d1`, the CRC worked out from the IHDR as it is now
- **expected:** `e61f7c73`, the CRC stored in the file

It may also warn about **extra data after the IEND chunk**. You'll come back to that in Step 7. If the `pngcheck` didn't warn about that you can ignore for now.

## Step 4 — The obvious fix (Flag Part 2)
In a hex editor, go to offset `0x1D` and write the computed CRC, `DB F9 35 D1`. *This will overwrite the exisitng CRC value of `E6 1F 7C 73` from bytes 29 to 32.* Save the file as `try1.png` or any filename of your choice and open it. You will have the second part of the flag.

> NOTE: Hex editor tools that can be used include `hexed.it` online, `HxD`, `ImHex`, `GHex` etc

> Greetings from Nairobi!  
> Part 2/4: `l3v3l_2_`

The image is very short (160 px), and parts 1 and 2 don't make a complete flag. **Something is still wrong.**

## Step 5 — Think about why the CRC was wrong
A CRC only fails to match if **the chunk's data was changed** after the CRC was calculated. So the stored CRC `e61f7c73` is probably the **original, correct** CRC, and one of the IHDR fields has been tampered with.

Examine the following offsets and dompare the IHDR fields below with the values expected by the PNG specification, and look for the one that doesn't fit. (*The specification and other useful tools are linked under **Resources** at the end of this document.*)

The IHDR layout:

| Offset | Bytes | Field | Value in file |
|---|---|---|---|
| `0x10` | 4 | Width | `00 00 03 84` = 900 |
| `0x14` | 4 | **Height** | `00 00 00 A0` = 160 ← suspicious. Width vs height not relative |
| `0x18` | 1 | Bit depth | 8 |
| `0x19` | 1 | Colour type | 2 (RGB) |
| `0x1A`–`0x1C` | 3 | Compression / filter / interlace | 0 / 0 / 0 |

Lowering the height is a classic way to hide rows. A decoder draws only as many rows as the *height* field says and ignores any extra pixel data. Here the image is 900 pixels wide but only 160 pixels tall, which looks oddly short. The challenge description hints at this too: *"...the card looks **shorter** than we remember..."*. The keyword is **shorter**: the height was deliberately reduced to hide something below the visible area.

## Step 6 — Recover the hidden rows (Flag Part 3)

The image is 900 pixels wide but only 160 pixels tall. The pixel data for the hidden rows is, if available, still in the file. You just need to make the height big enough for a viewer to draw them. A good first attempt is a **square image**: set the height equal to the width, 900.

### 6.1 Set the height to 900
900 in hex is `0x384`. Height is a 4-byte big-endian number, so it's written as `00 00 03 84`.

You can start from the original `postcard_v2` or from `try1.png`. Both work.

1. Open the file in any hex editor of your choice.
2. Go to offset `0x14` (the height field). You should see `00 00 00 A0` (160).
3. Overwrite those 4 bytes with the height of 900 which is `00 00 03 84`.
4. Check the second row (offset `0x10`) now starts with:
   ```
   00 00 03 84 00 00 03 84
   ```
   The first 4 bytes are the width (900) and the next 4 are the new height (900).
5. Save it as a new file, `try2.png`. Don't overwrite your original.

### 6.2 Get the new CRC from pngcheck
You've changed the IHDR data, so the stored CRC no longer matches:
```bash
pngcheck -v try2.png
```
pngcheck reports a **CRC error in chunk IHDR**, and the **computed** value is `762c686c`. You get the same computed value whether you started from the original or from `try1.png`, because it depends only on the IHDR contents. Only the stored value differs (`e61f7c73` or `dbf935d1`).

> *Some image viewers ignore CRC errors and can view the image as is from the export to `try2.png from 6.1 step. This would meann no need to do this 6.2 step*

### 6.3 Write the new CRC
1. In the hex editor, go to offset `0x1D`.
2. Overwrite the 4 bytes there with `76 2C 68 6C`.
3. Check the second row now reads:
   ```
   00 00 03 84 00 00 03 84 08 02 00 00 00 76 2C 68
   ```
   with `6C` as the first byte of the next row (offset `0x20`).
4. Save `try2.png` and run `pngcheck try2.png` again. The IHDR CRC error should be gone. A warning about extra data after IEND is expected (but could be missing too), and you'll deal with it in Step 7. Such error would be *"..additional data after IEND chunk.."*

### 6.4 Open the image
Open `try2.png`. The postcard is now taller and shows:

> Part 3/4: `p1x3ls_h1d3_b3l0w_`  
> Not every story stops at the END.

The real image is shorter than 900 pixels, so the bottom of the canvas is empty (black, grey or transparent, depending on the viewer). That's expected. There's no more pixel data to draw there.

**Part 3 found.** You now have three of the four parts.

---

### Optional: Alternative Method to find the exact original height
This isn't required, because you already have Part 3. The **stored** CRC from the original file (`e61f7c73`) was calculated with the real height. A script can try every height until the CRC matches it.

You can write the script yourself or ask an AI assistant. For example:

> *"Write a Python script that reads a PNG file, takes the stored IHDR CRC at bytes 29–32 (expected value e61f7c73), and tries every height from 1 to 9999 in the IHDR height field (bytes 20–23, big-endian) until the CRC-32 of bytes 12–28 matches. Print the matching height and save a copy of the file with that height."*

A sample script, `find_height.py`:
```python
import sys, zlib, struct

f = sys.argv[1]
d = bytearray(open(f, "rb").read())
stored = struct.unpack(">I", d[29:33])[0]
print(f"Stored IHDR CRC: {stored:08x}")

for h in range(1, 10000):
    ihdr = bytearray(d[12:29])                # "IHDR" + 13 data bytes
    ihdr[8:12] = struct.pack(">I", h)         # height = bytes 8-11 of this slice
    if zlib.crc32(ihdr) & 0xffffffff == stored:
        print(f"Real height: {h} (0x{h:08X})")
        d[20:24] = struct.pack(">I", h)       # patch height, keep stored CRC
        open("try2_exact.png", "wb").write(d)
        print("Written: try2_exact.png")
        break
else:
    print("No matching height found.")
```
Run it on the **original** file, because `try1.png` no longer holds the original CRC:
```bash
python3 find_height.py postcard_v2
```
It finds **420** (`00 00 01 A4`). With that height, the original CRC `E6 1F 7C 73` is correct, so in a hex editor you only need to change `0x14` to `00 00 01 A4` and leave the CRC alone.

> > **Note:** Some image viewers (such as Windows Photos) ignore CRC errors. They may show the image with an edited height even when the CRC hasn't been fixed. The same applies to the original challenge file, which may open straight away in your viewer. This is expected, and we know about it. A forgiving viewer only shortcuts the CRC fix. You still need the steps above to understand and recover each part of the flag.

## Step 7 — Look after IEND (Part 4)

### 7.1 Why look after IEND?
Run `pngcheck` on any file from the earlier steps that has no CRC errors, for example `try2.png`:
```bash
pngcheck try2.png
```
pngcheck reports **additional data after the IEND chunk**. pngcheck stops at the first error, so this message only appears once the CRC errors are fixed.

`IEND` is the chunk that marks the end of a PNG image. Viewers stop reading there and ignore anything after it. This message means the file holds **more than just the image**.

The challenge description hinted at this too: *"...the envelope feels suspiciously **heavy**."* The postcard (the image) is fine, but the envelope (the file) contains something extra.

### 7.2 Find the IEND chunks and the hidden image
List every `IEND` and `IHDR` in the file, with their byte positions:
```bash
grep -obUa "IEND" postcard_v2
grep -obUa "IHDR" postcard_v2
```
Each search finds **two** matches. A PNG has exactly one `IHDR` (at the start) and one `IEND` (at the end), so two of each means **two images** in one file.

Take note of the **first** `IEND` position. It's offset `0x4A09` (decimal 18953).

You can also see this in a hex editor by searching for the text `IEND` and `IHDR`. Each appears twice.

### 7.3 Why automatic tools don't find it
A PNG ends with a 12-byte `IEND` chunk:
```
00 00 00 00   49 45 4E 44   AE 42 60 82
 length = 0     "IEND"         CRC
```
The text `IEND` is at `0x4A09`, and its CRC ends at `0x4A10`. The second image starts right after, at offset `0x4A11`. Look at the bytes there:
```bash
xxd -s 0x4A11 -l 16 postcard_v2
```
They start with `00 00 00 0D 49 48 44 52`, a length of 13 followed by `IHDR`. The text `IHDR` itself is at `0x4A15`, which is where a search for `IHDR` lands.

Every PNG starts with an 8-byte **signature**: `89 50 4E 47 0D 0A 1A 0A`. The second image's signature has been **removed**, so it begins straight at its IHDR chunk. Tools such as `file`, `binwalk` and `foremost` look for that signature, so they miss this image

> `binwalk` shows this too:

    
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             PNG image, 900 x 160, 8-bit/color RGB, non-interlaced
    101           0x65            Zlib compressed data, default compression
    18994         0x4A32          Zlib compressed data, default compression
    

It finds only **one** PNG image. It also lists two blocks of **zlib** data, because PNG pixel data in `IDAT` chunks is zlib-compressed:

- `0x65` is the first image's pixel data. This is normal for any PNG.
- `0x4A32` comes **after the first IEND** (`0x4A09`), where the file should already have ended. This is the hidden image's pixel data.

Extracting that zlib block alone won't give you a picture. It's raw pixel data without the width, height and colour information stored in IHDR. You need to rebuild the complete PNG, as shown below.

To recover it: **cut out everything after the first IEND chunk, and put the signature back** at the start.

### 7.4 Option A — Terminal
```bash
# 1. The second image starts 8 bytes after the first "IEND" (4 letters + 4-byte CRC)
OFF=$(( $(grep -obUa "IEND" postcard_v2 | head -1 | cut -d: -f1) + 8 ))
echo "$OFF (hex: $(printf '%X' $OFF))"

# 2. Confirm the second image's IHDR starts there
xxd -s $OFF -l 16 postcard_v2

# 3. Rebuild it: signature + everything from OFF to the end of the file
printf '\211PNG\r\n\032\n' > part4.png
tail -c +$((OFF + 1)) postcard_v2 >> part4.png

# 4. Check
xxd part4.png | head -1     # should start: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452
file part4.png              # PNG image data, 500 x 100
pngcheck part4.png          # OK
```
- In this build, step 1 prints `18961 (hex: 4A11)`.
- `printf` uses octal escapes (`\211` = `0x89`, `\032` = `0x1A`) because they work in every shell.
- `tail -c +N` counts from 1, while offsets count from 0, so the command adds `+1`.

### 7.5 Option B — Any hex editor (no inserting needed)
1. Open a **fresh copy** of `postcard_v2`.
2. Search for the text `IEND` and stop at the **first** match (offset `0x4A09`). It's followed by `AE 42 60 82`.
3. Select everything from offset `0x0` up to `0x4A08`, the byte just **before** the `I` of `IEND`, and **delete** it.
4. The file now starts with:
   ```
   49 45 4E 44 AE 42 60 82 00 00 00 0D 49 48 44 52
   ```
   `IEND` + its CRC is exactly **8 bytes**, the same length as a PNG signature.
5. **Overwrite** those first 8 bytes with `89 50 4E 47 0D 0A 1A 0A`.
6. The first row now reads:
   ```
   89 50 4E 47 0D 0A 1A 0A 00 00 00 0D 49 48 44 52
   ```
7. Save the file as `part4.png`.

### 7.6 Open the image
Open `part4.png`:

> Part 4/4: `th3_3nd}`

**Part 4 found.** You now have all four parts of the flag.

## Flag
Using the four parts found as listed below, construct the flag by concatenating the 4 parts in order to form one flag.

Individual flag parts obtained

    Part 1/4: flag{inm_p0stc4rd_
    part 2/4 l3v3l_h4rd_
    Part 3/4 p1x3ls_h1d3n_b3l0w_
    Part 4/4 th3_3nd}


Concatenated flag value
```
flag{inm_p0stc4rd_l3v3l_h4rd_p1x3ls_h1d3n_b3l0w_th3_3nd}
```

## Key Takeaways
- Check metadata first. `tEXt` chunks are plain text and easy to miss.
- A CRC mismatch tells you **something changed**. Don't just overwrite the stored CRC. Ask what was changed and use the stored CRC to recover the original value.
- The IHDR height controls how many rows are drawn, so hidden rows can be recovered by brute-forcing the height against the CRC.
- Automatic file carvers rely on magic bytes. If a file's signature has been removed, you can still recognise it from its structure (`IHDR` … `IEND`) and rebuild it by hand.

## Resources
- PNG specification: http://www.libpng.org/pub/png/spec/1.2/PNG-Contents.html
- PNG chunk specification (IHDR, tEXt, CRC): https://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html
- pngcheck: https://man.cx/pngcheck(1)
- ExifTool: https://exiftool.org/
- hexed.it: https://hexed.it/
