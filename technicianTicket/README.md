# Get-Rescue Technician Widget

Field-facing Zoho Creator widget for the **RSR (repair)** on-road flow. Built 2026-07-31 from a "PROMPT FOR WIDGET CODE GENERATION" spec plus the Screen-by-Screen Flow Atlas mock, both shared the same day as the agent-facing `getRescueTicket` widget's own "Action 4" reconciliation round. Mobile-first (single-column, bottom-anchored actions), no build step — same Express dev server pattern as every other project in this repo.

See the sibling **`driverTicket`** (TOW path) and **`vendorTicket`** (both branches, plus a fleet hand-off screen) — all three share the same architecture and were built together. See `getRescueTicket/README.md` and `getRescueTicket/QUESTIONS_TO_ASK.md` for the fuller confirmed-facts/open-questions log this whole build relies on; not duplicated here.

**Field names**: see **`../FIELDS.md`** (repo root) — the single source of truth for every field API name this widget (and every other Get-Rescue widget) uses, organized step-by-step. Corrections happen there first.

**Zoho access**: see **`../ACCESS.md`** (repo root) — which Forms/Reports this widget needs permission to.

## Key decisions this build relies on (all user-confirmed 2026-07-31)

- **Data source is `Create_Case` only**, same `Agent_Ticket_Report`/`updateRecordById` pattern as `getRescueTicket` — the field-generation spec's own mention of an "Invites" form is **not** used anywhere in this widget.
- **No Google Maps API key yet.** `calcDistanceETA()`/`calcRoundTripKm()` are a plain Haversine straight-line distance (assumed 30 km/h average speed for a rough ETA) — a deliberate, isolated stand-in for a real Google Maps Distance Matrix API call, so swapping one in later only means editing these two functions, not every call site. Marked `TODO(Google Maps)` in the code.
- **The online/offline toggle is folded directly into this widget's topbar** (reads/writes `technicians_Report`'s `Availability_Status`, same report/field `toggleGetRescue` already used) — no separate toggle widget for Technician/Driver/Vendor, per explicit instruction.
- **Camera + GPS/timestamp photo watermarking** (`openCamera`/`capturePhoto`): opens the device camera via `getUserMedia`, captures a frame to `<canvas>`, overlays the current `navigator.geolocation` lat/long + local timestamp as burned-in text, then uploads the result. Falls back to a plain `<input type=file capture>` if camera/permission isn't available. **Only verifiable on a real mobile browser with camera + location permissions** — not exercised by this repo's Node-based syntax/logic checks.
- **Dashboard ("My Tickets")** shows `Create_Case` records where `Service_Type=="RSR"`, `Status` is one of the still-open RSR statuses, and the logged-in user's email matches one of `Assigned_Technician_Email`/`Technician_Email` (both candidates tried — **not confirmed live** which one a given ticket actually uses).
- **`REMAINING FEE DUE`** (Work Completed's own target status) and **`RSP CLOSED`** (Payment Received's own target status) are used exactly as the field-generation spec states — these are *better* evidence than `getRescueTicket`'s own long-standing `payment`-stage guesses (`RSP COMPLETE`), which were corrected there the same day to match (see its own README §5).

## What's genuinely unconfirmed (see `getRescueTicket/QUESTIONS_TO_ASK.md` for the full running list)

