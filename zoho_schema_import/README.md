# Zoho Creator — Import Schema files

One CSV per form (matches `../FIELDS.md` Part 1 exactly), built for Zoho Creator's **Form → Import Schema** flow ("On my own" → "Import Schema" → upload the CSV). Each file's header row is the field's **API name with underscores swapped for spaces** (e.g. `Vehicle_Registration_Number` → `Vehicle Registration Number`) — Zoho derives the new field's API name back from that header by replacing spaces with underscores again, so importing these gives you the *exact* API names the widget code already expects. A few sample data rows are included in each file so Zoho's column-type auto-detection has something real to guess from (dates look like dates, numbers look like numbers, etc.) — pick each column's type carefully on the mapping screen before finishing the import, using the "Zoho Field Type" column in `FIELDS.md` as the source of truth.

**Import order matters** — create the master/lookup-target forms first, then `Create_Case`, then `Invites`/`Task_Rejections` last (their Lookup fields point at forms that need to exist first):

1. `Client.csv` → creates the `Client` form
2. `Vehicle_Master.csv` → creates the `Vehicle_Master` form
3. `Vehicle_Issue.csv` → creates the `Vehicle_Issue` form
4. `Vendors.csv` → creates the `Vendors` form
5. `Technicians.csv` → creates the `Technicians` form
6. `Rate_Master.csv` → creates the `Rate_Master` form
7. `Create_Case.csv` → creates the `Create_Case` form (the big one)
8. `Task_Rejections.csv` → creates the `Task_Rejections` form
9. `Invites.csv` → creates the `Invites` form

After each import, **rename the form's default report** to match what the code expects (e.g. `Create_Case`'s own default report → `Agent_Ticket_Report`) — see `../FIELDS.md`'s own "Form X (report: ...)" heading for each form's target report name, and `../ACCESS.md` for who needs access to it.

## What this import gets right automatically

- Every field's **API name** (assuming you keep the header text as given — don't "clean up" the header text before uploading, or the derived API name won't match what the code expects).
- A reasonable first guess at **data type** for plain text/number/date columns, since the sample rows use realistic values.

## What you still have to fix by hand after each import — this is the important part

- **Dropdown / Radio option lists.** Zoho's schema import does not reliably pre-populate a Dropdown's options from the sample data across all import paths — after import, open every field marked "Dropdown"/"Radio" in `FIELDS.md` and type in its exact option list yourself (case and spelling matter, the widgets match on the literal string). The shared `REJECT_REASONS` list (5 options, see `FIELDS.md`) is reused by ~10 different fields across these forms — worth setting up once and checking each field against it.
- **Lookup fields.** CSV/Excel import cannot create a real Lookup relationship — `Client`, `Vehicle`, `Vendors1`, `Assigned_Vendor` (on `Create_Case`), `Vehicle`/`RSID` (on `Task_Rejections`), and `RSID`/`Vendor`/`Technician` (on `Invites`) will all come in as plain Single Line text fields. You must manually change each one's type to **Lookup** and point it at the correct target form (named in `FIELDS.md`'s own "Zoho Field Type" column, e.g. `Client → Client_Report`). `Vendors1` also needs **Multi-Lookup** (allow multiple), not single.
- **File Upload fields — not in these CSVs at all** (a spreadsheet import can't carry file attachments), add these manually on `Create_Case` after import: `Image_Upload`, `Image_Upload2`, `Image_Upload3`, `Image_Upload4`, `Image_Upload5`, `Image_Upload6`, `Expense_Photo`, `Pre_service_Photo`, `Image_Upload_On_truck`, `VCRF`, `Drop_Location_Photo`, `Unloaded_Images`, `VCRF_Image`, `Handover_Image`, `Upload_Photo_of_receipt`, `Closure_Extra_Photo`.
- **Multi Select fields.** `Vehicle_Issue` (on `Create_Case`) and `Vehicle_Category` (on `Vehicle_Issue`) should be **Multi Select**, not plain text — the `Vehicle_Issue.csv` sample row for "Battery Jumpstart" deliberately shows `2W,4W` in one cell as a hint, but you'll likely still need to switch the field type manually and re-enter the option list (`2W`, `4W`).
- **Required flags, defaults, and field type nuances** called out in `FIELDS.md` (e.g. `Status` recommended as Single Line — not Dropdown — on purpose; `RSP_Start_Latitude`/`Longitude` must be Text, not Decimal; `Case_ID` must not be an auto-number field). None of that survives a spreadsheet import — go through `FIELDS.md` Part 1 field-by-field after each import and correct anything the auto-import guessed differently.

**Bottom line**: this import gets your field *names* exactly right on the first try (the tedious, error-prone part when typing ~100 fields by hand) and saves you from typing every single field label yourself — but every Dropdown's options, every Lookup's target, every File Upload field, and every Multi Select still need a manual pass against `FIELDS.md` afterward. Budget for that pass; don't assume the form is done the moment the import finishes.
