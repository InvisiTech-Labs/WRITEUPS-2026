# Do Not Enter: Solution

**Category:** Warm Up

**Difficulty:** Easy

---

## Overview

Websites use a public file called `robots.txt` to tell search engine crawlers which paths they should not index. It is only a request to crawlers. It does not block anyone, and because anyone can read it, it often shows exactly where the interesting content is.

---

## Step 1: Visit the Website

Open the challenge URL. The homepage has nothing useful on it other than the CTF's landing page.

---

## Step 2: Check robots.txt

Every website keeps this file at the root. Open it:

```
https://ctf.imbank.com/robots.txt
```

or with curl:

```bash
curl https://ctf.imbank.com/robots.txt
```

Output:

```
User-agent: *
Disallow: /admin
Disallow: /backup/
Disallow: /assets/
```

**What this means:**
- `User-agent: *` means the rules apply to all crawlers.
- `Disallow:` lists paths that crawlers are asked to skip.

Three paths are listed. Since the site doesn't want them found, check each one.

---

## Step 3: Explore the Disallowed Paths

| Path | Result |
|---|---|
| `/admin` | Dead end |
| `/backup/` | Decoy, nothing useful |
| `/assets/` | Shows a page that looks empty |

The `/assets/` page only shows some generic text. The flag isn't visible on the page itself.

---

## Step 4: View the Page Source

Developers often leave notes in HTML comments. Comments don't appear on the rendered page, but they are still sent to the browser.

View the source using any of these:
- Right-click the page and choose **View Page Source**
- Press **Ctrl + U**
- Use curl:

```bash
curl https://ctf.imbank.com/assets/
```

Near the bottom of the source:

```html
<!-- Dev note: remove before going live -->
<!-- r0b0ts_c4nt_k33p_s3cr3ts -->
```

---

## Step 5: Format the Flag

The comment has the raw flag value. The event [rules](https://ctf.imbank.com/rules) say every flag must be submitted in the `flag{...}` format. Either of these is accepted:

```
flag{r0b0ts_c4nt_k33p_s3cr3ts}
```

```
flag{inm_r0b0ts_c4nt_k33p_s3cr3ts}
```

---

## Key Takeaways

1. **robots.txt is not a security control.** It is public, and listing sensitive paths in it points attackers straight to them.
2. **Always view the page source.** HTML comments, hidden fields and inline scripts often contain information that the page doesn't display.
3. **Read the rules.** Knowing the expected flag format is part of solving the challenge.

---