- Whether `Break_Down_Location1` is a real display/navigation field (treated here as one, falling back to a `maps.google.com` link built from `Latitude`/`Longitude` if absent) or something else entirely.
- The exact real option list for `Payment_Method1` beyond "Payment Gateway / Direct / Cash" (the spec's own "/ Etc" implies more).
- `Task_Rejections` (the best-effort rejection-log report referenced by the spec) — unconfirmed report name, and the write is non-blocking on purpose.

## 2026-07-31 (later) — Action 5 reconciliation: in-place navigation, reject-time field, expanded rejection log

A separate Action 5 mock (Service Acceptance/Rejection RSR) confirmed a real navigation gap: every successful action here previously called `backToDashboard()`, forcing the tech to re-tap into the ticket list to reach the next screen, even though the flow spec explicitly says e.g. "Accept Service ... navigate to 'Reach Breakdown Location' page." Fixed with a new `goToStatus(newStatus)` helper — sets the ticket's status locally and re-renders in place, staying on the same ticket. Applied everywhere a screen's own success handler advances the ticket to its *next* action (Accept → Reach, Reached → WIP, Work Completed/CX Reject → Payment); left as `backToDashboard()` wherever this technician's own involvement with the ticket actually ends (Reject, Cancel, Payment Received) — see `driverTicket`/`vendorTicket` for the identical pattern applied to their own screens.

Also this round: `renderTicketInfo()` now shows the breakdown Location too (not just Vehicle/Issue); a distinct `Service_Reject_Time` field is now stamped on Reject (was incorrectly reusing `Service_Acceptance` for both accept and reject); the Reject panel has an explicit "Back" button to cancel out of it; and the best-effort `Task_Rejections` log now includes the RSP's own name plus vehicle/issue details, per the mock's own "RSP name, times & Task details" instruction.

**One unresolved conflict, one now resolved — see `getRescueTicket/QUESTIONS_TO_ASK.md`:** (1) this mock names the ETA/Distance fields "RSP ETA"/"RSP Distance," while the original generation prompt (already implemented) said `ETA`/`Distance_To_Breakdown` — **still open**, since the equivalent Action 5a (TOW) mock repeats the identical generic phrase for a *different* pair of fields, weakening the case it's a precise name. (2) whether Reject writes `Status:"READY FOR ASSIGNMENT"` or `Status:"RSP REJECT"` — **resolved 2026-07-31**: the Action 5a (TOW) mock explicitly confirms `RSP REJECT`, matching what was already shipped.

## 2026-07-31 (later still) — Action 6 reconciliation: cancel-time/location capture, ETA fallback, Back button

The Action 6 mock (Reach Breakdown Location / Cancel — RSR) added several things this screen didn't have yet:

- **ETA is now shown and always editable** on the Reach screen, pre-filled from whatever `acceptService()` auto-captured — matches "Captured from app click / if null - entered manually." Whatever's in the box when "Reached Location" is tapped gets saved, overwriting the auto-captured value only if the tech actually changed it.
- **The Cancel panel now has a "Back" button** (same treatment the Reject panel got in the Action 5 round).
- **New cancel-time/location/distance capture**, per the mock's own footnote ("the location at time of button click... Distance... from RSP Start location to present location"): `acceptService()` now also saves the tech's own GPS position as `RSP_Start_Lat`/`RSP_Start_Lon`; `confirmCancel()` now stamps a `Cancellation_Time`, captures a fresh position as `Cancel_Location_Lat`/`Cancel_Location_Lon`, and computes `Cancel_Distance` from the saved start point to that position (via the same Haversine stand-in as everywhere else). All four field names are guessed, not confirmed — see `FIELDS.md`.
- **Flagged, not implemented**: the Back-Office-only "Send ETA To Cx via WhatsApp" button (no role-detection in this field-only app — same open question as the recurring REASSIGN button); a possible literal `RSP_`-prefixed naming convention across several fields (this mock's "RSP Reach time"/"RSP distance," combined with Action 5's "RSP ETA"/"RSP Distance," might mean the real Zoho fields are `RSP_ETA`, `RSP_Distance`, `RSP_Reach_Time`, etc. rather than the plainer names already shipped) — see `getRescueTicket/QUESTIONS_TO_ASK.md` for both.

## 2026-07-31 (even later) — Action 7 reconciliation: mandatory Issue Resolved, shared completion time, unresolved note

The Action 7 mock (WIP / Cx Reject — RSR) confirmed and added a few things:

- **`Issue Resolved?` is now actually validated as required** before "Work Completed" proceeds (was previously just optionally included in the payload if selected).
- **`RSP_Completion_Time` replaces `Work_Completed_Time`, and is now written by BOTH "Work Completed" and "Service Reject"** — per the mock's own footnote ("Capture times of clicks of 'work completed' button... or 'SERVICE REJECT' button... and saves it in 'RSP Completion time' field"). This is the **third** mock in a row using "RSP X" phrasing for a time/distance field (see the consolidated question in `QUESTIONS_TO_ASK.md`) — unlike the still-unresolved Action 5 conflict, this one wasn't contradicting an already-shipped name, so it was renamed directly rather than left as a flagged conflict.
- **Added a "Back" button and renamed "Confirm" → "Service Reject"** on the Cx Rejection panel, matching the mock's own button label; the Rejection Reason label was relabeled "Cx Rejection Reason" to match too (same underlying `Rejection_Reason` field, cosmetic only).
- **"Issue not resolved – add voice note?"** — the mock phrases this as an open question itself. Implemented as a plain-text `Unresolved_Note` field shown conditionally, **not real voice/audio recording** — flagged in `QUESTIONS_TO_ASK.md` rather than silently building a much bigger microphone-capture feature on a guess.

## 2026-07-31 (yet later) — Action 8 reconciliation: explicit Payment Status

The Action 8 mock (Final Payment Collection — common RSR & TOW) confirmed the payment method/receipt-photo/QR logic already matched, but **`Payment Status` needed to become a real, visible field** rather than something only inferred internally:

- Added a read-only "Payment Status" display (`Pending`/`Success`) to the screen. Non-gateway methods auto-flip it to `Success` the moment a receipt photo is attached; gateway payments can only truly reach `Success` via a real payment-gateway webhook, which doesn't exist in this app — so the record's own already-saved `Payment_Status` is also honored (`alreadySuccess`), meaning the "Payment Received" button will correctly enable itself once real gateway integration exists, with zero further code changes needed here.
- `confirmPaymentReceived()` now always writes `Payment_Status:"Success"` — clicking that button is itself the attestation, regardless of which path got it there.
- **Not implemented**: `Transaction_ID`/`Time_Of_Receipt`, which the mock says a real gateway "success" should also capture — there's no real gateway to source these from honestly yet; see `QUESTIONS_TO_ASK.md`.

## 2026-07-31 (yet later still) — Action 5a (TOW) mostly confirms what's already built

The Action 5a mock (Service Acceptance/Rejection — TOW) matches `driverTicket`'s existing Accept/Reject screen almost field-for-field (Start Odometer, Back button, `Service_Reject_Time`, `RSP_Start_Lat`/`RSP_Start_Lon` capture, `Task_Rejections` log) — no changes needed there. It also **resolves** the Action 5 Reject-status question in `QUESTIONS_TO_ASK.md`: this mock explicitly says Reject writes `Status:"RSP REJECT"`, matching what was already shipped, not the `READY FOR ASSIGNMENT` phrasing the earlier RSR mock seemed to suggest.

## 2026-07-31 (later again) — Task_Rejections form built; Case_ID/Vehicle corrected to Lookup writes

The user built the `Task_Rejections` form in Zoho Studio with `Case_ID` and `Vehicle` as real **Lookup** fields (not plain text as originally guessed). Corrected `rejectService()`'s best-effort log write to send `Case_ID:r.ID` (this ticket's own record ID) and `Vehicle:r[CONFIG.vehicleField]` (the raw fetched value, unconverted) instead of display strings — Lookup fields need the linked record's ID, not a label. `Vehicle`'s exact write shape is unconfirmed and needs live testing (see `getRescueTicket/QUESTIONS_TO_ASK.md`); the write stays non-blocking either way.

