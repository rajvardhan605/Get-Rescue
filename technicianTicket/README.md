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

## 2026-08-03 (later still) — Diagnostic logging for Payment Received

Same fix as `vendorTicket`'s own matching entry — added payload/error logging to `confirmPaymentReceived()` for the `Payment_Status` "Invalid column value" error.

## 2026-08-03 (yet later still) — Real bug fixed: `Payment_Status` resolved to `PAID`/`PENDING`/`NOT APPLICABLE`

Same fix as `vendorTicket`'s own matching entry — see its README for the full explanation. `Payment_Status` is the same field as the Quote step's own one, not a separate field; this widget's Payment screen now reads/writes `PAID`/`PENDING` instead of the guessed `Success`/`Pending`.

## 2026-08-03 (later again) — Real bug fixed: `Payment_Method1`'s actual option list; QR/gateway logic removed; camera+gallery both offered on every photo field

Same fixes as `vendorTicket`'s own matching entry (see its README for the full explanation) — `Payment_Method1` now lists the real `Cash`/`UPI`/`Card`/`Net Banking` options (default `Cash`), the QR-code/"Payment Gateway"-gated logic is gone (every method now requires the receipt photo or a zero balance), and every photo-capture slot offers both a 📷 camera tile and a new 🖼️ gallery tile (`openGallery()` + a new hidden `galleryInput` with no `capture` attribute).

## 2026-08-03 (later still) — All Zoho forms deleted and rebuilt from scratch: schema rebuild, renames, and a real Invites bug fixed

Same schema rebuild as `vendorTicket`'s own matching entry (see its README for the full explanation, and `../FIELDS.md`/`../ACCESS.md` for the complete build spec) — this widget's own relevant changes:

