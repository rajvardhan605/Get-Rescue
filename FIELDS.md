# Get-Rescue — Field Registry (single source of truth)

This file is the **one place** every Zoho field API/link name used anywhere in this repo is listed. Going forward:

- **Before using a field name in code, check here first.** If a name here conflicts with what's already in a widget's code, this file wins — update the code to match.
- **To correct a wrong field name, edit it in this file only.** The correction then gets carried into whichever widget(s) use that field the next time that widget is touched (or immediately, if you ask for it).
- **Every new field any future feature needs gets added here too**, in the right step section, at the same time it's added to the actual widget code — not just left living in a code comment.
- Organized **step-by-step**, matching the ticket's real lifecycle (Action 1 → Final Closure), not alphabetically — read it top to bottom in the order a ticket actually moves through it.

**Status column key:**
- ✅ **Confirmed** — verified against a real Zoho record or a real Zoho API error message.
- 🟡 **From spec** — given directly by a client-provided flow spec/screenshot, not independently re-checked in Zoho Studio.
- ❓ **Guessed** — a pattern-matched name with no direct source; most likely to need correcting.

**Widget column** shows every widget that reads or writes that field: **Agent** (`getRescueTicket`), **Kanban** (`ticketKanbanGetRescue` — shares the Agent's own field set for its dashboard/edit-resume wizard, not listed separately below unless it uses something extra), **Technician** (`technicianTicket`), **Driver** (`driverTicket`), **Vendor** (`vendorTicket`), **Toggle** (`toggleGetRescue`, superseded by the toggle now built into Technician/Driver/Vendor directly — kept for reference).

---

## Report / Form reference

| Report or Form | Purpose |
|---|---|
| `Create_Case` (form) / `Agent_Ticket_Report` (report) | The single ticket record — every field below that isn't explicitly a master-table field lives here. All profiles read/write through `Agent_Ticket_Report`. |
| `vehicle_master_Report` | Vehicle master (make/model, `Category`, `Segment`) |
| `Vehicle_Issue_Report` | Issue master (`Issue_Name`, `VEHICLE_TYPE`, `Issue_Type`) |
| `Client_Report` | Client master (`Client_Name`) |
| `vendors_Report` | Vendor master (name, phone, priority, online status, location, Individual/Fleet type) |
| `technicians_Report` | Technician/Driver master (name, email, online status, owning fleet vendor) |
| `My_Availability_Vendor` | A vendor's own online/offline record (separate from `vendors_Report`) |
| `Rate_Master_Report` | Fee calculation rules (🟡 report name itself not confirmed live) |
| `Task_Rejections` (form) | Best-effort rejection log written by Technician/Driver/Vendor on Reject. Built by the user 2026-07-31 with fields: `Case_ID` (Lookup → `Create_Case`), `RSP_Name` (Single Line), `Reason` (Single Line), `Rejected_At` (Date-Time), `Vehicle` (Lookup → the same vehicle-master form `Create_Case`'s own Vehicle field points to), `Vehicle_Issue` (Single Line). See Step 5/5a below for the exact payload and the Lookup-field write caveat |
| `Invites` (form) / `Invites_Report` (report) | **New 2026-08-03, per the user.** One record per invited vendor (and, per "invite covers all", per assigned technician/driver too) per ticket — reverses the earlier "Create_Case only" decision. `RSID` (Lookup → `Create_Case`) and `Vendor` (Lookup → whoever's invited) are both real Lookups, sent as bare IDs. Created by `getRescueTicket` on Assignment-step save (`syncInvites()`) and by `vendorTicket`'s fleet hand-off (`assignTechnician()`). `vendorTicket`/`technicianTicket`/`driverTicket` now read their own "my tickets" list from here first (matching `Vendor` by name), falling back to the older `Vendor_Emails`/`Technician_Emails` match for pre-existing tickets. See Step 5/5a and `QUESTIONS_TO_ASK.md` for exactly what's implemented vs still pending (only Accept/Reject write to `Invites` so far, via a guessed `Service_Acceptance_Next` field) |

---

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
| `Vehicle` | Vehicle (Lookup → `vehicle_master_Report`) | select | Agent | ✅ |
| `Vehicle_Registration_Number` | Vehicle Reg. Number | text | Agent | 🟡 |
| `Vehicle_Issue` | Vehicle Issue (plain multiselect text, not a Lookup) | multiselect | Agent | ✅ |
| `Service_Type` | Service Type ("RSR"/"TOW", hidden, derived from `Vehicle_Issue`) | badge | Agent, Technician, Driver, Vendor | ✅ |
| `Vehicle_Status` | Vehicle Status (`ONROAD`/`SAFE PARKING`) | select | Agent | 🟡 |
| `Time_of_service` | Time of Service (`NOW`/`LATER`) | radio | Agent | 🟡 |
| `Schedule_Date` | Schedule Date (when `LATER`) | date | Agent | 🟡 |
| `Service_Time` | Service Time slot (when `LATER`) | select | Agent | 🟡 |
| `Computed_Service_Time` | Computed service time (NOW = +30min, LATER = Schedule_Date+Service_Time), 12-hour+AM/PM+seconds format | (hidden, `extra()`) | Agent | ✅ (format confirmed live) |
| `Lead_Source` | Always `"Call"` for this agent-facing widget | (hidden, `extra()`) | Agent | ✅ |
| `Remarks` | Remarks (optional, editable at multiple steps — see Steps 2/3/4 too) | textarea | Agent | 🟡 |
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
| `Payment_Method` | Payment Method — **a different field from Step 8's `Payment_Method1`** | select | Agent | ❓ (kept as-is 2026-07-31 despite a flow spec showing a different list — user-confirmed) |
| `Payment_Status` | Payment Status (`PAID`/`PENDING`/`NOT APPLICABLE`) | select | Agent | ❓ (kept as-is 2026-07-31 despite a flow spec showing "Success/Pending" — user-confirmed) |
| `Total_Service_Fee1` | Total Service Fee (Payment Record) — likely-duplicate of `Total_Service_Fee`, relationship unconfirmed | number | Agent | ❓ |
| `Image_Upload` | Receipt Photo (locked/readonly here) | file | Agent | 🟡 |

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
| `Vendors1` | Vendors (multi-Lookup → `vendors_Report`) — **renamed from `Vendors` 2026-07-31**; **corrected 2026-07-31 (same day)**: confirmed by the user to be a real Lookup field, not plain multiselect text — now saves real `vendors_Report` record IDs (`useId:true`, via `idsFor()`) instead of the old composite `"[priority] Name \| Phone \| Status"` string, which a genuine Lookup field was silently rejecting (root cause of "no vendor is assigned after I click assign vendor"). **Write-shape experiment reverted, same day**: briefly tried wrapping each ID as an `{"ID":"..."}` object per a user report of Zoho's usual Lookup format, but that broke saving entirely (even the Create step stopped submitting) — reverted back to the plain bare-ID value, which is what actually works. Priority/phone/online-status no longer ride along on the saved value itself — display-only, resolved fresh from `vendors_Report` at picker time, not persisted. **Also read by `vendorTicket`'s own dashboard**: matched via the new `Vendor_Emails` field first, falling back to a name-comparison for older tickets — see Open Questions | Lookup (bare ID) | Agent, Vendor (read-only, dashboard matching only) | ✅ (Lookup type confirmed live 2026-07-31; bare-ID write shape confirmed live 2026-07-31) |
| `Assigned_Vendor` | **New field, added 2026-07-31 per the user**: a single Lookup → (assumed) `vendors_Report`, set by the vendor's own Accept action once they accept the ticket (distinct from `Vendors1`, which only holds the invited *candidates*). Displayed read-only in `getRescueTicket`'s `acceptance`/`acceptanceTow` stages. **Not yet written by any widget** — `vendorTicket`'s own `acceptService()` needs to send this, but doesn't yet have a way to resolve its own `vendors_Report` record ID (it only knows its `My_Availability_Vendor` record) — see Open Questions | Lookup (single) | Agent (view-only) | ❓ (name given by user; target report + write-side not yet implemented) |
| `Vendor_Emails` | **New hidden field, added 2026-07-31 per the user**: auto-populated (comma-separated) with the email of every vendor selected in `Vendors1`, resolved from `vendors_Report` via the new `email` entry in `CONFIG.vendorFieldCandidates` — not shown anywhere in this widget's own UI (user will hide it on the real Zoho form too). Used by `vendorTicket`'s own dashboard matching as a more reliable alternative to name-matching | text (hidden) | Agent (write, silent) | 🟡 |
| `Vendor_Email` | Vendor Email — **moved 2026-07-31** off the Assignment screen onto `acceptance`/`acceptanceTow`'s own read-only display, per the Action 4 mock (which only shows the Assign-To picker + Remarks on this screen; outcome fields belong on the next page) | text | Agent (view-only, Acceptance step) | 🟡 |
| ~~`Assignment_Sent`~~ | **Removed 2026-07-31** — nothing ever read it, no real notification system is wired to it, and it isn't in the mock. Flag if there's a real reason to bring it back | radio | — | removed |
| `Assigned_Technician` | Assigned Technician — **moved 2026-07-31**, same reasoning as `Vendor_Email` above | select | Agent (view-only, Acceptance step), Vendor (write, on fleet hand-off) | 🟡 |
| `Assigned_Technician_Email` | Technician Email — **moved 2026-07-31**, same reasoning; also one of the "assigned to me" match candidates in Technician/Driver's own dashboards | text | Agent (view-only, Acceptance step), Technician, Driver, Vendor | ❓ (dashboard-matching candidate, not confirmed which field a given ticket actually uses) |

## Step 5 / 5a — Service Acceptance & Rejection (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Service_Request` | Response (RSR): `Accepted`/`Rejected` | radio | Agent (view-only), Technician, Vendor | 🟡 |
| `Service_Request1` | Response (TOW): `Accepted`/`Rejected` | radio | Agent (view-only), Driver, Vendor | 🟡 |
| `Assign_Technican` | Assign Technician? (RSR) | radio | Agent (view-only) | 🟡 |
| `Assign_Technican1` | Assign Driver? (TOW) | radio | Agent (view-only) | 🟡 |
| `Reject_Reason` | Reject Reason (RSR) | select | Agent (view-only), Technician, Vendor | 🟡 |
| `Reject_Reason1` | Reject Reason (TOW) | select | Agent (view-only), Driver, Vendor | 🟡 |
| `Vendor_Email1` | Vendor Email (Acceptance step's own copy, RSR) | text | Agent (view-only) | 🟡 |
| `Vendor_Email2` | Vendor Email (Acceptance step's own copy, TOW) | text | Agent (view-only) | 🟡 |
| `Technician_Email` | Technician Email (RSR) — also a dashboard-matching candidate | text | Agent (view-only), Technician | 🟡 |
| `Technician_Email1` | Driver Email (TOW) — also a dashboard-matching candidate | text | Agent (view-only), Driver | 🟡 |
| `Vendor_Email` / `Assigned_Technician` / `Assigned_Technician_Email` | **Moved here from Step 4 (Assignment) 2026-07-31** — outcome fields, shown read-only once someone's accepted, not something the agent fills in on the Assignment screen itself (see Step 4's own row for the rationale) | text/select/text | Agent (view-only) | 🟡 |
| `Technician_Emails` | **New hidden field, added 2026-07-31 per the user**: written by `vendorTicket`'s fleet hand-off (`assignTechnician()`) alongside `Assigned_Technician_Email` — currently just mirrors that one technician's email (since the hand-off still direct-assigns a single technician, not a multi-select request), but will hold a comma-separated list once the fleet "request several, whoever accepts" flow exists (see Open Questions). Also checked by `technicianTicket`/`driverTicket`'s own dashboard matching | text (hidden) | Vendor (write), Technician, Driver (dashboard-matching read) | 🟡 |
| `Assigned_Vendor` | Single Lookup, set by the vendor's own Accept action — see Step 4's own row | Lookup (single) | Agent (view-only) | ❓ (target report + write-side not yet implemented) |
| `Service_Acceptance` | Accept timestamp (RSR) | datetime | Technician, Vendor | ❓ (from flow-generation spec, not re-confirmed) |
| `Service_Acceptance_For_Tow` | Accept timestamp (TOW) | datetime | Driver, Vendor | ❓ |
| `Service_Reject_Time` | Reject timestamp (both branches) — added 2026-07-31, was previously (incorrectly) reusing `Service_Acceptance` for rejects too | datetime | Technician, Driver, Vendor | 🟡 (from the Action 5 flow spec) |
| `ETA` | ETA in minutes to breakdown, RSR (Haversine stand-in for Google Maps Distance Matrix API — see `FIELDS.md` note at bottom) | number | Agent (view-only), Technician, Vendor | 🟡 — **possible naming conflict, see Open Questions**: a separate Action 5 mock names this "RSP ETA" instead |
| `Distance_To_Breakdown` | Distance to breakdown in km, RSR | number | Agent (view-only), Technician, Vendor | 🟡 — **possible naming conflict, see Open Questions**: the same mock names this "RSP Distance" instead |
| `ETA_TOW` | ETA in minutes to breakdown, TOW | number | Driver, Vendor | ❓ |
| `Distance_To_Breakdown_TOW` | Distance to breakdown in km, TOW | number | Driver, Vendor | ❓ |
| `Odometer_Reading` | Start odometer (TOW, captured at Accept) — **same field name reused at Step 7b for the drop-side odometer reading; not two separate fields** | number | Driver, Vendor | ❓ |
| `Navigate_To_Drop_Link` | Drop location display (see Step 3) | (display only) | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.RSP_Name` / `.Reason` / `.Rejected_At` / `.Vehicle_Issue` | Expanded rejection-log fields (2026-07-31, per the Action 5 flow spec's "RSP name... & Task details") — best-effort, non-blocking write, plain Single Line/Date-Time values | text/datetime | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.Case_ID` | The user built this as a real Lookup field (→ `Create_Case`) in Zoho Studio — now sends the ticket's own record ID (`r.ID`), same ID `apiUpdate(r.ID,...)` targets everywhere else, instead of the display Case ID string. **A same-day `{"ID":...}` object-wrapping experiment was tried and reverted** — it broke saving, so this is back to the plain bare-ID value | Lookup (bare ID) | Technician, Driver, Vendor | 🟡 |
| `Task_Rejections.Vehicle` | Also a real Lookup field — sends the raw `r[CONFIG.vehicleField]` value as fetched from `Create_Case` (unconverted). **Same object-wrapping experiment tried and reverted** — see `Case_ID`'s own row. Still unconfirmed/needs live testing whether this raw pass-through shape is actually correct — the write is non-blocking, so a mismatch fails silently | Lookup (bare, unconverted) | Technician, Driver, Vendor | ❓ (needs live verification) |

## Step 6 / 6a — Reach Breakdown Location & Cancel (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Action_field` | Action (RSR): `Reached`/`Cancel` | radio | Agent (view-only) | 🟡 |
| `Action_field2` | Action (TOW Loading step): `Loaded`/`Cancel` | radio | Agent (view-only) | 🟡 |
| `Navigation_Link` | Navigation Link (RSR, free text) | textarea | Agent (view-only) | 🟡 |
| `Navigate_To_Breakdown_Link` | Navigate to Breakdown Link (TOW) | textarea | Agent (view-only) | 🟡 |
| `Image_Upload2` | Arrival Photo (RSR) / also reused as the RSR Cancel photo | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Image_Upload5` | Arrival Photo (TOW) / also reused as the TOW Cancel photo | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Cancel_Reason` | Cancel Reason (RSR) | select | Agent (view-only), Technician, Vendor | 🟡 |
| `Reject_Reason2` | Reject/Cancel Reason (TOW Reach step) | select | Agent (view-only), Driver, Vendor | 🟡 |
| `Reach_Time` | Reach/arrival timestamp (both branches) — the Action 6 mock calls this "RSP Reach time"; possible naming conflict, see Open Questions | datetime | Technician, Driver, Vendor | ❓ |
| `Roundtrip_Distance` | Round-trip distance in km — set here for RSR (office→breakdown→office Haversine), set again at Step 7b for TOW's 4-point version | number | Agent (view-only), Technician, Driver, Vendor | 🟡 |
| `Odometer_reading_at_Reached_Location` | Odometer at Reached (TOW) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `RSP_Start_Latitude` / `RSP_Start_Longitude` | The tech/driver/vendor's own GPS position at Accept time (Step 5) — added 2026-07-31 so Step 6's Cancel flow can compute a distance from here to wherever Cancel is tapped. **Renamed + changed to Text 2026-08-03** (was `RSP_Start_Lat`/`RSP_Start_Lon`, Decimal) — the user hit a real "exceeded its maximum digits" error on the Decimal version (raw GPS coordinates can have 17 decimal places) and switched these two specifically to plain Text to sidestep it entirely | text | Technician, Driver, Vendor | 🟡 (user-renamed 2026-08-03) |
| `Cancellation_Time` | Timestamp of the Cancel Task click | datetime | Technician, Driver, Vendor | ❓ (added 2026-07-31, guessed) |
| `Cancel_Location_Lat` / `Cancel_Location_Lon` | GPS position at the moment Cancel is confirmed | decimal | Technician, Driver, Vendor | ❓ (added 2026-07-31, guessed) |
| `Cancel_Distance` | Distance in km from `RSP_Start_Latitude`/`Longitude` to `Cancel_Location_Lat`/`Lon` (Haversine stand-in) — the Action 6 mock calls this "RSP distance"; **deliberately a different field from Step 5's own "RSP distance" (`Distance_To_Breakdown`)** since they're computed from different points — see Open Questions. `Cancel_Location_Lat`/`Lon` themselves are still Decimal (only `RSP_Start_Latitude`/`Longitude` were switched to Text) | number | Technician, Driver, Vendor | ❓ (added 2026-07-31, guessed) |
| `Toll_Charges` | Toll Charges? (TOW Reach step) | radio | Agent (view-only), Driver, Vendor | 🟡 |

## Step 7 — Work In Progress (Technician, RSR only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Issue_Resolved` | Issue Resolved? — required as of 2026-07-31 (Action 7 flow spec marks it mandatory; Technician/Vendor now validate it before "Work Completed" proceeds) | radio | Agent (view-only), Technician, Vendor | 🟡 |
| `Unresolved_Note` | Free-text note when Issue Resolved = No — added 2026-07-31 as a plain-text stand-in for the mock's own (uncertain, "?") "add voice note" idea; **not real audio capture**, see Open Questions | textarea | Technician, Vendor | ❓ (guessed) |
| `Image_Upload3` | Pre-service Photo(s) — min 2, max 4 per the flow spec | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Image_Upload4` | Post-service Photo(s) — min 2, max 4 | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Expense_Amount` | Expense Amount | number | Agent (view-only), Technician, Vendor | 🟡 |
| `Expense_Photo` | Expense Photo(s) — up to 2 | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Office_To_Mechanic` | Office → Mechanic leg (km) — RSR's own 3-leg round-trip | number | Agent (view-only) | 🟡 |
| `Mechanic_To_Breakdown` | Mechanic → Breakdown leg (km) | number | Agent (view-only) | 🟡 |
| `Breakdown_To_Office` | Breakdown → Office leg (km) | number | Agent (view-only) | 🟡 |
| `RSP_Completion_Time` | Completion timestamp — **shared by both "Work Completed" and "Service Reject"** (renamed 2026-07-31 from `Work_Completed_Time`, which only "Work Completed" ever set; the Action 7 flow spec explicitly says both buttons stamp the same field) | datetime | Technician, Vendor | ❓ (renamed 2026-07-31 per explicit spec wording, still guessed — part of the recurring "RSP_" naming pattern, see Open Questions) |
| `Rejection_Reason` | Customer-reject reason (Action 7's own CX-reject branch) — **confirmed 2026-07-31 to be the same field reused at Step 7a's own Cx-reject branch** (the TOW Loading screen), not a distinct per-branch field | select | Technician, Driver, Vendor | 🟡 |

## Step 7a — WIP: Loading (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Pre_service_Photo` | Pre-service Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Image_Upload_On_truck` | On-truck Photos — 3 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `VCRF` | VCRF Form upload — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Image_Upload6` | Additional Photo (agent-side field only — not exercised by Driver/Vendor's own Loading screen) | file | Agent (view-only) | 🟡 |
| `Rejection_Reason` | Cx-reject reason at this step — **corrected 2026-07-31**: this stage's alternate outcome is "Customer Rejected" → `Status:"REMAINING FEE DUE"` (same as Step 7's own Cx-reject), sharing the one `Rejection_Reason` field above; a previously-guessed distinct `Rejection_Reason1` (and a synthetic, never-written `Action_field2`) were removed — they didn't match this actual screen and were also silently breaking `getRescueTicket`'s own status derivation for this stage, see Open Questions | select | Technician, Driver, Vendor | 🟡 |
| `RSP_Completion_Time` | Shared completion timestamp, same field as Step 7 — stamped by this stage's own "Service Reject" too, per the Action 7a mock | datetime | Technician, Driver, Vendor | ❓ |
| `Pickup_Time` | Vehicle-picked timestamp | datetime | Driver, Vendor | ❓ |
| `Breakdown_To_Drop_Distance` | Breakdown → Drop leg (km) — set here from Haversine, then recomputed as part of the 4-point total at Step 7b. **As of 2026-07-31, computed from the driver's own live GPS fix at the moment "Vehicle Picked" is tapped (falling back to the ticket's static breakdown Lat/Lon if GPS is unavailable), per the Action 7a mock's "based on current GPS location of Driver"** — previously always used the static breakdown coordinates | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Return_Journey_ETA` | Return ETA in minutes — same live-GPS-origin change as `Breakdown_To_Drop_Distance` above. **Naming conflict, not renamed**: the Action 7a mock calls these two fields "Drop Distance" and "DROP ETA" respectively — a different name pair from what's shipped here (from the original field-generation spec's own matrix); kept the original names pending confirmation, same treatment as the Step 5 ETA/Distance conflict, see Open Questions | number | Agent (view-only), Driver, Vendor | 🟡 |

## Step 7b — WIP: Reached Drop Location (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Odometer_Reading` | Odometer reading at drop (same field name as Step 5/5a's start-odometer — see that row's note) — **corrected 2026-07-31: this is a plain number field, not a photo upload** (`getRescueTicket` was showing it as a file field; fixed to match what Driver/Vendor actually send) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Drop_Location_Photo` | Drop Location Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Toll_Charges1` | Toll Charges? (drop side) | radio | Agent (view-only), Driver, Vendor | 🟡 |
| `Office_To_Mechanic` / `Mechanic_To_Breakdown` / `Breakdown_To_Drop_Distance` / `Drop_To_Office_Distance` | The 4-point roundtrip's individual legs — **"Mechanic" waypoint question resolved 2026-07-31**: the Action 7b mock's own footnote describes the roundtrip as "RESCUE[static] to Breakdown Location to Drop Location to RESCUE[static]" — i.e. a 3-leg Office→Breakdown→Drop→Office trip, with no separate "Mechanic" waypoint at all. "Mechanic" in the original field-generation spec's naming appears to just be that same static office/"RESCUE" point under a different label. The already-shipped calculation (`officeToMechanic` = Office→Breakdown, `mechanicToBreakdown` hardcoded `0`) already produces exactly this 3-leg total, so **no code change was needed** — only the field *names* stay a mismatch against this new understanding (kept as-is to avoid an unnecessary rename of already-shipped, cross-referenced fields) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Roundtrip_Distance` | Total of the 4 legs | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Drop_Location_Arrival_Time` | Timestamp when "Reached Drop Location" is tapped — added 2026-07-31 per the Action 7b mock ("time is captured as 'drop location arrival time'") | datetime | Agent (view-only), Driver, Vendor | ❓ (guessed name) |
| `RSP_Drop_Location_Lat` / `RSP_Drop_Location_Lon` | The driver/vendor's own live GPS fix at that same click — added 2026-07-31 per the mock's "Location is captured as 'RSP Drop Location'"; separate from the ticket's static `DropLocationLat`/`DropLocationLong` (the customer-specified drop point, still used for the roundtrip calc) | decimal | Agent (view-only), Driver, Vendor | ❓ (guessed name) |

## Step 7c — WIP: Unloading & Handover (Driver / Vendor, TOW only)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Unloaded_Images` | Unloaded Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `VCRF_Image` | VCRF Image — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_Image` | Handover Photo — 1 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_to_Name` | Handover to (Name) | text | Agent (view-only), Driver, Vendor | 🟡 |
| `Handover_to_Number` | Handover Number — **corrected 2026-07-31 to a number input**, per the Action 7c mock's explicit "[Number field]" (was a `tel`-type text box) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `RSP_Completion_Time` | Shared completion timestamp, same field as Step 7/7a — added 2026-07-31: the Action 7c mock says "Capture Time of click of 'DROPPED' button as the 'work complete time'" | datetime | Agent (view-only), Technician, Driver, Vendor | ❓ |
| `Handover_to_Designation` | Handover Location (`Home`/`Office`/`Work Shop`) | select | Agent (view-only), Driver, Vendor | 🟡 |

## Step 8 — Final Payment Collection (Technician / Driver / Vendor, shared)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Remaining_Fee` | Remaining Fee (readonly display here — same field as Step 2's own copy) | decimal | Agent, Technician, Driver, Vendor | 🟡 |
| `Payment_Method1` | Payment Method — **a different field from Step 2's `Payment_Method`** | select | Agent, Technician, Driver, Vendor | 🟡 — options corrected 2026-07-31 to `["Payment Gateway","Direct","Cash"]`, default `"Payment Gateway"`; the flow spec's own "/ Etc" implies more may exist (see Open Questions) |
| `Payment_received` | Payment Received? | radio | Agent, Technician, Driver, Vendor | 🟡 |
| `QR_Code` | QR / Reference | text | Agent | 🟡 |
| `Send_Payment_Link1` | Send payment link | check | Agent | 🟡 |
| `Upload_Photo_of_receipt` | Receipt Photo — mandatory when Payment Method isn't Payment Gateway | file | Agent, Technician, Driver, Vendor | 🟡 |
| `Payment_Status` | `Pending`/`Success` — added 2026-07-31 as an explicit field (was previously only inferred internally). Technician/Driver/Vendor auto-flip it to `Success` once a receipt photo is attached for a non-gateway method, or read it as already `Success` if some other real integration set it; **"Payment Received" is only enabled once this is `Success` or the balance is zero**. Mirrored read-only into `getRescueTicket`'s own `payment` stage/Final Closure display 2026-07-31 (Final Closure reconciliation) — wasn't shown to the agent at all before | select | Agent (view-only), Technician, Driver, Vendor | 🟡 |
| `Transaction_ID` / `Time_Of_Receipt` | The mock's own "transaction ID, time of receipt... captured in their respective fields" once a real Payment Gateway returns success — **not implemented**, since there's no real gateway webhook in this app to source these from honestly (would need a real integration, not a client-side guess) | text/datetime | — | ❓ (not set by any widget yet — see Open Questions) |

## Final Closure (Agent only)

Reachable at any point in a ticket's timeline (dashboard row action + a permanent wizard button), not gated by the stepper. Every field from every step above is also shown here (read-only display, or click-to-edit — see `getRescueTicket/README.md`'s Final Closure entries for the full mechanism). Fields unique to this screen:

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

### `vendors_Report` (vendor master)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `Vendor_Name` / `Name` / `vendor_name` | Display label candidates | Agent | ✅ (label resolves live) |
| `Vendor_Phone` / `Phone` / `phone_number` / `Contact_Number` / `Mobile` / `Mobile_Number` | Phone candidates | Agent | ❓ |
| `Priority` / `Vendor_Priority` / `Rank` | Priority candidates | Agent | ❓ |
| `Availability_Status` / `Status` / `Vendor_Status` | Online/Offline candidates | Agent | ❓ |
| `Latitude` / `Vendor_Latitude` / `Lat` and `Longitude` / `Vendor_Longitude` / `Long` / `Lng` | Vendor's own location candidates | Agent | ❓ |
| `Vendor_Type` | Individual vs. Fleet | Vendor | ❓ (guessed, drives `IS_FLEET`) |

### `My_Availability_Vendor` (a vendor's own toggle record)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `Email` | Match to logged-in user | Vendor, Toggle | ✅ (mechanism), 🟡 (exact field) |
| `Availability_Status` | `Online`/`Offline` | Vendor, Toggle | 🟡 |
| `vendor_name` | Display name | Vendor, Toggle | 🟡 |

### `technicians_Report` (technician/driver master)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `Email` | Match to logged-in user | Technician, Driver, Vendor (fleet hand-off), Toggle | 🟡 |
| `Availability_Status` | `Online`/`Offline` | Technician, Driver, Toggle | 🟡 |
| `technician_name` | Display name | Technician, Driver, Vendor, Toggle | 🟡 |
| `Fleet_Vendor` | Links a technician back to their owning fleet vendor (matched by name string, not ID) | Vendor | ❓ (guessed) |

### `vehicle_master_Report`
| Field | Purpose | Status |
|---|---|---|
| `Name` | Display label | ✅ |
| `Category` | 2W/4W | ✅ |
| `Segment` | A–F, used in fee calc | ✅ |

### `Vehicle_Issue_Report`
| Field | Purpose | Status |
|---|---|---|
| `Issue_Name` / `Name` | Display label | ✅ |
| `VEHICLE_TYPE` | Matched against vehicle's `Category` | ✅ |
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

## Dropdown / Radio / Select values — exact option lists used in code

Every select/radio/dropdown field across all four widgets, with the **exact** values the code sends — build the Zoho Creator dropdown/radio fields with these exact options (case and spelling matter; Zoho matches on the literal string). `Status` itself is its own table right below this one, since it's the biggest and most step-dependent.

> ⚠️ **Naming collision found 2026-07-31, not yet resolved — needs your decision before building.** Two *different* fields both ended up named `Payment_Status` on `Create_Case`:
> 1. The **original `quote`-stage field** (booking-fee payment status): options `PAID` / `PENDING` / `NOT APPLICABLE`. Been in this app since early on, currently `disabled:true` in the wizard (display-only there).
> 2. A **new field added 2026-07-31** for the Action 8 reconciliation (final/remaining-fee payment status, read/written by `technicianTicket`/`driverTicket`/`vendorTicket`'s own Payment screen and mirrored into `getRescueTicket`'s `payment` stage): options `Pending` / `Success`.
>
> These can't both really be one field — the value sets don't overlap and mean different things (booking fee vs. final fee). **Please tell me which of these is the real, already-existing `Payment_Status` field in your Zoho form** (if either), so I can rename the other one to something distinct (e.g. `Final_Payment_Status`) across `getRescueTicket` and all three field widgets. Until this is resolved, don't create a single `Payment_Status` dropdown expecting it to serve both purposes — it needs to be two separate fields.

### Shared reason list — `REJECT_REASONS`
Used, unmodified, by every "reason" dropdown below (rejections, cancellations, cx-rejects, refunds, closure) unless noted otherwise:
`Too Far / Outside Service Area`, `Already Engaged With Another Job`, `Vehicle/Equipment Not Suitable`, `Not Available (Off Duty)`, `Other`

| Field | Values (exact, in order) | Default | Used at | Widgets |
|---|---|---|---|---|
| `Vehicle_Status` | `ONROAD`, `SAFE PARKING` | — | Step 1 (Create) | Agent |
| `Time_of_service` | `NOW`, `LATER` | — | Step 1 (Create) | Agent |
| `Service_Type` | `RSR`, `TOW` | — | Step 1 (derived from the chosen issue's `Issue_Type`, not picked directly) | Agent |
| `Payment_Method` | `Cash`, `UPI`, `Card`, `Net Banking` | — | Step 2 (Quote/Booking Fee) — currently `disabled:true`, display-only | Agent |
| `Payment_Status` (quote-stage one) | `PAID`, `PENDING`, `NOT APPLICABLE` | — | Step 2 (Quote/Booking Fee) — **see naming-collision callout above** | Agent |
| `Preferred_Vendor` | `Yes`, `No` | `No` | Step 4 (Assignment) | Agent |
| `Assignment_Sent` | `Yes`, `No` | — | Step 4 (Assignment) | Agent |
| `Assign_Technican` (RSR) / `Assign_Technican1` (TOW) | `Yes`, `No` | — | Step 5/5a (Acceptance) | Agent (view-only) |
| `Reject_Reason` (RSR) / `Reject_Reason1` (TOW) | *`REJECT_REASONS`* | — | Step 5/5a (Reject) | Technician/Driver/Vendor, Agent (view-only) |
| `Cancel_Reason` (RSR) / `Reject_Reason2` (TOW) | *`REJECT_REASONS`* | — | Step 6/6a (Cancel) | Technician/Driver/Vendor, Agent (view-only) |
| `Issue_Resolved` | `Yes`, `No` | — | Step 7 (RSR WIP) | Technician, Agent (view-only) |
| `Rejection_Reason` | *`REJECT_REASONS`* | — | Step 7 (RSR Cx-reject) & Step 7a (TOW Loading Cx-reject) — same field, shared across both | Technician/Driver/Vendor, Agent (view-only) |
| `Toll_Charges` (Step 6a) / `Toll_Charges1` (Step 7b) | `Yes`, `No` | — | Step 6a / 7b | Driver/Vendor, Agent (view-only) |
| `Handover_to_Designation` | `Home`, `Office`, `Work Shop` | — | Step 7c (Unloading) | Driver/Vendor, Agent (view-only) |
| `Payment_Method1` | `Payment Gateway`, `Direct`, `Cash` | `Payment Gateway` | Step 8 (Payment) — the flow spec's own "/ Etc" implies more may exist, unconfirmed | Technician/Driver/Vendor, Agent |
| `Payment_received` | `Yes`, `No` | — | Step 8 (Payment) | Agent |
| `Payment_Status` (payment-stage one) | `Pending`, `Success` | — | Step 8 (Payment) — **see naming-collision callout above** | Technician/Driver/Vendor, Agent (view-only) |
| `Refund_Due` | `Yes`, `No` | `No` | Final Closure | Agent |
| `Refund_Reason` | *`REJECT_REASONS`* | — | Final Closure — reused list, not a dedicated refund-reason set; unconfirmed if that's actually correct | Agent |
| `Closure_Status` | `Not Converted`, `Cancelled`, `Completed` | — | Final Closure — kept as 3 options, not split into "Cancelled Billable"/"Cancelled Not Billable" (see README §5, 2026-07-30) | Agent |
| `Closure_Reason` | *`REJECT_REASONS`* | — | Final Closure (shown when Closure_Status is Cancelled/Not Converted) — reused list, unconfirmed if correct for this purpose | Agent |
| `Cx_Feedback_Score` | `1`,`2`,`3`,`4`,`5`,`6`,`7`,`8`,`9`,`10` | — | Final Closure | Agent |
| `B2B_GST_Invoice_Required` | `Yes`, `No` | `No` | Final Closure | Agent |
| `Vendor_Type` | `Individual`, `Fleet` (matched case-insensitively in code, but these are the two canonical values) | — | Vendor's own profile record (`vendors_Report`/`My_Availability_Vendor`), not `Create_Case` — guessed field name, unconfirmed | `vendorTicket` |
| `Availability_Status` | `Online`, `Offline` | — | `technicians_Report` / `My_Availability_Vendor`, not `Create_Case` | All three field widgets, `toggleGetRescue` |

## Status values written to `Status` (Create_Case) — step-by-step

| Value | Written by (step) | Resumes at (reopening a ticket) |
|---|---|---|
| `SERVICE FEE QUOTE` | Step 1 (Create Case, on save) | Step 2 |
| `BOOKING FEE PENDING` | Step 2 | Step 2 |
| `LOCATION REQUEST` | Step 2 (fee-free, link not yet sent) or Step 3 (own default) | Step 3 |
| `LOCATION PENDING` | Step 2 (once link sent / paid) | Step 3 |
| `READY FOR ASSIGNMENT` | Step 3 (once a breakdown location exists) | Step 4 |
| `RSP IRA (INVITE RESPONSE AWAITED)` | Step 4 | Step 4 |
| `RSP REJECT` | Step 5/5a | Step 4 (re-assign) — an earlier Action 5 mock seemed to suggest `READY FOR ASSIGNMENT` instead; **resolved 2026-07-31**, the Action 5a (TOW) mock explicitly confirms `RSP REJECT` |
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

See `getRescueTicket/QUESTIONS_TO_ASK.md` for the current live list (Individual vs. Fleet field, the "Mechanic" waypoint, App Status vs. Login Status, Payment_Method1's full option list, and others) — not duplicated here to avoid two copies drifting apart. Fix the field name **here** once an answer comes in, then that question can be checked off there.
