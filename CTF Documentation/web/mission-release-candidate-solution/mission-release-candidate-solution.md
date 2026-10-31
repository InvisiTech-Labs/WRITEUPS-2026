# Mission: Release Candidate

## Post-Event Solution

Challenge 1 revealed the phpMyAdmin route, but the retained backup also stated that its password had been rotated. The correct next step is not to guess credentials. Instead, use the release metadata recovered from the asset inventory to identify the portal component associated with the current release record.

The solution path is:

```text
Challenge 1 evidence
PM-274 + platform-comms + 2024-q3-transition
        |
        v
Products page
        |
        v
Annual Savings Planner functionality
        |
        v
/assets/js/portal.js
        |
        v
/assets/js/portal.js.map
        |
        v
/assets/release-notes/2024-q3-transition.txt
        |
        +--> Current service-account credentials
        |
        v
/pma/ -> appdb -> operational_notes
        |
        +--> Flag 2
        +--> CC-17 for Challenge 3
```

## What You Have at the Start

After solving **The Last Backup**, you should have retained:

```text
Administrative service: /pma/
Migration reference: PM-274
Collection: platform-comms
Record: 2024-q3-transition
```

Opening `/pma/` displays a phpMyAdmin login page, but the retired password from the backup does not work. Default accounts, blank passwords, and password guessing are not part of the solution.

The collection name `platform-comms` and record name `2024-q3-transition` indicate that another released portal asset should contain the current evidence. Return to the main website and continue reviewing its pages and browser-loaded files.

## Step 1 — Open the Products Page

Use the portal's navigation bar to open the **Products** page.

The page contains two main areas:

1. A products explorer section with controls for filtering the displayed products.
2. An **Annual Savings Planner** calculator below the product information.

Enter a monthly amount in the calculator. The page immediately displays the projected annual amount in Kenyan shillings.

This interaction confirms that the Products page is using client-side JavaScript. Open the browser's Developer Tools and use one of these approaches:

- Select the **Network** panel, filter by `JS`, and refresh the Products page.
- Select the **Sources** panel in Debugger and expand the site's `/assets/js/` directory.
- Use **View Page Source** and review the scripts loaded near the end of the HTML.

You should find:

```text
/assets/js/portal.js
```

Open it from the Developer Tools interface or directly in a new browser tab:

```text
https://<challenge-hostname>/assets/js/portal.js
```

## Step 2 — Understand What `portal.js` Does

The JavaScript file is short and readable. Its visible code supports the functionality you just observed on the Products page.

The first section handles the product filters:

```javascript
var controls = document.querySelectorAll("[data-service-filter]");
var cards = document.querySelectorAll("[data-service]");
```

When a filter is selected, the script shows only the product cards matching the requested service category.

The second section supports the Annual Savings Planner:

```javascript
var calculator = document.querySelector("#savings-calculator");
```

When the monthly amount changes, the script multiplies it by 12 and formats the result as Kenyan shillings:

```javascript
amount * 12
```

This explains why `portal.js` is loaded on the Products page. It is a legitimate interface script, not an obviously suspicious file. The useful evidence appears at the very end:

```javascript
//# sourceMappingURL=/assets/js/portal.js.map
```

This source-map directive identifies an additional development artifact retained on the server.

## Step 3 — Inspect the Source Map

Open the referenced map in a new browser tab:

```text
https://<challenge-hostname>/assets/js/portal.js.map
```

The browser displays a small JSON document. Use the browser's JSON viewer or **Ctrl+F** to search for the evidence retained from Challenge 1:

```text
platform-comms
2024-q3-transition
PM-274
```

The source-map information maps the release identifiers to:

```text
/assets/release-notes/2024-q3-transition.txt
```

This is the current release record associated with the migration evidence.

## Step 4 — Read the Release Record

Open the discovered file:

```text
https://<challenge-hostname>/assets/release-notes/2024-q3-transition.txt
```

The browser displays a plain-text transition record containing the active read-only service account:

```text
username: svc_console
password: C0nsole-2026-Release!84
```

These credentials are stored in the retained release evidence rather than the old backup. They are the intended login for the database console.

## Step 5 — Sign In to phpMyAdmin

Open the administrative route recovered in Challenge 1:

```text
https://<challenge-hostname>/pma/
```

Sign in with:

```text
Username: svc_console
Password: C0nsole-2026-Release!84
```

After authentication, phpMyAdmin displays the database objects available to this account. The service account is intentionally read-only and should show:

```text
appdb
```

There is no need to alter server settings, create tables, upload files, or exploit phpMyAdmin.

## Step 6 — Browse `operational_notes`

Select **appdb** in the left navigation panel. phpMyAdmin displays the tables within the application database.

Open:

```text
operational_notes
```

Select the **Browse** tab to view its contents. The operational_note table contains Flag 2. Enumerate the rows and identify the one with console migration details:

```text
flag{inm_c0ns0l3_3num3r4t10n_w1ns}
```

Submit this value to the **Mission: Release Candidate** challenge.

## Step 7 — Preserve the Challenge 3 Reference

Continue reading the operational note after finding the flag. It also identifies:

```text
CC-17
```

This is a change-control reference, not another credential. It indicates that another table in `appdb` contains the approved verification record required for Challenge 3.

At the end of Challenge 2, you should have:

```text
Flag 2:
flag{inm_c0ns0l3_3num3r4t10n_w1ns}

Continuation reference:
CC-17
```

Remain within `appdb` for the next stage. The next task is to correlate `CC-17` with the database's change-control table records.

## Optional SQL View

The intended route is to browse the table through the phpMyAdmin interface. The same information can also be viewed from phpMyAdmin's **SQL** tab with this SQL query:

```sql
SELECT subject, note
FROM appdb.operational_notes;
```


## Optional Command-Line Verification

The static files can also be retrieved with `curl` after their paths have been discovered in the browser:

```bash
BASE='https://<challenge-hostname>'

curl -s "$BASE/assets/js/portal.js"
curl -s "$BASE/assets/js/portal.js.map"
curl -s "$BASE/assets/release-notes/2024-q3-transition.txt"
```

This is an alternative way to read the same browser-accessible evidence.

## Solution Summary

The Products page loads `portal.js` to support its service filters and Annual Savings Planner. Inspecting that small script reveals a source-map reference. The source map correlates the retained release identifiers from Challenge 1 with the `2024-q3-transition` release record.

The release record exposes the current `svc_console` credentials. Those credentials provide read-only access to `appdb`, where the `operational_notes` table contains Flag 2 and the `CC-17` reference required for the next challenge.
