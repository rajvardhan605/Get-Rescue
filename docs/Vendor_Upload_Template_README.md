# Vendor upload template


- Row 1 = the exact Zoho field link names. Do not rename them.
- Row 2 = the field type; **delete row 2 and the EXAMPLE row before importing.**
- `Vendor_Type`: `Individual` or `Fleet`. `Engagement_Type`: `Own Mechanic` or `Tie-Up`.
- `Email` must be the vendor's portal login email (lower or upper case both work); the vendor app matches the login to it.
- `Address` is a Zoho composite field: in the Zoho import screen map its parts (line 1, city, state, postal code) to separate columns if you need them; it is left as one column here.
- `NIGHT_START_HOUR` / `NIGHT_END_HOUR` are Zoho Time fields, e.g. `22:00` and `06:00`.
- KM slabs (`KM_Slab_*`) follow the format Operations already uses in Zoho; copy one existing vendor's values if unsure.
- Left out on purpose (the apps fill them): Current_Latitude, Current_Longitude, Location_Last_Updated_At, Last_Online_At, Availability_Status, Active_Device_ID, Active_Device_At, Zoho_Books_Vendor_ID.

Import: Zoho Creator -> Vendors form -> Import data -> upload this CSV -> check the column mapping -> import.
