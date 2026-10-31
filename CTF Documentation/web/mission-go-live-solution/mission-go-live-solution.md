# Mission: Go Live

## Post-Event Solution

The change-control verification provides a continuation artifact containing the final token and endpoint. To complete attestation, combine that material with the original release reference and submit a correctly constructed API request.

## Step 1 — Read the Handoff

Open the continuation path returned by Challenge 3 in your browser:

```text
https://<challenge-hostname>/assets/operations/level4-access.php.bak
```

The file is served as text and provides:

```text
token: 8daf72c7610e5ccbcedf3668a7a1e9af
endpoint: /operations/attest.php
```

The surrounding comment associates attestation with the original console migration release and states that only the approved request method is accepted. The token is request material, not a flag.

## Step 2 — Follow the API's Feedback

Open the endpoint in the browser:

```text
https://<challenge-hostname>/operations/attest.php
```

Browser navigation sends GET. The response reports:

```text
This request method is not approved for attestation. Review the request method.
```

Use browser Developer Tools or an API client to control the request. Testing **PUT** advances past this check and produces:

```text
Required request header missing: Release-Reference.
```

As you complete the request, further errors explain that the body must be a JSON object containing a non-empty string named `token`. Read each response as feedback on the first requirement that has not been satisfied.

## Step 3 — Recover the Release Reference

Return to the landing page and select **View Page Source**. The retained comment contains:

```html
<!-- Retained release reference: PM-274. Preserve this reference when reviewing console migration. -->
```

The header must therefore contain:

```text
Release-Reference: PM-274
```

Do not use `CC-17`: that identifies the change-control review, not the console migration release. With a correct token but an incorrect release value, the API reports that the release reference is invalid or incorrectly formatted.

## Step 4 — Submit the Attestation

Open Developer Tools on a page within the challenge hostname and select **Console**. Send this same-origin request:

```javascript
fetch('/operations/attest.php', {
  method: 'PUT',
  headers: {
    'Release-Reference': 'PM-274',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    token: '8daf72c7610e5ccbcedf3668a7a1e9af'
  })
}).then(async response => {
  console.log(response.status, await response.json());
});
```

The successful response is HTTP 200 with:

```json
{
  "status": "attested",
  "flag": "flag{inm_r3l34s3_v3r1f13d_c4s3_cl0s3d}"
}
```

Submit the flag to **Go Live**. The final checklist is complete: the release reference, recovered token, and request method all match. The migration is cleared for production.

## Alternative — curl

The same request can be sent from a terminal:

```bash
curl -i -X PUT \
  'https://<challenge-hostname>/operations/attest.php' \
  -H 'Release-Reference: PM-274' \
  -H 'Content-Type: application/json' \
  --data '{"token":"8daf72c7610e5ccbcedf3668a7a1e9af"}'
```

## Alternative — Postman or Insomnia

1. Create a new HTTP request and select **PUT**.
2. Enter the endpoint:

   ```text
   https://<challenge-hostname>/operations/attest.php
   ```

3. Under **Headers**, add:

   ```text
   Release-Reference: PM-274
   Content-Type: application/json
   ```

4. Configure the request body:
   - **Postman:** Select **Body → raw → JSON**.
   - **Insomnia:** Select **Body → JSON**.

5. Enter the recovered token as a JSON string:

   ```json
   {
     "token": "8daf72c7610e5ccbcedf3668a7a1e9af"
   }
   ```

6. Click **Send**. A successful request returns HTTP **200** with:

   ```json
   {
     "status": "attested",
     "flag": "flag{inm_r3l34s3_v3r1f13d_c4s3_cl0s3d}"
   }
   ```

If the request is rejected, read the response message and check the method, headers, and JSON body.