## 2026-07-31 (later still again) — Real bug fixed: technicians couldn't see their tickets at all (invalid `max_records`)

`loadAvailability()` (called first thing in `boot()`, before `loadDashboard()`) was fetching the technician's own availability record with `max_records:1` — Zoho Creator's `getRecords` only accepts **200, 500, or 1000** for that parameter (error code 9250, "Please enter a valid input for 'max_records' key"), so this call failed on every single boot. Fixed to `200`. Same bug existed identically in `driverTicket` and `vendorTicket` (the latter also had a second occurrence, `100`, in its fleet hand-off technician-list fetch) — all fixed together, see each one's own README.

## 2026-07-31 (later still again, once more) — Dashboard matching also checks the new hidden `Technician_Emails` field

Per the user's request, `vendorTicket`'s fleet hand-off now writes a hidden, comma-separated `Technician_Emails` field alongside `Assigned_Technician_Email`. `isMine()` here now also checks it (in addition to the existing `Assigned_Technician_Email`/`Technician_Email` candidates), so this will keep working once the hand-off becomes a real multi-candidate request flow instead of a direct single assign.

## 2026-07-31 (later still again, one more) — Lookup write shape: tried an object wrapper, reverted same day

Briefly added a `toLookupRef()` helper and used it for `Case_ID`/`Vehicle` in the `Task_Rejections` write, wrapping each as `{"ID":"..."}` based on a reported Zoho Lookup response shape. **Reverted the same day** — this broke saving elsewhere in the app (see `getRescueTicket/README.md`'s own matching entry), so `Case_ID`/`Vehicle` are back to a bare `r.ID` / the unconverted raw `Vehicle` value, same as before this round. `toLookupRef()` removed entirely.

## 2026-08-03 — New "Invites" architecture: dashboard now finds "my tickets" via Invites_Report first

Same architecture change as `vendorTicket`'s own matching entry (see its README for the full explanation) — this widget now tracks its own name (`MY_TECH_NAME`, set in `loadAvailability()`) and uses it to match `Invites_Report` records (`loadMyInvites()`), falling back to the older `Technician_Emails`/email match for tickets without a matching Invites record yet. `acceptService()`/`rejectService()` also now write to this technician's own Invites record via the new `updateMyInvite()`. These Invites records themselves get created by `vendorTicket`'s fleet hand-off (`assignTechnician()`) when a fleet vendor picks this technician. Added `apiUpdateReport()`/`unwrapLookup()`/`lookupId()` helpers to support this — same as `vendorTicket`.

## 2026-08-03 — Real bug fixed: "RSP_Start_Lon has exceeded its maximum digits" on Accept

`getPositionSafe()` was passing the browser's raw GPS coordinates straight through — these can have 17 decimal places, overflowing whatever Decimal field `RSP_Start_Lat`/`RSP_Start_Lon` are on `Create_Case`. Rounded at the source so every write that uses this position (Accept, Cancel's distance calc, etc.) is safe automatically. **Updated same day**: the user widened those fields to 15 decimal places, so the rounding changed from 6 → 15 places, via `toFixed(15)` (not a `*1e15` multiply, which can exceed `Number.MAX_SAFE_INTEGER` for lat/lon values).

