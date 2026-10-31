# Layered CTF Writeup: The Washroom Sign Challenge

**Category:** OSINT / Forensics / Steganography / Crypto **Given file:** `Image.jpg` **Flag format:** `flag{...}`

This challenge is a chain of layers. Each layer gives you what you need for the next one. This writeup explains every step, the tool used, and how that tool works, so anyone on the team can follow it.

## Overview of the layers

|#|Layer|Technique|Tool|
|-|-|-|-|
|1|Metadata|Read the EXIF hint and spot the decoy GPS|exiftool|
|2|OSINT|Read the blurred word on the real sign|your eyes (and a short walk)|
|3|File carving|A ZIP is hidden after the end of the JPEG|grep, dd, binwalk|
|4|Encrypted ZIP|Password is the base64 of the sign word|7z, base64|
|5|LSB steganography|Data hidden in the blue channel of a PNG|Python + Pillow (or zsteg)|
|6|Symmetric crypto|AES-256-CBC with PBKDF2|openssl|
|7|Weak RSA|Primes are too close together: Fermat factoring|Python (or RsaCtfTool)|
|8|XOR + encoding|XOR with the recovered key, then Base32|Python (or CyberChef)|

## Tools to install (Kali / Debian / Ubuntu)

```bash
sudo apt update
sudo apt install -y libimage-exiftool-perl binwalk p7zip-full openssl python3 python3-pip
pip install pillow --break-system-packages
# optional extras
sudo gem install zsteg
```

\---

## Step 1: Look at the file and its metadata

Always start by asking what the file really is and what it says about itself.

```bash
file Image.jpg
exiftool Image.jpg
```

**How it works:** `file` identifies a file by its magic bytes (the first few bytes), not by its extension. `exiftool` reads the metadata blocks (EXIF, GPS, comments) that cameras and editors store inside an image.

**What you find:**

* `ImageDescription` contains a hint: take a break, walk to the washroom, and you might see something. One word on a sign is blurred in this photo, but you can read it in person. That word is the password: lowercase, *wrapped for the wire*.
* GPS tags: about 4.04 S, 39.67 E. That is **Mombasa**.

**Takeaways:**

1. The password is a single word from a sign, in lowercase.
2. *Wrapped for the wire* is a hint for **base64**, the encoding used to send binary or text safely over a wire/protocol.
3. The GPS location is a **decoy**. The CTF is being held in Nairobi, and we will need the real city later. Metadata can be edited, so never trust it blindly.

## Step 2: OSINT, read the blurred word

Open the image. On the right-hand column of the sign you will see:

* Lift lobby
* Finance
* *(blurred word)*
* HumanResource
* Washrooms

The third line is blurred, so you cannot read it from the photo. Follow the hint: go to the washroom area, find this sign, and read the word.

**The word is `Procurement`.**

In lowercase this is `procurement`.

> OSINT tip: the photo itself gives you clues about where to look: the sign style, the ceiling lights, the glass door under the sign, and the names of the departments on it.

## Step 3: Find the hidden ZIP appended to the JPEG

A JPEG file ends with the two bytes `FF D9` (the End Of Image marker). Image viewers stop reading there. Anything placed after that marker is ignored by viewers but is still in the file. This is called **appended data**.

Check the file size. It is far larger than a normal photo of this kind, which is suspicious.

### Option A: binwalk (easiest)

```bash
binwalk Image.jpg
binwalk -e Image.jpg
```

**How it works:** binwalk scans every offset of a file for known signatures (magic bytes) of formats like ZIP, PNG, gzip and so on, then prints where each one starts. `-e` extracts them automatically.

### Option B: manual carving

A ZIP begins with the bytes `50 4B 03 04` (`PK\\x03\\x04`). Find the first one:

```bash
grep -obUaP 'PK\\x03\\x04' Image.jpg | head -1
```

This prints `OFFSET:PK`. Using that offset (for example `752000`), cut from there to the end of the file:

