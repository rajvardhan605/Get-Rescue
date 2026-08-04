# Get-Rescue Driver Widget

Field-facing Zoho Creator widget for the **TOW** on-road flow. Sibling of `technicianTicket` (RSR) — same architecture, CONFIG shape, camera/GPS watermark modal, and Haversine distance/ETA stand-in, just the TOW-specific screens (Accept/Reject → Reach → Loading → Reached Drop Location → Unloading & Handover → Final Payment) instead of RSR's.

See `technicianTicket/README.md` for the full set of shared, user-confirmed decisions this build relies on (Create_Case-only data source, no Google Maps key yet, toggle folded in, camera/GPS best-effort, etc.) — not repeated here. See `getRescueTicket/README.md` / `getRescueTicket/QUESTIONS_TO_ASK.md` for the underlying flow-spec reconciliation both widgets were built from.

**Field names**: see **`../FIELDS.md`** (repo root) — the single source of truth for every field API name across all Get-Rescue widgets, organized step-by-step. Corrections happen there first.

**Zoho access**: see **`../ACCESS.md`** (repo root) — which Forms/Reports this widget needs permission to.

## TOW-specific notes

- **Dashboard** matches `Service_Type=="TOW"` (instead of `"RSR"`), against `MY_STATUSES` covering the full TOW lifecycle: `RSP IRA (INVITE RESPONSE AWAITED)` → `RSP ON THE WAY` → `LOADING` → `TO DROP LOCATION` → `UNLOADING` → `REMAINING FEE DUE`.
- **The 4-point roundtrip** (`calc4PointRoundtrip()`, used on "Reached Drop Location"): the field-generation spec names four legs — Office→Mechanic→Breakdown→Drop→Office — implying a distinct "Mechanic" waypoint separate from both the office and the breakdown site, which has no corresponding location field anywhere in this codebase. Implemented as a 3-leg calculation (Office→Breakdown, Breakdown→Drop, Drop→Office) with `mechanicToBreakdown` hardcoded to `0` — flagged in `getRescueTicket/QUESTIONS_TO_ASK.md`, not silently assumed away.
- **"Vehicle Picked" (Loading screen)** is disabled until `Navigate_To_Drop_Link` has a value, per the spec's own button-state rule.
- **Handover fields** (`Handover_to_Name`/`Handover_to_Number`/`Handover_to_Designation`) are all required before "Dropped" can be confirmed, matching the spec's own "Validate all inputs and images."

## 2026-07-31 (later) — Action 5 reconciliation

