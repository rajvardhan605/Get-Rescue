# Vendor upload template (updated 17-Nov-2026)

- Row 1 = the exact Zoho field link names. Do not rename them.
- Row 2 = the field type; **delete row 2 and the EXAMPLE row before importing.**

## Required (the import fails without these)
- `vendor_name` - must be unique.
- `service_type` - `RSR`, `TOW` or `Both`. It also decides which tickets the vendor is offered: a TOW-only vendor never appears for a repair ticket.
- `Availability_Status` - put `Offline`. The vendor app sets Online/Offline itself once the vendor logs in.
- `Email` - must be unique, and must be the vendor's portal login email (upper or lower case both work). The vendor app matches the login to it.

## Dropdown values (type exactly)
- `Vendor_Priority`: 1, 2, 3 or 4
- `Approval_Status`: Active / Deactive
- `Display_in_list_crm`: Show / Hide
- `Vendor_Type`: Fleet / Individual
- `Engagement_Type`: Own Mechanic / Tie-Up

## Vendor rates
- The 24 DAY / NIGHT rate columns (BASE PAY, BASE KMS, EXTRA KM for Repair, Tow 2W FBT, Tow 4W FBT and Tow 4W MT) are what the vendor fee is calculated from.
- Fill every category the vendor serves.
- `NIGHT_START_HOUR` / `NIGHT_END_HOUR` are Time fields, e.g. `22:00` and `06:00`. A plain `22` may be rejected by the import.
- **`Base_Fee`, `Base_KM` and `KM_Rate` are a fallback.** They are used only when the matching day/night rate is blank. If every rate is filled, they are never used, and you can treat them as data-entry fields. Recommended: fill them with the vendor's day Repair rates as a safety net. If both a rate and its fallback are blank, that part of the fee is calculated as 0.

## Other columns
- `TDS_Percentage` - e.g. 1 or 2.
- `KM_Slab_*` - follow the format Operations already uses in Zoho; copy one existing vendor's values if unsure. `KM_Slab_Tow_2W_MT` is removed because it is not in the Vendors table.
- `Base_Location_1..5` - the vendor's fleet locations (lat/long). Used for distance to the breakdown when the vendor is offline. Set `Vendor_Base_Location_Count` to how many are filled.
- `Latitude` / `Longitude` (GPS HOME LOCATION) - optional. Used only as a last fallback when the vendor has no base location and no address location.
- `Address` - Zoho composite field: in the import screen map its parts (line 1, city, state, postal code) to separate columns if needed.
- `Vehicle_Issue` - optional. It is a multi-select of issues; easiest to set in Zoho after the import. The apps do not use it to filter vendors.

## Left out on purpose
- **Filled by the apps:** Current_Latitude, Current_Longitude, Location_Last_Updated_At, Last_Online_At, Last_Seen_At, Active_Device_ID, Active_Device_At, Zoho_Books_Vendor_ID.
- **Documents and photos** (can't go through a CSV; upload them per vendor in Zoho): Vendor_Photo, Address_Proof, Aadhaar/DL/PAN photos, Cancelled_Cheque, UPI_Scan_Code.
- **Other fields** (can be added if you want them): bank details, Aadhaar/DL/PAN numbers, Payment Schedule, Payment Type, overtime and wait charges.

## How to import
Zoho Creator -> Vendors form -> Import data -> upload this CSV -> check the column mapping -> import.
