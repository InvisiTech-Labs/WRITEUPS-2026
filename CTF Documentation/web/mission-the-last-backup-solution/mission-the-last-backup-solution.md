# Mission: The Last Backup

## Post-Event Solution

The challenge begins on a polished demonstration portal. The first flag is not displayed on any visible page and does not require a login or an exploit. The intended approach is to inspect the resources that the browser loads, then follow the retained deployment evidence from one file to the next.

The solution path is:

```text
Landing-page source
        |
        v
/assets/js/app.js
        |
        v
/assets/js/app.js.map
        |
        v
/assets/manifest/asset-inventory.json
        |
        v
/backup/config.php.bak
        |
        +--> Flag 1
        +--> /pma/
        +--> Evidence needed for Challenge 2
```

## What You See Initially

Open the challenge URL in a browser. You are presented with a normal demonstration portal containing navigation links, informational pages, images, styling, and a few client-side scripts.

There is no obvious flag, login form, or vulnerable input on the landing page. The Products, Insights, Careers, Support, Security, and the other sections help create a realistic website, but they do not contain alternative flags or bypasses.

The useful evidence begins in the HTML source and the files already loaded by the browser.

## Step 1 — Inspect the Landing Page Source

On the landing page, right-click and select **View Page Source**. You can also open the browser's Developer Tools and use the **Elements**, **Sources**, or **Network** panel.

Within the HTML source, look at the comments and the JavaScript files included near the bottom of the page. You should notice retained-release context associated with:

```text
PM-274
```

The page also loads:

```text
/assets/js/app.js
```

The `PM-274` value identifies the migration or release activity behind the portal. Keep it in mind because the same context connects this challenge to later evidence.

Open the JavaScript file directly by appending its path to the challenge hostname:

```text
https://<challenge-hostname>/assets/js/app.js
```

Alternatively, select `app.js` from the browser's **Sources** or **Network** panel.

## Step 2 — Follow the JavaScript Source Map

The JavaScript file is small and can be read directly in the browser. At the end of the file, you should find:

```javascript
//# sourceMappingURL=/assets/js/app.js.map
```

This line tells the browser that a source map exists for the script.

A source map links a generated minified JavaScript file to development-time (also known as original, un-minified source code) source information. If it is left accessible in production, it may expose original filenames, comments, paths, or other details that are not visible in the rendered portal.

Open the referenced map in a new browser tab:

```text
https://<challenge-hostname>/assets/js/app.js.map
```

The browser displays a small JSON document. You can read it directly, use the browser's JSON viewer, or search the page with **Ctrl+F** for terms such as `asset`, `manifest`, or `inventory`.

Within the source information, you should find:

```text
/assets/manifest/asset-inventory.json
```

## Step 3 — Read the Asset Inventory

Open the discovered path in the browser:

```text
https://<challenge-hostname>/assets/manifest/asset-inventory.json
```

The response is another readable JSON document containing retained migration metadata.

Review the entries rather than searching only for the word `flag`. Under the retired deployment artifacts, the inventory records:

```text
/backup/config.php.bak
```

The same inventory also includes two values relevant to the next challenge:

```text
collection: platform-comms
record: 2024-q3-transition
```

Record these values. They are not decorative metadata. They describe where the migration team's current release evidence belongs and will help you locate the valid service account later in the chain.

## Step 4 — Open the Retained Backup

Open the exact backup path from the inventory:

```text
https://<challenge-hostname>/backup/config.php.bak
```

The file gets downloaded to your machine. Open it, read it, extract the flag alongside any other information that will be useful as you proceed to the next challenge in the mission.

The retained file contains Flag 1:

```text
flag{inm_r3c0n_b4ckup_f1l3s_l34k}
```

Submit this value to the **Mission: The Last Backup** challenge.

## Step 5 — Preserve the Evidence for Challenge 2

Do not stop reading after finding the flag. The backup also identifies the database administration interface:

```text
/pma/
```

You can confirm that the service exists by opening:

```text
https://<challenge-hostname>/pma/
```

The result is a phpMyAdmin login page. However, the backup explains that its old configuration password was rotated. The backup therefore reveals where the next stage takes place, but it does not provide a working login.

At the end of Challenge 1, you should have collected:

```text
Flag 1:
flag{inm_r3c0n_b4ckup_f1l3s_l34k}

Next service:
/pma/

Release correlation:
PM-274
collection: platform-comms
record: 2024-q3-transition
```

Challenge 2 uses this release correlation to locate the current service-account evidence. Guessing passwords at `/pma/` is not required.

## Optional Command-Line Verification

The entire path can be followed in a browser. If you prefer to confirm the discovered files from a terminal, define the challenge URL:

```bash
BASE='https://<challenge-hostname>'
```

Then request each artifact:

```bash
curl -s "$BASE/assets/js/app.js"
curl -s "$BASE/assets/js/app.js.map"
curl -s "$BASE/assets/manifest/asset-inventory.json"
curl -s "$BASE/backup/config.php.bak"
```

If `jq` is installed, it can format the JSON responses:

```bash
curl -s "$BASE/assets/js/app.js.map" | jq .
curl -s "$BASE/assets/manifest/asset-inventory.json" | jq .
```

These commands are an alternative way to view evidence already available through the browser. They are not required to solve the challenge.

## Why Browsing `/backup/` Does Not Work

Opening the directory itself:

```text
https://<challenge-hostname>/backup/
```

returns `403 Forbidden` because directory listing is disabled. The server will not show all files stored in that directory.

This is why the exact filename must be recovered from the asset inventory. The file is accessible when requested directly, even though the directory contents cannot be listed.

You can optionally confirm the response code from a terminal:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$BASE/backup/"
```

Expected result:

```text
403
```

## Other Clues and Deliberate Decoys

The portal's `robots.txt` contains broad crawler-specific collection references. It may help confirm that retained asset collections exist, but it does not directly reveal the flag, the current password, or the complete route to the backup.

The portal's additional pages, editorial content, CSS files, and auxiliary scripts are benign decoys. They provide a realistic application footprint but do not contain hidden credentials, alternative flags, write functionality, or another solution path.

A sufficiently large content-discovery wordlist might eventually locate the backup file. However, the intended path is faster and provides the context required for Challenge 2:

```text
HTML source -> JavaScript -> source map -> asset inventory -> backup file
```

## Common Wrong Turns

### Looking only at the rendered portal

The first evidence exists in the HTML source and the files loaded by the browser. There is no visible button that reveals the flag.

### Brute-forcing directories immediately

Directory brute force is unnecessary. Every exact path is disclosed by the preceding artifact.

### Searching every file only for `flag{`

The asset inventory contains continuation evidence as well as the backup path. Reading the surrounding metadata is important because the `PM-274`, `platform-comms`, and `2024-q3-transition` values are needed in Challenge 2.

### Guessing phpMyAdmin credentials

Challenge 1 reveals `/pma/`, but not a working password. The old credential in the backup was rotated. The valid service-account evidence is recovered through the next part of the mission.

### Trying to execute the `.php.bak` file

The backup is intentionally served as readable text. The weakness is the disclosure of its contents, not server-side code execution.

### Treating every portal page as meaningful

Several pages and assets are deliberate decoys. Evidence connected to `PM-274`, source maps, retained assets, and migration records is what advances the chain.