Same round of fixes as `technicianTicket` (see its own README's matching entry, not repeated here): in-place navigation between screens via `goToStatus()` instead of always dropping back to the dashboard, a distinct `Service_Reject_Time` field, a "Back" button on the Reject panel, an expanded `Task_Rejections` log, and (at the time) two unresolved client-source conflicts around ETA/Distance field naming and the Reject status value — see `getRescueTicket/QUESTIONS_TO_ASK.md`.

## 2026-07-31 (later still) — Action 6 reconciliation

Same round of fixes as `technicianTicket` (see its own README's matching entry, not repeated here): ETA now shown/editable on the Reach screen, a "Back" button on the Cancel panel, and the new `RSP_Start_Lat`/`RSP_Start_Lon` (captured at Accept) + `Cancellation_Time`/`Cancel_Location_Lat`/`Cancel_Location_Lon`/`Cancel_Distance` (captured at Cancel) fields — all guessed names, see `FIELDS.md`.

## 2026-07-31 (yet later) — Action 8 reconciliation

Same explicit `Payment_Status` (Pending/Success) fix as `technicianTicket` (see its own README's matching entry, not repeated here) — the button-enable logic now also honors an already-`Success` record so a future real gateway integration "just works" without touching this code again.

## 2026-07-31 (yet later still) — Action 5a (TOW) confirms this screen as-is

The Action 5a mock (Service Acceptance/Rejection — TOW) is this widget's own Accept/Reject screen, and it matches what was already built almost exactly (Start Odometer, Back button, `Service_Reject_Time`, `RSP_Start_Lat`/`RSP_Start_Lon`, `Task_Rejections` log) — no changes needed. It also resolves the Action 5 Reject-status question: `Status:"RSP REJECT"` is confirmed correct, per `getRescueTicket/QUESTIONS_TO_ASK.md`.

## 2026-07-31 (later again) — Action 6a (TOW) confirms this screen as-is

The Action 6a mock (Reach Breakdown Location / Cancel — TOW) matches this widget's Action 6 screen field-for-field (`Odometer_reading_at_Reached_Location`, `Toll_Charges`, `Reach_Time`, mandatory photo on both Reached/Cancel, `Status:"RSP CANCELLED"` on Cancel, the `Cancellation_Time`/`Cancel_Location_Lat`/`Lon`/`Cancel_Distance` capture) — no code changes needed. Its repeat of "RSP Reach time"/"RSP distance" phrasing (identical to the Action 6 RSR mock) further reinforces treating that as generic language, not a literal field prefix.

## 2026-07-31 (later again still) — Action 7a (TOW) reconciliation: added the missing Cx-Reject panel, live-GPS drop distance

The Action 7a mock (WIP - Loading / Cx Reject — TOW) revealed this screen was missing an entire outcome: it only had "Vehicle Picked," with no "Customer Rejected" path at all. Added a `Customer Rejected` button opening a Cx-Rejection panel (Cx Rejection Reason dropdown, Back button, "Service Reject" button) that writes the same shared `Rejection_Reason`/`RSP_Completion_Time` fields as `technicianTicket`'s own RSR WIP Cx-reject and sets `Status:"REMAINING FEE DUE"`.

Also: `vehiclePicked()` now computes the Drop distance/ETA from the driver's own **live GPS position** at click time (falling back to the ticket's static breakdown Lat/Lon only if unavailable), per the mock's explicit "based on current GPS location of Driver" — previously always used the static location. Field-name conflict flagged, not renamed: the mock calls these two fields "Drop Distance"/"DROP ETA," different from the already-shipped `Breakdown_To_Drop_Distance`/`Return_Journey_ETA` — see `getRescueTicket/QUESTIONS_TO_ASK.md`.

This round also surfaced (and fixed) a real bug in the sibling `getRescueTicket` agent widget: its `wipLoadingTow`/`reach`/`reachTow`/`acceptance`/`acceptanceTow` stages were deriving the ticket's current status from synthetic fields none of these three widgets ever write — see `getRescueTicket/README.md`'s own matching entry.

## 2026-07-31 (later again yet still) — Action 7b (Reached Drop Location) & Action 7c (Unloading) reconciliation

**Action 7b** resolves the long-open "Mechanic waypoint" question: the mock's own footnote describes the roundtrip as "RESCUE[static] to Breakdown Location to Drop Location to RESCUE[static]" — a plain 3-leg trip, no separate Mechanic location — which is exactly what `calc4PointRoundtrip()` already computes (see `getRescueTicket/QUESTIONS_TO_ASK.md`), so no calculation change was needed. Two new fields were added, both stamped in `reachedDrop()`: `Drop_Location_Arrival_Time` (click timestamp) and `RSP_Drop_Location_Lat`/`RSP_Drop_Location_Lon` (this widget's own live GPS fix at that moment, via `getPositionSafe()`) — separate from the ticket's static `DropLocationLat`/`Long` (the customer's specified drop point, still used for the distance calc).

**Action 7c** added the previously-missing `RSP_Completion_Time` stamp to `confirmDropped()` (per "Capture Time of click of 'DROPPED' button as the 'work complete time'" — same shared field as Steps 7/7a), and corrected `Handover_to_Number` from a `tel`-type text box to a real number input, per the mock's explicit "[Number field]".

## 2026-07-31 (later again yet still further) — Task_Rejections form built; Case_ID/Vehicle corrected to Lookup writes

Same fix as `technicianTicket`'s own matching entry (see its README) — `Case_ID`/`Vehicle` in the now-built `Task_Rejections` form are real Lookups, so `rejectService()` now sends `r.ID`/`r[CONFIG.vehicleField]` instead of display strings.

## 2026-07-31 (later still again) — Real bug fixed: drivers couldn't see their tickets at all (invalid `max_records`)