```bash
dd if=Image.jpg of=carved.zip bs=1 skip=752000
# or, faster:
tail -c +$((752000+1)) Image.jpg > carved.zip
```

**How it works:** `grep -obUaP` treats the file as binary (`-U`, `-a`), uses a Perl regex (`-P`), and prints the byte offset (`-b`) of each match (`-o`). `dd` or `tail` then copies everything from that offset onward.

Verify:

```bash
file carved.zip
7z l carved.zip
```

You will see two files, `readme.txt` and `layer2.png`, and the listing will show they are encrypted.

## Step 4: Open the encrypted ZIP

The password is the sign word, lowercase, then base64 encoded (the *wrapped for the wire* hint).

```bash
echo -n 'procurement' | base64
# cHJvY3VyZW1lbnQ=
```

> \*\*Important:\*\* use `echo -n`. Without `-n`, echo adds a newline character and you get a different, wrong base64 string.

Now extract with 7z:

```bash
7z x carved.zip -p'cHJvY3VyZW1lbnQ='
```

**How it works:** `base64` converts every 3 bytes of input into 4 printable characters. It is an *encoding*, not encryption: anyone can reverse it. The ZIP uses **AES-256** encryption, which many older `unzip` versions cannot open, so use `7z`.

You now have:

* `readme.txt`
* `layer2.png`

## Step 5: Read the readme

```bash
cat readme.txt
```

```
layer 2 is sealed in the low bits.
what you pull out: base64, openssl aes-256-cbc, pbkdf2, 600000 iterations.
key = city + \_ + the word from the sign, lowercase, joined by an underscore.
the city is where the photo was really taken, not where the tags say.
```

This tells us three things:

1. The data is hidden in the **least significant bits (LSB)** of the PNG.
2. What comes out is base64 of an OpenSSL AES-256-CBC blob, using PBKDF2 with 600000 iterations.
3. The decryption key is `city\_word`. The city is the **real** location (Nairobi, where you are physically), not the GPS decoy (Mombasa). So the key is `nairobi\_procurement`.

## Step 6: Extract the LSB-hidden data from the PNG

**Background:** every pixel has red, green and blue values from 0 to 255. Changing the last bit of a value changes the colour by at most 1 out of 255, which is invisible to the eye. Hiding data this way is called **LSB steganography**.

Only the **blue** channel is used here. Bits are stored in row-major pixel order, most significant bit first, and the first 32 bits give the payload length in bytes.

### Quick look with zsteg (optional)

```bash
zsteg layer2.png
zsteg -E 'b1,b,lsb,xy' layer2.png > raw.bin
```

The notation `b1,b,lsb,xy` means: 1 bit, blue channel, LSB first-or-last order, scanning x then y. The output starts with a 4-byte length, so you would need to skip the first 4 bytes. Python is more reliable for this challenge.

### Python extractor

```python
from PIL import Image

px = list(Image.open('layer2.png').convert('RGB').getdata())
bits = ''.join(str(p\[2] \& 1) for p in px)      # p\[2] is the BLUE channel

length = int(bits\[:32], 2)                      # first 4 bytes = length
payload = bits\[32:32 + length \* 8]
data = bytes(int(payload\[i:i+8], 2) for i in range(0, len(payload), 8))

open('blob.txt', 'wb').write(data)
print(data\[:60])
```

The result is a long base64 string (it begins with `U2FsdGVkX1`, which is base64 for `Salted\_\_`, the header OpenSSL puts on encrypted data).

## Step 7: Decrypt the OpenSSL blob

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 -a -A \\
  -in blob.txt -pass pass:nairobi\_procurement -out vault.txt
