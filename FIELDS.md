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
| `Added_Time` | Creation Time | *(Zoho built-in system field — nothing to create)* | — | **Used starting 2026-09-02 (build-plan item #6, dashboard live timer)** — the wireframe's own "Creation Time" auto-field. **Bug found live 2026-09-04**: `vendorPerformance`/`agentPerformance` both got `Added_Time: undefined` on every single record from `Agent_Ticket_Report`, despite passing `field_config:"all"` — confirmed via Zoho's own API v2.1 docs that `"all"` only "fetches the fields included both in the detailed view and quick view layout," i.e. it's scoped to whatever fields a report's own layout was built with, **not** literally every field on the form. `Added_Time` is a system field that was apparently never added to `Agent_Ticket_Report`'s layout — this was a report-configuration gap, not something fixable in widget code. **Fixed in Zoho 2026-09-04/05 (user-confirmed live)**: "Added Time" added to `Agent_Ticket_Report`'s own layout (Report Builder → Configure Fields → "+ Add Fields") — a follow-up console dump showed real timestamps flowing through for `vendorPerformance`/`agentPerformance`. Since this is a report-level layout change (not per-widget), it should transparently also cover `getRescueTicket`'s own dashboard timer (build-plan item #6), which reads the same field off the same report — not independently re-verified live for that specific widget, but no separate fix should be needed. `parseAnyDateTime()` (tries native `Date` parsing, covers ISO 8601, falls back to this app's own two custom formats) handles whatever shape the value comes back in. Also used by `vendorPerformance`/`agentPerformance` for date-range filtering |
| `Created_By_Agent_Email` | (hidden) | Single Line | — | ❓ **New field, added 2026-09-03** for the new `agentPerformance` widget — needs creating in Zoho before it can save. Stamped once, only on the FIRST save of a new ticket (`create` stage's own `extra()`, `!r.ID` guard) with the logged-in agent's own email — never touched on any later resave, so reopening/editing an existing ticket under a different agent's login never reattributes it. Tickets created before this field existed have no value here and won't be attributed to any agent |
| `Assignment_Time` | (hidden) | Date-Time | — | ❓ **New field, added 2026-09-03** for `agentPerformance`'s "Avg Time (Creation to Assignment)" metric — needs creating in Zoho before it can save. Stamped once, only the FIRST time the `assignment` stage is successfully saved (`Vendors1` is `req:true`, so a successful save always means a real vendor was picked) — never overwritten on a later re-invite/reassign of the same ticket |
| `Agent_Wizard_Step` | Agent Wizard Step | Single Line | one of the wizard's own step `key`s, e.g. `acceptance` | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified against a live record showing real data flowing through it. Unlocks the "resume at the exact step the agent was on" feature (see `getRescueTicket/README.md`'s matching entry). Hidden; `getRescueTicket` writes its current step here on every step change and reads it back on reopen, alongside `Status`, to disambiguate cases where `Status` alone is shared by more than one step |
| `Locked_By` | Locked By | Single Line | the holding agent's login email | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live. Unlocks the ticket-locking feature (see `getRescueTicket/README.md`'s matching entry — "if two agents try to open the same ticket, only one can work on it at a time"). Hidden; agent-vs-agent only, scoped to `getRescueTicket` — `vendorTicket`/`technicianTicket`/`driverTicket` don't read or write either of these two fields at all |
| `Locked_At` | Locked At | Date-Time | — | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live. Paired with `Locked_By` above; refreshed roughly every 60s while the locking agent's wizard stays open (a heartbeat, not just a one-time stamp) so a genuinely active session's lock never goes stale mid-edit. A lock older than 15 minutes with no heartbeat is treated as abandoned and silently released for the next agent |
| `Confirmation_Time` | (hidden) | Date-Time | — | 🟡 **Created in Zoho 2026-09-07 (user-confirmed) and wired same day** — user-provided LOCATIONS-DISTANCES-DATETIME spec, "#5 Task Confirmation Time." Stamped once, the first time `Status` reaches `SCHEDULED` or `READY FOR ASSIGNMENT` (whichever earlier) — two independent `extra()` hooks (`create` and `locations` stages, the two places that transition can happen) both check `!r.Confirmation_Time` so whichever fires first wins, never overwritten after |
| `Remaining_Fee_Receipt_Time` | (hidden) | Date-Time | — | 🟡 **Created in Zoho 2026-09-07 (user-confirmed) and wired same day** — spec's "#14 Remaining Fee Receipt Time" (cash branch only — the Zoho-Payment-gateway branch remains unbuilt, no gateway integration exists). Stamped at the same click as `RSP_Closure_Time` below, in all three vendor-facing apps' own `confirmPaymentReceived()` plus `getRescueTicket`'s own agent-run `payment` stage |
| `RSP_Closure_Time` | (hidden) | Date-Time | — | 🟡 **Created in Zoho 2026-09-07 (user-confirmed) and wired same day** — spec's "#15 RSP Closure Time." Shares the exact same trigger as `Remaining_Fee_Receipt_Time` above per the spec's own wording (cash "Payment Received" click) — kept as two separate fields matching the spec's literal numbering, not merged |
| `Agent_Closure_Time` | (hidden) | Date-Time | — | 🟡 **Created in Zoho 2026-09-07 (user-confirmed) and wired same day** — spec's "#16 Task Closure Time (Agent)." Written in `saveClosure()`'s own payload on every save of Final Closure (not stamped-once — always reflects the most recent Close action, same as every other field in that payload) |
| `Distance_For_Quote_RT_KM` | Distance for Quote (RT KM) | Number | — | TOW only |
| `Suggested_Rate` | Suggested Rate | Number | — | Computed/read-only in the widget |
| `Total_Service_Fee` | Total Service Fee | Number | — | Also displayed read-only again at Step 8 (Payment) — see `payment` stage note below; the old separate `Total_Service_Fee1` field was dropped, this one field now serves both |
| `Booking_Fee` | Booking Fee | Number | — | |
| `Remaining_Fee` | Remaining Fee | Decimal | — | = Total − Booking; re-displayed at Payment |
| `Client_Charge` | Client Charge | Number | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New (On-Demand Invoicing/Vendor+Client Accounting prep, user-provided spec) — editable only when `Client` ≠ the special `"ON DEMAND"` walk-in value; null/disabled when it is |
| `Rescue_Charge` | Rescue Charge | Decimal | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, wired into the widget. Read-only, computed `Total_Service_Fee − Client_Charge − Booking_Fee` (`Client_Charge` treated as 0 when blank — user-confirmed 2026-08-25; originally stayed `null` until `Client_Charge` was entered, and originally didn't net out `Booking_Fee` at all — user reported live "Rescue Charge is not updated when booking fee enter"), same live-recompute pattern as `Remaining_Fee`. Only forced `null` for the `"ON DEMAND"` walk-in Client |
| `Rate_Breakup` | Rate Breakup | Multi-line | — | |
| `Send_Link_For_The_Breakdown_Location` | Request Location | Radio (`Yes`/`No`, or Checkbox) | — | Action-trigger field, not a real dropdown a human picks — value just gets flipped by the "Request Location" button |
| `Payment_Method` | Payment Method | Dropdown | `Cash`, `UPI`, `Card`, `Net Banking` | Booking-fee payment method — editable since the 2026-08-07 restore (this row's own "kept locked/display-only" note was stale, corrected 2026-09-02) |
| `Booking_Fee_Transaction_ID` | BF Transaction ID | Single Line | — | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live (added to the widget 2026-08-07, user request; was never added to this file at all until a 2026-08-11 field audit caught the gap). Purely a reference number the agent records against the booking-fee payment; nothing else in the app reads or writes it |
| `Payment_Status` | Payment Status | Dropdown | `PAID`, `PENDING`, `NOT APPLICABLE` | **The same field is reused at Step 8** (Final Payment) — one field, not two, confirmed live 2026-08-03 |
| `Image_Upload` | BF Receipt Photo (label changed 2026-09-02, build-plan item #8 — was plain "Receipt Photo", identical to `Upload_Photo_of_receipt`'s own label, so a photo's fee-type wasn't distinguishable by label alone) | File Upload | — | ✅ **Created in Zoho, user-confirmed 2026-09-10** (was 🔴 confirmed missing earlier the same day — Zoho code 3710 "No field named Image_Upload found" — see git history). No widget-side code change was needed; the widget already targeted this exact Link Name. Mandatory whenever `Payment_Method` has a value and isn't `"Payment Gateway"` — matches the wireframe's "mandatory if Payment Method is not Payment gateway" rule. Since `Payment_Method`'s real option list has no "Payment Gateway" choice at all today, this is effectively mandatory whenever any payment method is selected |
| `Latitude` / `Longitude` | Breakdown Latitude / Longitude | **Single Line Text** (✅ converted from Decimal 2026-09-11, user-confirmed — sidesteps the same "exceeded maximum digits" error class `RSP_Start_Latitude`/`Longitude` was already converted for) | — | Pasted as `lat, lon` text and split client-side |
| `Send_Link_For_The_Drop_Location` | Send Drop Location Link | Checkbox | — | TOW only |
| `DropLocationLat` / `DropLocationLong` | Drop Latitude / Longitude | **Single Line Text** (✅ converted from Decimal 2026-09-11, user-confirmed, same reason as `Latitude`/`Longitude` above) | — | TOW only — the customer's specified drop point |
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
| `ETA` (RSR) / `ETA_TOW` (TOW) | ETA (mins) | Number | — | Haversine stand-in for Google Maps Distance Matrix. Written once, at Accept time, by the vendor/technician/driver's own app — kept intact for reporting from then on. **Locked once real 2026-08-11** (user request) in both `getRescueTicket`'s Close Case screen and `Ticket Kanban`'s Reach step: read-only the moment it holds a real value, still manually editable while blank (fallback for a ticket the vendor's app never touched at all). Customer-communication edits (`Send_ETA_To_Customer`) never touch this field either way — they read/write the separate `Agent_ETA_Override`(`_TOW`) field instead, see that field's own row below |
| `Distance_To_Breakdown` (RSR) / `Distance_To_Breakdown_TOW` (TOW) | Distance to Breakdown (km) | Number | — | |
| `Odometer_Reading` | Odometer Reading | Number | — | **Fixed 2026-09-02 (user request)**: used to be reused for both the Step 5a start-odometer AND the Step 7b drop-side reading (one field, silently overwritten) — now Step 5a's start-odometer only. See `Odometer_reading_at_Drop_Location` below for the Step 7b reading, split into its own field |
| `Navigation_Link` | Navigation Link | Multi-line | — | RSR |
| `Navigate_To_Breakdown_Link` | Navigate to Breakdown Link | Multi-line | — | TOW |
| `Image_Upload2` (RSR) / `Image_Upload5` (TOW) | Arrival / Cancel Photo | File Upload | — | Reused for both Arrival and Cancel on the same branch |
| `Cancel_Reason` (RSR) / `Reject_Reason2` (TOW) | Cancel Reason | Dropdown | *`REJECT_REASONS`* | |
| `Reach_Time` | Reach Time | Date-Time | — | Shared by both branches |
| `Reach_Location_Lat` / `Reach_Location_Lon` | Reach Location Lat / Lon | Single Line | — | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live (user request, "Missing Fields & Field Behavior" spec — Reach never captured a live GPS fix before this at all). Text, not Decimal — same digit-limit reasoning as `RSP_Start_Latitude`/`Longitude`. Shared by both branches, written by `vendorTicket`/`technicianTicket`/`driverTicket`'s own Reach-click handler |
| `Distance_Reach_To_Breakdown` (RSR) / `Distance_Reach_To_Breakdown_TOW` (TOW) | Reach Location → Breakdown (km) | **Decimal** | — | 🟡 **Type corrected to Decimal in Zoho 2026-08-11 (user-confirmed)** — was originally created as "Number" per this file's own earlier (wrong) spec, which rejected the real one-decimal-place km value live with "Enter a valid number." Now matches every other km-distance field in this app (`Distance_To_Breakdown`, `Cancel_Distance`, etc.). Not yet re-tested live since the correction — worth confirming "Reached Location" saves cleanly now. Deliberately a NEW field, separate from `Roundtrip_Distance` (Office→Breakdown→Office, unchanged) — a plain one-way distance from the Reach-click position to the breakdown location, same shape as `Distance_To_Breakdown` at Accept |
| `Roundtrip_Distance` | Round-Trip Distance (km) | Number | — | RSR's 3-leg total; TOW's 4-point total (Step 7b) |
| `Odometer_reading_at_Reached_Location` | Odometer at Reached | Number | — | TOW |
| `RSP_Start_Latitude` / `RSP_Start_Longitude` | RSP Start Latitude / Longitude | Single Line | — | **Text, not Decimal** — sidesteps a real "exceeded maximum digits" error raw GPS coordinates (up to 17 decimal places) hit on a Decimal field. Reused 2026-08-04 for TOW's own drop-side GPS fix too (see `RSP_Drop_Location_Lat`/`Lon`'s own row below) — one field, not two |
| `Cancellation_Time` | Cancellation Time | Date-Time | — | |
| `Cancel_Location_Lat` / `Cancel_Location_Lon` | Cancel Location Lat / Lon | **Single Line Text** (✅ converted from Decimal 2026-09-11, user-confirmed — last remaining Decimal coordinate pair in the app, same "exceeded maximum digits" reason as every other lat/lon field) | — | |
| `Cancel_Distance` | Cancel Distance (km) | Number | — | Distance from `RSP_Start_Latitude`/`Longitude` to `Cancel_Location_Lat`/`Lon` — deliberately a different field from `Distance_To_Breakdown` |
| `Distance_Cancel_To_Breakdown` | Cancel Location → Breakdown (km) | **Decimal** | — | 🟡 **Type corrected to Decimal in Zoho 2026-08-11 (user-confirmed)**, same fix as `Distance_Reach_To_Breakdown`'s own row above — not yet independently re-tested live. Deliberately a different field from `Cancel_Distance` just above (Accept location → Cancel location) — this is Cancel location → the ticket's own static breakdown location. Shared by both branches |
| `Toll_Charges` (Step 6a) / `Toll_Charges1` (Step 7b) | Toll Charges? | Radio | `Yes`, `No` | |
| `Agent_ETA_Override` (RSR) / `Agent_ETA_Override_TOW` (TOW) | Agent ETA Override (mins) | Number | — | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live (this row existed since 2026-08-07 but was missing this marker, a documentation gap caught during a later field audit). Agent-entering mode only. Deliberately separate from `ETA`/`ETA_TOW` above — never written in the same save as those, see that field's own note |
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
| ~~`RSP_Drop_Location_Lat` / `RSP_Drop_Location_Lon`~~ | — | Decimal | — | **Removed 2026-08-04 (user request)** — never a confirmed field; consolidated into `RSP_Start_Latitude`/`RSP_Start_Longitude` (Text) above, reused for this same live GPS fix |
| `Unloaded_Images` | Unloaded Photo | File Upload | — | 4 mandatory |
| `VCRF_Image` | VCRF Image | File Upload | — | 1 mandatory |
| `Handover_Image` | Handover Photo | File Upload | — | 1 mandatory |
| `Handover_to_Name` | Handover to (Name) | Single Line | — | |
| `Handover_to_Number` | Handover Number | Number | — | |
| `Handover_to_Designation` | Handover Location | Dropdown | `Home`, `Office`, `Work Shop` | |
| `Payment_Method_Final` | Payment Method | Dropdown | `Cash`, `UPI`, `Card`, `Net Banking`, default `Cash` | **Renamed 2026-08-03 from `Payment_Method1`** — a separate field from `Payment_Method` above (records how the *remaining* fee was paid, vs. the booking fee). **Editable from the Final Closure summary regardless of Acceptance_Mode as of 2026-09-02 (build-plan item #5)** — was previously locked there too whenever the vendor used their own app, contradicting the spec's explicit closure-page carve-out; the wizard's own live Payment step is unaffected, still locks as before |
| `Payment_received` | Payment Received? | Radio | `Yes`, `No` | Same 2026-09-02 always-editable-from-Closure fix as `Payment_Method_Final` above |
| `QR_Code` | QR / Reference | Single Line | — | Vestigial — kept for a future real payment-gateway integration, no widget currently shows a QR block |
| `Send_Payment_Link1` | Send Payment Link | Checkbox | — | |
| `Upload_Photo_of_receipt` | Remaining Fee Receipt Photo (label changed 2026-09-02, build-plan item #8 — see `Image_Upload`'s own matching note) | File Upload | — | Mandatory for every payment method now (no more "unless Payment Gateway" carve-out). Same 2026-09-02 always-editable-from-Closure fix as `Payment_Method_Final`/`Payment_received` above |
| `Refund_Due` | Refund Due? | Radio | `Yes`, `No`, default `No` | Final Closure |
| `Refund_Reason` | Refund Reason | Dropdown | *`REJECT_REASONS`* | |
| `Refund_Amount` | Refund Amount | Number | — | |
| `Closure_Status` | Closure Status | Dropdown | `Not Converted`, `Cancelled`, `Completed`, **`Cancelled - Billable`** | 🟡 **4th option added in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget — distinguishes a billable cancellation (vendor genuinely dispatched/travelled before the job was cancelled) from a plain non-billable one, drives the Vendor Round Trip Distance calc below. **Confirmed live 2026-09-04**: this is set only by the agent's own Final Closure step, which in real usage is almost never actually performed — a live sample of 37 real tickets on one vendor had this field blank on every single one, including several already at `Status:"RSP CLOSED"`. `vendorPerformance`/`agentPerformance` both originally keyed their "Completed" count off `Closure_Status==="Completed"` alone, which is why it undercounted; fixed to also treat `Status:"RSP CLOSED"` (Step 8, Payment Received — the vendor's own real completion signal, set automatically, long before Final Closure) as Completed |
| `Closure_Reason` | Not Converted / Cancelled Reason | Dropdown | *`REJECT_REASONS`* | |
| `Cx_Feedback_Score` | Customer Feedback Score | Radio | `1`,`2`,`3`,`4`,`5`,`6`,`7`,`8`,`9`,`10` | Single overall score, unchanged. (A same-day 3-score split — service provider/call agent/overall — was proposed 2026-09-07 then reverted per direct user feedback: "Single feedback as you have now is good enough." No new fields needed.) |
| `Cx_Feedback_Text` | Customer Feedback Notes | Multi-line | — | |
| `B2B_GST_Invoice_Required` | B2B GST Invoice Required? | Radio | `Yes`, `No`, default `No` | |
| `GST_Number` | GST Number | Single Line | — | Shown when `B2B_GST_Invoice_Required = Yes`. This is the CUSTOMER's own GST number for their B2B invoice — see `GST_Name`/`GST_Address` below, which auto-fetch off this same field, and don't confuse with `Vendor_GST_Number` on `Vendors_Report` (the vendor's own GSTIN, a different concept) |
| `Zoho_Books_Invoice_ID` | Zoho Books Invoice ID | Single Line | — | Written by `onDemandInvoicing()`'s real push (the `Send_Payment_Link` button, Quote stage) — holds the **Booking Fee** invoice specifically since 2026-09-07 (previously held an invoice for the ticket's full `Total_Service_Fee`, a confirmed mismatch against what the customer page's own `booking_payment` mode showed as due — see `getRescueTicket/README.md`'s matching entry) |
| `Zoho_Books_Invoice_ID_Remaining` | Zoho Books Invoice ID (Remaining) | Single Line | — | **New field, added 2026-09-07** — needs creating in Zoho before `Send_Payment_Link1` (Payment stage)'s own invoice write-back actually persists. A SEPARATE invoice from the one above, for exactly `Remaining_Fee` — kept in its own field so the two invoices never clobber each other's ID. Read by `getCustomerTicketInfo.deluge`'s `remaining_payment` mode, mirroring how `Zoho_Books_Invoice_ID` already feeds `booking_payment` |
| `Zoho_Books_Invoice_URL` | Zoho Books Invoice URL | Single Line | — | ✅ **Created in Zoho, user-confirmed 2026-09-11** (Task Tracker PDF item 27, "Invoice Link"). `onDemandInvoicing()` already returns a real invoice/payment URL from Zoho Books — this field is where that button's own call (`Generate_OnDemand_Invoice`) writes it, via that function's existing `paymentUrlField` parameter. Shown via `__OnDemandInvoiceLinkDisplay` at the end of the Closure page. **Not yet live-tested end-to-end** — worth clicking "Generate Invoice" on a real ON DEMAND ticket to confirm this field actually populates and the link renders/opens correctly |
| `Zoho_Payment_URL` / `Zoho_Payment_URL_Remaining` | Zoho Payment URL / (Remaining) | Single Line | — | ✅ **Created in Zoho, user-confirmed 2026-09-10** (renamed from the earlier placeholder `Zoho_Books_Payment_URL`/`Zoho_Books_Payment_URL_Remaining`, which was never confirmed to exist). Holds the REAL, payable link for the Booking Fee / Remaining Fee respectively, generated server-side by `onDemandInvoicing.deluge` via the real Zoho Payments "Payment Links" API (`payments.zoho.in/api/v1/paymentlinks`) and shown to the customer as the `invoiceUrl` `getCustomerTicketInfo.deluge` returns to the Customer Page's own "Pay Now" button. Falls back to the old hand-built `books.zoho.in/invoices/{id}?organization_id=...` invoice-view URL automatically if blank (older tickets, or if payment-link generation fails) |
| `Closure_Extra_Photo` | Additional Photo | File Upload | — | |
| `Vendor_Fleet_Location_Lat` / `Vendor_Fleet_Location_Lon` | Vendor Fleet Location Lat/Lon | Single Line | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New (user-provided spec) — Text, not Decimal, same digit-limit reasoning as every other raw GPS field in this app. Auto-picked at Closure as whichever of the assigned vendor's own `Base_Location_N_Lat`/`Lon` pairs (see `Vendors_Report` below) is nearest to the ticket's own `Latitude`/`Longitude` (crow-flight/Haversine, not Google API) — for both RSR and TOW |
| `Vendor_Round_Trip_Distance` | Vendor Round Trip Distance (km) | Decimal | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — TOW only, Google API multi-leg: Vendor Fleet Location → Breakdown → Drop → Vendor Fleet Location when `Closure_Status="Completed"`, or Vendor Fleet Location → Breakdown → Vendor Fleet Location when `Closure_Status="Cancelled - Billable"`. Always left `null` for RSR (see `Vendor_Distance` below instead) |
| `Vendor_Distance` | Vendor Distance (km) | Decimal | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — RSR-completed's one-way equivalent of `Vendor_Round_Trip_Distance` above: Google API distance from Vendor Fleet Location to the breakdown location only, no return leg |
| `Google_Feedback_Requested` | Google Feedback | Checkbox | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — when checked at Closure, triggers a Google-review-request message send (mechanism TBD, likely a WhatsApp template like `Send_Payment_Link`'s own) |
| `Extra_Payable` | Extra Payable | Number | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — plain agent-editable number on the Agent Closure page, no dependencies |
| `GST_Name` / `GST_Address` | GST Name / GST Address | Single Line / Multi-line | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — intended to auto-fetch off the ticket's own `GST_Number` above via a GSTIN lookup API; **that lookup integration itself is deferred** (user decision 2026-08-17: "fields/UI now, wiring later") — until it's built, these are plain manually-typed fields, same "ships safely" degrade as every other not-yet-wired field here |
| `Zoho_Books_Invoice_ID` / `Zoho_Books_Bill_ID` | Zoho Books Invoice ID / Bill ID | Single Line | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — reference IDs for the Zoho Books invoice (customer) and bill (vendor payment) once that integration is built; **wiring deferred** (2026-08-17 decision), same as `GST_Name`/`GST_Address` above. This is the same Zoho Books work scoped in the separate accounting-integration estimate, not new/duplicate scope |
| `Airtable_Record_ID` / `Airtable_Synced_At` | Airtable Record ID / Synced At | Single Line / Date-Time | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. New — reference fields for the "send full case detail to Airtable" Agent Closure action; **wiring deferred** (2026-08-17 decision, needs an Airtable API key + base/table ID, plus a Deluge Custom API proxy since this widget's CSP blocks a direct browser call to airtable.com — same reason WhatsApp sending goes through a Custom API instead of `fetch()`) |

**Dropped from the schema entirely (2026-08-03)** — do not create these; nothing in the current code writes or needs them:
`Assign_Technican`, `Assign_Technican1`, `Service_Request`, `Service_Request1`, `Action_field`, `Action_field2`, `Assignment_Sent`, `Total_Service_Fee1`. All were either display-only guesses nothing ever populated, or synthetic fields the widgets never actually wrote (they set `Status` directly instead) — see each widget's README changelog for the individual history.

## Form 2 — `Task_Rejections` (report: `Task_Rejections_Report` — **confirmed live 2026-09-04** via the report's own edit-page URL in Zoho; the widgets always pass the exact form name `Task_Rejections` to `addRecords`, so the report name itself doesn't matter for writes, only for reads)

**Read side added 2026-08-06** (user request — a full audit of every workflow timestamp turned up one real gap: `Rejected_At`/`RSP_Name`/`Reason` were being written correctly by all three field-facing widgets, but `getRescueTicket` never fetched or displayed this report at all, so a reject event was structurally invisible to the agent regardless of field-name correctness). New "Rejection History" section in Final Closure (`loadRejectionHistory()`, `CONFIG.taskRejectionsReport`), guessed as `Task_Rejections_Report` by analogy with every other report in this app. Calls `getRecords` directly (not through the usual `apiGetReport()` helper, which silently swallows every error into an empty list) specifically so a wrong guess here shows up as a console warning instead of looking identical to "this ticket genuinely has no rejections."

**Bug found and fixed 2026-09-04**: `vendorPerformance` (built 2026-09-03) independently guessed the *wrong* value — the bare form name `Task_Rejections` — instead of reusing this already-correct `Task_Rejections_Report` value, causing a live 404 on every load (silently swallowed by its `apiGetReport()` helper, so only the "Rejected / Timed Out" card came up empty — nothing crashed). Fixed by pointing `vendorPerformance`'s `CONFIG.taskRejectionsReport` at the same confirmed value. Lesson for future new widgets: grep existing widgets' `CONFIG` blocks for a report/field name before guessing a fresh one.

Best-effort rejection log, written by Technician/Driver/Vendor whenever they tap Reject. Non-blocking — a failed write here never blocks the actual rejection.

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `RSID` | RSID | Lookup → `Create_Case` | **Renamed 2026-08-03 from `Case_ID`**, to match `Invites`' own naming for the same "Lookup back to the ticket" concept. Sent as a bare record ID |
| `RSP_Name` | RSP Name | Single Line | The technician/driver/vendor's own display name |
| `Reason` | Reason | Single Line | Recommend Single Line, **not** a Dropdown, even though the app always sends one of the 5 `REJECT_REASONS` values — avoids a repeat of this session's "Invalid column value" class of bug on a field nothing lets a human mistype anyway |
| `Rejected_At` | Rejected At | Date-Time | |
| `Vehicle` | Vehicle | Lookup → `Vehicle_Master_Report` | Sent as the raw value read off the ticket's own `Vehicle` field |
| `Vehicle_Issue` | Vehicle Issue | Single Line | The ticket's `Vehicle_Issue` display string at the time of rejection |

## Vendor Accounting fields (on `Create_Case` / `Agent_Ticket_Report`)

New fields for the vendor-payout workflow — user-provided `Zoho - Vendor Accounting.pdf` spec. **Confirmed created in Zoho 2026-08-27**; the fee formula was built+tested the same day but had no screen until the Level 1 (Operations Approval) screen shipped 2026-09-04 in `getRescueTicket` (gated by `IS_OPS_MANAGER`, same as Ops Map). Level 2 (Accounts Payable) shipped the same day too — see `Accounting_Integration_Estimate.csv`/`_Detailed.csv` (repo root) for the original full scope breakdown this was drawn from.

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `Extras` | Extras | Number | Manual entry, editable on the Level 1 screen. **Not** the same field as the pre-existing `Extra_Payable` (a different, older concept — the two are deliberately distinct, do not confuse them) |
| `Cash_Collected` | Cash Collected | Number | Per the PDF's own "Case System" source column — **currently has no writer anywhere in this app**; renders as 0/blank on the Level 1 screen until a future increment wires its real source (most likely the vendor's own cash-collection record from the on-road payment step). Flagged in the screen's own UI text, not silently guessed at |
| `Vendor_KM_Fee` / `Vendor_Gross_Fee` / `Vendor_Net_Fee` / `Vendor_Balance_Before_TDS` | Vendor KM Fee / Gross Fee / Net Fee / Balance (before TDS) | Number ×4 | Written by the Level 1 screen's "Operations Approved" action — the computed output of `computeVendorFeeForTicket()`/`computeVendorFee()`. TDS itself is a Level 2 concern, not applied here |
| `Settlement_Status` | Settlement Status | Dropdown | **Value list confirmed 2026-09-04**: `Pending Approval` (or blank) → `Payable` → `Paid`, matching the PDF's own Case-completed → Operations-Approved → Payment-completed flow. **Verify this exact 3-option list is actually set on the live Zoho dropdown before relying on it** — named constants (`SETTLEMENT_STATUS_PENDING`/`_PAYABLE`/`_PAID`) in `getRescueTicket/app/widget.html` make a 1-line fix easy if the real wording differs. The Level 1 screen sets this to `Payable` on approval; a case only appears in the pending-approval list while this is blank or `Pending Approval` |
| `Ops_Approval_Comments` | Comments | Multi-line | Editable on the Level 1 screen, saved alongside the fee fields on approval |

**Vendor ID display code** (e.g. the PDF's `RSPID00XXX`): explicit 2026-09-04 user decision — **not building a separate code field**, the vendor's existing Zoho record/name is sufficient.

### Level 2 (Accounts Payable) fields — 🔴 NOT YET CREATED, needed before this screen works live

New fields for the batch vendor-payment screen (`accountsPayableBtn`/`recordVendorPayment.deluge`, shipped 2026-09-04). None of these exist in Zoho yet — same "specify here first, then create, then it gets wired" pattern as every other new field this session.

On `Create_Case` / `Agent_Ticket_Report`:

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `TDS_Amount` / `Final_Payable` / `Vendor_Amount_Paid` | TDS Amount / Final Payable / Amount Paid | Number ×3 | Written by `recordVendorPayment()` after a successful Zoho Books payment — each case's own computed values, even when paid together with other cases in one transaction |
| `Vendor_Payment_Date` | Payment Date | Date | Shared across every case paid in the same batch transaction |
| `Vendor_UTR_No` | UTR / Reference No. | Single Line | Shared across every case paid in the same batch transaction |
| `Vendor_Payment_Status` | Payment Status | Dropdown: `Payable`, `Paid` | **Distinct from the existing customer-side `Payment_Status`** (PAID/PENDING/NOT APPLICABLE, `FIELDS.md` §Create_Case) — do not confuse the two, they track different money |
| `Zoho_Books_Bill_ID` | Zoho Books Bill ID | Single Line | The real Bill created in Zoho Books for this case's payout — same traceability pattern as `Zoho_Books_Invoice_ID` for on-demand customer invoicing |

On `Vendors_Report`:

| API Name | Label | Zoho Field Type | Notes |
|---|---|---|---|
| `TDS_Percentage` | TDS % | Number | Per-vendor TDS rate (e.g. `1` for 1%) — explicit 2026-09-04 decision: per-vendor, not a single global config value, since real TDS rates can vary by vendor entity type |
| `Zoho_Books_Vendor_ID` | Zoho Books Vendor ID | Single Line | Caches the matched/created Books vendor contact so repeat payments skip the find-or-create round trip — same pattern as `On_Demand_Invoicing`'s customer resolution |

**GST is explicitly out of scope for Level 2** (2026-09-04 decision) — no `GST_Payable`/`GST_Paid` fields, no UI. `Vendor_GST_Number` (already on `Vendors_Report`) stays unwired.

**Zoho Books config values needed in `recordVendorPayment.deluge` itself** (not Creator fields — left as empty-string placeholders in the script, same pattern as `onDemandInvoicing.deluge`'s own `gst18_tax_id` before it was confirmed): `EXPENSE_ACCOUNT_ID` (Chart of Accounts entry the Bill's line items post to) and `PAID_THROUGH_ACCOUNT_ID` (the bank/cash account Vendor Payments are recorded from).

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
| `Invited_At` | Invited At | Date-Time | **New, created in Zoho 2026-09-07 (user-confirmed) and wired same day** — spec's "#6 RSP Invite Time." Stamped once, the moment this Invites record is created (`getRescueTicket`'s own invite-creation call, right alongside `RSID`/`Vendor`). Must also be added to `Invites_Report`'s own layout (Configure Fields), same "system/report field_config:'all' is layout-scoped, not form-scoped" gotcha `Added_Time` already hit once on `Agent_Ticket_Report` |

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
| `Current_Latitude` / `Current_Longitude` | Current Latitude / Longitude | Single Line | — | 🔴 **NOT YET CREATED** as of 2026-08-17 — see the cross-cutting `Vendors_Report` section further down for the full writeup (live-location heartbeat, `vendorTicket`) |
| `Vendor_GST_Number` | GST No. | Single Line | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — the vendor's own GSTIN, for vendor-payment/Zoho Books purposes. Deliberately not named `GST_Number` — that name is already taken on `Create_Case` for the customer's own B2B invoice GST, a different concept |
| `KM_Slab_Repair` / `KM_Slab_Tow_2W_FBT` / `KM_Slab_Tow_4W_FBT` / `KM_Slab_Tow_4W_MT` / `KM_Slab_Tow_2W_MT` | KM Slab – Repair / Tow 2W FBT / Tow 4W FBT / Tow 4W MT / Tow 2W MT | Number ×5 | — | 🟢 **Wired 2026-09-04** into the new Vendor Accounting Level 1 screen (`getRescueTicket`, `vendorKmSlabFieldFor()`/`computeVendorFeeForTicket()`) — per-vendor, per-vehicle-category distance-rate slabs for the vendor payout KM Fee. `KM_Slab_Tow_2W_MT` added later (2026-08-27, user-confirmed) than the other 4 (2026-08-17) — was previously missing here, corrected |
| `Base_Fee` / `Base_KM` / `KM_Rate` | Base Fee / Base KM / KM Rate | Number ×3 | — | 🟢 **Created 2026-08-27 (user-confirmed), wired 2026-09-04** into the Vendor Accounting Level 1 screen. Per-vendor flat rate card (deliberate user decision: NOT per-vehicle-type, a different granularity from the `KM_Slab_*` fields above — see `getRescueTicket/app/widget.html`'s own "Vendor Accounting" comment block for the flagged inconsistency). Was previously missing from this file — only documented in `getRescueTicket/README.md`'s changelog |
| `Vendor_Base_Location_Count` | Number of Base Locations | Number | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — how many of the 5 `Base_Location_N_Lat`/`Lon` pairs below are actually populated for this vendor |
| `Base_Location_1_Lat`/`_Lon` … `Base_Location_5_Lat`/`_Lon` | Base Location 1–5 Lat/Lon | Single Line ×10 | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — capped-fixed-fields approach (not a Subform — this app has never used one, chosen as the lower-risk option 2026-08-17), one pair per fleet base location. `getRescueTicket`'s Closure step picks whichever populated pair is nearest (crow-flight) to the ticket's breakdown location as `Vendor_Fleet_Location_Lat`/`Lon` |
| `Location_Last_Updated_At` | Location Last Updated | Date-Time | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — intended to be stamped by `vendorTicket`'s own location heartbeat alongside `Current_Latitude`/`Longitude`; the heartbeat shipped last turn does **not** write this yet — a follow-up patch to `pingCurrentLocation()` is needed once the Operations Manager Map View work starts |
| `Last_Online_At` | Last Online At | Date-Time | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — intended to be stamped on toggle-on by `onToggleClick()`, for the Operations Manager Map View's own "last online date/time" display |
| `Engagement_Type` | Own Mechanic / Tie-Up | Dropdown | `Own Mechanic`, `Tie-Up` | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. New — deliberately kept separate from the existing `Vendor_Type` (Individual/Fleet) above, a different classification; drives the Operations Manager Map View's own filter |

## Form 5 — `Technicians` (report: `Technicians_Report`)

Covers both independent technicians/drivers who log in directly, and technicians who belong to a Fleet vendor.

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `Technician_Name` | Technician Name | Single Line | — | Display label (**renamed from lowercase `technician_name`**) |
| `Email` | Email | Single Line | — | Matches the logged-in technician/driver to their own record |
| `Availability_Status` | Availability Status | Dropdown | `Online`, `Offline` | |
| `Fleet_Vendor` | Fleet Vendor | Single Line | — | This technician's owning fleet vendor, matched by name string (not a Lookup) in `vendorTicket`'s own hand-off panel |
| `Current_Latitude` / `Current_Longitude` | Current Latitude / Longitude | Single Line | — | 🔴 **NOT YET CREATED** as of 2026-08-17 — mirrors `Vendors_Report`'s own pair, see the cross-cutting `Technicians_Report` section further down |
| `Location_Last_Updated_At` | Location Last Updated | Date-Time | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. Mirrors `Vendors_Report`'s own field above, for `technicianTicket`/`driverTicket`'s own heartbeats |
| `Last_Online_At` | Last Online At | Date-Time | — | 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into any widget. Mirrors `Vendors_Report`'s own field above |

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

## Form 10 — `On_Demand_Invoicing` (report: `On_Demand_Invoicing_Report`, guessed by the same `_Report` suffix convention as every other form/report pair above — not yet independently confirmed)

New 2026-08-20, per the user-provided "Ondemand Workflow for Invoicing" PDF spec. One record per invoiced ticket, created by `getRescueTicket`'s own `maybeCreateOnDemandInvoice()` the moment a ticket is closed with `Closure_Status="Completed"` **and** actually has something to invoice (`Total_Service_Fee` for an `"ON DEMAND"` walk-in, `Rescue_Charge` for any other/billed `Client`, only when that figure is `>0`). This is Step 1 of the PDF's own two-stage plan ("create invoice first in Zoho Creator, then Zoho Books") — the Zoho Books-specific columns below stay blank until `sendToZohoBooks()` (in `getRescueTicket/app/widget.html`) is wired with real org ID/OAuth credentials.

| API Name | Label | Zoho Field Type | Options | Notes |
|---|---|---|---|---|
| `RSID` | RSID | Lookup → `Agent_Ticket_Report` | — | 🟡 **Created in Zoho 2026-08-20 (user-confirmed), now wired** into `maybeCreateOnDemandInvoice()`. Doubles as the duplicate-invoice guard — that function reads this report and skips creating a second record for a ticket that already has one |
| `Customer_Name` | Customer Name | Single Line | — | 🟡 Snapshotted from the ticket's own `Customer_Name` at invoice time, not a live link |
| `Customer_Phone` | Customer Phone | Single Line | — | 🟡 From `Phone_Number1` |
| `Customer_Email` | Customer Email | Email | — | 🟡 From `Email` — needed to actually send the invoice once Books is wired |
| `GST_Number` | GST Number | Single Line | — | 🟡 From the ticket's own `GST_Number` (blank unless `B2B_GST_Invoice_Required="Yes"`) |
| `GST_Name` | GST Name | Single Line | — | 🟡 From `GST_Name` |
| `GST_Address` | GST Address | Multi-Line | — | 🟡 From `GST_Address` |
| `Vehicle` | Vehicle | Single Line | — | 🟡 Resolved via `lookupNamesFor(r.Vehicle, OPTIONS.vehicles)` — same resolver the dashboard's own ticket cards use |
| `Service` | Service | Single Line | — | 🟡 `Service_Type` + the resolved Vehicle Issue name(s), e.g. `"TOW - Battery Dead"` |
| `Client` | Client | Single Line | — | 🟡 `"ON DEMAND"` for a walk-in, or the resolved corporate client's display name otherwise |
| `Billing_Basis` | Billing Basis | Dropdown | `Final Amount (On Demand)`, `Rescue Charge (Client)` | 🟡 Records which rule decided `Invoice_Amount`, so it's auditable rather than re-derived later |
| `Invoice_Amount` | Invoice Amount | Decimal | — | 🟡 The GST-inclusive amount actually invoiced |
| `Service_Charges` | Service Charges | Decimal | — | 🟡 `Invoice_Amount × 100 / 118`, rounded to 2 decimals — exact PDF formula |
| `CGST_Amount` | CGST Amount | Decimal | — | 🟡 `Service_Charges × 9%` |
| `SGST_Amount` | SGST Amount | Decimal | — | 🟡 `Service_Charges × 9%` |
| `Round_Off` | Round Off | Decimal | — | 🟡 `Invoice_Amount − (Service_Charges + CGST_Amount + SGST_Amount)` — can be positive OR negative (the PDF's own ₹999 worked example gives `+0.01`, not `0.00`) |
| `Place_Of_Supply` | Place of Supply | Single Line | — | 🟡 Defaults to `"Karnataka"` per the PDF; not currently overridden per-ticket |
| `Zoho_Books_Contact_ID` | Zoho Books Contact ID | Single Line | — | 🔴 **Not yet written** — populated once `sendToZohoBooks()` is wired with real credentials |
| `Zoho_Books_Invoice_ID` | Zoho Books Invoice ID | Single Line | — | 🔴 **Not yet written** — same as above |
| `Invoice_Number` | Invoice Number | Single Line | — | 🔴 **Not yet written** — the PDF's own "return Invoice Number to ticket" step also isn't implemented yet; needs a decision on which ticket-side field should mirror it (flagged in `getRescueTicket/README.md`'s own 2026-08-20 entry) |
| `Invoice_PDF_URL` | Invoice PDF URL | Single Line | — | 🔴 **Not yet written** |
| `Sync_Status` | Sync Status | Dropdown | `Pending`, `Created in Creator`, `Sent to Books`, `Failed`, `Emailed` | 🟡 Currently always written as `"Created in Creator"` by `maybeCreateOnDemandInvoice()` — the later states are for the not-yet-built Books push |
| `Sync_Error` | Sync Error | Multi-Line | — | 🔴 **Not yet written** — reserved for the Books push's own error message |
| `Emailed_At` | Emailed At | Date-Time | — | 🔴 **Not yet written** |

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
| `Agent_Report` | (agent identities, user-managed) | `Agent_Name` ✅, `Email` 🟡 |

**`Agent_Report` added 2026-08-05** (user request — show the agent's real name, not their login email, on new Remarks entries). Report name AND name field both **user-confirmed**: `Agent_Name` — the initial guess ("cloned from `Technicians_Report`, so it's probably still literally `Technician_Name`") turned out wrong, it was renamed after all. `Email` (used to match the logged-in agent to their record) is still an unconfirmed guess by analogy with every other master report in this app. `getRescueTicket` falls back to the login email itself if no match is found (including if `Email` turns out wrong) — see `agentDisplayName()`'s own comment; a console warning fires on every boot with no match, showing the first loaded record's real keys so a wrong `Email` guess can be corrected quickly.

**`User_Type` added 2026-08-17** (user-provided spec — Operations Manager role) — Dropdown, `Agent`/`Operations Manager`. 🟡 **Created in Zoho 2026-08-17 (user-confirmed)**, not yet wired into the widget. No role/permission concept existed anywhere in this app before this; once wired, `getRescueTicket` will read this off the same `Agent_Report` record `agentDisplayName()` already resolves by login email, and conditionally show the new Map View only for `Operations Manager` — every existing agent keeps seeing exactly what they see today.

**`User_Type` resolution given a fallback report 2026-09-07 (real production bug, user-directed fix)**: live console logs confirmed `Agent_Report` returns `403 Forbidden` for a real agent's own Portal login in production (development's shared/admin account has broader access, which hid this). `agentUserTypeFor()` still tries `Agent_Report` FIRST (unchanged, same as `agentDisplayName()`'s own name lookup) — only when that yields no match at all (exactly what the 403 produces) does it fall through to a **separate report, `My_Availability_Agent`** (same `Email`/`User_Type` field names assumed, not independently re-confirmed against this new report specifically — its own diagnostic log will show immediately if wrong), mirroring the exact "own login/toggle profile, Portal-readable" pattern `My_Availability_Vendor` already uses for vendors above. `agentDisplayName()`'s own name lookup was deliberately left reading only `Agent_Report`, no fallback — still likely broken in production by the same 403 until that report's Sharing permissions are separately fixed in Zoho (not a code fix, out of scope for this change).

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
| `Agent_Wizard_Step` | **Created in Zoho 2026-08-11 (user-confirmed)** — the wizard step `key` the agent was last on (e.g. `"acceptance"`), written on every step change so reopening the ticket resumes exactly there instead of guessing from `Status` alone. See `getRescueTicket/README.md`'s matching entry for why `Status` alone isn't enough. Written/read only by `getRescueTicket` | (hidden) | Agent | 🟡 (feature shipped in code 2026-08-11; field created same day, not yet independently verified against a live record) |

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
| `Booking_Fee_Transaction_ID` | BF Transaction ID — reference number the agent records against the booking-fee payment | text | Agent | 🟡 **Created in Zoho 2026-08-11 (user-confirmed)** — not yet independently verified live (added to the widget 2026-08-07, user request; never added to this file until a 2026-08-11 field audit caught the gap) |
| `Payment_Status` | Payment Status (`PAID`/`PENDING`/`NOT APPLICABLE`) — **confirmed 2026-08-03 via the live Zoho field config**: this is the ONE real field, also used (same values) at Step 8's own Payment screen | select | Agent, Technician, Driver, Vendor | ✅ (confirmed live 2026-08-03) |
| `Image_Upload` | Receipt Photo — **wired live 2026-09-02** (was never actually added to this step's field list before, despite this row's own note; now editable, mandatory whenever `Payment_Method` has a value and isn't `"Payment Gateway"`, per `reqFile()`) | file | Agent | ✅ **created in Zoho, user-confirmed 2026-09-10** (was 🔴 confirmed missing earlier the same day) |

**Dropped 2026-08-03**: `Total_Service_Fee1` ("likely-duplicate, relationship unconfirmed") — the Step 8 payment screen now just displays `Total_Service_Fee` read-only instead of a second field.

## Step 3 — Capture Locations (Agent)

**Real bug fixed 2026-08-04**: this step's own advance-gate (`canAdvance`) only ever checked that a *breakdown* location existed — a TOW ticket could advance to Assignment (and all the way to the vendor's own Loading screen) with **no drop location at all**, at which point the vendor's "Vehicle Picked" button stayed permanently disabled with no way back to this step to fix it short of reopening the ticket from the dashboard. Now requires both `Latitude`/`Longitude` AND `DropLocationLat`/`DropLocationLong` for a TOW ticket before advancing (RSR is unaffected — it never has a drop location).

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Send_Link_For_The_Breakdown_Location` | Send Breakdown Location Link (same field as Step 2's copy) | button | Agent | 🟡 |
| `Latitude` / `Longitude` | Breakdown coordinates (pasted as text, split via `parseTarget`) | Zoho: **Single Line Text** (✅ converted 2026-09-11); widget's own internal field type tag (`t:"decimal"`, purely display/render — a `step="any"` numeric-styled input) is unchanged and doesn't need to match Zoho's schema type | Agent, Technician, Driver, Vendor (read) | ✅ |
| `Send_Link_For_The_Drop_Location` | Send Drop Location Link, TOW only | button | Agent | 🟡 |
| `DropLocationLat` / `DropLocationLong` | Drop coordinates, TOW only | Zoho: **Single Line Text** (✅ converted 2026-09-11), same widget-tag note as `Latitude`/`Longitude` above | Agent, Driver, Vendor (read) | ✅ |
| `RT_KM` | Round-Trip KM, TOW only (agent-side Haversine calc) | number | Agent | 🟡 |
| `Location_Received_Time` | Timestamp the breakdown location was first captured — stamped once, never overwritten | (hidden, `extra()`) | Agent | ❓ (field name/type user-confirmed 2026-07-31, not independently re-verified live) |
| `Break_Down_Location1` | Display/navigation field the field-facing widgets link to (falls back to a `maps.google.com` link built from `Latitude`/`Longitude` if empty) | (display only) | Technician, Driver, Vendor | ❓ (not confirmed whether this is a real formula field or something else) |
| `Navigate_To_Drop_Link` | Drop-location display/navigation field, TOW only | (display only) | Technician, Driver, Vendor | ❓ |

## Step 4 — Vendor / Mechanic / Driver Assignment (Agent)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Preferred_Vendor` | Preferred Vendor? (Yes = show all vendors; No/default = geofenced) | radio | Agent | ✅ |
| `Vendors_Report.service_type` | **New field, created in Zoho 2026-09-02 (user-confirmed, exact lowercase API name)** — options `RSR`/`TOW`/`Both`. Build-plan item #3 (wireframe: "Employee list is filtered by service type: Mechanics for RSR, Drivers for Towing"). Gates `vendorsFor()`'s own candidate list before the existing geofence/priority sort — a vendor with this field blank still shows (missing data shows rather than silently vanishing, same philosophy as the rest of `vendorsFor()`) | select (on `Vendors_Report`, not `Create_Case`) | Agent (read, via `OPTIONS.vendors`) | ✅ (user-confirmed exact field name) |
| `Technicians_Report.service_capability` | **New field, created in Zoho 2026-09-02 (user-confirmed, exact lowercase API name)** — options `RSR`/`TOW`/`Both`, same purpose as `Vendors_Report.service_type` above but for `vendorTicket`'s own fleet-handoff technician list (`showHandoffPanel()`) — a fleet vendor with both RSR and TOW technicians can no longer hand a job to the wrong kind | select (on `Technicians_Report`, not `Create_Case`) | Vendor (read, fleet hand-off only) | ✅ (user-confirmed exact field name) |
| `Vendors1` | Vendors (multi-Lookup → `Vendors_Report`) — saves real record IDs (`useId:true`, via `idsFor()`), bare (not object-wrapped — an object-wrapping experiment was tried and reverted the same day, broke saving entirely) | Lookup (bare ID) | Agent, Vendor (read-only, dashboard matching only) | ✅ (Lookup type confirmed live 2026-07-31; bare-ID write shape confirmed live 2026-07-31) |
| `Assigned_Vendor` | A single Lookup → `Vendors_Report`, set by the vendor's own Accept action once they accept the ticket (distinct from `Vendors1`, which only holds the invited *candidates*). Displayed read-only in `getRescueTicket`'s `acceptance`/`acceptanceTow` stages. **Write-side implemented 2026-08-04** — `vendorTicket`'s `acceptService()` now sends `currentVendorRecord.id` (unblocked by the schema rebuild merging `My_Availability_Vendor` into `Vendors_Report`, so vendorTicket now always has its own `Vendors_Report` ID on hand). Also now read back by `vendorTicket`'s own `loadDashboard()` to implement "1st person to accept, task goes away from other invited vendors' apps" (Action 4 mock) — falls back to the old "show any invited candidate" behavior for any ticket accepted before this existed (no `Assigned_Vendor` recorded yet) | Lookup (single) | Agent (view-only), Vendor (write on Accept, read in dashboard filtering) | 🟡 (write shape follows the same bare-ID convention as every other Lookup here; best-effort only — two vendors accepting within the same instant could both still write it, no server-side lock available from a plain widget) |
| `Vendor_Emails` | Auto-populated (comma-separated) with the email of every vendor selected in `Vendors1`, resolved from `Vendors_Report` via `CONFIG.vendorEmailField` — not shown anywhere in this widget's own UI. Used by `vendorTicket`'s own dashboard matching as a more reliable alternative to name-matching | text (hidden) | Agent (write, silent) | 🟡 |
| `Vendor_Email` | Vendor Email — shown once someone's actually accepted, not something the agent fills in upfront | text | Agent (view-only, Acceptance step) | 🟡 |
| `Assigned_Technician` | Assigned Technician | select | Agent (view-only, Acceptance step), Vendor (write, on fleet hand-off) | 🟡 |
| `Assigned_Technician_Email` | Technician Email | text | Agent (view-only, Acceptance step), Technician, Driver, Vendor | ❓ (dashboard-matching candidate, not confirmed which field a given ticket actually uses) |

## Step 5 / 5a — Service Acceptance & Rejection (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Acceptance_Mode` | Agent-only radio: "Vendor Accepting via App" (default) vs "Agent Entering on Behalf of Vendor" — added 2026-08-07, drives `seedAcceptanceVendorFields()`'s auto-fill of `Assigned_Vendor`/`Vendor_Email`*. **Never independently confirmed as a real `Create_Case` field** — added directly to this widget's own STAGES without a matching FIELDS.md entry until now. **Under live suspicion 2026-08-11**: user reports the selection reverts after leaving and reopening a ticket, exactly the symptom a nonexistent field would produce (Zoho silently drops unrecognized payload keys rather than erroring). Diagnostic logging added to `saveStep()`/`openTicket()` to confirm — if `openTicket`'s own console dump shows this key entirely absent from the raw Zoho record after a save that included it, the field needs to be added to `Create_Case` in Zoho Studio (Radio or Single Line, exact option text `"Vendor Accepting via App"` / `"Agent Entering on Behalf of Vendor"`) | radio | Agent only | ❓ (unconfirmed, likely missing from Zoho) |
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
| `Odometer_Reading` | Start odometer (TOW, captured at Accept). **Fixed 2026-09-02** — used to be reused at Step 7b for the drop-side reading too; now Start-only | number | Driver, Vendor | ❓ |
| `Navigate_To_Drop_Link` | Drop location display (see Step 3). **Also now shown in the field apps' own ticket-info header (Case ID/Customer/Location/etc.) as of 2026-09-02 — previously only appeared as a map-link button on the Reached-Drop screen** | (display only) | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.RSP_Name` / `.Reason` / `.Rejected_At` / `.Vehicle_Issue` | Rejection-log fields — best-effort, non-blocking write | text/datetime | Technician, Driver, Vendor | ❓ |
| `Task_Rejections.RSID` | **Renamed 2026-08-03 from `Task_Rejections.Case_ID`** to match `Invites`' own naming — a real Lookup field, sends the ticket's own record ID (`r.ID`) | Lookup (bare ID) | Technician, Driver, Vendor | 🟡 |
| `Task_Rejections.Vehicle` | Also a real Lookup field — sends the raw `r[CONFIG.vehicleField]` value as fetched from `Create_Case` (unconverted) | Lookup (bare, unconverted) | Technician, Driver, Vendor | ❓ (needs live verification) |

**Dropped 2026-08-03**: `Service_Request`, `Service_Request1`, `Assign_Technican`, `Assign_Technican1` — synthetic/display-only fields no widget ever actually wrote (they set `Status` directly instead).

## Step 6 / 6a — Reach Breakdown Location & Cancel (Technician / Driver / Vendor)

| Field | Label | Type | Widgets | Status |
|---|---|---|---|---|
| `Navigation_Link` | Navigation Link (RSR) | textarea (no writer — see below) | Agent (view-only) | 🟡 |
| `Navigate_To_Breakdown_Link` | Navigate to Breakdown Link (TOW) | textarea (no writer — see below) | Agent (view-only) | 🟡 |
| `Image_Upload2` | Arrival Photo (RSR) / also reused as the RSR Cancel photo | file | Agent (view-only), Technician, Vendor | 🟡 |
| `Image_Upload5` | Arrival Photo (TOW) / also reused as the TOW Cancel photo | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Cancel_Reason` | Cancel Reason (RSR) | select | Agent (view-only), Technician, Vendor | 🟡 |
| `Reject_Reason2` | Reject/Cancel Reason (TOW Reach step) | select | Agent (view-only), Driver, Vendor | 🟡 |
| `Reach_Time` | Reach/arrival timestamp (both branches) | datetime | Technician, Driver, Vendor | ❓ |
| `Reach_Location_Lat` / `Reach_Location_Lon` | The tech/driver/vendor's own GPS position at the moment Reach/Reached is clicked — **new 2026-08-11** (user request, "Missing Fields & Field Behavior" spec; Reach never captured a live GPS fix before this). Text (not Decimal), same reasoning as `RSP_Start_Latitude`/`Longitude`. Shared by both branches | text | Technician, Driver, Vendor | 🟡 — created in Zoho 2026-08-11 (user-confirmed), not yet independently verified live |
| `Distance_Reach_To_Breakdown` (RSR) / `Distance_Reach_To_Breakdown_TOW` (TOW) | One-way distance from the Reach-click position (just above) to the ticket's own static breakdown location — **new 2026-08-11**, deliberately separate from `Roundtrip_Distance` just below (that one stays Office→Breakdown→Office, untouched) | decimal | Technician, Driver, Vendor | 🟡 Type corrected to Decimal in Zoho 2026-08-11 (user-confirmed) — see Part 1's own matching row for the full story; not yet re-tested live since the correction |
| `Roundtrip_Distance` | Round-trip distance in km — set here for RSR (office→breakdown→office Haversine), set again at Step 7b for TOW's 4-point version | number | Agent (view-only), Technician, Driver, Vendor | 🟡 |
| `Odometer_reading_at_Reached_Location` | Odometer at Reached (TOW) | number | Agent (view-only), Driver, Vendor | 🟡 |
| `RSP_Start_Latitude` / `RSP_Start_Longitude` | The tech/driver/vendor's own GPS position at Accept time (Step 5), used to compute the Step 6 Cancel distance. Text (not Decimal) to sidestep a real "exceeded maximum digits" error on raw GPS coordinates. Reused 2026-08-04 (user request) at Step 7b's own "Reached Drop Location" for TOW — see that step's own row below, now removed in favor of this same field | text | Technician, Driver, Vendor | 🟡 (user-renamed 2026-08-03) |
| `Cancellation_Time` | Timestamp of the Cancel Task click | datetime | Technician, Driver, Vendor | ❓ |
| `Cancel_Location_Lat` / `Cancel_Location_Lon` | GPS position at the moment Cancel is confirmed | Zoho: **Single Line Text** (✅ converted 2026-09-11) | Technician, Driver, Vendor | ✅ |
| `Cancel_Distance` | Distance in km from `RSP_Start_Latitude`/`Longitude` to `Cancel_Location_Lat`/`Lon` — deliberately a different field from Step 5's own `Distance_To_Breakdown` | number | Technician, Driver, Vendor | ❓ |
| `Distance_Cancel_To_Breakdown` | Distance in km from `Cancel_Location_Lat`/`Lon` (just above) to the ticket's own static breakdown location — **new 2026-08-11** (user request, spec item 2; no field for this existed under any name before). Deliberately a different field from `Cancel_Distance` just above | decimal | Technician, Driver, Vendor | 🟡 Type corrected to Decimal in Zoho 2026-08-11 (user-confirmed), same fix as `Distance_Reach_To_Breakdown` — not yet re-tested live |
| `Toll_Charges` | Toll Charges? (TOW Reach step) | radio | Agent (view-only), Driver, Vendor | 🟡 |
| `Agent_ETA_Override` (RSR) / `Agent_ETA_Override_TOW` (TOW) | Agent's own ETA override (mins), Agent-entering mode only | number | Agent | 🟡 — created in Zoho 2026-08-11 (user-confirmed; this row existed since 2026-08-07 but was missing that marker, a documentation gap caught during a later field audit). Not yet independently confirmed against a live record showing real data. **Deliberately a separate field from `ETA`/`ETA_TOW`** (see that field's own row in the Step 5/5a table above) — `expectedArrivalTime()` prefers this value when set (for what actually gets sent to the customer via `Send_ETA_To_Customer`), but saving it never writes to `ETA`/`ETA_TOW` — confirmed via a dedicated 2026-08-11 audit that the two never share a save payload |

**`Navigation_Link`/`Navigate_To_Breakdown_Link` fixed 2026-08-04**: neither field ever had a writer — no widget populates them, so `getRescueTicket`'s own Reach screen always showed them blank even though the field-facing widgets show a working "🧭 Navigate to Breakdown" link. `getRescueTicket` now renders these as an actual clickable link too (new `navlink` field type), built the same way the field-facing widgets already do: `Break_Down_Location1` if present, else a `maps.google.com` URL from the ticket's own `Latitude`/`Longitude`. Computed, not stored — nothing is written back to `Navigation_Link`/`Navigate_To_Breakdown_Link` themselves.

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
| `Odometer_reading_at_Drop_Location` | Odometer reading at drop — a plain number, not a photo upload. **New field, added 2026-09-02 (user request)** — previously reused `Odometer_Reading` (Step 5/5a's own start-odometer field), silently overwriting it; now its own field so both readings persist. **Created in Zoho 2026-09-02 (user-confirmed)** | number | Agent (view-only), Driver, Vendor | 🟡 user-confirmed created, not independently re-verified via a real record |
| `Drop_Location_Photo` | Drop Location Photos — 4 mandatory | file | Agent (view-only), Driver, Vendor | 🟡 |
| `Toll_Charges1` | Toll Charges? (drop side) | radio | Agent (view-only), Driver, Vendor | 🟡 |
| `Office_To_Mechanic` / `Mechanic_To_Breakdown` / `Breakdown_To_Drop_Distance` / `Drop_To_Office_Distance` | The 4-point roundtrip's individual legs — really a 3-leg Office→Breakdown→Drop→Office trip (no separate "Mechanic" waypoint exists), `Mechanic_To_Breakdown` is hardcoded `0` — field *names* kept as-is to avoid an unnecessary rename of already-shipped, cross-referenced fields | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Roundtrip_Distance` | Total of the 4 legs | number | Agent (view-only), Driver, Vendor | 🟡 |
| `Drop_Location_Arrival_Time` | Timestamp when "Reached Drop Location" is tapped | datetime | Agent (view-only), Driver, Vendor | ❓ (guessed name) |
| ~~`RSP_Drop_Location_Lat` / `RSP_Drop_Location_Lon`~~ | **Removed 2026-08-04 (user request)** — the driver/vendor's own live GPS fix at that same click now reuses `RSP_Start_Latitude`/`RSP_Start_Longitude` (Step 5's own row above) instead of this separate, never-confirmed pair. Still separate from the ticket's static `DropLocationLat`/`DropLocationLong` (the customer-specified drop point, still used for the roundtrip calc) | — | — | — |

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
| `Upload_Photo_of_receipt` | Label "Remaining Fee Receipt Photo" as of 2026-09-02 (build-plan item #8, agent side only — see that field's own note) — mandatory for every Payment Method now — both a live-camera capture (GPS/time watermark) and a plain gallery picker are offered | file | Agent, Technician, Driver, Vendor | 🟡 |
| `Payment_Status` | The SAME field as Step 2's own `Payment_Status` (`PAID`/`PENDING`/`NOT APPLICABLE`) — not a separate `Pending`/`Success` field as originally guessed. Technician/Driver/Vendor auto-flip it to `PAID` once a receipt photo is attached, or read it as already `PAID` if some other real integration set it; "Payment Received" is only enabled once this is `PAID` or the balance is zero | select | Agent, Technician, Driver, Vendor | ✅ (confirmed live 2026-08-03) |
| `Transaction_ID` / `Time_Of_Receipt` | The mock's own "transaction ID, time of receipt" once a real Payment Gateway returns success — **not implemented**, no real gateway webhook exists to source these from honestly | text/datetime | — | ❓ (not set by any widget yet) |

## Final Closure (Agent only)

Reachable at any point in a ticket's timeline (dashboard row action + a permanent wizard button), not gated by the stepper. Every field from every step above is also shown here (read-only display, or click-to-edit — see `getRescueTicket/README.md`'s Final Closure entries for the full mechanism). Fields unique to this screen:

**"Distance & ETA Metrics" group (added 2026-08-04, per user request)**: `ETA`, `Distance_To_Breakdown`, `ETA_TOW`, `Distance_To_Breakdown_TOW`, `Cancel_Distance`, `Office_To_Mechanic`, `Mechanic_To_Breakdown`, `Breakdown_To_Office`, `Roundtrip_Distance`, `Breakdown_To_Drop_Distance`, `Drop_To_Office_Distance`, `Return_Journey_ETA` — all removed from their previous per-stage wizard display (`reach`/`wip`/`wipUnloadingTow`) and shown **only** in this dedicated, read-only Final Closure group, not tied to any wizard step. These are all Google Maps/Haversine-computed by the vendor/technician/driver widgets, not agent input — see each field's own Step-N row above for where it's written.

**`_ETA_Met_Status` (added 2026-09-02, build-plan item #7)** — synthetic, computed, nothing stored in Zoho at all (`t:"computed"`, the first one ever shown in this summary — `closureFieldValueDisplay()`/`closureFieldRowHtml()` both got a new branch for this). Compares the promised arrival deadline (Accept time + ETA, Agent Override taking priority when set — same resolution `expectedArrivalTime()` already uses for the customer-facing message, just returning the raw `Date` instead of a display string via the new `expectedArrivalDeadline()`) against `Reach_Time`. Shows "✅ ETA Met" / "❌ ETA Missed" / a "not available yet" hint when there isn't enough data (no ETA captured, unparseable accept time, or the vendor hasn't reached yet). Rendered as a plain, non-clickable row — deliberately not the usual clickable `.crow` — since there's nothing real underneath it to open an editor for.

**"Field Activity Log" group (added 2026-08-04, per a user audit request — "all data we are saving through process should be visible in Final Closure")**: `Service_Acceptance`, `Service_Acceptance_For_Tow`, `Service_Reject_Time`, `Reach_Time`, `Pickup_Time`, `Cancellation_Time`, `Cancel_Location_Lat`/`Lon`, `RSP_Start_Latitude`/`Longitude`. An agent-facing audit turned up these 8 fields being written by `technicianTicket`/`driverTicket`/`vendorTicket`'s own Accept/Reject/Reach/Cancel actions but never shown *anywhere* in `getRescueTicket` before — not in a wizard stage, not in Final Closure. Read-only, shown regardless of confirmation status (all 8 are still guessed field names per the writing widgets' own comments, same ❓ confidence as everything else unconfirmed in this file) — if a name turns out wrong, the row will just always read "—", which is itself a useful signal. `RSP_Start_Latitude`/`Longitude`'s own row was relabeled "RSP Location — Accept/Drop" the same day, once it started being reused for TOW's own drop-side GPS fix too (see its Step 5 row above) — this one row now shows whichever of the two fixes was written most recently.

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

### `Vendors_Report` (vendor master)

**2026-08-05 (user request): the online/offline toggle no longer lives here** — the 2026-08-03 unification below didn't work for the vendor portal, so the toggle was moved back to its own separate `My_Availability_Vendor` report (`Email`/`Availability_Status`, same field names, matched independently — its record id is never used for ticket matching). Everything else in this section is unaffected.

| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `vendor_name` | Display label — **corrected 2026-08-04, confirmed live**: real field is lowercase, the planned `Vendor_Name` rename never landed | Agent, Vendor | ✅ (confirmed live 2026-08-04) |
| `Email` | Match to logged-in vendor | Vendor | ✅ (mechanism), 🟡 (exact field) |
| `Mobile_Number_01` | Phone — **corrected 2026-08-04, user-confirmed**: real field is `Mobile_Number_01`, not `Phone` | Agent | ✅ (confirmed live 2026-08-04) |
| `Vendor_Priority` | Assignment sort priority — **corrected 2026-08-04, confirmed live**: real field is `Vendor_Priority`, not `Priority` | Agent | ✅ (confirmed live 2026-08-04) |
| `Availability_Status` | `Online`/`Offline` | Agent, Vendor | ✅ (confirmed live 2026-08-04) |
| `Address` (composite) | Vendor's own location — **corrected 2026-08-04, confirmed live**: NOT flat `Latitude`/`Longitude` fields, lives nested as `Address.latitude`/`Address.longitude` inside Zoho's own composite Address field type | Agent | ✅ (confirmed live 2026-08-04) |
| `Vendor_Type` | Individual vs. Fleet | Vendor | 🟡 (drives `IS_FLEET`) |
| `Current_Latitude` / `Current_Longitude` | **New 2026-08-17** (user request: "when vendor is online, show distance from the vendor's current location (latest location from app); when offline, show distance from the vendor's static location (as in vendor table)"). Written every ~2.5 min by `vendorTicket`'s own heartbeat (`pingCurrentLocation()`/`startLocationHeartbeat()`) while the vendor is toggled Online — via `My_Availability_Vendor`, same underlying `Vendors_Report` record. Read by `getRescueTicket`'s `vendorInfo()`/`vendorLiveLatLon()`, preferred over the static `Address` above only when the vendor's `Availability_Status` is `Online` **and** a live fix has actually landed; falls back to static `Address` otherwise (never blank for an online vendor with no fix yet). **Type: Single Line Text, not Decimal** — same digit-limit reasoning as every other raw GPS field in this app (see `RSP_Start_Latitude`/`Reach_Location_Lat` etc. above) | Agent (read), Vendor (write, silent heartbeat) | 🔴 **NOT YET CREATED** — code degrades safely (falls back to static `Address`) until these exist in Zoho Studio |

### `Technicians_Report` (technician/driver master)
| Field | Purpose | Widgets | Status |
|---|---|---|---|
| `Technician_Name` | Display name (renamed from lowercase `technician_name`) | Technician, Driver, Vendor | 🟡 |
| `Email` | Match to logged-in user | Technician, Driver, Vendor (fleet hand-off) | 🟡 |
| `Availability_Status` | `Online`/`Offline` | Technician, Driver | 🟡 |
| `Fleet_Vendor` | Links a technician back to their owning fleet vendor (matched by name string, not ID) | Vendor | ❓ (guessed) |
| `Current_Latitude` / `Current_Longitude` | **New 2026-08-17** — same feature/rationale as `Vendors_Report`'s own pair just above, mirrored for technicians/drivers per user's explicit scope confirmation ("Vendors + Technicians + Drivers"). Written every ~2.5 min by `technicianTicket`'s and `driverTicket`'s own heartbeats while toggled Online. Not yet read anywhere in `getRescueTicket` (that widget's Assignment step only ever picks a *vendor*, not a technician/driver directly) — added now so the schema is ready if/when technician-direct-assignment location display is needed | Technician, Driver (write, silent heartbeat) | 🔴 **NOT YET CREATED** |

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
| `SERVICE FEE QUOTE` | Step 1 (Create Case, on save) — **changed 2026-08-06**: written regardless of `Time_of_service`; a "LATER" ticket used to jump straight to `SCHEDULED` here (added 2026-08-04), before Booking Fee or Location were even touched, which put it on the dashboard's "Ready & Invited" bucket while neither was settled. Now goes through the exact same pipeline as a "NOW" ticket. | Step 2 |
| `BOOKING FEE PENDING` | Step 2 | Step 2 |
| `LOCATION REQUEST` | Step 2 (fee-free, link not yet sent) or Step 3 (own default) | Step 3 |
| `LOCATION PENDING` | Step 2 (once link sent / paid) | Step 3 |
| `SCHEDULED` | Step 3 — **moved here 2026-08-06** (from Step 1, see above). Written once Booking Fee AND Location are BOTH settled (Step 3 only ever reaches this point once `locationReady()` is true) AND the ticket's own `Computed_Service_Time` is still more than 2 hours away. **Resume target changed 2026-08-11** (user-reported live: reopening landed on Step 3 instead of Step 4) — now resumes at Step 4 like `READY FOR ASSIGNMENT` below, not Step 3. Step 4 (Assignment) has no gate of its own preventing an early invite on a still-far-future ticket; the auto-promotion feature (client-side `autoPromoteScheduledTickets()` + the optional Deluge Scheduler) is what actually flips this to `READY FOR ASSIGNMENT` once the 2-hour window closes, independent of anyone reopening the ticket. | Step 4 |
| `READY FOR ASSIGNMENT` | Step 3 (once a breakdown location exists, AND — 2026-08-06 — the service time is ≤2 hours away, or `Time_of_service = NOW`) | Step 4 |
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

## WhatsApp template catalog (added 2026-09-06)

No single catalog of WhatsApp template names existed anywhere in this repo before now (confirmed via a full audit across all four widgets that ever touch messaging). User provided a 13-template spec (`1_MESSAGE_TEMPLATES.pdf`) plus RSR/TOW trigger-logic PDFs (`2_LOGIC_REPAIR.pdf`/`3_LOGIC_TOW.pdf`). This table maps the spec's own numbering to the actual template-name identifiers used in code, so future work stays consistent rather than each widget inventing its own naming.

All send calls go through `sendWhatsAppTemplate(phone, templateName, params)` → Custom API `sendWhatsAppMessage` (defined once in `getRescueTicket/app/widget.html`, copied verbatim into `technicianTicket`/`driverTicket`/`vendorTicket` on 2026-09-06 since none of the three had any WhatsApp code before that).

| Spec # | Purpose | Template name in code | Params | Status |
|---|---|---|---|---|
| 1 | Initiate conversation | — | — | ❌ Not sent as a distinct message anywhere |
| 2 | Automated-chat disclaimer note | — | — | ❌ Not sent anywhere |
| 3 | Booking fee + total fee | `booking_fees_link` | fee amount, fee/link text, **+ link as its own param3 (added 2026-09-06)** | ⚠️ Sent via `getRescueTicket`'s `Send_Payment_Link`/`Send_Payment_Link1` — exact text match to the real approved template unconfirmed. `param3` added additively alongside the already-working embedded-link text in param2, in case the real template has grown a 3rd placeholder |
| 4 | Customer location details | `location` | customer name (+ optional page link) or ETA string (reused, see caveat below) | ✅ Implemented, `getRescueTicket` only |
| 5 | Confirmation (READY, prev ≠ SCHEDULED) | — | — | ❌ Not implemented |
| 6 | Scheduled confirmation | — | — | ❌ Not implemented |
| 7 | Cancellation policy (paired with 5 or 6) | — | — | ❌ Not implemented |
| 8 | Dispatch + ETA, vendor/mechanic name **and phone number** | `dispatch_eta` | name, phone, ETA (order is a best-effort guess) | ✅ **Implemented 2026-09-06** in `technicianTicket`/`driverTicket`/`vendorTicket`'s `acceptService()` — `Technicians_Report.Phone` (user-confirmed) and `Vendors_Report.Mobile_Number_01` (already confirmed) resolved at boot into `MY_TECH_PHONE`/`MY_VENDOR_PHONE`. **Not yet implemented** in `getRescueTicket`'s own "agent entering on behalf of vendor" Accept path — deferred, that stage can assign a vendor or a specific fleet technician and needs its own careful pass |
| 9 | "Contact call center" note, 5 min after 8 | — | — | ❌ Not implemented — needs a delayed/scheduled-send mechanism, none exists in this codebase yet |
| 10 | Technician on-site / Reach | `reach_time` | none (no visible parameter slots in the spec's own template text) | ✅ **Implemented 2026-09-06** in `technicianTicket`/`driverTicket`/`vendorTicket`'s Reach handlers — template name is a best-effort guess, not confirmed against a real approved WhatsApp Business template |
| 11 | Service complete | — | — | ❌ Not implemented in any field app. `getRescueTicket`'s reach-step "Send ETA to Customer" button currently reuses the `location` template with an ETA string as its param — a likely mismatch/miswiring against the real templates 8/10/11, not yet corrected |
| 12 | Balance payment link | `booking_fees_link` (reused) | remaining fee, fee/link text | ⚠️ Sent via `getRescueTicket`'s `Send_Payment_Link1`, but unconditionally — the spec's Remaining-Fee=0-vs->0 message-count branching (11 alone vs. 11+12) is not implemented; also field apps have no payment-link-generation capability to send this themselves |
| 13 | Google feedback | `google_feedback_link` | customer name, `GOOGLE_REVIEW_LINK` | ✅ Implemented in `getRescueTicket`, wired to the `Google_Feedback_Requested` checkbox on Agent Closure — currently a guarded no-op since `GOOGLE_REVIEW_LINK` is still an empty placeholder |
| — | Customer feedback (not in the 13-template spec — built earlier this session for the `customerPage` widget) | `customer_feedback_link` | customer name, customer-page feedback link | ✅ Implemented in `getRescueTicket`, same trigger pattern as 13 |

**Known naming caveat (template 4)**: `getRescueTicket`'s two "Send ETA to Customer" call sites reuse the `location` template with an ETA string in the slot the rest of the app uses for a customer-page link — this doesn't match any of the 13 spec templates cleanly and is flagged as a likely miswiring, not yet corrected.

---

## Open questions this file doesn't resolve on its own

See `getRescueTicket/QUESTIONS_TO_ASK.md` for the current live list (Individual vs. Fleet field, the "Mechanic" waypoint, App Status vs. Login Status, and others) — not duplicated here to avoid two copies drifting apart. Fix the field name **here** once an answer comes in, then that question can be checked off there.
