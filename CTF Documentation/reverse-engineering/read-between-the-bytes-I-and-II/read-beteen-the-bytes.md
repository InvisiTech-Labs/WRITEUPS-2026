# Solution Doc — "Read Between the Bytes"

## Overview

This challenge is split into two linked parts: static analysis of an Android application to recover a hidden Firestore endpoint and its flag (**Challenge I**), then using what's uncovered there to chain into a second, deeper Firestore-based puzzle that ends in cracking a password-protected zip (**Challenge II**).

---

## Challenge I — Read Between the Bytes

### Objective

Perform static analysis on the Android application to locate and recover the first flag.

### Tools used

- [MobSF](https://mobsf.live/)
- Android SDK tools — APK inspection
- CyberChef or a Base64 decoder — string decoding

### Steps

1. **Obtain the APK.** Download the provided APK and prepare it for static analysis.
2. **Upload to MobSF.** Submit the APK for static analysis and let it complete.
3. **Inspect extracted URLs.** In the MobSF report, review the list of URLs/strings pulled from the app. Among them is a Firestore REST endpoint:

   ```
   https://firestore.googleapis.com/v1/projects/ctfd-45f4f/databases/(default)/documents/ctfd/leve12
   ```

4. **Query the endpoint.**

   ```bash
   curl "https://firestore.googleapis.com/v1/projects/ctfd-45f4f/databases/(default)/documents/ctfd/leve12"
   ```

   Response:

   ```json
   {
     "name": "projects/ctfd-45f4f/databases/(default)/documents/ctfd/leve12",
     "fields": {
       "flag": {
         "stringValue": "16Vpi4PR4ndalltqyNUoJy4Ryx4JGf9XG"
       }
     },
     "createTime": "2026-10-23T01:07:34.187723Z",
     "updateTime": "2026-10-23T01:48:06.417017Z"
   }
   ```

5. **Recover the flag.** `fields.flag.stringValue` is the Challenge I flag.

### Challenge I Flag

```
Decode 16Vpi4PR4ndalltqyNUoJy4Ryx4JGf9XG from Base62 to get flag{inm_congrats_nerd}


```

---

## Challenge II — Deeper in the Bytes

### Objective

Use what was found in Challenge I to continue probing the app's Firestore backend, recover a leaked Python "decode" script, locate the archive it references, and crack it to obtain the final flag.

### Tools used

- MobSF (for the initial URL list, reused from Challenge I)
- `curl` — querying Firestore REST endpoints
- `fcrackzip` — zip password cracking
- `rockyou.txt` wordlist

### Steps

1. **Enumerate further Firestore documents.** Going back through the fuller list of URLs surfaced by MobSF, several additional Firestore document paths return `UNAUTHORIZED`. Two, however, are accessible without auth and matter here:
   - one returns a **code base** describing a decode/extraction flow
   - one returns the **base64 of a zip file**, referenced by name inside that code base

2. **Fetch the code base.**

   ```bash
   curl "https://firestore.googleapis.com/v1/projects/ctfd-45f4f/databases/(default)/documents/ctfd/level8"
   ```

   The `DECODE` field contains a Python script:

   ```python
   #!/usr/bin/env python3
   """
   This script shows the retrieval + extraction flow. The zip password
   has been intentionally left blank below -- that's the part you need
   to figure out.
   """

   import base64
   import io
   import zipfile
   import urllib.request
   import json


   FIRESTORE_URL = "https://firestore.googleapis.com/v1/projects/ctfd-45f4f/databases/(default)/documents/ctfd/BASE64ZIP"

   def fetch_base64_zip(url: str) -> str:
       with urllib.request.urlopen(url) as resp:
           doc = json.loads(resp.read().decode())
       # Firestore REST responses wrap field values like {"fields": {"data": {"stringValue": "..."}}}
       return doc["fields"]["data"]["stringValue"]


   def decode_zip(b64_string: str) -> bytes:
       return base64.b64decode(b64_string)


   ZIP_PASSWORD = ""  # <-- figure this out

   def extract_and_display(zip_bytes: bytes, password: str):
       with zipfile.ZipFile(io.BytesIO(zip_bytes)) as zf:
           for name in zf.namelist():
               with zf.open(name, pwd=password.encode()) as f:
                   content = f.read().decode(errors="replace")
                   print(f"--- {name} ---")
                   print(content)


   def main():
       print("[*] Fetching base64 payload from Firestore...")
       b64_data = fetch_base64_zip(FIRESTORE_URL)

       print("[*] Decoding base64 -> zip bytes...")
       zip_bytes = decode_zip(b64_data)

       if not ZIP_PASSWORD:
           print("[!] ZIP_PASSWORD is empty. Set it before extraction will work.")
           return

       print("[*] Extracting archive contents...")
       extract_and_display(zip_bytes, ZIP_PASSWORD)


   if __name__ == "__main__":
       main()
   ```

   This tells the solver exactly what's needed: fetch a base64-encoded zip from a Firestore doc (placeholder name `BASE64ZIP` in the script), decode it, and crack its password.

3. **Fetch the base64 zip payload.**

   ```bash
   curl "https://firestore.googleapis.com/v1/projects/ctfd-45f4f/databases/(default)/documents/ctfd/BASE64ZIP"
   ```

   Response (truncated):

   ```json
   {
     "name": "projects/ctfd-45f4f/databases/(default)/documents/ctfd/BASE64ZIP",
     "fields": {
       "base64zipstring": {
         "stringValue": "UEsDBBQACQATAAOWpWF2TtK1wwAAAAAwBAAAKABwAc2hhZG93LnR4dFVUCQADrvXcalH13Gp1eAsAAQToAwAABOgDAADA9W6FLfqOTdGwe6E3Hmp6pey0mAqGUR+U..."
       }
     },
     "createTime": "2026-10-26T14:54:08.134974Z",
     "updateTime": "2026-10-26T14:54:08.134974Z"
   }
   ```

4. **Decode the base64 string into a zip file.**

   ```bash
   echo "<base64zipstring value>" | base64 -d > shadow.zip
   ```

5. **Crack the zip password with fcrackzip.**

   ```bash
   fcrackzip -u -D -p rockyou.txt shadow.zip
   ```

   Output:

   ```
   PASSWORD FOUND!!!!: pw == superman
   ```

6. **Extract and read the archive.**

   ```bash
   unzip -P superman shadow.zip
   cat shadow.txt
   ```

   `shadow.txt` contains a list of decoy `user:password` entries. The final entry holds the flag in place of a password:

   ```
   testuser01:T3st!Alpha_91
   testuser02:Br4vo#Lake_27
   testuser03:Cyb3r@Stone_84
   testuser04:Mango!Cloud_62
   testuser05:N0va#River_53
   testuser06:Delta@Fox_718
   testuser07:K3nya!Lab_426
   testuser08:Quartz#Moon_39
   testuser09:P1xel@Storm_85
   testuser10:flag{inm_Shell_Yeah_Buddy}
   ```

### Challenge II Flag

```
flag{inm_Shell_Yeah_Buddy}
```

---

## Summary
 
Challenge I MobSF static analysis → Firestore URL enumeration: flag{inm_congrats_nerd}

Challenge II  Firestore doc chaining (`level8` → `BASE64ZIP`) → base64 decode → zip password crack  `flag{inm_Shell_Yeah_Buddy}` 