## 2026-08-03 (later) — `RSP_Start_Lat`/`RSP_Start_Lon` changed to Text, renamed `RSP_Start_Latitude`/`RSP_Start_Longitude`

The user resolved the "maximum digits" error a different way than more precision-tuning: changed these two fields to plain **Text** on the real Create_Case form (sidesteps any digit limit entirely) and renamed them `RSP_Start_Latitude`/`RSP_Start_Longitude`. Renamed every reference in this widget to match — `acceptService()`'s own write, the diagnostic logging added for this same issue, and `confirmCancel()`'s own read of the saved start position for its distance calc (`Number(r.RSP_Start_Latitude)`/`Number(r.RSP_Start_Longitude)` still works fine — converting a numeric-looking Text value back to a JS number for the Haversine calc). `Cancel_Location_Lat`/`Lon` and `RSP_Drop_Location_Lat`/`Lon` are still Decimal fields, unaffected by this change — `getPositionSafe()`'s own 15-decimal rounding still matters for those.

## 2026-08-03 (later) — Real bugs fixed: null-button crash on every action, invalid `Service_Acceptance_Next`, diagnostic logging for Work Completed

Same fixes as `vendorTicket`'s own matching entry (see its README for the full explanation) — added `setDisabled()` and replaced all `.disabled=` sites in this file (~14), changed `Service_Acceptance_Next` from `true` to `"Yes"`, and added payload/error logging to `workCompleted()`.

## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