- **`Payment_Method1` renamed to `Payment_Method_Final`** throughout.
- **`technicians_Report` renamed to `Technicians_Report`**, `technician_name` renamed to `Technician_Name` — `CONFIG.techReport`/`techNameField` updated to match.
- **`Task_Rejections.Case_ID` renamed to `Task_Rejections.RSID`** — `rejectService()`'s write updated to match.
- **Real bug fixed: `loadMyInvites()` was matching the wrong Invites field.** It compared `inv.Vendor` against `MY_TECH_NAME`, but `vendorTicket`'s fleet hand-off never actually wrote a technician's name into `Vendor` (a Zoho Lookup can only target one form — see `vendorTicket/README.md`'s own entry). `Invites` now has a separate `Technician` field for exactly this, and this widget matches against that instead.

## 2026-08-11 — Missing-fields pass: Reach now captures a live GPS fix, two brand-new one-way distance fields

Per the user's own "Missing Fields & Field Behavior" spec, this round closes a gap the earlier RSR reconciliation rounds left open: Accept (`RSP_Start_Latitude`/`RSP_Start_Longitude`, see the 2026-07-31 Action 6 entry above) and Cancel (`Cancel_Location_Lat`/`Cancel_Location_Lon`) both already grab a fresh `getPositionSafe()` GPS fix at their own moment of click — Reach never did, at all, until now. Same change made identically in `vendorTicket`'s own RSR reach handler (`markReachedRsr()`) — see its README for the mirrored entry.

- **`markReached()`** now also captures a GPS fix the instant "Reached Location" is tapped, saved as **`Reach_Location_Lat`/`Reach_Location_Lon`** (shared field names — not split RSR/TOW, since this app has no TOW branch to split from anyway), right before the existing `apiUpdate()` call. From that same fix, **`Distance_Reach_To_Breakdown`** is computed via the existing `calcDistanceETA()` (this click's own position → the ticket's static `Latitude`/`Longitude` breakdown point) — deliberately a plain one-way distance, kept separate from `Roundtrip_Distance` just above it (which stays the untouched Office→Breakdown→Office 2-leg trip). Mirrors exactly how `Distance_To_Breakdown` already works over in `acceptService()`. Best-effort: a denied/unavailable location simply skips both new fields, same tolerance Accept/Cancel's own fixes already have — never blocks the save.
- **`confirmCancel()`** now also computes **`Distance_Cancel_To_Breakdown`** right after its existing `Cancel_Distance` calc, from the same Cancel-click GPS fix (`pos`) to the ticket's own static `Latitude`/`Longitude` breakdown point. Deliberately distinct from `Cancel_Distance` just above it (Accept location → Cancel location) — no field or calc covered "Cancel location → breakdown" before this at all.

**All four fields touched here (`Reach_Location_Lat`, `Reach_Location_Lon`, `Distance_Reach_To_Breakdown`, `Distance_Cancel_To_Breakdown`) are brand new and do not exist on `Create_Case` yet** — they still need to be created in Zoho Studio before this code's writes will actually persist (same caveat as every other guessed-name field flagged elsewhere in this README/`../FIELDS.md`). `../FIELDS.md` itself is being updated separately alongside this change.

**Real caveat worth flagging** (see `vendorTicket/README.md`'s own matching entry for the full reasoning): these four are bundled into the SAME `apiUpdate()` payload as `Reach_Time`/`Status`/`Cancellation_Time` — the fields Reach/Cancel actually depend on to advance the ticket at all. This repo's own precedent (new fields added straight into an existing save, dozens of times this session, no reported failures) suggests Zoho's `updateRecordById` likely tolerates an unrecognized key rather than rejecting the whole request — but that's inferred, not independently confirmed for this exact case. If Reach or Cancel stops working before these fields are created in Zoho, that's the signal this inference was wrong.

## 2026-08-11 (later still) — User confirmed: all four fields above now created in Zoho Studio

See `getRescueTicket/README.md`'s own matching entry for the full picture (all 11 fields flagged as pending across this session, not just this widget's four) — `FIELDS.md` updated accordingly. The payload-rejection caveat above is now moot for these four; worth a real Reach/Cancel test on a live ticket to confirm the values actually land correctly.

## 2026-08-17 — Live location heartbeat while Online: `Current_Latitude`/`Current_Longitude`

User request: "when vendor is online it is supposed to show the distance from the vendor's current location (latest location from app); when vendor is offline it is supposed to show the distance from the vendor's static location (as in vendor table)" — user confirmed this applies to Technicians/Drivers too, not just Vendors, and that the refresh should happen roughly every 2-3 minutes while online. Until now, `Technicians_Report` only ever held a technician's static/home location (whatever it was set to once, off-app) — there was no way for `getRescueTicket`'s own Assignment-step vendor/technician picker to show a distance based on where an Online technician actually is *right now*.

Added a live location heartbeat, folded into this widget's own toggle logic (`onToggleClick()`/`loadAvailability()`, around widget.html:1337/1313) alongside the existing toggle-in-topbar pattern:

- **Two new `CONFIG` fields** (widget.html, right after `techNameField`): `techCurrentLatField: "Current_Latitude"`, `techCurrentLonField: "Current_Longitude"` — same field names `getRescueTicket`'s own picker expects, written directly to this technician's own `Technicians_Report` record (`CONFIG.techReport`) — unlike `vendorTicket`, this widget's toggle already targets `Technicians_Report` itself, no separate `My_Availability_Vendor`-style report in between.
- **`pingCurrentLocation()`** — takes a fresh `getPositionSafe()` fix and writes it to `Current_Latitude`/`Current_Longitude` via `updateRecordById`, guarded on `currentTechRecord` being set; best-effort throughout (a denied/unavailable fix, or a failed write, just silently skips that one tick, logged via `console.warn("[Get-Rescue] pingCurrentLocation failed...")`).
- **`startLocationHeartbeat()`/`stopLocationHeartbeat()`** — `start` fires an immediate ping (no waiting for the first interval tick) then runs `pingCurrentLocation()` on a `setInterval` every `LOCATION_HEARTBEAT_MS` (150000ms / 2.5 min — the midpoint of the user's own 2-3 min range); `stop` clears the timer. `start` always calls `stop` first so repeated toggling never stacks two timers.
- **Wired into the toggle**: `onToggleClick()` (widget.html, in the toggle's success path) now calls `startLocationHeartbeat()` when going Online and `stopLocationHeartbeat()` when going Offline. `loadAvailability()` (the boot-time loader) also calls `startLocationHeartbeat()` immediately after setting the toggle's initial on/off UI state, if the technician is already Online when the widget loads — so a page refresh/reopen while Online doesn't require toggling off and back on to resume fresh pings.

**Two brand-new fields — not yet created in Zoho Studio as of this entry**: `Current_Latitude`, `Current_Longitude` on `Technicians_Report` (same "ships safely ahead of the field existing" pattern used throughout this app — the write is best-effort and non-blocking, so nothing else breaks until they're created; see `FIELDS.md`'s own note).

The identical change (same field names, same 2.5-min heartbeat, same immediate-fix-on-toggle-on pattern) was made the same day in `vendorTicket/app/widget.html` (`pingCurrentLocation()`/`startLocationHeartbeat()`, around its own `CONFIG.availabilityCurrentLatField`/`availabilityCurrentLonField`) — see that file's own matching comments for the fuller cross-widget picture, including why vendor's version routes through its separate `My_Availability_Vendor` report rather than `Vendors_Report` directly (a wrinkle this widget doesn't have, since its own toggle already targets `Technicians_Report`).

Syntax-checked (`node -e` script against the `<script>` block — see this repo's own established check) and packed (`technicianTicket.zip`).

## 2026-08-20 — Real bug fixed: after rejecting a ticket, it silently vanished from "My Tickets" with no lasting sign it was ever rejected (user-reported live)

User's exact report, confirmed after a round of clarifying questions: "if rejected... vendor acceptance status is not being displayed" — turned out to mean this widget's OWN "My Tickets" list, not the agent's `getRescueTicket` dashboard or the per-vendor Invite Status panel there (both already handle Rejected correctly, see `getRescueTicket/README.md`). Confirmed root cause: `rejectService()` writes `Status:"RSP REJECT"`, but `RSP REJECT` was never in `MY_STATUSES` — so the instant that save landed, the ticket dropped out of `fetchMyTickets()`'s own filter entirely. The reject toast fades a couple seconds later and the ticket is just gone, with nothing left anywhere on this widget showing "you rejected this."

**Confirmed fix** (user's own choice between two options — keep it in the main list vs. a separate "recent activity" section): stays in the main **"My Tickets" list** with a clearly-styled **Rejected** pill, rather than a separate screen/tab.

**Fix**:
- `MY_STATUSES` now includes `"RSP REJECT"`.
- `STATUS_PILL["RSP REJECT"]` added — `{cls:"rej", label:"Rejected"}`, styled with the same `--stop`/`--stop-soft` red already used for Reject Reason/Cancel Reason required-field markers elsewhere in this file (new `.st.rej` CSS rule).
- `renderActionScreen()` gained a dedicated `RSP REJECT` branch — tapping back into a rejected ticket now shows a clear "You Rejected This Ticket" panel with the real reject reason (`Reject_Reason` — this widget is RSR-only), instead of falling through to the generic "Nothing to do on this ticket right now" filler (which used to be unreachable for a rejected ticket anyway, since it never stayed in the list long enough to tap back open).

**Worth knowing — a real consequence of this change, not yet separately reported as a problem**: `Status` is a single ticket-level field. If a ticket was invited to **multiple** vendors and only ONE of them rejects, the ticket now shows "Rejected" in **every** invited vendor's own "My Tickets" too — including ones who haven't responded at all yet — since there's no per-vendor status at this widget's own dashboard level (only `getRescueTicket`'s separate Invites-based panel tracks that distinction). Flagging rather than silently deciding it away: if this turns out to be confusing in practice with multi-vendor invites, the fix would need to check this vendor's own Invites-report row instead of the ticket-level `Status` before showing the Rejected pill — not implemented here since the user's own confirmed scope was the single-vendor case.

**Tested** against the real shipped code (Node-VM harness, all three widgets — `vendorTicket`/`technicianTicket`/`driverTicket`) — 10 checks: `MY_STATUSES`/`STATUS_PILL` updated correctly in each, and a live `renderActionScreen()` call confirms the new "You Rejected This Ticket" panel actually renders the real reject reason (checking both `Reject_Reason`/`Reject_Reason1` where technicianTicket branches by `Service_Type`). Re-ran every other same-day vendorTicket test suite (vendor-identity retry/fallback, fleet hand-off, live-location heartbeat) to confirm no regression — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-21 — Real bug fixed: agent's own "Vendor Invite Status" panel kept showing "Awaiting response" even after a genuine accept (user-reported live: "acceptance status is not being displayed")

User's exact report: on the Assignment page in the agent portal, invited vendors' own acceptance status wasn't showing correctly even after they'd genuinely accepted.

**Root cause, traced through the same identity-resolution gap found earlier this session**: `state.inviteByTicketId` (the cache `updateMyInvite()` uses to find which Invites row belongs to this ticket) is built exactly ONCE, at `loadDashboard()`'s own `fetchMyTickets()`/`loadMyInvites()` call, by matching against whatever currentTechRecord held AT THAT MOMENT. If currentTechRecord hadn't resolved yet then (the exact gap `resolveVendorRecord()`'s own 2026-08-19 retry was built for), this cache stayed empty for that vendor for the rest of the session — even after a LATER retry inside `acceptService()` itself successfully resolved currentTechRecord, `updateMyInvite()` kept trusting the stale, empty cache and silently no-op'd on every write, since it never re-checked. `Service_Acceptance_Next`/`Status` on the real Invites row never got written — and since the agent's own "Vendor Invite Status" panel (`assignInviteStatusHtml()` in `getRescueTicket`) reads directly off that same Invites record, it kept showing "Awaiting response" forever, even though the vendor genuinely accepted and the ticket's own `Status` field (written directly, not through Invites) updated correctly the whole time.

**Fix**: `updateMyInvite()` now retries a real, direct `CONFIG.invitesReport` lookup (matched on the Technician Lookup field, same "read the report, filter client-side by id" pattern used everywhere else in this app for this exact report) whenever the cached invite isn't found — instead of silently giving up. A successful retry also backfills `state.inviteByTicketId` so later calls in the same session don't need to retry again. If currentTechRecord is STILL null even at retry time (a genuine identity gap, not just a stale cache), this still safely no-ops with a clear diagnostic — never guesses at which Invites row to write to.

**Tested** against the real shipped code (Node-VM harness, stubbing `apiGetReport()`/`apiUpdateReport()`) — confirmed the retry finds and writes to the real Invites row despite a stale/empty cache, confirmed it backfills the cache for next time, confirmed no write is attempted (and nothing throws) when the identity genuinely never resolved, and confirmed the already-correct-cache case still writes directly with no regression. Re-ran the vendor-identity retry/fallback and fleet hand-off suites from earlier this session to confirm no interaction — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-21 — Real bug fixed: reopening the app after a photo was already uploaded showed the field as empty and re-blocked progress (user-reported live)

User's exact report: "when photo is clicked, it is being sent immediately and is visible on the agent portal. but if vendor closes app and opens his app again and comes back to the same page, the fields display as 'empty'. so vendor has to click all photos again in order to be able to move forward."

**Root cause**: every photo field's own display (`renderPhotoGrid()`) and every "N photos required" validation check across this widget only ever looked at `state.photos[field]` — a plain array of browser `File` objects picked/captured THIS session. That starts completely empty on every fresh app load, regardless of what's genuinely already saved on the ticket record itself (the upload really did succeed — that's exactly why it was already visible on the agent portal). So the grid rendered nothing, and the "at least N required" gate blocked the vendor from proceeding until they re-took photos that were never actually missing.

**Fix**:
- `fileCountFor(rawValue)`/`zohoFileDownloadUrl(recordId, fieldName, rawValue, index)` — ported directly from `getRescueTicket`'s own already-working versions (same account/app/report, so this isn't a new guess).
- `effectivePhotoCount(fieldName)` — the real count now used by every "N photos required" check: whatever's already on the ticket record (from a PREVIOUS session — `state.current` is only ever set fresh by `openTicket()`, never mutated after an upload, so this can never double-count) plus whatever's been picked THIS session.
- `renderPhotoGrid()` now renders a tile for each already-uploaded photo too — a real thumbnail via `zohoFileDownloadUrl()` where a usable URL can be built, hydrated asynchronously by the new `hydrateAlreadyUploadedThumbs()` (same onerror-recovery idea as `getRescueTicket`'s own file previews — reverts cleanly to a plain "✓ Saved" badge instead of leaving a broken image icon if the URL fails to load), otherwise falling straight to that plain badge. These tiles have no ✕ remove button — there's no "delete an already-uploaded photo" flow, only newly-picked-but-not-yet-final files can be removed before upload.
- Every "N photos required" check (Arrival, Cancel, Pre/Post-service, on-truck, VCRF, drop-location, unloaded, handover, receipt — every photo field in this widget) now uses `effectivePhotoCount()` instead of the raw session-only count.

**Tested** against the real shipped code (Node-VM harness with a richer fake DOM tracking appended children) — confirmed `effectivePhotoCount()` correctly counts an already-saved photo from a previous session even with zero picked this session (the exact reported scenario), correctly still blocks when genuinely nothing exists either way, and correctly avoids double-counting when both an already-saved AND a freshly-picked photo exist together; confirmed `renderPhotoGrid()` actually renders the already-saved placeholder tile and the count label reflects it instead of showing empty. Re-ran the vendor-identity retry/fallback, invite-status retry, and "Rejected stays visible" suites from earlier this session to confirm no regression — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-21 (later) — Two real bugs fixed: "Vehicle" showed a raw record id instead of a name, and "Location" showed raw decimal Latitude/Longitude digits (user-reported live, screenshot)

User's exact report/screenshot: on the Service Acceptance screen, `VEHICLE` showed `448881000000058404` (a raw `Vehicle_Master_Report` id) instead of a real vehicle name, and `LOCATION` showed `28.53000000000000, 77.17000000000000` (raw coordinates) with an explicit follow-up request to hide the lat/long fields.

**Vehicle name root cause**: `CONFIG.vehicleNameField` (`"Name"`) was a single, never-independently-reconfirmed guess for `Vehicle_Master_Report`'s real name column. If the real field is named something else on this account, `loadVehicleNames()` would build an entirely empty `VEHICLE_NAMES` map — every single vehicle, not just one — and `vehicleDisplayFor()`'s own fallback would show the raw id instead, exactly matching the screenshot.

**Fix**: `loadVehicleNames()` now tries several plausible field-name candidates (`Vehicle_Name`, `vehicle_name`, `Name`, `name`, plus the original config value) instead of one guess — same pattern this file's own `FLEET_TECH_NAME_CANDIDATES` already uses for an identical class of gap. If every candidate still comes up empty for every record, it now falls back to the raw id (unchanged, still correct — never guesses a wrong name) but with a clear console diagnostic explaining why, instead of silently doing nothing.

**Location fix**: the raw `Latitude`/`Longitude` fallback in `renderTicketInfo()` is removed — `Location` now only shows when there's a real `Break_Down_Location1` address string; otherwise the row is hidden entirely (the existing row-filter already drops any falsy value, so this required no new logic, just removing the fallback).

**Tested** against the real shipped code (Node-VM harness) — 20 checks across all three widgets: vehicle name resolves correctly via a fallback candidate when the primary guess is wrong (the exact reported case); a genuine total miss still safely falls back to the raw id with a diagnostic, not silently; `renderTicketInfo` no longer shows raw coordinates; a real address string still displays correctly (no regression). Re-ran the photo-persistence, vendor-identity retry, and invite-status retry suites from earlier this session — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-21 (later still) — New: "Navigate to Breakdown" added to the Accept screen (user request: "shows based on the service type like RSR and TOW")

User's exact ask: "NAVIGATE TO BREAKDOWN & NAVIGATE TO DROP LOCATION Buttons to be available in vendor app and these buttons shows based on the service type like RSR and TOW." Checking every screen turned up a real, consistent gap: the Reach screen already has "Navigate to Breakdown" (and, for TOW, "Navigate to Drop Location" already exists at the appropriate later step too), but the Accept screen — one step earlier — never had a breakdown nav button at all, in any of the three field widgets. `driverTicket`'s own Accept screen already had "Navigate to Drop Location" but no breakdown button; `technicianTicket`'s had neither.

**Fix**: `renderAcceptScreen()` now shows the same breakdown-nav link this file's own Reach screen already uses — this widget is RSR-only, so there's no Drop Location concept here at all.

**Tested** against the real shipped code (Node-VM harness) — 8 checks across all three widgets: Accept screen now shows "Navigate to Breakdown" for both RSR and TOW (where applicable); RSR correctly never shows "Navigate to Drop Location" (no drop leg exists for that service type); TOW/driverTicket's pre-existing "Navigate to Drop Location" is confirmed unchanged (no regression), with the new Breakdown link correctly appearing first, matching the real order of the flow (pick up, then drop). Re-ran the vehicle-name/hide-latlong, photo-persistence, and vendor-identity-retry suites from earlier this session to confirm no interaction — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-24 — Real bug fixed: Online/Offline toggle silently did nothing when `currentTechRecord` hadn't resolved (user-reported live: "online/offline toggle is not working")

User's exact report: the availability toggle just doesn't work.

**Root cause**: `onToggleClick()` used to just `return` immediately whenever `currentTechRecord` hadn't resolved — no error, no toast, nothing. Every other identity-resolution bug found this session (vendorTicket's `currentVendorRecord`, the Invites-status write) traced back to the exact same shape of gap: a one-time match at boot time that can fail to resolve (a slow connection, a login-email mismatch) and then never gets retried, even though the underlying data might be perfectly fine. A silently-dead button reads exactly like "not working" from the technician's side, with zero clue why.

**Fix**: the boot-time match (previously inline in `loadAvailability()`) is now its own `resolveTechRecord(logContext)`, reused as a real retry inside `onToggleClick()` — if `currentTechRecord` is still null right when the technician taps the toggle, it retries the actual `CONFIG.techReport` query once before giving up. If the identity genuinely still can't be resolved after that, the technician now sees a clear error toast instead of a dead, unresponsive switch. Also bumped the fetch from 200 to 1000 records, same reasoning as vendorTicket's own `resolveVendorRecord()` fix (rules out `Technicians_Report` simply exceeding the old cap).

**Tested** against the real shipped code (Node-VM harness, stubbing `apiGetReport()`/`updateRecordById`) — confirmed a retry that finds the real record lets the toggle go through normally (a success toast, not an error); confirmed a genuine failure (no matching record at all) now shows a clear error toast instead of silently doing nothing. Re-ran the invite-status-retry, Accept-screen-nav-buttons, vehicle-name, and photo-persistence suites from earlier this session to confirm no regression — all still pass.

Syntax-checked and packed (`technicianTicket.zip`).


## 2026-08-27 — Real gap fixed: Google Distance Matrix calls now use two-wheeler routing (mirrored from vendorTicket)

User's exact ask (originally against `vendorTicket`, then: "fixed mirrored there" for this app and `driverTicket`): "for all distance calculations that use google api - ensure that travel type has to be four wheeler for TOW cases and distance type should be two wheeler for REPAIR/RSR cases." This app only ever handles RSR tickets — `loadDashboard()`'s own query is hardcoded `Service_Type == "RSR"`, no TOW branch exists anywhere in the file — so every one of its distance calls was silently routing a mechanic on a two-wheeler using Google's car/four-wheeler `"DRIVING"` mode the whole time, which can miss narrower streets/one-ways a bike can legally take.

**Fix**: `googleDistanceMatrix`/`calcDistanceETA`/`calcRoundTripKm`/`officeRoundTripLegs` all take a new, optional trailing `travelMode` param — omitted, it still defaults to `"DRIVING"` (backward compatible). Added `travelModeFor(record)`, mirroring vendorTicket's own function name/shape — but since this app can never see a TOW ticket, it always resolves to `"TWO_WHEELER"` rather than branching on the record. All 7 real call sites (`acceptService`, `markReached`, `confirmCancel` ×3, `workCompleted`'s `officeRoundTripLegs` call) now pass it through explicitly.

**Tested** against the real shipped code (Node-VM harness, functions extracted directly from `widget.html` via brace-matching) — confirmed `travelModeFor` always returns `"TWO_WHEELER"` regardless of input, and that `calcDistanceETA`/`calcRoundTripKm`/`officeRoundTripLegs` all correctly resolve to `"TWO_WHEELER"` on the real Google API call shape; confirmed the module-level default (an omitted `travelMode` arg) still falls back to `"DRIVING"`, unchanged. Syntax-checked and packed (`technicianTicket.zip`).


### 2026-09-01 — Real gap fixed: dashboard auto-refresh only handled brand-new tickets, not status changes/removals on already-known ones

Found while fixing the identical gap in `getRescueTicket`'s own dashboard poll (user-reported live: a new ticket wasn't appearing in a second open tab) — audited every widget's own auto-refresh for the same class of issue. `pollForNewCases()` here only ever re-rendered when a genuinely new ticket arrived; an already-known ticket's status silently changing elsewhere or a known ticket dropping out of "my tickets" entirely updated `state.tickets` in the background but never refreshed what's actually on screen.

**Fix**: added `KNOWN_TICKET_STATUSES` (companion to the existing `KNOWN_TICKET_IDS`) to track each known ticket's own last-seen status. `pollForNewCases()` now also re-renders on a status change or a removal — but deliberately doesn't play the new-case sound/notification for either (that's reserved for a genuinely new arrival), so this only fixes staleness, not notification behavior.

**Tested** — new suite (`test_field_dashboard_auto_refresh.js`, 24 checks across vendorTicket/technicianTicket/driverTicket) extracting the real shipped functions: brand-new ticket still notifies+renders; a status change on a known ticket renders but doesn't notify; a known ticket disappearing renders but doesn't notify; nothing changed renders/notifies nothing; the status snapshot itself stays current. Syntax-checked and packed (`technicianTicket.zip`).

### 2026-09-06 — New: WhatsApp integration added (was zero before this) — Reach message wired, more to follow

User provided three PDFs ("RESCUE - WhatsApp Message Templates," "RESCUE - Repair Service Logic Flow," "RESCUE - Towing Service Logic Flow") specifying 13 numbered WhatsApp templates and their trigger logic across the whole ticket lifecycle. A full audit found this widget (and `driverTicket`/`vendorTicket`) had **zero** WhatsApp code at all — every send in the app lived only in `getRescueTicket`, covering just 4 of the 13 templates. Proceeding incrementally, safest piece first, per explicit user instruction to not risk breaking existing functionality.

**Built**: `formatIndianPhone()`/`sendWhatsAppTemplate()` copied verbatim from `getRescueTicket`'s own already-proven implementation (same `sendWhatsAppMessage` Custom API, same param shape) — this file had neither before. Wired into `markReached()`: after the existing Reach save/photo-upload/toast completes, fires the `reach_time` template (spec item 10 — "Mechanic/Driver app - Reach Breakdown Location page, on click REACH button, Send 10") to the customer's `Phone_Number`. Deliberately **not awaited** — a failed/slow send can never block or delay the real Reach action, matching this app's existing tolerance for best-effort GPS capture.

**Deliberately not built yet, blocked on real answers rather than guessed**:
- **Accept-time message (template 8, "vendor/mechanic name and number")** — user confirmed live that the phone number IS a real template parameter, but `Technicians_Report` has no confirmed phone field anywhere in this codebase (unlike `Vendors_Report`'s confirmed `Mobile_Number_01`) — user is checking the real field name before this gets built, to avoid sending a broken/blank parameter to real customers.
- **5-minutes-later follow-up (template 9)** and the **multi-step CTA-triggered thread (templates 1→2→3)** — both need an architecture decision (a delayed/scheduled send survives an agent closing their tab; the CTA case needs a webhook receiving the customer's own button-tap) before building either.
- **Complete-message with Remaining-Fee branching (templates 11/12)** — the field apps have no payment-link-generation capability at all today (that lives only in `getRescueTicket`); needs a design decision on whether to add it here or route through the agent side.

**Template name flagged, not confirmed**: `reach_time` is a best-effort guess (no numbered/named catalog exists anywhere, matching every other template name in this app) — the spec's own row title is "Share rescuer Reach Time." Also has no visible parameter slots in the spec's own template text, so it's sent with zero params — correct this once the real approved WhatsApp Business template name/shape is confirmed.

**Tested** — new suite (`test_field_apps_whatsapp.js`, 12 checks across technicianTicket/driverTicket/vendorTicket): `formatIndianPhone()` correctness, and that the Reach handler in each app actually calls `sendWhatsAppTemplate` with the `reach_time` template, unawaited (confirming it truly can't block navigation). Full existing suite re-run — no regressions. Syntax-checked and packed (`technicianTicket.zip`).

### 2026-09-06 (later) — New: WhatsApp dispatch message (template 8) wired into Accept

User confirmed the real phone field on `Technicians_Report` is `Phone` (previously unconfirmed anywhere in this codebase — the only prior guess was a multi-candidate fallback list in `vendorTicket`'s own fleet-technician display), and confirmed the phone number genuinely is a 3rd dynamic value in the real approved template (name + phone + ETA), not just descriptive wording in the logic-flow doc.

**Real architecture gap found and fixed**: `sendWhatsAppTemplate()` only ever forwarded 2 parameters (`param1`/`param2`) — a 3rd was needed. The server-side `sendWhatsAppMessage` Custom API's Deluge source isn't checked into this repo (unlike every other Custom API), so rather than guess whether it would even forward a `param3`, asked the user to paste its actual contents — confirmed it already builds its `parameters` array conditionally including `param3` if non-blank, so extending the JS side was safe. Updated in all four widgets (`getRescueTicket` included, even though it doesn't call this with 3 params yet) for consistency — existing 2-param call sites are unaffected, `param3` simply defaults to an empty string for them.

**Built**: `techPhoneField: "Phone"` added to `CONFIG`; `MY_TECH_PHONE` resolved in `resolveTechRecord()` alongside the existing `MY_TECH_NAME`. `acceptService()` now sends the `dispatch_eta` template (spec template 8 — "ACCEPT/REJECT page, on click ACCEPT button, Send 8 with vendor/mechanic name and number") with `[MY_TECH_NAME, MY_TECH_PHONE, ETA]`, fire-and-forget, right after the existing accept save/toast, same non-blocking convention as the Reach message.

**Not built yet, explicitly deferred**: `getRescueTicket`'s own "agent entering on behalf of vendor" Accept trigger (the PDF lists this as a second place template 8 should fire) — that stage can assign either a vendor or a specific technician under a fleet vendor, and its save routes through a generic multi-stage config mechanism rather than one dedicated function like this app's own `acceptService()`. Needs its own careful pass rather than rushing it into this one.

**Template name/param order flagged, not confirmed**: `dispatch_eta` and the `[name, phone, ETA]` ordering are a best-effort guess — the spec's own template text only shows 2 visible blanks (name, ETA), the phone slot's exact position was inferred from the user's own confirmation it exists, not from directly seeing 3 blanks in the extracted PDF text.

**Tested** — extended `test_field_apps_whatsapp.js` to 25 checks: confirms `param3` support across all four widgets, and the exact `dispatch_eta` wiring/param order in each app's Accept handler. Full suite re-run — same pre-existing, already-investigated failures, nothing new. Syntax-checked and packed (`technicianTicket.zip`).

### 2026-09-07 — Location/distance fixes from the user-provided LOCATIONS-DISTANCES-DATETIME spec

Full gap analysis + fix writeup lives in `getRescueTicket/README.md`'s own matching entry — this app's own changes (RSR-only, no drop leg, so only 2 of the fixes apply here):
- `OFFICE_LAT`/`OFFICE_LON` corrected to `12.967945122837245, 77.6110507612277` (previous value was off by ~13-20m).
- **Real bug fixed**: `markReached()`'s `Roundtrip_Distance` used to always route through the static ticket breakdown location, even though a live GPS fix is captured a few lines later in the same function at this exact Reach click — the fix just never got fed into the round-trip calc. Now uses that live fix (falls back to the static location only if GPS is denied).

**Tested** — new `test_locations_distances_2026_09_07.js` (32 checks, shared with `vendorTicket`/`driverTicket`/`getRescueTicket`). Full existing `test_field_apps_whatsapp.js` suite re-run — no regressions. Syntax-checked (`node --check` on the extracted script block) and packed.

### 2026-09-07 (later) — New: `Remaining_Fee_Receipt_Time`/`RSP_Closure_Time` wired into `confirmPaymentReceived()`

User created all 5 new timestamp fields from the same spec (see `getRescueTicket/README.md`'s own matching entry for the full field list/rationale, including the other 3 which are `getRescueTicket`-only). This app's own change: `confirmPaymentReceived()`'s payload now also includes `Remaining_Fee_Receipt_Time` and `RSP_Closure_Time` (both stamped at this same click, kept as two separate fields per the spec's own literal numbering) — no extra guard needed, since this function only ever runs on a genuine "Payment Received" click.

**Tested** — new `test_new_timestamps_2026_09_07.js` (15 checks, shared with `vendorTicket`/`driverTicket`/`getRescueTicket`). Full existing suites re-run — no regressions. Syntax-checked and packed.

### 2026-09-07 (later still) — Real bug fixed: photo capture appeared frozen during the GPS wait, inviting repeated taps

Found during a full 24-point feature-coverage audit. `capturePhoto()`'s `getPositionSafe()` call can take up to 8 seconds for a real GPS fix, and nothing disabled the shutter or showed progress during that wait — it just looked frozen, and each repeated tap started a fully separate concurrent capture (duplicate photo + duplicate upload). Fixed with a simple `capturingPhoto` in-flight flag (extra taps become a no-op) plus a visibly disabled/dimmed shutter button for the duration — the actual capture/watermark/upload logic is completely unchanged, only wrapped. The two camera/gallery buttons themselves were confirmed intentional (2026-08-03 user request), not the cause.

**Tested** — new `test_photo_capture_debounce_2026_09_07.js` (36 checks, shared with `vendorTicket`/`driverTicket`), a real execution test with a controllable-delay GPS mock proving a double-tap produces exactly one photo, not two. Full existing suite re-run — no regressions. Syntax-checked and packed.

### 2026-09-08 — Google Distance Matrix API scope-restricted to only getRescueTicket's Final Closure calculation

Explicit user decision: *"use Google api's only for final distance calculations.. round trip, vendor travel distances and vendor round trip"* — confirmed via AskUserQuestion to mean Google's billed Distance Matrix API should be used ONLY inside `getRescueTicket`'s Final Closure calculation (`Roundtrip_Distance`/`Vendor_Distance`/`Vendor_Round_Trip_Distance`), and reverted everywhere else. See `vendorTicket/README.md`'s matching entry for the full rationale — this app's own 7 call sites (Accept, Reach, Cancel, Work Completed) are fixed identically: `loadGoogleMaps()` short-circuits to `return Promise.resolve(false);` at the top, so the Google Maps script is never requested (zero network calls, zero billing risk) and every call site falls straight into its existing Haversine fallback. `travelModeFor()` (always `TWO_WHEELER` in this RSR-only app) is unchanged, just inert now.

**Needs redeploying**: none — client-side JS only.

**Tested** — `test_technician_driver_travelmode.js` rewritten (same "Google never reached, Haversine still works" pattern as vendorTicket's matching test). New `test_google_api_scope_2026_09_08.js` (29 checks, shared across all 4 widgets). Full existing suite re-run — no new regressions (pre-existing, unrelated failures only, none touching distance/API code). Syntax-checked and packed.

## 2026-09-10 — quick-win fixes from the client's soft-test audit

Two fixes from the full soft-test gap audit (see the published findings), picked as the lowest-effort items involving this widget:

**Notification sound made distinct.** vendorTicket/technicianTicket/driverTicket previously played the byte-identical 2-beep 880Hz sine tone — impossible to tell which app a new ticket landed on by sound alone. This app now plays 3 sharper beeps at a higher pitch on a square wave (vs. vendorTicket's 2 sine beeps, driverTicket's rising triangle sweep) — genuinely different timbre and count, not just a pitch tweak — and louder (`gain` 0.28 → 0.5).

**Customer phone number in the ticket-info summary is now click-to-dial.** Same fix as vendorTicket's own matching entry — `renderTicketInfo()`'s "Phone" row is now a `tel:` link via `formatIndianPhone()` (already proven elsewhere in this file for WhatsApp sending), visible text unchanged.

**Needs redeploying**: none — client-side JS only.

**Tested**: syntax-checked and packed — not yet independently live-tested.

## 2026-09-10 (later) — least-effort-first pass: Android nav-link fix + image capture downscale

Same two fixes as vendorTicket's own matching entry (see `getRescueTicket/README.md` for the full shared writeup) — this widget's smaller scope (RSR-only, no drop-location flow):

**Item #12** — new `buildNavUrl(lat, lon)` helper (near `esc()`); this widget's 2 nav-link call sites (`renderAcceptScreen()`, `renderReachScreen()`) now route through it. Android gets a `geo:` URI; iOS/desktop unchanged.

**Item #14** — new `CAPTURE_MAX_DIM = 1600` constant; `capturePhoto()`'s canvas now caps at that dimension before encoding, same as vendorTicket's own copy of this identical function.

**Tested**: syntax-checked and packed. **Not yet live-tested** — needs a real Android device for the nav-link fix specifically.

## 2026-09-11 — Location+timestamp capture: Work Completed now captures GPS too

Part of an app-wide "capture location+time at every relevant step" request — full lifecycle audit and field list in `getRescueTicket/README.md`'s own matching entry. This app is RSR-only, so it only needed one of the fixes: `workCompleted()`/`confirmCxReject()` both had a timestamp (`RSP_Completion_Time`) but no location capture at all. Added a `getPositionSafe()` call (this app's own already-proven helper, same one Accept/Reach/Cancel already use) + new `Work_Completion_Lat`/`Work_Completion_Lon` fields (Single Line Text) to both — they're the two possible outcomes of the same "work completion" moment.

**Tested**: syntax check passed, `zet pack` re-run, both new field writes confirmed present via fresh grep. **Not yet live-tested** — needs a real RSR ticket walked through to Work Completed (and separately, Customer Reject) to confirm the location saves correctly.

## 2026-09-11 — 2 real bugs fixed, from a client "Task Tracker" audit (same fixes as vendorTicket, applied here identically)

**Bug 1**: `confirmPaymentReceived()` called `apiUpdate()` (marking the ticket closed/paid) **before** `uploadPendingPhotos()` — a genuine photo-upload failure could leave the ticket closed and marked PAID without its mandatory receipt photo actually saved. Fixed by reordering: the photo upload now runs first, and a real failure blocks the close.

**Bug 2**: `openGallery()` (choose-from-gallery) pushed the picked file straight into `state.photos` with zero compression, unlike `capturePhoto()` (the in-app camera), which already resizes to 1600px/quality 0.85. New `compressImageFile()` helper now applies that same treatment to gallery picks too, falling back to the original file if compression ever fails.

See `vendorTicket/README.md`'s own matching entry for the full root-cause writeup — identical code, identical reasoning, ported here.

**Needs redeploying**: `dist/technicianTicket.zip` (re-packed). **Not yet live-tested**.

## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
