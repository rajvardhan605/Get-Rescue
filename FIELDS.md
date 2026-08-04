# Get-Rescue — Field Registry & Zoho Creator Build Spec (single source of truth)

This file is the **one place** every Zoho field API/link name used anywhere in this repo is listed. Going forward:

- **Before using a field name in code, check here first.** If a name here conflicts with what's already in a widget's code, this file wins — update the code to match.
- **To correct a wrong field name, edit it in this file only.** The correction then gets carried into whichever widget(s) use that field the next time that widget is touched (or immediately, if you ask for it).
- **Every new field any future feature needs gets added here too**, in the right section, at the same time it's added to the actual widget code — not just left living in a code comment.

**2026-08-03 — all Zoho forms/reports were deleted and rebuilt from scratch.** The tables in this file are now the **authoritative build spec** — create every form/field below exactly as listed (API names already follow the `Label_With_Underscores` convention, matching the field's own label) and all six widgets' code already targets these exact names. Where a field name changed from what shipped before 2026-08-03, or a field was dropped entirely, that's called out inline.

**Status column key** (used in the narrative "Step" tables further down):
- ✅ **Confirmed** — verified against a real Zoho record or a real Zoho API error message.
- 🟡 **From spec** — given directly by a client-provided flow spec/screenshot, not independently re-checked in Zoho Studio.
- ❓ **Guessed** — a pattern-matched name with no direct source; most likely to need correcting once real data starts flowing.

**Widget column** shows every widget that reads or writes that field: **Agent** (`getRescueTicket`), **Kanban** (`ticketKanbanGetRescue` — a second view over the same Create_Case data, not listed separately unless it uses something extra), **Technician** (`technicianTicket`), **Driver** (`driverTicket`), **Vendor** (`vendorTicket`), **Toggle** (`toggleGetRescue` — superseded, kept only for reference/rollback, not part of the rebuilt schema).

---

# PART 1 — Forms & Reports to create in Zoho Creator

Nine forms total. `Create_Case` is the only large one — every other form is small. Zoho auto-creates a default report per form; the app always reads/writes through the renamed report shown here, not the form's own default report name.

**Fast start**: `../zoho_schema_import/` (repo root) has one CSV per form, pre-built for Zoho Creator's own **Import Schema** feature — gets every field's exact API name right on the first try. See that folder's own README for the import order and, importantly, what it *doesn't* get right automatically (Dropdown option lists, Lookup targets, File Upload fields, Multi Select) that still needs a manual pass against the tables below.

## Shared option list — `REJECT_REASONS`

Reused, unmodified, by every "reason"-style Dropdown below (rejections, cancellations, cx-rejects, refunds, closure) unless noted otherwise. Build this as a Dropdown with exactly these 5 options, in this order:
`Too Far / Outside Service Area`, `Already Engaged With Another Job`, `Vehicle/Equipment Not Suitable`, `Not Available (Off Duty)`, `Other`

## Form 1 — `Create_Case` (report: `Agent_Ticket_Report`)

The single ticket record. Every profile reads/writes through `Agent_Ticket_Report`, never `Create_Case`'s own default report.

**A note on `Status`**: this field is never shown as a picker in any widget — it's always set programmatically (17+ distinct values, see the Status Values table near the end of this file, and the list keeps growing as flow gets refined). Given this session's repeated "Invalid column value" bugs on fields that *were* strict Dropdowns, **build `Status` as Single Line Text, not a Dropdown** — it removes an entire class of save failures with no downside, since nothing ever presents its options to a human to pick from. (If you'd rather have stricter reporting/filtering in native Zoho views, a Dropdown works too — just remember to add every value in the Status Values table as an option, and add new ones there first whenever a new stage is introduced.)

| API Name | Label | Zoho Field Type | Options (exact, in order) / Default | Notes |
|---|---|---|---|---|
| `Customer_Name` | Customer Name | Single Line | — | Required |
| `Phone_Number1` | Phone Number | Single Line | — | Required. This is the box the agent actually types into |
| `Phone_Number` | Phone Number (Primary) | Single Line | — | Required by Zoho on save; silently mirrored from `Phone_Number1` by the widget (`extra()` hook) — the agent never sees this field |
| `Alternate_Phone_Number` | Alternate Phone Number | Single Line | — | Optional |
| `Email` | Email | Single Line | — | Use Single Line (not Zoho's Email type) to avoid format-validation save failures |
| `Client` | Client | Lookup → `Client_Report` | — | Required |
| `Client_Tracking_Id` | Client Tracking ID | Single Line | — | Hidden in the UI whenever `Client` is the special "ON DEMAND" walk-in entry |
| `Vehicle` | Vehicle | Lookup → `Vehicle_Master_Report` | — | Required |
| `Vehicle_Registration_Number` | Vehicle Reg. Number | Single Line | — | Optional |
| `Vehicle_Issue` | Vehicle Issue | Multi Select **Lookup** → `Vehicle_Issue_Report` | populated from `Vehicle_Issue_Report.Issue_Name` | Required. **Corrected 2026-08-04, user-confirmed**: this is a real multiselect Lookup (`useId:true`, saves record IDs) — reverses the earlier "plain multi-select text, not a Lookup" note. Filtered client-side to the chosen Vehicle's `Category` matched against the issue's own `VEHICLE_TYPE`, and only offers issues whose `DISPLAY_STATUS` is `SHOW` |
| `Service_Type` | Service Type | Dropdown | `RSR`, `TOW` | Never picked directly by a human — the widget derives and writes it from the chosen `Vehicle_Issue`'s own `Issue_Type` |
| `Vehicle_Status` | Vehicle Status | Dropdown | `ONROAD`, `SAFE PARKING` | Required |
| `Time_of_service` | Time of Service | Radio | `NOW`, `LATER` | Required |
| `Schedule_Date` | Schedule Date | Date | — | Required when `Time_of_service = LATER` |
| `Service_Time` | Service Time | Dropdown | Half-hour slots covering the full 24-hour day (e.g. `12:00 AM - 12:30 AM`, `12:30 AM - 1:00 AM`, … `11:30 PM - 12:00 AM`) — **widened 2026-08-04** from the earlier 9:00 AM–5:00 PM-only window | Required when `Time_of_service = LATER` |
| `Computed_Service_Time` | Computed Service Time | Date-Time | — | Hidden; auto-computed (NOW → +30 min from save time; LATER → picked Schedule_Date + Service_Time) |
| `Lead_Source` | Lead Source | Dropdown | `Call`, `Website`, `WhatsApp`, `App`, default `Call` | Always written as `Call` by this agent-facing widget; hidden from the UI |
| `Remarks` | Remarks | Multi-line | — | Append-only in the widget (past entries read-only, only new timestamped lines addable) — same physical field reused at every step below, not a separate field per step |
| `Status` | Status | Single Line (see note above) | see Status Values table | Hidden; drives which stage every widget resumes a ticket at |
| `Case_ID` | Case ID | Single Line | — | Hidden; the widget writes a human-readable value itself (`"RSID"` + last 6 digits of the record's own ID) right after creating the record — do **not** make this an auto-number field, the app needs to set its own value |
| `Distance_For_Quote_RT_KM` | Distance for Quote (RT KM) | Number | — | TOW only |
| `Suggested_Rate` | Suggested Rate | Number | — | Computed/read-only in the widget |
| `Total_Service_Fee` | Total Service Fee | Number | — | Also displayed read-only again at Step 8 (Payment) — see `payment` stage note below; the old separate `Total_Service_Fee1` field was dropped, this one field now serves both |
| `Booking_Fee` | Booking Fee | Number | — | |
| `Remaining_Fee` | Remaining Fee | Decimal | — | = Total − Booking; re-displayed at Payment |
| `Rate_Breakup` | Rate Breakup | Multi-line | — | |
| `Send_Link_For_The_Breakdown_Location` | Request Location | Radio (`Yes`/`No`, or Checkbox) | — | Action-trigger field, not a real dropdown a human picks — value just gets flipped by the "Request Location" button |
| `Payment_Method` | Payment Method | Dropdown | `Cash`, `UPI`, `Card`, `Net Banking` | Booking-fee payment method — kept locked/display-only in the wizard |
| `Payment_Status` | Payment Status | Dropdown | `PAID`, `PENDING`, `NOT APPLICABLE` | **The same field is reused at Step 8** (Final Payment) — one field, not two, confirmed live 2026-08-03 |
| `Image_Upload` | Receipt Photo | File Upload | — | Booking-fee receipt |
| `Latitude` / `Longitude` | Breakdown Latitude / Longitude | Decimal | — | Pasted as `lat, lon` text and split client-side |
| `Send_Link_For_The_Drop_Location` | Send Drop Location Link | Checkbox | — | TOW only |
| `DropLocationLat` / `DropLocationLong` | Drop Latitude / Longitude | Decimal | — | TOW only — the customer's specified drop point |
| `RT_KM` | Round-Trip KM | Number | — | TOW only, agent-side Haversine calc |
| `Location_Received_Time` | Location Received Time | Date-Time | — | Hidden; stamped once, never overwritten |
| `Break_Down_Location1` | Breakdown Location Link | Single Line or Formula | — | Field widgets fall back to a `maps.google.com` link built from `Latitude`/`Longitude` if this is empty |
| `Navigate_To_Drop_Link` | Navigate to Drop Link | Multi-line | — | TOW only |
| `Preferred_Vendor` | Preferred Vendor? | Radio | `Yes`, `No`, default `No` | Yes = show every vendor; No = geofenced (8km RSR / 20km TOW) |
| `Vendors1` | Vendors | Lookup (Multi) → `Vendors_Report` | — | Saves real `Vendors_Report` record IDs |
| `Assigned_Vendor` | Assigned Vendor | Lookup → `Vendors_Report` | — | Set by the vendor's own Accept action, not the agent |
| `Vendor_Emails` | Vendor Emails | Single Line | — | Hidden; comma-separated emails of everyone in `Vendors1`, auto-populated on save |
| `Vendor_Email` | Vendor Email (Assignment) | Single Line | — | Outcome field, shown once someone's accepted |
| `Assigned_Technician` | Assigned Technician | Dropdown, populated from `Technicians_Report.Technician_Name` | — | Set on a fleet vendor's hand-off |
| `Assigned_Technician_Email` | Technician Email (Assignment) | Single Line | — | |
| `Reject_Reason` (RSR) / `Reject_Reason1` (TOW) | Reject Reason | Dropdown | *`REJECT_REASONS`* | |
| `Vendor_Email1` (RSR) / `Vendor_Email2` (TOW) | Vendor Email | Single Line | — | Acceptance step's own copy |
| `Technician_Email` (RSR) / `Technician_Email1` (TOW) | Technician/Driver Email | Single Line | — | Also an "assigned to me" dashboard-matching candidate |
| `Technician_Emails` | Technician Emails | Single Line | — | Hidden; mirrors the assigned technician's email, will hold a comma-separated list once multi-request hand-off exists |
| `Service_Acceptance` (RSR) / `Service_Acceptance_For_Tow` (TOW) | Accept Time | Date-Time | — | |
| `Service_Reject_Time` | Reject Time | Date-Time | — | Shared by both branches |
| `ETA` (RSR) / `ETA_TOW` (TOW) | ETA (mins) | Number | — | Haversine stand-in for Google Maps Distance Matrix |
| `Distance_To_Breakdown` (RSR) / `Distance_To_Breakdown_TOW` (TOW) | Distance to Breakdown (km) | Number | — | |
| `Odometer_Reading` | Odometer Reading | Number | — | Reused for both the Step 5a start-odometer AND the Step 7b drop-side reading — one field, not two |
| `Navigation_Link` | Navigation Link | Multi-line | — | RSR |
| `Navigate_To_Breakdown_Link` | Navigate to Breakdown Link | Multi-line | — | TOW |
| `Image_Upload2` (RSR) / `Image_Upload5` (TOW) | Arrival / Cancel Photo | File Upload | — | Reused for both Arrival and Cancel on the same branch |
| `Cancel_Reason` (RSR) / `Reject_Reason2` (TOW) | Cancel Reason | Dropdown | *`REJECT_REASONS`* | |
| `Reach_Time` | Reach Time | Date-Time | — | Shared by both branches |
| `Roundtrip_Distance` | Round-Trip Distance (km) | Number | — | RSR's 3-leg total; TOW's 4-point total (Step 7b) |
| `Odometer_reading_at_Reached_Location` | Odometer at Reached | Number | — | TOW |
| `RSP_Start_Latitude` / `RSP_Start_Longitude` | RSP Start Latitude / Longitude | Single Line | — | **Text, not Decimal** — sidesteps a real "exceeded maximum digits" error raw GPS coordinates (up to 17 decimal places) hit on a Decimal field |
| `Cancellation_Time` | Cancellation Time | Date-Time | — | |
| `Cancel_Location_Lat` / `Cancel_Location_Lon` | Cancel Location Lat / Lon | Decimal | — | |
| `Cancel_Distance` | Cancel Distance (km) | Number | — | Distance from `RSP_Start_Latitude`/`Longitude` to `Cancel_Location_Lat`/`Lon` — deliberately a different field from `Distance_To_Breakdown` |
| `Toll_Charges` (Step 6a) / `Toll_Charges1` (Step 7b) | Toll Charges? | Radio | `Yes`, `No` | |
| `Issue_Resolved` | Issue Resolved? | Radio | `Yes`, `No` | RSR, required before Work Completed |
| `Unresolved_Note` | Issue Not Resolved — Notes | Multi-line | — | Shown when `Issue_Resolved = No` |
| `Image_Upload3` | Pre-service Photo | File Upload | — | RSR, min 2 / max 4 |
| `Image_Upload4` | Post-service Photo | File Upload | — | RSR, min 2 / max 4 |
| `Expense_Amount` | Expense Amount | Number | — | |
| `Expense_Photo` | Expense Photo | File Upload | — | Up to 2 |
| `Office_To_Mechanic` | Office → Mechanic (km) | Number | — | Part of RSR's 3-leg round trip and TOW's 4-point total |
| `Mechanic_To_Breakdown` | Mechanic → Breakdown (km) | Number | — | |
| `Breakdown_To_Office` | Breakdown → Office (km) | Number | — | RSR |
| `RSP_Completion_Time` | Completion Time | Date-Time | — | Shared by Work Completed, Service Reject, Cx-Reject and Dropped across every branch — one field |
| `Rejection_Reason` | Cx Rejection Reason | Dropdown | *`REJECT_REASONS`* | Shared by RSR's own Cx-reject and TOW Loading's own Cx-reject |
| `Pre_service_Photo` | Pre-service Photo | File Upload | — | TOW, 4 mandatory |
| `Image_Upload_On_truck` | On-truck Photo | File Upload | — | TOW, 3 mandatory |
| `VCRF` | VCRF | File Upload | — | TOW, 1 mandatory |
| `Image_Upload6` | Additional Photo | File Upload | — | |
| `Pickup_Time` | Pickup Time | Date-Time | — | TOW |
| `Breakdown_To_Drop_Distance` | Breakdown → Drop (km) | Number | — | Computed from live GPS at "Vehicle Picked," recomputed as part of the 4-point total at Step 7b |
| `Return_Journey_ETA` | Return ETA (mins) | Number | — | |
| `Drop_Location_Photo` | Drop Location Photo | File Upload | — | TOW, 4 mandatory |
| `Drop_To_Office_Distance` | Drop → Office (km) | Number | — | |
| `Drop_Location_Arrival_Time` | Drop Arrival Time | Date-Time | — | |
| `RSP_Drop_Location_Lat` / `RSP_Drop_Location_Lon` | RSP Drop Location Lat / Lon | Decimal | — | Field widget's own live GPS fix — separate from the customer's static `DropLocationLat`/`Long` |
| `Unloaded_Images` | Unloaded Photo | File Upload | — | 4 mandatory |
| `VCRF_Image` | VCRF Image | File Upload | — | 1 mandatory |
| `Handover_Image` | Handover Photo | File Upload | — | 1 mandatory |
| `Handover_to_Name` | Handover to (Name) | Single Line | — | |
| `Handover_to_Number` | Handover Number | Number | — | |
| `Handover_to_Designation` | Handover Location | Dropdown | `Home`, `Office`, `Work Shop` | |
| `Payment_Method_Final` | Payment Method | Dropdown | `Cash`, `UPI`, `Card`, `Net Banking`, default `Cash` | **Renamed 2026-08-03 from `Payment_Method1`** — a separate field from `Payment_Method` above (records how the *remaining* fee was paid, vs. the booking fee) |
| `Payment_received` | Payment Received? | Radio | `Yes`, `No` | |
| `QR_Code` | QR / Reference | Single Line | — | Vestigial — kept for a future real payment-gateway integration, no widget currently shows a QR block |
| `Send_Payment_Link1` | Send Payment Link | Checkbox | — | |
| `Upload_Photo_of_receipt` | Receipt Photo | File Upload | — | Mandatory for every payment method now (no more "unless Payment Gateway" carve-out) |
| `Refund_Due` | Refund Due? | Radio | `Yes`, `No`, default `No` | Final Closure |
| `Refund_Reason` | Refund Reason | Dropdown | *`REJECT_REASONS`* | |
| `Refund_Amount` | Refund Amount | Number | — | |
| `Closure_Status` | Closure Status | Dropdown | `Not Converted`, `Cancelled`, `Completed` | |
| `Closure_Reason` | Not Converted / Cancelled Reason | Dropdown | *`REJECT_REASONS`* | |
| `Cx_Feedback_Score` | Customer Feedback Score | Radio | `1`,`2`,`3`,`4`,`5`,`6`,`7`,`8`,`9`,`10` | |
| `Cx_Feedback_Text` | Customer Feedback Notes | Multi-line | — | |
| `B2B_GST_Invoice_Required` | B2B GST Invoice Required? | Radio | `Yes`, `No`, default `No` | |
| `GST_Number` | GST Number | Single Line | — | Shown when `B2B_GST_Invoice_Required = Yes` |
| `Closure_Extra_Photo` | Additional Photo | File Upload | — | |

**Dropped from the schema entirely (2026-08-03)** — do not create these; nothing in the current code writes or needs them:
`Assign_Technican`, `Assign_Technican1`, `Service_Request`, `Service_Request1`, `Action_field`, `Action_field2`, `Assignment_Sent`, `Total_Service_Fee1`. All were either display-only guesses nothing ever populated, or synthetic fields the widgets never actually wrote (they set `Status` directly instead) — see each widget's README changelog for the individual history.

## Form 2 — `Task_Rejections` (report: `Task_Rejections_Report`, or whatever default report name Zoho gives it — the widgets always pass the exact form name `Task_Rejections` to `addRecords`, so the report name itself doesn't matter for writes)

Best-effort rejection log, written by Technician/Driver/Vendor whenever they tap Reject. Non-blocking — a failed write here never blocks the actual rejection.

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `RSID` | RSID | Lookup → `Create_Case` | **Renamed 2026-08-03 from `Case_ID`**, to match `Invites`' own naming for the same "Lookup back to the ticket" concept. Sent as a bare record ID |
| `RSP_Name` | RSP Name | Single Line | The technician/driver/vendor's own display name |
| `Reason` | Reason | Single Line | Recommend Single Line, **not** a Dropdown, even though the app always sends one of the 5 `REJECT_REASONS` values — avoids a repeat of this session's "Invalid column value" class of bug on a field nothing lets a human mistype anyway |
| `Rejected_At` | Rejected At | Date-Time | |
| `Vehicle` | Vehicle | Lookup → `Vehicle_Master_Report` | Sent as the raw value read off the ticket's own `Vehicle` field |
| `Vehicle_Issue` | Vehicle Issue | Single Line | The ticket's `Vehicle_Issue` display string at the time of rejection |

## Form 3 — `Invites` (report: `Invites_Report`)

One record per invited vendor **and** per assigned technician/driver, per ticket — created by `getRescueTicket` (vendor invites, on Assignment-step save) and by `vendorTicket`'s own fleet hand-off (technician invites).

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `RSID` | RSID | Lookup → `Create_Case` | |
| `Vendor` | Vendor | Lookup → `Vendors_Report` | Populated only for a vendor invite |
| `Technician` | Technician | Lookup → `Technicians_Report` | **New 2026-08-03** — populated only for a technician/driver invite (fleet hand-off). A real bug is fixed by splitting this out: the field used to just be `Vendor` for both cases, but a Zoho Lookup can only target one form, so a technician's own `Technicians_Report` ID could never actually have resolved through a Lookup pointed at `Vendors_Report` |
| `Service_Acceptance_Next` | Service Acceptance Next | Single Line or Dropdown (`Yes`/`No`) | Written `"Yes"` on Accept (confirmed live — a boolean `true` was rejected) |
| `Status` | Status | Single Line | Written `"RSP REJECT"` on Reject, mirroring the ticket's own value |
| `Reject_Reason` | Reject Reason | Dropdown | *`REJECT_REASONS`* — written on Reject |

## Form 4 — `Vendors` (report: `Vendors_Report`)

**Unified 2026-08-03** — this single form now covers both what used to be a separate `vendors_Report` (the agent's Assignment-step picker/master data) and a separate `My_Availability_Vendor` (a vendor's own login/toggle profile). There is no real reason for a vendor to have two different records across two different forms; one Vendors master serves both purposes.

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `vendor_name` | Vendor Name | Single Line | — | **Corrected 2026-08-04, confirmed live via console dump**: the real field is lowercase `vendor_name` — the planned rename to `Vendor_Name` never actually landed on the real form (same class of surprise as `Vehicle_Issue`'s `VEHICLE_TYPE`). Display label everywhere |
| `Email` | Email | Single Line | — | Matches the logged-in vendor to their own record in `vendorTicket` |
| `Mobile_Number_01` | Phone | Single Line | — | **Corrected 2026-08-04, user-confirmed**: the real field is `Mobile_Number_01` (this app's own `Mobile_Number_NN` naming convention), not a plain `Phone` field |
| `Vendor_Priority` | Priority | Number | — | **Corrected 2026-08-04, confirmed live**: the real field is `Vendor_Priority`, not `Priority`. Lower = higher priority in the Assignment-step sort |
| `Availability_Status` | Availability Status | Dropdown | `Online`, `Offline` | Confirmed live — matches as originally specified |
| `Address` | Address | Address (Zoho's composite field type) | — | **Corrected 2026-08-04, confirmed live**: the vendor's own location is NOT flat `Latitude`/`Longitude` fields — it's `Address.latitude`/`Address.longitude`, nested inside Zoho's own composite Address field (which also carries `country`, `district_city`, `address_line_1`, etc.). The Assignment-step distance calc reads `Address.latitude`/`Address.longitude` first, falling back to flat `Latitude`/`Longitude` fields only if `Address` itself is absent |
| `Vendor_Type` | Vendor Type | Dropdown | `Individual`, `Fleet` | Individual vendors do the job themselves; Fleet vendors hand off to one of their own Technicians after accepting |

## Form 5 — `Technicians` (report: `Technicians_Report`)

Covers both independent technicians/drivers who log in directly, and technicians who belong to a Fleet vendor.

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `Technician_Name` | Technician Name | Single Line | — | Display label (**renamed from lowercase `technician_name`**) |
| `Email` | Email | Single Line | — | Matches the logged-in technician/driver to their own record |
| `Availability_Status` | Availability Status | Dropdown | `Online`, `Offline` | |
| `Fleet_Vendor` | Fleet Vendor | Single Line | — | This technician's owning fleet vendor, matched by name string (not a Lookup) in `vendorTicket`'s own hand-off panel |

## Form 6 — `Vehicle_Master` (report: `Vehicle_Master_Report`)

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `Name` | Name | Single Line | — | Display label (make/model) |
| `Category` | Category | Dropdown (single-select) | `2W`, `4W` | Matched against `Vehicle_Issue_Report.VEHICLE_TYPE` |
| `Segment` | Segment | Dropdown | `A`–`F` (or whatever the real fee-rate segments are) | Used in the Rate_Master fee lookup |

## Form 7 — `Vehicle_Issue` (report: `Vehicle_Issue_Report`)

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `Issue_Name` | Issue Name | Single Line | — | Display label |
| `VEHICLE_TYPE` | Vehicle Type | Multi Select | `2W`, `4W` | **Corrected 2026-08-04, user-confirmed against real Zoho Studio schema**: the field is `VEHICLE_TYPE` (all-caps) — the 2026-08-03 note claiming a rename to `Vehicle_Category` was wrong (either never landed or was reverted); which vehicle categories this issue applies to, matched against `Vehicle_Master_Report.Category` |
| `DISPLAY_STATUS` | Display Status | Dropdown | `SHOW` (and presumably a hidden counterpart) | **New 2026-08-04, user-confirmed.** Only issues with `DISPLAY_STATUS = "SHOW"` are offered in the Vehicle Issue picker, on top of the `VEHICLE_TYPE` category match |
| `Issue_Type` | Issue Type | Dropdown | `REPAIR`, `TOW` (and any others the real catalog needs) | Contains `TOW` → drives the ticket's own `Service_Type` |

## Form 8 — `Client` (report: `Client_Report`)

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `Client_Name` | Client Name | Single Line | Display label |

## Form 9 — `Rate_Master` (report: `Rate_Master_Report`)

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `Vehicle_Type` | Vehicle Type | Dropdown | `2W`, `4W` | Matched against the ticket's own vehicle `Category` |
| `Vehicle_Segment` | Vehicle Segment | Dropdown | `A`–`F` | Matched against `Vehicle_Master.Segment` |
| `Issue` | Issue | Single Line | — | Matched against the chosen `Vehicle_Issue.Issue_Name` |
| `Rate_Type` | Rate Type | Dropdown | `FIXED`, `FORMULA` | |
| `Fixed_Rate` | Fixed Rate | Number | — | Used when `Rate_Type = FIXED` |
| `Formula_Base_Amount` | Formula Base Amount | Number | — | Used when `Rate_Type = FORMULA` |
| `Formula_KM_Buffer` | Formula KM Buffer | Number | — | |
| `Formula_Per_KM_Rate` | Formula Per KM Rate | Number | — | |
| `Formula_Min_Amount` | Formula Min Amount | Number | — | |

---

# PART 2 — Reports to create (List View / Detail View)

Zoho Creator lets you pick which fields show in a report's List View (the row/grid you scan) vs. its Detail View (opened record, shows everything by default — you generally don't need to trim this down). Recommendations below are for List View; Detail View can safely just show "all fields" in every case.

| Report | Built from | Recommended List View columns |
|---|---|---|
| `Agent_Ticket_Report` | `Create_Case` | `Case_ID`, `Customer_Name`, `Phone_Number1`, `Vehicle`, `Service_Type`, `Status`, `Client`, `Time_of_service` |
| `Vehicle_Master_Report` | `Vehicle_Master` | `Name`, `Category`, `Segment` |
| `Vehicle_Issue_Report` | `Vehicle_Issue` | `Issue_Name`, `VEHICLE_TYPE`, `DISPLAY_STATUS`, `Issue_Type` |
| `Client_Report` | `Client` | `Client_Name` |
| `Vendors_Report` | `Vendors` | `Vendor_Name`, `Phone`, `Priority`, `Availability_Status`, `Vendor_Type` |
| `Technicians_Report` | `Technicians` | `Technician_Name`, `Email`, `Availability_Status`, `Fleet_Vendor` |
| `Rate_Master_Report` | `Rate_Master` | `Vehicle_Type`, `Vehicle_Segment`, `Issue`, `Rate_Type` |
| `Task_Rejections` (default report) | `Task_Rejections` | `RSID`, `RSP_Name`, `Reason`, `Rejected_At` |
| `Invites_Report` | `Invites` | `RSID`, `Vendor`, `Technician`, `Service_Acceptance_Next`, `Status` |

**Access note**: which profile needs View vs. View+Edit vs. Add on each of these is tracked separately in `ACCESS.md` (same folder) — keep both files in sync when a report's access requirements change.

---

# PART 3 — Narrative reference (why each field exists, by workflow step)

The tables below repeat much of Part 1's information but organized by the ticket's real lifecycle (Action 1 → Final Closure) instead of by form — useful for understanding *why* a field exists and what mock/decision it traces back to. Part 1 is the one to build from; this part is context.

## Step 1 — Create Service Ticket (Agent)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Customer_Name` | Customer Name | text | Agent | ✅ |
| `Phone_Number1` | Phone Number (shown on form) | tel | Agent | ✅ |
| `Phone_Number` | Phone Number (real required field, mirrored from `Phone_Number1`) | tel | Agent | ✅ |
| `Alternate_Phone_Number` | Alternate Phone Number | tel | Agent | ❓ (re-added 2026-07-31, name carried over from before its 2026-07-25 removal, not re-confirmed) |
| `Email` | Email | email | Agent | 🟡 |
| `Client` | Client (Lookup → `Client_Report`) | select | Agent | ✅ |
| `Client_Tracking_Id` | Client Tracking ID | text | Agent | 🟡 |
| `Vehicle` | Vehicle (Lookup → `Vehicle_Master_Report`) | select | Agent | ✅ |
| `Vehicle_Registration_Number` | Vehicle Reg. Number | text | Agent | 🟡 |
| `Vehicle_Issue` | Vehicle Issue (plain multiselect text, not a Lookup) | multiselect | Agent | ✅ |
| `Service_Type` | Service Type ("RSR"/"TOW", hidden, derived from `Vehicle_Issue`) | badge | Agent, Technician, Driver, Vendor | ✅ |
| `Vehicle_Status` | Vehicle Status (`ONROAD`/`SAFE PARKING`) | select | Agent | 🟡 |
| `Time_of_service` | Time of Service (`NOW`/`LATER`) | radio | Agent | 🟡 |
| `Schedule_Date` | Schedule Date (when `LATER`) | date | Agent | 🟡 |
| `Service_Time` | Service Time slot (when `LATER`) | select | Agent | 🟡 |
| `Computed_Service_Time` | Computed service time (NOW = +30min, LATER = Schedule_Date+Service_Time), 12-hour+AM/PM+seconds format | (hidden, `extra()`) | Agent | ✅ (format confirmed live) |
| `Lead_Source` | Always `"Call"` for this agent-facing widget | (hidden, `extra()`) | Agent | ✅ |
| `Remarks` | Remarks (optional, editable at multiple steps — see Steps 2/3/4 too) | remarks (append-only) | Agent | 🟡 |
| `Status` | The ticket's current stage — see the Status Values table at the bottom of this file | (hidden) | all | ✅ (picklist itself confirmed; not every value) |
| `Case_ID` | Human-readable case ID (`"RSID"` + last 6 digits of the record's own Zoho ID) | (hidden) | all | ✅ (scheme), 🟡 (whether Zoho ever rejects the override — see `getRescueTicket/README.md` §4) |

## Step 2 — Service Fee Quote & Booking Fee Collection (Agent)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Distance_For_Quote_RT_KM` | Distance for Quote (RT KM), TOW only | number | Agent | 🟡 |
| `Suggested_Rate` | Suggested Rate (readonly, computed) | number | Agent | ✅ (computation logic confirmed via native Deluge) |
| `Total_Service_Fee` | Total Service Fee | number | Agent | ✅ |
| `Booking_Fee` | Booking Fee | number | Agent | ✅ |
| `Remaining_Fee` | Remaining Fee (= Total − Booking here; re-appears in Step 8) | decimal | Agent, Technician, Driver, Vendor | 🟡 |
| `Rate_Breakup` | Rate Breakup (editable text) | textarea | Agent | ✅ |
| `Send_Link_For_The_Breakdown_Location` | Bundled "send payment link + request location" action (fee>0 case) — same field as Step 3's own copy | button | Agent | 🟡 |
| `Payment_Method` | Payment Method — a different field from Step 8's `Payment_Method_Final` | select | Agent | ❓ (kept as-is 2026-07-31 despite a flow spec showing a different list — user-confirmed) |
| `Payment_Status` | Payment Status (`PAID`/`PENDING`/`NOT APPLICABLE`) — **confirmed 2026-08-03 via the live Zoho field config**: this is the ONE real field, also used (same values) at Step 8's own Payment screen | select | Agent, Technician, Driver, Vendor | ✅ (confirmed live 2026-08-03) |
| `Image_Upload` | Receipt Photo (locked/readonly here) | file | Agent | 🟡 |

**Dropped 2026-08-03**: `Total_Service_Fee1` ("likely-duplicate, relationship unconfirmed") — the Step 8 payment screen now just displays `Total_Service_Fee` read-only instead of a second field.

## Step 3 — Capture Locations (Agent)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Send_Link_For_The_Breakdown_Location` | Send Breakdown Location Link (same field as Step 2's copy) | button | Agent | 🟡 |
| `Latitude` / `Longitude` | Breakdown coordinates (pasted as text, split via `parseTarget`) | decimal | Agent, Technician, Driver, Vendor (read) | ✅ |
| `Send_Link_For_The_Drop_Location` | Send Drop Location Link, TOW only | button | Agent | 🟡 |
| `DropLocationLat` / `DropLocationLong` | Drop coordinates, TOW only | decimal | Agent, Driver, Vendor (read) | 🟡 |
| `RT_KM` | Round-Trip KM, TOW only (agent-side Haversine calc) | number | Agent | 🟡 |
| `Location_Received_Time` | Timestamp the breakdown location was first captured — stamped once, never overwritten | (hidden, `extra()`) | Agent | ❓ (field name/type user-confirmed 2026-07-31, not independently re-verified live) |
| `Break_Down_Location1` | Display/navigation field the field-facing widgets link to (falls back to a `maps.google.com` link built from `Latitude`/`Longitude` if empty) | (display only) | Technician, Driver, Vendor | ❓ (not confirmed whether this is a real formula field or something else) |
| `Navigate_To_Drop_Link` | Drop-location display/navigation field, TOW only | (display only) | Technician, Driver, Vendor | ❓ |

## Step 4 — Vendor / Mechanic / Driver Assignment (Agent)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Preferred_Vendor` | Preferred Vendor? (Yes = show all vendors; No/default = geofenced) | radio | Agent | ✅ |
| `Vendors1` | Vendors (multi-Lookup → `Vendors_Report`) — saves real record IDs (`useId:true`, via `idsFor()`), bare (not object-wrapped — an object-wrapping experiment was tried and reverted the same day, broke saving entirely) | Lookup (bare ID) | Agent, Vendor (read-only, dashboard matching only) | ✅ (Lookup type confirmed live 2026-07-31; bare-ID write shape confirmed live 2026-07-31) |
| `Assigned_Vendor` | A single Lookup → `Vendors_Report`, set by the vendor's own Accept action once they accept the ticket (distinct from `Vendors1`, which only holds the invited *candidates*). Displayed read-only in `getRescueTicket`'s `acceptance`/`acceptanceTow` stages. **Write-side implemented 2026-08-04** — `vendorTicket`'s `acceptService()` now sends `currentVendorRecord.id` (unblocked by the schema rebuild merging `My_Availability_Vendor` into `Vendors_Report`, so vendorTicket now always has its own `Vendors_Report` ID on hand). Also now read back by `vendorTicket`'s own `loadDashboard()` to implement "1st person to accept, task goes away from other invited vendors' apps" (Action 4 mock) — falls back to the old "show any invited candidate" behavior for any ticket accepted before this existed (no `Assigned_Vendor` recorded yet) | Lookup (single) | Agent (view-only), Vendor (write on Accept, read in dashboard filtering) | 🟡 (write shape follows the same bare-ID convention as every other Lookup here; best-effort only — two vendors accepting within the same instant could both still write it, no server-side lock available from a plain widget) |
| `Vendor_Emails` | Auto-populated (comma-separated) with the email of every vendor selected in `Vendors1`, resolved from `Vendors_Report` via `CONFIG.vendorEmailField` — not shown anywhere in this widget's own UI. Used by `vendorTicket`'s own dashboard matching as a more reliable alternative to name-matching | text (hidden) | Agent (write, silent) | 🟡 |
| `Vendor_Email` | Vendor Email — shown once someone's actually accepted, not something the agent fills in upfront | text | Agent (view-only, Acceptance step) | 🟡 |
| `Assigned_Technician` | Assigned Technician | select | Agent (view-only, Acceptance step), Vendor (write, on fleet hand-off) | 🟡 |
| `Assigned_Technician_Email` | Technician Email | text | Agent (view-only, Acceptance step), Technician, Driver, Vendor | ❓ (dashboard-matching candidate, not confirmed which field a given ticket actually uses) |

## Step 5 / 5a — Service Acceptance & Rejection (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Reject_Reason` | Reject Reason (RSR) | select | Agent (view-only), Technician, Vendor | 🟡 |
| `Reject_Reason1` | Reject Reason (TOW) | select | Agent (view-only), Driver, Vendor | 🟡 |
| `Vendor_Email1` | Vendor Email (Acceptance step's own copy, RSR) | text | Agent (view-only) | 🟡 |
| `Vendor_Email2` | Vendor Email (Acceptance step's own copy, TOW) | text | Agent (view-only) | 🟡 |
| `Technician_Email` | Technician Email (RSR) — also a dashboard-matching candidate | text | Agent (view-only), Technician | 🟡 |
| `Technician_Email1` | Driver Email (TOW) — also a dashboard-matching candidate | text | Agent (view-only), Driver | 🟡 |
| `Technician_Emails` | Written by `vendorTicket`'s fleet hand-off (`assignTechnician()`) alongside `Assigned_Technician_Email` — currently just mirrors that one technician's email, will hold a comma-separated list once the fleet "request several, whoever accepts" flow exists. Also checked by `technicianTicket`/`driverTicket`'s own dashboard matching | text (hidden) | Vendor (write), Technician, Driver (dashboard-matching read) | 🟡 |
| `Service_Acceptance` | Accept timestamp (RSR) | datetime | Technician, Vendor | ❓ (from flow-generation spec, not re-confirmed) |
| `Service_Acceptance_For_Tow` | Accept timestamp (TOW) | datetime | Driver, Vendor | ❓ |
| `Service_Reject_Time` | Reject timestamp (both branches) | datetime | Technician, Driver, Vendor | 🟡 (from the Action 5 flow spec) |
| `ETA` | ETA in minutes to breakdown, RSR (Haversine stand-in for Google Maps Distance Matrix API) | number | Agent (view-only), Technician, Vendor | 🟡 |
| `Distance_To_Breakdown` | Distance to breakdown in km, RSR | number | Agent (view-only), Technician, Vendor | 🟡 |
| `ETA_TOW` | ETA in minutes to breakdown, TOW | number | Driver, Vendor | ❓ |
| `Distance_To_Breakdown_TOW` | Distance to breakdown in km, TOW | number | Driver, Vendor | ❓ |
| `Odometer_Reading` | Start odometer (TOW, captured at Accept) — **same field name reused at Step 7b for the drop-side odometer reading; not two separate fields** | number | Driver, Vendor | ❓ |
| `Navigate_To_Drop_Link` | Drop location display (see Step 3) | (display only) | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.RSP_Name` / `.Reason` / `.Rejected_At` / `.Vehicle_Issue` | Rejection-log fields — best-effort, non-blocking write | text/datetime | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.RSID` | **Renamed 2026-08-03 from `Task_Rejections.Case_ID`** to match `Invites`' own naming — a real Lookup field, sends the ticket's own record ID (`r.ID`) | Lookup (bare ID) | Technician, Driver, Vendor | 🟡 |
| `Task_Rejections.Vehicle` | Also a real Lookup field — sends the raw `r[CONFIG.vehicleField]` value as fetched from `Create_Case` (unconverted) | Lookup (bare, unconverted) | Technician, Driver, Vendor | ❓ (needs live verification) |

**Dropped 2026-08-03**: `Service_Request`, `Service_Request1`, `Assign_Technican`, `Assign_Technican1` — synthetic/display-only fields no widget ever actually wrote (they set `Status` directly instead).

## Step 6 / 6a — Reach Breakdown Location & Cancel (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Navigation_Link` | Navigation Link (RSR, free text) | textarea | Agent (view-only) | 🟡 |
| `Navigate_To_Breakdown_Link` | Navigate to Breakdown Link (TOW) | textarea | Agent (view-only) | 🟡 |
| `Image_Upload2` | Arrival Photo (RSR) / also reused as the RSR Cancel photo | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Image_Upload5` | Arrival Photo (TOW) / also reused as the TOW Cancel photo | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Cancel_Reason` | Cancel Reason (RSR) | select | Agent (view-only), Technician, Vendor | 🟡 |
| `Reject_Reason2` | Reject/Cancel Reason (TOW Reach step) | select | Agent (view-only), Driver, Vendor | 🟡 |
| `Reach_Time` | Reach/arrival timestamp (both branches) | datetime | Technician, Driver, Vendor | ❓ |
| `Roundtrip_Distance` | Round-trip distance in km — set here for RSR (office→breakdown→office Haversine), set again at Step 7b for TOW's 4-point version | number | Agent (view-only), Technician, Driver, Vendor | 🟡 |
| `Odometer_reading_at_Reached_Location` | Odometer at Reached (TOW) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `RSP_Start_Latitude` / `RSP_Start_Longitude` | The tech/driver/vendor's own GPS position at Accept time (Step 5), used to compute the Step 6 Cancel distance. Text (not Decimal) to sidestep a real "exceeded maximum digits" error on raw GPS coordinates | text | Technician, Driver, Vendor | 🟡 (user-renamed 2026-08-03) |
| `Cancellation_Time` | Timestamp of the Cancel Task click | datetime | Technician, Driver, Vendor | ❓ |
| `Cancel_Location_Lat` / `Cancel_Location_Lon` | GPS position at the moment Cancel is confirmed | decimal | Technician, Driver, Vendor | ❓ |
| `Cancel_Distance` | Distance in km from `RSP_Start_Latitude`/`Longitude` to `Cancel_Location_Lat`/`Lon` — deliberately a different field from Step 5's own `Distance_To_Breakdown` | number | Technician, Driver, Vendor | ❓ |
| `Toll_Charges` | Toll Charges? (TOW Reach step) | radio | Agent (view-only), Driver, Vendor | 🟡 |

## Step 7 — Work In Progress (Technician, RSR only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Issue_Resolved` | Issue Resolved? — required (Technician/Vendor validate it before "Work Completed" proceeds) | radio | Agent (view-only), Technician, Vendor | 🟡 |
| `Unresolved_Note` | Free-text note when Issue Resolved = No — plain-text stand-in for the mock's own "add voice note" idea; not real audio capture | textarea | Technician, Vendor | ❓ (guessed) |
| `Image_Upload3` | Pre-service Photo(s) — min 2, max 4 | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Image_Upload4` | Post-service Photo(s) — min 2, max 4 | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Expense_Amount` | Expense Amount | number | Agent (view-only), Technician, Vendor | 🟡 |
| `Expense_Photo` | Expense Photo(s) — up to 2 | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Office_To_Mechanic` | Office → Mechanic leg (km) — RSR's own 3-leg round-trip | number | Agent (view-only) | 🟡 |
| `Mechanic_To_Breakdown` | Mechanic → Breakdown leg (km) | number | Agent (view-only) | 🟡 |
| `Breakdown_To_Office` | Breakdown → Office leg (km) | number | Agent (view-only) | 🟡 |
| `RSP_Completion_Time` | Completion timestamp — shared by both "Work Completed" and "Service Reject" | datetime | Technician, Vendor | ❓ |
| `Rejection_Reason` | Customer-reject reason (Action 7's own CX-reject branch) — the same field reused at Step 7a's own Cx-reject branch (TOW Loading), not a distinct per-branch field | select | Technician, Driver, Vendor | 🟡 |

## Step 7a — WIP: Loading (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Pre_service_Photo` | Pre-service Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Image_Upload_On_truck` | On-truck Photos — 3 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `VCRF` | VCRF Form upload — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Image_Upload6` | Additional Photo (agent-side field only) | file | Agent (view-only) | 🟡 |
| `Rejection_Reason` | Cx-reject reason at this step — this stage's alternate outcome is "Customer Rejected" → `Status:"REMAINING FEE DUE"` (same as Step 7's own Cx-reject), sharing the one `Rejection_Reason` field above | select | Technician, Driver, Vendor | 🟡 |
| `RSP_Completion_Time` | Shared completion timestamp, same field as Step 7 — stamped by this stage's own "Service Reject" too | datetime | Technician, Driver, Vendor | ❓ |
| `Pickup_Time` | Vehicle-picked timestamp | datetime | Driver, Vendor | ❓ |
| `Breakdown_To_Drop_Distance` | Breakdown → Drop leg (km) — computed from the driver's own live GPS fix at the moment "Vehicle Picked" is tapped (falling back to the ticket's static breakdown Lat/Lon if GPS is unavailable) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Return_Journey_ETA` | Return ETA in minutes — same live-GPS-origin as `Breakdown_To_Drop_Distance` above | number | Agent (view-only), Driver, Vendor | 🟡 |

## Step 7b — WIP: Reached Drop Location (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Odometer_Reading` | Odometer reading at drop (same field name as Step 5/5a's start-odometer) — a plain number, not a photo upload | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Drop_Location_Photo` | Drop Location Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Toll_Charges1` | Toll Charges? (drop side) | radio | Agent (view-only), Driver, Vendor | 🟡 |
| `Office_To_Mechanic` / `Mechanic_To_Breakdown` / `Breakdown_To_Drop_Distance` / `Drop_To_Office_Distance` | The 4-point roundtrip's individual legs — really a 3-leg Office→Breakdown→Drop→Office trip (no separate "Mechanic" waypoint exists), `Mechanic_To_Breakdown` is hardcoded `0` — field *names* kept as-is to avoid an unnecessary rename of already-shipped, cross-referenced fields | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Roundtrip_Distance` | Total of the 4 legs | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Drop_Location_Arrival_Time` | Timestamp when "Reached Drop Location" is tapped | datetime | Agent (view-only), Driver, Vendor | ❓ (guessed name) |
| `RSP_Drop_Location_Lat` / `RSP_Drop_Location_Lon` | The driver/vendor's own live GPS fix at that same click — separate from the ticket's static `DropLocationLat`/`DropLocationLong` (the customer-specified drop point, still used for the roundtrip calc) | decimal | Agent (view-only), Driver, Vendor | ❓ (guessed name) |

## Step 7c — WIP: Unloading & Handover (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Unloaded_Images` | Unloaded Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `VCRF_Image` | VCRF Image — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_Image` | Handover Photo — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_to_Name` | Handover to (Name) | text | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_to_Number` | Handover Number — plain number input | number | Agent (view-only), Driver, Vendor | 🟡 |
| `RSP_Completion_Time` | Shared completion timestamp, same field as Step 7/7a — "Capture Time of click of 'DROPPED' button as the 'work complete time'" | datetime | Agent (view-only), Technician, Driver, Vendor | ❓ |
| `Handover_to_Designation` | Handover Location (`Home`/`Office`/`Work Shop`) | select | Agent (view-only), Driver, Vendor | 🟡 |

## Step 8 — Final Payment Collection (Technician / Driver / Vendor, shared)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Total_Service_Fee` | Total Service Fee (readonly display here — same field as Step 2's own copy; the old separate `Total_Service_Fee1` was dropped 2026-08-03) | number | Agent, Technician, Driver, Vendor | ✅ |
| `Remaining_Fee` | Remaining Fee (readonly display here — same field as Step 2's own copy) | decimal | Agent, Technician, Driver, Vendor | 🟡 |
| `Payment_Method_Final` | Payment Method — same option list as Step 2's own `Payment_Method`, but tracked as a separate field. **Renamed 2026-08-03 from `Payment_Method1`** | select | Agent, Technician, Driver, Vendor | ✅ — options `["Cash","UPI","Card","Net Banking"]`, default `"Cash"`, confirmed 2026-08-03 via user's own Zoho field screenshot |
| `Payment_received` | Payment Received? | radio | Agent, Technician, Driver, Vendor | 🟡 |
| `QR_Code` | QR / Reference | text | Agent | 🟡 (vestigial now that `Payment_Method_Final` has no "Payment Gateway" option — no widget shows a QR block anymore) |
| `Send_Payment_Link1` | Send payment link | check | Agent | 🟡 |
| `Upload_Photo_of_receipt` | Receipt Photo — mandatory for every Payment Method now — both a live-camera capture (GPS/time watermark) and a plain gallery picker are offered | file | Agent, Technician, Driver, Vendor | 🟡 |
| `Payment_Status` | The SAME field as Step 2's own `Payment_Status` (`PAID`/`PENDING`/`NOT APPLICABLE`) — not a separate `Pending`/`Success` field as originally guessed. Technician/Driver/Vendor auto-flip it to `PAID` once a receipt photo is attached, or read it as already `PAID` if some other real integration set it; "Payment Received" is only enabled once this is `PAID` or the balance is zero | select | Agent, Technician, Driver, Vendor | ✅ (confirmed live 2026-08-03) |
| `Transaction_ID` / `Time_Of_Receipt` | The mock's own "transaction ID, time of receipt" once a real Payment Gateway returns success — **not implemented**, no real gateway webhook exists to source these from honestly | text/datetime | — | ❓ (not set by any widget yet) |

## Final Closure (Agent only)

Reachable at any point in a ticket's timeline (dashboard row action + a permanent wizard button), not gated by the stepper. Every field from every step above is also shown here (read-only display, or click-to-edit — see `getRescueTicket/README.md`'s Final Closure entries for the full mechanism). Fields unique to this screen:

**"Distance & ETA Metrics" group (added 2026-08-04, per user request)**: `ETA`, `Distance_To_Breakdown`, `ETA_TOW`, `Distance_To_Breakdown_TOW`, `Cancel_Distance`, `Office_To_Mechanic`, `Mechanic_To_Breakdown`, `Breakdown_To_Office`, `Roundtrip_Distance`, `Breakdown_To_Drop_Distance`, `Drop_To_Office_Distance`, `Return_Journey_ETA` — all removed from their previous per-stage wizard display (`reach`/`wip`/`wipUnloadingTow`) and shown **only** in this dedicated, read-only Final Closure group, not tied to any wizard step. These are all Google Maps/Haversine-computed by the vendor/technician/driver widgets, not agent input — see each field's own Step-N row above for where it's written.

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Refund_Due` | Refund Due? | radio | Agent | ❓ |
| `Refund_Reason` | Refund Reason | select | Agent | ❓ |
| `Refund_Amount` | Refund Amount | number | Agent | ❓ |
| `Closure_Status` | Closure Status (`Not Converted`/`Cancelled`/`Completed`) | select | Agent | ❓ |
| `Closure_Reason` | Not Converted / Cancelled Reason | select | Agent | ❓ |
| `Cx_Feedback_Score` | Customer Feedback score (1–10) | radio | Agent | ❓ |
| `Cx_Feedback_Text` | Customer Feedback notes | textarea | Agent | ❓ |
| `B2B_GST_Invoice_Required` | B2B GST Invoice Required? | radio | Agent | ❓ |
| `GST_Number` | GST Number | text | Agent | ❓ |
| `Closure_Extra_Photo` | Additional Photo | file | Agent | ❓ |

None of these ten have ever been confirmed against a real `Create_Case` record — verify each in Zoho Studio and correct here first if any are wrong.

---

## Cross-cutting: master-table fields (not on `Create_Case`)

### `Vendors_Report` (vendor master — unified 2026-08-03, absorbs the old separate `My_Availability_Vendor`)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `vendor_name` | Display label — **corrected 2026-08-04, confirmed live**: real field is lowercase, the planned `Vendor_Name` rename never landed | Agent, Vendor | ✅ (confirmed live 2026-08-04) |
| `Email` | Match to logged-in vendor | Vendor | ✅ (mechanism), 🟡 (exact field) |
| `Mobile_Number_01` | Phone — **corrected 2026-08-04, user-confirmed**: real field is `Mobile_Number_01`, not `Phone` | Agent | ✅ (confirmed live 2026-08-04) |
| `Vendor_Priority` | Assignment sort priority — **corrected 2026-08-04, confirmed live**: real field is `Vendor_Priority`, not `Priority` | Agent | ✅ (confirmed live 2026-08-04) |
| `Availability_Status` | `Online`/`Offline` | Agent, Vendor | ✅ (confirmed live 2026-08-04) |
| `Address` (composite) | Vendor's own location — **corrected 2026-08-04, confirmed live**: NOT flat `Latitude`/`Longitude` fields, lives nested as `Address.latitude`/`Address.longitude` inside Zoho's own composite Address field type | Agent | ✅ (confirmed live 2026-08-04) |
| `Vendor_Type` | Individual vs. Fleet | Vendor | 🟡 (drives `IS_FLEET`) |

### `Technicians_Report` (technician/driver master)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `Technician_Name` | Display name (renamed from lowercase `technician_name`) | Technician, Driver, Vendor | 🟡 |
| `Email` | Match to logged-in user | Technician, Driver, Vendor (fleet hand-off) | 🟡 |
| `Availability_Status` | `Online`/`Offline` | Technician, Driver | 🟡 |
| `Fleet_Vendor` | Links a technician back to their owning fleet vendor (matched by name string, not ID) | Vendor | ❓ (guessed) |

### `Vehicle_Master_Report`
| Field | Purpose | Status |
|---|---|---|
| `Name` | Display label | ✅ |
| `Category` | 2W/4W | ✅ |
| `Segment` | A–F, used in fee calc | ✅ |

### `Vehicle_Issue_Report`
| Field | Purpose | Status |
|---|---|---|
| `Issue_Name` | Display label | ✅ |
| `VEHICLE_TYPE` | Multi Select — matched against vehicle's `Category`. **Corrected 2026-08-04**: the 2026-08-03 note claiming this was renamed to `Vehicle_Category` was wrong — `VEHICLE_TYPE` is the real, current field name, user-confirmed against live Zoho Studio | ✅ |
| `DISPLAY_STATUS` | Only issues with value `SHOW` are offered in the picker — new 2026-08-04, user-confirmed | ✅ |
| `Issue_Type` | Contains `TOW` → drives `Service_Type` | ✅ |

### `Rate_Master_Report`
| Field | Purpose | Status |
|---|---|---|
| `Vehicle_Type`, `Vehicle_Segment`, `Issue`, `Rate_Type`, `Fixed_Rate`, `Formula_Base_Amount`, `Formula_KM_Buffer`, `Formula_Per_KM_Rate`, `Formula_Min_Amount` | Fee calculation | ✅ (fields, ported from real Deluge) / 🟡 (report name itself) |

### `Client_Report`
| Field | Purpose | Status |
|---|---|---|
| `Client_Name` | Display label | ✅ |

---

## Status values written to `Status` (Create_Case) — step-by-step

| Value | Written by (step) | Resumes at (reopening a ticket) |
|---|---|---|
| `SERVICE FEE QUOTE` | Step 1 (Create Case, on save, when `Time_of_service = NOW`) | Step 2 |
| `SCHEDULED` | Step 1 (Create Case, on save, when `Time_of_service = LATER`) — **added 2026-08-04** | Step 2 |
| `BOOKING FEE PENDING` | Step 2 | Step 2 |
| `LOCATION REQUEST` | Step 2 (fee-free, link not yet sent) or Step 3 (own default) | Step 3 |
| `LOCATION PENDING` | Step 2 (once link sent / paid) | Step 3 |
| `READY FOR ASSIGNMENT` | Step 3 (once a breakdown location exists) | Step 4 |
| `RSP IRA (INVITE RESPONSE AWAITED)` | Step 4 | Step 4 |
| `RSP REJECT` | Step 5/5a | Step 4 (re-assign) |
| `RSP ON THE WAY` | Step 5/5a (Accept) | Step 6/6a |
| `RSP CANCELLED` | Step 6/6a (Cancel) | — (terminal-ish, not currently resumed anywhere specific) |
| `WORK IN PROGRESS` | Step 6 (RSR Reached) | Step 7 |
| `LOADING` | Step 6a (TOW Reached) | Step 7a |
| `TO DROP LOCATION` | Step 7a (Vehicle Picked) | Step 7b |
| `UNLOADING` | Step 7b (Reached Drop) | Step 7c |
| `REMAINING FEE DUE` | Step 7 (Work Completed / CX Reject) or Step 7c (Dropped) | Step 8 |
| `RSP CLOSED` | Step 8 (Payment Received) | — (closed; Final Closure can still reopen it) |
| `CLOSED` | Final Closure (Close Case) | — (closed) |

---

## Open questions this file doesn't resolve on its own

See `getRescueTicket/QUESTIONS_TO_ASK.md` for the current live list (Individual vs. Fleet field, the "Mechanic" waypoint, App Status vs. Login Status, and others) — not duplicated here to avoid two copies drifting apart. Fix the field name **here** once an answer comes in, then that question can be checked off there.