Same bug and fix as `technicianTicket`'s own matching entry (see its README) — `loadAvailability()` was calling `getRecords` with `max_records:1`, which Zoho rejects (only 200/500/1000 allowed, error 9250), breaking `boot()` before it ever reached `loadDashboard()`. Fixed to `200`.

## 2026-07-31 (later again yet still further once more) — Dashboard matching also checks the new hidden `Technician_Emails` field

Same fix as `technicianTicket`'s own matching entry (see its README) — `isMine()` here now also checks the new `Technician_Emails` field.

## 2026-07-31 (later again yet still further once more, again) — Lookup write shape: tried an object wrapper, reverted same day

Same as `technicianTicket`'s own matching entry (see its README) — `toLookupRef()`/the `{ID}` object wrapping for `Case_ID`/`Vehicle` was tried and reverted the same day; back to a bare `r.ID` / the unconverted raw `Vehicle` value.

## 2026-08-03 — New "Invites" architecture: dashboard now finds "my tickets" via Invites_Report first

Same architecture change as `technicianTicket`'s own matching entry (see its README) — this widget now tracks `MY_TECH_NAME` and matches `Invites_Report` records the same way, falling back to the older email match. `acceptService()`/`rejectService()` write to this driver's own Invites record via `updateMyInvite()`.

## 2026-08-03 — Real bug fixed: "RSP_Start_Lon has exceeded its maximum digits" on Accept

Same fix as `technicianTicket`'s own matching entry — `getPositionSafe()` now rounds to 15 decimal places (updated same day from an initial 6, once the user widened the Decimal fields) via `toFixed(15)` at the source.

## 2026-08-03 (later) — `RSP_Start_Lat`/`RSP_Start_Lon` changed to Text, renamed `RSP_Start_Latitude`/`RSP_Start_Longitude`

Same change as `technicianTicket`'s own matching entry — renamed throughout this widget too.

## 2026-08-03 (later) — Real bugs fixed: null-button crash on every action, invalid `Service_Acceptance_Next`

Same fixes as `vendorTicket`'s own matching entry (see its README) — added `setDisabled()` and replaced all `.disabled=` sites in this file (~18), changed `Service_Acceptance_Next` from `true` to `"Yes"`. (This widget has no `workCompleted()` — TOW's own equivalent outcome is `vehiclePicked()`/`confirmDropped()`, not affected by the Status error reported this round.)

## 2026-08-03 (later still) — Diagnostic logging for Payment Received

Same fix as `vendorTicket`'s own matching entry — added payload/error logging to `confirmPaymentReceived()` for the `Payment_Status` "Invalid column value" error.

## 2026-08-03 (yet later still) — Real bug fixed: `Payment_Status` resolved to `PAID`/`PENDING`/`NOT APPLICABLE`

Same fix as `vendorTicket`'s own matching entry — see its README for the full explanation.

## 2026-08-03 (later again) — Real bug fixed: `Payment_Method1`'s actual option list; QR/gateway logic removed; camera+gallery both offered on every photo field

Same fixes as `vendorTicket`'s own matching entry (see its README for the full explanation) — `Payment_Method1` now lists the real `Cash`/`UPI`/`Card`/`Net Banking` options (default `Cash`), the QR-code/"Payment Gateway"-gated logic is gone (every method now requires the receipt photo or a zero balance), and every photo-capture slot offers both a 📷 camera tile and a new 🖼️ gallery tile (`openGallery()` + a new hidden `galleryInput` with no `capture` attribute).

## 2026-08-03 (later still) — All Zoho forms deleted and rebuilt from scratch: schema rebuild, renames, and a real Invites bug fixed

Same schema rebuild as `vendorTicket`'s/`technicianTicket`'s own matching entries (see their READMEs for the full explanation, and `../FIELDS.md`/`../ACCESS.md` for the complete build spec) — this widget's own relevant changes: `Payment_Method1` → `Payment_Method_Final`; `technicians_Report`/`technician_name` → `Technicians_Report`/`Technician_Name`; `Task_Rejections.Case_ID` → `RSID`; and the same `loadMyInvites()` fix (now matches `inv.Technician` instead of the never-actually-populated `inv.Vendor`).

## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
