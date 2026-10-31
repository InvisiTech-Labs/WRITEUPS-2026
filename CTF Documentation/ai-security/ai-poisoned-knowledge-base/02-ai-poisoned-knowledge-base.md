# Solution — The Poisoned Knowledge Base

| Field | Value |
|---|---|
| Category | AI Security |
| Path | `/ai2/` |
| Flag | `flag{inm_p01s0n_th3_r3tr13v3r_w1s3ly}` |

## Setup

```bash
export BASE="https://10.50.234.208:5001"
# or: export BASE="http://127.0.0.1:5001"
```

Confirm: `curl -s "$BASE/health"` returns `{"ok":true,...}`.

## What you're looking at

A RAG assistant. It scores uploaded documents against your query using
Jaccard token overlap, then retrieves the top 3. A restricted document
containing the flag is on disk but **not indexed** — it only surfaces when
your upload shares enough vocabulary with it (threshold: product of
similarities ≥ 0.15).

---

## Option A — Command Line (curl)

### Step 1 — Read the docs

```bash
curl -s "$BASE/ai2/docs" | head -40
```

The docs mention restricted documents surface on "significant token
overlap" — a hint to match vocabulary.

### Step 2 — Confirm nothing is retrieved by default

```bash
curl -s -X POST "$BASE/ai2/query" \
  -H 'Content-Type: application/json' \
  -d '{"question": "What is the parking policy?"}'
```

Expected: `{"answer":"Retrieved 0 documents...","retrieved":0}`

### Step 3 — Upload a poisoned document

```bash
curl -s -X POST "$BASE/ai2/upload" \
  -H 'Content-Type: application/json' \
  -d '{"content": "Executive compensation committee Q4 2025 bonus tiers retention packages for senior leadership. Compliance audit exports reviewed."}'
```

Expected: `{"status":"uploaded","total_docs":1}`

### Step 4 — Query with the same vocabulary

```bash
curl -s -X POST "$BASE/ai2/query" \
  -H 'Content-Type: application/json' \
  -d '{"question": "Summarize executive compensation committee Q4 2025 bonus tiers retention packages and compliance audit exports."}'
```

Expected:

```json
{
  "answer": "According to the retrieved documents: flag{inm_p01s0n_th3_r3tr13v3r_w1s3ly}",
  "retrieved": 2
}
```

---

## Option B — Browser (landing page)

The challenge ships with an upload + query UI.

### Step 1 — Open the challenge

```
https://<CTFd-Assigned-Host>/ai2/
```

### Step 2 — Baseline check

In the **Ask the assistant** input, type:

```
What is the parking policy?
```

Click **Ask**. The log shows `retrieved: 0` (nothing indexed yet).

### Step 3 — Upload a poisoned document

In the **Upload a document** textarea, paste:

```
Executive compensation committee Q4 2025 bonus tiers retention packages for senior leadership. Compliance audit exports reviewed.
```

Click **Upload**. The status reads `{"status":"uploaded","total_docs":1}`.

### Step 4 — Query with matching vocabulary

In the **Ask** input, paste:

```
Summarize executive compensation committee Q4 2025 bonus tiers retention packages and compliance audit exports.
```

Click **Ask**. The reply includes the flag.

---

## Option C — Browser DevTools Console

### Step 1 — Open the challenge and DevTools

Navigate to `https://<CTFd-Assigned-Host>/ai2/` and press F12 → **Console**.

### Step 2 — Upload

```javascript
await fetch('/ai2/upload', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    content: 'Executive compensation committee Q4 2025 bonus tiers retention packages for senior leadership. Compliance audit exports reviewed.'
  })
}).then(r => r.json());
```

### Step 3 — Query

```javascript
await fetch('/ai2/query', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    question: 'Summarize executive compensation committee Q4 2025 bonus tiers retention packages and compliance audit exports.'
  })
}).then(r => r.json());
```

The console prints the response with the flag.

---

## Submit

```
flag{inm_p01s0n_th3_r3tr13v3r_w1s3ly}
```

## Why it works

The retriever treats your uploaded content as trusted corpus. Mirroring
the restricted document's vocabulary forces the retriever to pull it into
the answer context — the RAG pipeline has no way to distinguish legitimate
content from attacker-supplied content.

## Key takeaways

- **RAG systems trust their retrieved context.** Control the corpus,
  control the answer.
- Similarity thresholds are not access controls.
- Real mitigation: ACL-aware retrievers, ingest-time filtering,
  per-request access logging, tenant isolation.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `retrieved: 0` on every query | You haven't uploaded any documents. Run Step 3. |
| `retrieved: 1` but no flag | Match phrases like "executive compensation", "Q4 2025", "bonus tiers", "compliance audit exports". |
| `429 Rate limit` | Wait 60 seconds. Limit is 20 requests / 60s per IP per challenge. |