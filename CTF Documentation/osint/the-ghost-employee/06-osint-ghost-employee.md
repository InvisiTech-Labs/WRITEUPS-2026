# Solution — The Ghost Employee

| Field | Value |
|---|---|
| Category | OSINT |
| Path | `/osint/` |
| Flag | `flag{inm_gh0st_3mpl0y33_h1dd3n_c0mm1t}` |

## Setup

```bash
export BASE="https://10.50.234.208:5001"
# or: export BASE="http://127.0.0.1:5001"
```

Confirm: `curl -s "$BASE/health"` returns `{"ok":true,...}`.

## What you're looking at

A simulated corporate site. A former employee committed a secret, then
force-pushed to remove it. The commit remains as a **dangling object** in
the git reflog. Find the employee, reach the git mirror, read the dangling
commit.

This is a browsing challenge — no exploitation.

---

## Option A — Command Line (curl)

### Step 1 — Read the blog

```bash
curl -s "$BASE/osint/api/blog" | jq
```

Expected:

```json
{
  "posts": [
    {
      "id": 1,
      "title": "Farewell — moving on to new adventures",
      "author": "jdoe",
      "body": "After five great years at Nexus Dynamics, I'm moving on. Keep an eye on the internal-tools repo — I may have left some things behind."
    }
  ]
}
```

Two pointers: employee **jdoe** and the **internal-tools repo**.

### Step 2 — Look up jdoe in the directory

```bash
curl -s "$BASE/osint/api/employees" | jq
```

Expected:

```json
{
  "employees": [
    {"name":"Jordan Doe","username":"jdoe","github":"jdoe-dev","university":"Westbrook University","left":true},
    {"name":"Priya Patel","username":"ppatel","github":"priya-p","university":"Riverside Tech","left":false},
    {"name":"Marcus Lee","username":"mlee","github":"mlee-ops","university":"Northgate College","left":false}
  ]
}
```

`jdoe` → GitHub username **`jdoe-dev`**. `left: true`. The other two are decoys.

### Step 3 — Fetch the git reflog

```bash
curl -s "$BASE/osint/git/reflog" | jq
```

Expected:

```json
{
  "commits": [
    {"sha":"a1b2c3…","message":"Initial commit","dangling":false},
    {"sha":"f1e2d3…","message":"WIP: internal tools bootstrap","dangling":false},
    {"sha":"deadbeef…","message":"TODO: remove hardcoded flag before release","dangling":true}
  ]
}
```

### Step 4 — Fetch the dangling commit

```bash
curl -s "$BASE/osint/git/commit/deadbeef" | jq
```

Expected: the diff contains
`AUDIT_FLAG = "flag{inm_gh0st_3mpl0y33_h1dd3n_c0mm1t}"`.

---

## Option B — Browser (landing page buttons)

The landing page has three buttons that hit the same API endpoints.

### Step 1 — Open the challenge

```
https://<CTFd-Assigned-Host>/osint/
```

### Step 2 — Read the latest blog post

Click **"Read the full post"** under "Latest from the blog". A `<pre>`
panel shows the blog JSON. Note `"author": "jdoe"` and the phrase
`"internal-tools repo"`.

### Step 3 — Load the employee directory

Click **Load directory**. The panel shows employee JSON. Note:

- `"username": "jdoe"`, `"github": "jdoe-dev"`, `"left": true`
- Two other employees with `left: false` are decoys.

### Step 4 — Fetch the git reflog

Click **GET /git/reflog**. The panel shows commit JSON. The third commit
is flagged `"dangling": true` with message
`"TODO: remove hardcoded flag before release"`.

### Step 5 — Fetch the dangling commit

The landing page doesn't have a button for this — navigate directly in
the **address bar**:

```
https://<CTFd-Assigned-Host>/osint/git/commit/deadbeef
```

The browser displays the JSON diff with the flag.

---

## Option C — Browser (cross-page click trail)

You don't need to hit API URLs — the site is designed to be browsed.

### Step 1 — Start at the landing page

```
https://<CTFd-Assigned-Host>/osint/
```

### Step 2 — Read the blog

Click **"Read the full post"**. Note the author `jdoe` and the phrase
`"internal-tools repo"`.

### Step 3 — Open a profile

Navigate directly to:

```
https://<CTFd-Assigned-Host>/osint/linkedin/jdoe-dev
```

Confirms identity: `"alumni_of": "Westbrook University"`,
`"github": "jdoe-dev"`, `"note": "Former Nexus Dynamics employee."`.

### Step 4 — Look at the git history

Back on the landing page, click **GET /git/reflog**. The third entry is
your target.

### Step 5 — Read the commit

Navigate to:

```
https://<CTFd-Assigned-Host>/osint/git/commit/deadbeef
```

The flag appears in the `diff` field.

---

## Option D — Browser DevTools (Network tab)

This mimics how an OSINT investigator watches traffic.

### Step 1 — Open `/osint/` and press F12

Switch to the **Network** tab.

### Step 2 — Click through the buttons

Each click fires a request. As you click:

1. **"Read the full post"** → `GET /osint/api/blog`
2. **"Load directory"** → `GET /osint/api/employees`
3. **"GET /git/reflog"** → `GET /osint/git/reflog`

The Network tab shows each request and its JSON response. Click a request
and view the **Response** tab to read the JSON.

### Step 3 — Fetch the dangling commit

In the URL bar at the top of the browser, navigate to:

```
https://<CTFd-Assigned-Host>/osint/git/commit/deadbeef
```

The flag is visible in the diff field.

---

## Option E — Browser + JSON formatters

If the inline JSON is hard to read:

1. Open `/osint/api/employees` in one tab and `/osint/git/reflog` in
   another.
2. Copy the JSON from each.
3. Paste into **https://jsonformatter.org** (or any JSON pretty-printer).
4. Read the formatted output — the `dangling: true` flag jumps out.

Then navigate to `/osint/git/commit/deadbeef`.

---

## Submit

```
flag{inm_gh0st_3mpl0y33_h1dd3n_c0mm1t}
```

## Why it works

Real incident pattern: developer commits a secret, force-pushes to remove
it. The commit remains reachable as a dangling object until Git garbage
collects. Anyone with reflog access (or a GitHub commit URL keyed by SHA)
can recover it.

## Common dead ends

| Attempt | Result |
|---|---|
| Look at `priya-p` or `mlee-ops` | No dangling commits — decoys |
| `/osint/api/profile/jdoe` | Works, but the GitHub username is `jdoe-dev` |
| Search the live site for "flag" | Not present — only in the diff |
| Look at current `master` | Not there — force-pushed |

## Key takeaways

- **Deleting a commit isn't deleting the data.**
- Force-pushing doesn't remove secrets from the object database.
  **Rotate the secret immediately.**
- OSINT is assembling a picture from scattered public sources.
- Real mitigation: `git filter-repo` or BFG to rewrite history, **and**
  rotate any exposed secret. Better: never commit secrets — use a secret
  manager and pre-commit hooks.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `{"error":"not found"}` on a profile | Wrong username. Try `jdoe-dev`. |
| `{"error":"unknown commit"}` | SHA prefix doesn't match. Use `deadbeef`. |
| No dangling commit in reflog | Stale deployment. Verify `$BASE/health` and redeploy. |