cat vault.txt
```

**How it works:**

* `-aes-256-cbc`: AES with a 256-bit key in CBC mode.
* `-pbkdf2 -iter 600000`: your password is not used directly. PBKDF2 stretches it through 600000 rounds of hashing to derive the real key and IV, which makes brute-forcing slow.
* `-a -A`: the input is base64 on a single line.
* If the password is wrong you get `bad decrypt`. If you tried `mombasa\_procurement`, this is where it fails.

Output:

```
n = 30138891662448154711455208619982092091757869369786686442...
e = 65537
c = ...
ct = ...
```

This is an RSA setup (`n`, `e`, `c`) plus a hex ciphertext `ct`.

## Step 8: Break the weak RSA with Fermat factoring

RSA is safe only if nobody can factor `n = p \* q`. Here the generator picked `q` very close to `p` (they differ by roughly 2^76, while each is about 2^128). When two primes are this close, **Fermat's factorisation method** finds them almost instantly.

**The idea:** write `n = a^2 - b^2 = (a - b)(a + b)`. Start at `a = ceil(sqrt(n))` and increase `a` until `a^2 - n` is a perfect square `b^2`. Then `p = a - b` and `q = a + b`. When `p` and `q` are close, `a` is already very near `sqrt(n)`, so the loop ends fast.

```python
import math, re

v = open('vault.txt').read()
g = dict(re.findall(r'(\\w+) = (\\w+)', v))
n, e, c = int(g\['n']), int(g\['e']), int(g\['c'])

a = math.isqrt(n) + 1
while True:
    b2 = a \* a - n
    b = math.isqrt(b2)
    if b \* b == b2:
        break
    a += 1

p, q = a - b, a + b
assert p \* q == n

d = pow(e, -1, (p - 1) \* (q - 1))              # RSA private exponent
m = pow(c, d, n)                                # decrypt c
key = m.to\_bytes((m.bit\_length() + 7) // 8, 'big')
print(key)                                      # b'Pkl4nds!'
```

**The XOR key is `Pkl4nds!`.**

Alternatives: paste `n` into [factordb.com](https://factordb.com), or run `RsaCtfTool --publickey ... --attack fermat`.

## Step 9: XOR and Base32 decode to get the flag

The hex string `ct` was XORed with the repeating key, and the result is Base32 text.

```python
import base64

ct = bytes.fromhex(g\['ct'])
xored = bytes(x ^ key\[i % len(key)] for i, x in enumerate(ct))
print(xored)                       # looks like base32 text (A-Z, 2-7, =)
print(base64.b32decode(xored))     # the flag
```

**How it works:** XOR flips bits where the key bit is 1. Doing it twice with the same key gives back the original, so XORing the ciphertext with the key undoes the encryption. **Base32** uses only `A-Z` and `2-7`, so seeing that alphabet in the output tells you which decoder to use.

### CyberChef alternative

Recipe: **From Hex** → **XOR** (key: `Pkl4nds!`, type UTF8) → **From Base32**.

## Flag

```
flag{inm\_l4y3r5ed}
```

\---

\---

## Common mistakes

|Problem|Cause and fix|
|-|-|
|ZIP password rejected|You used `echo` without `-n`, left capitals in the word, or forgot to base64 it. The password is the base64 of `procurement`.|
|`unzip` fails with an unsupported method error|The ZIP uses AES-256. Use `7z x`.|
|`bad decrypt` from openssl|Wrong key. Use the real city (`nairobi`), not the GPS decoy (`mombasa`), and keep the underscore: `nairobi\_procurement`.|
|LSB output is garbage|You used the wrong channel or bit order. It is the **blue** channel, row-major, MSB first, with a 4-byte length first.|
|Cannot find the ZIP|Look for `PK\\x03\\x04` **after** the JPEG end marker `FF D9`. Use `binwalk` if unsure.|
|Do not know the word|The word is blurred on purpose. Go to the real sign near the washroom and read it.|

## Skills practised

* Metadata analysis and spotting decoys
* OSINT and physical observation
* File carving and appended-data detection
* Password-protected archives and encoding (base64)
* LSB image steganography
* AES with PBKDF2 key derivation
* Weak RSA key generation (Fermat factoring)
* XOR and Base32 decoding

