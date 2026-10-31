# Mission: Change Control

## Post-Event Solution

The previous challenge ends in `appdb.operational_notes`, but the continuation is deliberately written as ordinary operational language rather than as a ready-made identifier.

The relevant note:

- Contains Flag 2.
- Appears on record **17**.
- Refers to **change control number seventeen**.

The task is to understand that evidence, inspect the database structure, normalize the reference as `CC-17`, and use it to find the correct change-control record among similar-looking noise.

## Step 1 — Interpret the Operational Note

Remain signed in to phpMyAdmin and return to:

```text
appdb.operational_notes
```

The table contains many unrelated rows. Use phpMyAdmin's **Search** or **SQL** tab to locate the row containing Flag 2 rather than reviewing every record manually.

For example:

```sql
SELECT *
FROM operational_notes
WHERE note LIKE '%flag{inm_%';
```

The matching record is row `17`. Its text refers to:

```text
change control number seventeen
```

This is not a second flag and it is not a complete URL. It tells you how to construct the change-control reference:

```text
CC-17
```

The hyphen matters. Other rows and columns may contain similar values such as `cc17` or `CC17`, but those are noise and do not represent the normalized reference used by the verification workflow.

## Step 2 — Inspect the Change-Control Structure

Open:

```text
appdb.change_control
```

Review the column names before writing the next query. The table contains numerous unrelated records, aliases, statuses, and review details. Some rows deliberately contain values resembling the target reference.

Use the normalized reference derived from the operational note:

```sql
SELECT *
FROM change_control
WHERE change_ref = 'CC-17';
```

The exact match returns the relevant record. Its verification path is stored as a template:

```text
/operations/verify.php?ref={change-ref-number}
```

## Step 3 — Complete the Verification Path

Replace the placeholder with the normalized reference:

```text
/operations/verify.php?ref=CC-17
```

Open the completed path in the browser:

```text
https://<challenge-hostname>/operations/verify.php?ref=CC-17
```

The endpoint returns a JSON response containing Flag 3:

```text
flag{inm_ch4ng3_c0ntr0l_d3c0d3d}
```

Submit this value to the **Change Control**.

## Step 4 — Preserve the Continuation Artifact

The same JSON response contains the path required for Challenge 4:

```text
/assets/operations/level4-access.php.bak
```

Keep the path exactly as returned.

## Alternative Filtering in phpMyAdmin

The challenge can be completed without writing SQL manually:

1. Open `operational_notes` and use **Search** to locate the Flag 2 row.
2. Note that it is record 17 and read the phrase “change control number seventeen.”
3. Open `change_control` and use **Search**.
4. Set `change_ref` to the exact value `CC-17`.
5. Open the verification path from the matching result.


## Solution Summary

Find the Flag 2 record in `operational_notes`, recognize that record 17 and “change control number seventeen” describe `CC-17`, then query `change_control` using that exact normalized reference. Substitute `CC-17` into the stored verification-path template and open the resulting URL to obtain Flag 3 and the Challenge 4 continuation artifact.
