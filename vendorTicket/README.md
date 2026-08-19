# Get-Rescue Vendor Widget

Field-facing Zoho Creator widget covering a vendor's **full lifecycle across both RSR and TOW tickets** — sibling of `technicianTicket`/`driverTicket` (same architecture, CONFIG shape, camera/GPS watermark modal, Haversine distance/ETA stand-in), but branches internally by `Service_Type` instead of being one branch per widget, plus an extra Individual-vs-Fleet fork.

See `technicianTicket/README.md` for the full set of shared, user-confirmed decisions this build relies on — not repeated here.

**Field names**: see **`../FIELDS.md`** (repo root) — the single source of truth for every field API name across all Get-Rescue widgets, organized step-by-step. Corrections happen there first.

**Zoho access**: see **`../ACCESS.md`** (repo root) — which Forms/Reports this widget needs permission to.

## Vendor-specific behavior (user-confirmed 2026-07-31: "Vendor does everything an individual, PLUS hands off if fleet")

- **Accept/Reject** is common to both service types (one screen, branches internally on `Service_Type` for which fields it writes — `Service_Acceptance`/`Reject_Reason` for RSR, `Service_Acceptance_For_Tow`/`Reject_Reason1` for TOW).
- **After Accept**: if the logged-in vendor is **Individual**, they proceed through the exact same on-road screens `technicianTicket`/`driverTicket` have (Reach→WIP→Payment for RSR, Reach→Loading→Reached Drop→Unloading→Payment for TOW) — literally the same logic, just branched by `Service_Type` inside this one widget instead of living in two separate ones.
- If the vendor is **Fleet**, Accept still writes `Status:"RSP ON THE WAY"` immediately, but a hand-off panel (`showHandoffPanel()`) then appears in place of returning to the dashboard — lists the fleet's own technicians (best-effort match against `technicians_Report`), lets the vendor assign one (writes `Assigned_Technician`/`Assigned_Technician_Email` plus the branch-specific `Technician_Email`/`Technician_Email1`), and the assigned technician then picks the ticket up in their own `technicianTicket`/`driverTicket` widget via the email match already built there.
- **Individual vs. Fleet detection** (`IS_FLEET`, set once at boot from the vendor's own profile record) and **the fleet→technician link** are both **unconfirmed guesses** (`Vendor_Type` field, `Fleet_Vendor` link field) — flagged clearly in code comments and `getRescueTicket/QUESTIONS_TO_ASK.md`, verify against real Zoho Studio schema.
- **Toggle** targets `My_Availability_Vendor` (the vendor-specific report `toggleGetRescue` already used), not `technicians_Report`.

## 2026-07-31 (later) — Action 5 reconciliation

Same round of fixes as `technicianTicket` (see its own README's matching entry, not repeated here): in-place navigation between screens via `goToStatus()` (the fleet hand-off panel already worked this way; the individual on-road path now does too), a distinct `Service_Reject_Time` field, a "Back" button on the Reject panel, an expanded `Task_Rejections` log, and (at the time) two unresolved client-source conflicts around ETA/Distance field naming and the Reject status value — see `getRescueTicket/QUESTIONS_TO_ASK.md`.

## 2026-07-31 (later still) — Action 6 reconciliation

Same round of fixes as `technicianTicket` (see its own README's matching entry, not repeated here): ETA now shown/editable on both Reach screens (RSR and TOW), a "Back" button on the Cancel panel, and the new `RSP_Start_Lat`/`RSP_Start_Lon` + `Cancellation_Time`/`Cancel_Location_Lat`/`Cancel_Location_Lon`/`Cancel_Distance` fields, shared by `confirmCancelCommon()` across both branches.

## 2026-07-31 (even later) — Action 7 reconciliation

Same round of fixes as `technicianTicket`'s own RSR WIP screen (see its own README's matching entry, not repeated here — this widget's `renderWipScreen()` is the same RSR-only screen, used when the vendor is Individual on an RSR ticket): mandatory Issue Resolved validation, `RSP_Completion_Time` shared by Work Completed and Service Reject, a "Back" button + relabeled "Service Reject" on the Cx Rejection panel, and a plain-text `Unresolved_Note` stand-in for the mock's own uncertain "voice note" idea.

## 2026-07-31 (yet later) — Action 8 reconciliation

Same explicit `Payment_Status` (Pending/Success) fix as `technicianTicket` (see its own README's matching entry, not repeated here) — shared by this widget's one `renderPaymentScreen()` across both RSR and TOW branches.

## 2026-07-31 (yet later still) — Action 5a (TOW) confirms this screen as-is

The Action 5a mock (Service Acceptance/Rejection — TOW) is this widget's own TOW-branch Accept/Reject screen, and it matches what was already built — no changes needed. It also resolves the Action 5 Reject-status question: `Status:"RSP REJECT"` is confirmed correct for both branches, per `getRescueTicket/QUESTIONS_TO_ASK.md`.

## 2026-07-31 (later again) — Action 6a (TOW) confirms this screen as-is

Same confirmation as `driverTicket`'s own matching entry (see its README) — this widget's `renderReachScreenTow()` already matched the Action 6a mock field-for-field, no code changes needed.

## 2026-07-31 (later again still) — Action 7a (TOW) reconciliation: added the missing Cx-Reject panel, live-GPS drop distance

Same fixes as `driverTicket`'s own matching entry (see its README, not repeated here): added a `Customer Rejected` → Cx-Rejection panel to this widget's TOW Loading screen (was missing entirely, only had "Vehicle Picked"), shared `Rejection_Reason`/`RSP_Completion_Time` fields, `Status:"REMAINING FEE DUE"`; `vehiclePicked()` now uses the vendor's own live GPS position for the Drop distance/ETA calculation instead of the static breakdown location.

Also fixed this round: a real status-derivation bug in the sibling `getRescueTicket` agent widget affecting all vendor-side viewOnly stages — see `getRescueTicket/README.md`'s own matching entry.

## 2026-07-31 (later again yet still) — Action 7b & Action 7c reconciliation

Same fixes as `driverTicket`'s own matching entry (see its README, not repeated here): the "Mechanic waypoint" question is resolved (no code change — the 3-leg calc already matches), `reachedDrop()` now also stamps `Drop_Location_Arrival_Time` + this vendor's own live GPS as `RSP_Drop_Location_Lat`/`Lon`, `confirmDropped()` now stamps the shared `RSP_Completion_Time`, and `Handover_to_Number` is now a real number input.

## 2026-07-31 (later again yet still further) — Task_Rejections form built; Case_ID/Vehicle corrected to Lookup writes

Same fix as `technicianTicket`'s own matching entry (see its README) — `Case_ID`/`Vehicle` in the now-built `Task_Rejections` form are real Lookups, so `rejectService()` now sends `r.ID`/`r[CONFIG.vehicleField]` instead of display strings.

## 2026-07-31 (later still again) — Real bug fixed: vendors couldn't see their tickets at all (invalid `max_records`)

Same root bug as `technicianTicket`'s own matching entry (see its README) — `loadAvailability()` was calling `getRecords` with `max_records:1`, which Zoho rejects (only 200/500/1000 allowed, error 9250), breaking `boot()` before it ever reached `loadDashboard()`. Fixed to `200`. This widget had a **second** occurrence of the same mistake in `showHandoffPanel()`'s fleet-technician-list fetch (`max_records:100`) — also fixed to `200`.

## 2026-07-31 (later again yet still further, once more) — Real bug fixed: invited vendors saw an empty dashboard

Reported live: the Agent Ticket Report clearly showed a vendor listed in `Vendors1` on several tickets at `RSP IRA (INVITE RESPONSE AWAITED)`, but that vendor's own "My Tickets" showed zero. Root cause: `isMine()` only checked `Vendor_Email`/`Vendor_Email1`/`Vendor_Email2`, none of which are set yet at the invite stage — only `Vendors1` (the invite-candidate multiselect) is. Fixed by also matching the logged-in vendor's own name (read from their `My_Availability_Vendor` record at boot, stored as `MY_VENDOR_NAME`) against each `Vendors1` chip's parsed name (`vendorChipName()`, stripping the same `"[priority] Name | Phone | Status"` formatting `getRescueTicket`'s own `vendorDisplayName()` does).

**Needs live verification**: this assumes `vendors_Report`'s label field (whichever `getRescueTicket` uses to build `Vendors1`'s chip text) produces the exact same string as `My_Availability_Vendor.vendor_name` for the same vendor — see `getRescueTicket/QUESTIONS_TO_ASK.md`'s matching entry if a vendor is still missing from their dashboard after this.

## 2026-07-31 (later again yet still further, once more, again) — Dropped the 3-field email guess for dashboard matching

`isMine()` no longer checks `Vendor_Email`/`Vendor_Email1`/`Vendor_Email2` at all — per the user's own feedback, guessing at three field names was unnecessary now that `Vendors1` is a real Lookup matched by membership (fixed earlier this session). Removed the now-dead `CONFIG.myEmailFieldCandidates` too.

## 2026-07-31 (later again yet still further, once more, again, and once more) — New hidden `Vendor_Emails`/`Technician_Emails` fields; dashboard matching now checks them first

Per the user's request: `getRescueTicket` now auto-populates a hidden `Vendor_Emails` field (comma-separated) whenever `Vendors1` is saved. `isMine()` here now checks it first (exact email match, more reliable than the name-comparison added earlier), falling back to the `Vendors1` name-match only for older tickets saved before `Vendor_Emails` existed.

`assignTechnician()` (the fleet hand-off) now also writes a hidden `Technician_Emails` field alongside `Assigned_Technician_Email` — currently just mirrors the one directly-assigned technician's email, since this hand-off still direct-assigns rather than requesting several candidates at once (that's still the open "technicians multi-lookup" question — see `getRescueTicket/QUESTIONS_TO_ASK.md`).

## 2026-07-31 (later again yet still further, once more, again, and once more, again) — Lookup write shape: tried an object wrapper, reverted same day

Same as `technicianTicket`'s own matching entry (see its README) — `toLookupRef()`/the `{ID}` object wrapping for `Case_ID`/`Vehicle` was tried and reverted the same day; back to a bare `r.ID` / the unconverted raw `Vehicle` value.

## 2026-08-03 — New "Invites" architecture: dashboard now finds "my tickets" via Invites_Report first

Per the user's confirmed answers (reversing the earlier "Create_Case only" decision): `getRescueTicket` now creates one `Invites` record per invited vendor when the agent saves the Assignment step. This widget's `loadMyInvites()` fetches all `Invites_Report` records and matches `Vendor` (unwrapped to its display name) against `MY_VENDOR_NAME`, building `state.inviteByTicketId` (ticket ID → that vendor's own Invites record). `loadDashboard()` now treats a ticket as "mine" if it has a matching Invites record **or** the older `Vendor_Emails`/`Vendors1`-name match still applies — so nothing that used to show up disappears.

`acceptService()`/`rejectService()` now also write to this vendor's own Invites record (`updateMyInvite()`, best-effort/non-blocking): `Service_Acceptance_Next: true` on Accept, `{Status:"RSP REJECT", Reject_Reason}` on Reject — field name/value format guessed by analogy, not independently confirmed (see `getRescueTicket/QUESTIONS_TO_ASK.md`).

`assignTechnician()` (the fleet hand-off) now also creates an Invites record for the picked technician (`RSID` → this ticket, `Vendor` → the technician's own `technicians_Report` ID), so `technicianTicket`/`driverTicket` can find it the same way.

**Not yet implemented**: the later stages (Reach/WIP/Loading/Drop/Unloading/Payment) don't write their own "_next" progress flags to the Invites record yet — only Action 5/5a (Accept/Reject) does, to keep this round reviewable. Added new `apiUpdateReport(report, id, data)` (generalizes the old report-hardcoded `apiUpdate`) and `unwrapLookup()`/`lookupId()` helpers to support this.

## 2026-08-03 — Real bug fixed: "RSP_Start_Lon has exceeded its maximum digits" on Accept

Same fix as `technicianTicket`'s own matching entry — `getPositionSafe()` now rounds to 15 decimal places (updated same day from an initial 6, once the user widened the Decimal fields) via `toFixed(15)` at the source.

## 2026-08-03 (later) — `RSP_Start_Lat`/`RSP_Start_Lon` changed to Text, renamed `RSP_Start_Latitude`/`RSP_Start_Longitude`

Same change as `technicianTicket`'s own matching entry — renamed throughout this widget too.

## 2026-08-03 (later) — Real bugs fixed: null-button crash on every action, invalid `Service_Acceptance_Next`, diagnostic logging for Work Completed

User-reported live via console: **`Cannot set properties of null (setting 'disabled')`** was throwing on `acceptService`, `markReachedRsr`, and presumably every other action — each one's `finally` block re-enabled its own button unconditionally, but a successful action typically navigates away first (`goToStatus()`), destroying that button's DOM element before `finally` ran. Added a `setDisabled(id,val)` helper (no-ops on a missing element) and replaced all ~20 `$("...Btn").disabled=...` sites in this file with it.

**`Service_Acceptance_Next: true` confirmed rejected** (`["Invalid column value for Service_Acceptance_Next"]`, visible in the same console dump) — changed to the string `"Yes"`, matching every other Yes/No field's convention in this codebase.

**Added diagnostic logging for `workCompleted()`** (`Invalid column value for Status` on Work Completed) — payload + full error now logged; cause not yet found, see `getRescueTicket/QUESTIONS_TO_ASK.md`.

## 2026-08-03 (later still) — Diagnostic logging for Payment Received

Same "Invalid column value" issue, now on `Payment_Status` — user-reported live on the Payment Received button. Added the same payload-logging + full-error pattern to `confirmPaymentReceived()`.

## 2026-08-03 (yet later still) — Real bug fixed: `Payment_Status` isn't a separate field — resolved the earlier naming collision

The user shared the real field's live Zoho config: `PAID` / `PENDING` / `NOT APPLICABLE` — this is the SAME `Payment_Status` field the Quote step already uses for the booking fee, not a distinct `Pending`/`Success` field as originally guessed for the Action 8 reconciliation. `renderPaymentScreen()`'s `alreadySuccess` check and its live status-preview, plus `confirmPaymentReceived()`'s own write, now all use `PAID`/`PENDING` instead of `Success`/`Pending`. See `FIELDS.md`'s own resolved callout for the fuller writeup.

## 2026-08-03 (later again) — Real bug fixed: `Payment_Method1`'s actual option list; QR/gateway logic removed; camera+gallery both offered on every photo field

User shared the real field's live Zoho config for `Payment_Method1`: `Cash` / `UPI` / `Card` / `Net Banking` — not the earlier guessed `Payment Gateway` / `Direct` / `Cash`. Since none of the real options is an automatic online payment gateway, `renderPaymentScreen()`'s QR-code block and its "Payment Gateway"-gated logic were removed entirely: the receipt-photo block is now always shown, and every method requires that photo (or a zero remaining balance) before "Payment Received" unlocks. `Payment_Method1`'s select now lists the four real options, defaulting to `Cash`.

Also per explicit request: every photo-capture slot (`renderPhotoGrid()`) now shows two distinct "add" tiles instead of one — 📷 opens the existing live-camera modal (GPS/time watermark), and a new 🖼️ opens a plain gallery file picker (`openGallery()`, backed by a new hidden `<input type="file" id="galleryInput" accept="image/*">` with no `capture` attribute, so the OS shows its normal photo library / files chooser). Previously the only way to attach a photo without a working camera was the `cameraFallbackInput` error path, which still hints `capture="environment"` on mobile; the new gallery button is the explicit, always-available alternative.

## 2026-08-03 (later still) — All Zoho forms deleted and rebuilt from scratch: schema rebuild, renames, and a real Invites bug fixed

The user deleted every Zoho form/report (accumulated mismatched field names, unnecessary data) and asked for a complete build spec instead — now in `../FIELDS.md` (Parts 1–2) and `../ACCESS.md`. This widget's own code changed to match:

- **`Payment_Method1` renamed to `Payment_Method_Final`** — the "1" suffix was a Zoho auto-dedup artifact, not a meaningful name; the field itself is unchanged (still the Final Payment step's own method, separate from `Payment_Method` at the Quote step).
- **`vendors_Report` and the separate `My_Availability_Vendor` form are unified into one `Vendors_Report`.** There was never a real reason for a vendor to have two different records across two forms — this widget's own `CONFIG.vendorReport` now points at `Vendors_Report`, and `vendorNameField` changed from lowercase `vendor_name` to `Vendor_Name`.
- **`Vendors_Report`'s own field names for phone/priority/status/lat/lon (read via `vendorInfo()` in `getRescueTicket`, not this widget directly)** collapsed from multi-candidate guesses into single confirmed names — doesn't change this widget's own code, but see `getRescueTicket/README.md`.
- **Real bug fixed: `assignTechnician()`'s fleet hand-off was writing the wrong Invites field.** It sent `{RSID:r.ID, Vendor:tech.ID}` — but `tech.ID` is a `Technicians_Report` record ID, and `Vendor` is a Lookup pointed at `Vendors_Report`. A Zoho Lookup can only target one form, so this could never actually have resolved. `Invites` now has a separate `Technician` Lookup field (→ `Technicians_Report`), and this widget writes `Technician:tech.ID` instead.
- **`Task_Rejections.Case_ID` renamed to `Task_Rejections.RSID`**, matching `Invites`' own naming for the same "Lookup back to the ticket" concept — this widget's own `rejectService()` write updated to match.
- **`Technicians_Report.technician_name` renamed to `Technician_Name`**, `fleetTechnicianNameField` updated to match.

## 2026-08-11 — Reach now captures a live GPS fix; two brand-new distance fields added ("Missing Fields & Field Behavior" spec, user request)

User-provided spec item 2 ("Reached Breakdown – Technician Page... REACH button → Round Trip Distance → Calculate the distance from the rescue/breakdown location based on the location captured when REACH is clicked") surfaced a real gap: Reach never captured a live GPS fix at all before this, unlike Accept (`RSP_Start_Latitude`/`Longitude`) and Cancel (`Cancel_Location_Lat`/`Lon`), both of which already do via `getPositionSafe()`. `Roundtrip_Distance` (already computed at Reach, in `markReachedRsr()`) turned out to be a different, pre-existing measurement — Office(fixed constant)→Breakdown(static ticket coordinates)→Office — not tied to any live fix at all. Confirmed with the user this should be a **new, separate field**, not a redefinition of `Roundtrip_Distance` (which stays exactly as-is, used elsewhere for cost/reporting).

- **`markReachedRsr()`** and **`markReachedTow()`** now both call `getPositionSafe()` right before their existing `apiUpdate()`, writing the fix into two brand-new shared fields, `Reach_Location_Lat`/`Reach_Location_Lon` (Text, same digit-limit reasoning as `RSP_Start_Latitude`/`Longitude`), and computing a one-way distance from that fix to the ticket's own static breakdown location (`r.Latitude`/`Longitude`) via the existing `calcDistanceETA()` helper — written to a new **RSR-specific** field, `Distance_Reach_To_Breakdown`, from `markReachedRsr()`, and a new **TOW-specific** field, `Distance_Reach_To_Breakdown_TOW`, from `markReachedTow()` (matches `Distance_To_Breakdown`/`Distance_To_Breakdown_TOW`'s own existing RSR/TOW split at Accept). Best-effort — a denied/unavailable location just skips these two fields, same tolerance Accept/Cancel's own fixes already have.
- **`confirmCancelCommon()`** — spec item 2 also asked for "Distance: Cancel Location → Rescue/Breakdown Location," which turned out not to exist under any name (the existing `Cancel_Distance` is a *different* measurement, Accept location → Cancel location — confirmed via audit, explicitly noted in `FIELDS.md`). Added right after the existing `Cancel_Distance` calc: a new shared field, `Distance_Cancel_To_Breakdown`, computed from the same Cancel-click position to `r.Latitude`/`Longitude` via `calcDistanceETA()`.
- **Companion changes**: `getRescueTicket` now shows/edits all of these (Agent-entering-mode only) on its own Reach step, and `technicianTicket`/`driverTicket` got the identical treatment mirrored into their own reach/cancel handlers — see those widgets' own README entries and `getRescueTicket/README.md`'s matching entry for the full cross-widget picture.

**Five brand-new `Create_Case` fields — not yet created in Zoho Studio as of this entry**: `Reach_Location_Lat`, `Reach_Location_Lon`, `Distance_Reach_To_Breakdown`, `Distance_Reach_To_Breakdown_TOW`, `Distance_Cancel_To_Breakdown` (Text/Text/Number/Number/Number respectively — see `FIELDS.md`'s own build-spec rows).

**Real caveat worth flagging, unlike the field-additions earlier this session that shipped as isolated, separate save calls**: these five are bundled into the SAME `apiUpdate()` payload as `Reach_Time`/`Status`/`Cancellation_Time` — the fields the vendor's REACH/CANCEL buttons actually depend on to advance the ticket at all. This repo's established pattern elsewhere (adding a brand-new field straight into an existing STAGES save, dozens of times this session with no reported failures) is the precedent this follows, which suggests Zoho's own `updateRecordById` most likely tolerates an unrecognized key by ignoring it rather than rejecting the whole request — but that's an inference from precedent, not something independently confirmed for this exact bulk-update shape. **If Reach or Cancel stops working (button click does nothing / silent failure toast) once these fields are live, before they're created in Zoho, that's the signal to create them immediately** — it would mean this inference was wrong for this case.

Syntax-checked (`node -e` script against the `<script>` block) and packed (`vendorTicket.zip`).

## 2026-08-11 (later still) — User confirmed: `Reach_Location_Lat`, `Reach_Location_Lon`, `Distance_Reach_To_Breakdown`, `Distance_Reach_To_Breakdown_TOW`, `Distance_Cancel_To_Breakdown` now created in Zoho Studio

See `getRescueTicket/README.md`'s own matching entry for the full picture (all 11 fields flagged as pending across this session, not just this widget's five) — `FIELDS.md` updated accordingly. The "does the whole payload get rejected on an unrecognized key" caveat this widget's own entry above flagged is now moot for these five specifically; worth a real Reach/Cancel test on a live ticket to confirm the values actually land correctly.

## 2026-08-17 — New: periodic live-location heartbeat while toggled Online (user request, feeds `getRescueTicket`'s own vendor picker)

User request: *"when vendor is offline it is supposed to show distance from the vendor's static location (as in vendor table); when vendor is online it is supposed to show the distance from the vendor's current location (latest location from app)"* — confirmed scope **Vendors + Technicians + Drivers**, frequency **every 2–3 minutes**, and to **use the Google API for the actual distance calculation** on `getRescueTicket`'s side (see that widget's own matching entry for the picker-side half of this feature). This widget's half is the actual location source: nothing anywhere in the codebase previously tracked a vendor's live position, only the static `Address` on `Vendors_Report`.

- **Two brand-new fields**, not yet created in Zoho Studio: `Current_Latitude`/`Current_Longitude` on `Vendors_Report`, written through `My_Availability_Vendor` (this widget's own toggle report) — `CONFIG.availabilityCurrentLatField`/`availabilityCurrentLonField`. See `../FIELDS.md`.
- **New heartbeat**: `LOCATION_HEARTBEAT_MS=150000` (2.5 min), `pingCurrentLocation()` (grabs a fix via the existing `getPositionSafe()` and writes it through `updateRecordById` against `currentAvailabilityRecord`, non-fatal/best-effort — a denied or unavailable fix just skips that tick and retries next interval), `startLocationHeartbeat()`/`stopLocationHeartbeat()`.
- **`onToggleClick()`** now calls `startLocationHeartbeat()`/`stopLocationHeartbeat()` right after the Online/Offline `Availability_Status` write succeeds, based on which way the vendor just toggled.
- **`loadAvailability()`** now also calls `startLocationHeartbeat()` at boot if the vendor is already Online when the app opens/reopens (previously the toggle's own on-click was the only place a heartbeat could ever have started, which would've missed this case entirely).

**Not yet verified live**: real device geolocation and the actual write landing on `Current_Latitude`/`Current_Longitude` can't be exercised outside a live browser/device session — syntax-checked via `node -e` against the `<script>` block only. Same mirrored heartbeat added to `technicianTicket`/`driverTicket` — see those widgets' own README entries (field/report names differ slightly there: no separate `My_Availability_Vendor`-style indirection, the toggle writes straight to `Technicians_Report`).

Packed (`vendorTicket.zip`).

## 2026-08-19 — Real bug fixed: Vendor Email / Vendor Email (Assignment) / Assigned Vendor all stayed blank on the agent's own Accept screen after a fleet hand-off (user-reported live, screenshot)

User's exact repro: a technician accepted a job that came through a Fleet vendor's own hand-off — Expected Arrival Time was correctly populated on the agent's own Accept screen (confirming the technician genuinely accepted), but Vendor Email, Vendor Email (Assignment), and Assigned Vendor all showed completely blank.

**Root cause, confirmed by reading every write path involved**: `assignTechnician()` (the fleet hand-off action) has never recorded the FLEET VENDOR's own identity anywhere — only the technician's (`Assigned_Technician`/`Assigned_Technician_Email`/`Technician_Email(1)`). `technicianTicket`'s/`driverTicket`'s own direct acceptance has no reliable way to backfill this later (a technician's own record doesn't reliably carry back to a specific vendor at accept time) — and since a technician/driver can only ever see a ticket via an `Invites` record this exact function creates, this is the one and only correct place to fix it, since `currentVendorRecord` (the fleet vendor, actively handing the job off right now) is directly and reliably available at this exact moment.

Also found and fixed in the same pass (not separately reported, but the same class of gap): the 2026-08-06 fix for a **direct** vendor acceptance (`acceptService()`) claimed in its own comment to cover "Vendor Email (Assignment)" too, but the code only ever set `Vendor_Email1`/`Vendor_Email2` — the bare `Vendor_Email` field (getRescueTicket's own "Vendor Email (Assignment)" label) was never actually written there either.

**Fix**:
- `assignTechnician()` now also writes `Assigned_Vendor` (this fleet vendor's own `Vendors_Report` id) and the vendor's own email into both the bare `Vendor_Email` field and the correct RSR/TOW branch field (`Vendor_Email1`/`Vendor_Email2`) — using `currentVendorRecord`, the exact same identity `acceptService()` already records for a direct acceptance. A fleet hand-off now looks, from the agent's own Accept screen, identical to a vendor accepting directly.
- `acceptService()` (direct vendor acceptance) now also writes the bare `Vendor_Email` field alongside its existing `Vendor_Email1`/`Vendor_Email2` write.

**Not touched**: `technicianTicket`'s/`driverTicket`'s own direct acceptance functions — they can only ever be reached through this same hand-off, so by the time either one runs, the vendor identity is already correctly on the record.

**Tested** against the real shipped code (Node-VM harness, stubbing `apiUpdate()` to capture the actual payload instead of hitting a real API) — 8 checks covering both the RSR and TOW hand-off branches: `Assigned_Vendor`/bare `Vendor_Email`/the correct branch-specific email field are all now set to the fleet vendor's own identity, the wrong branch's own field is correctly left unset, and the technician's own identity is confirmed still recorded correctly alongside it. All passed.

Syntax-checked and packed (`vendorTicket.zip`).


## 2026-08-19 (later) — Investigating: Vendor Email/Assigned Vendor still blank after a DIRECT vendor acceptance, even with the same-day fix redeployed — diagnostics added, not yet a confirmed fix

User's follow-up report: on a different ticket, a vendor accepted directly on their own phone (not a fleet hand-off) — Assigned Vendor/Vendor Email/Vendor Email (Assignment) were all still blank on the agent's own Accept screen, and the user confirmed the fix from the entry above **was** already redeployed to Zoho Creator. So this is a genuinely different situation from the fleet hand-off gap fixed earlier the same day.

**Leading hypothesis, not yet confirmed with live data**: `currentVendorRecord` (matched against `CONFIG.vendorReport`/`Vendors_Report`, used for `Assigned_Vendor`/`Vendor_Email*`) and `currentAvailabilityRecord` (matched against the SEPARATE `My_Availability_Vendor` report — moved back out on its own 2026-08-05, see that entry) are two INDEPENDENT email matches. The vendor's toggle/dashboard working at all only proves the latter resolved — nothing guarantees the two report's own record sets stay in sync for every vendor. If `currentVendorRecord` itself never matched for this specific vendor, `Assigned_Vendor`/`Vendor_Email*` would stay blank regardless of the accept-time fix, since that fix only ever had a real vendor identity to write if this match succeeded in the first place.

**Deliberately did NOT add a fallback** (e.g. reusing `currentAvailabilityRecord`'s own id if `currentVendorRecord` fails) — if that's genuinely a different underlying record, writing its id into `Assigned_Vendor` (a Lookup targeting `Vendors_Report` specifically) risks silently saving a WRONG vendor reference, which is worse than a clearly blank one. Writing unverified data into a financially/operationally meaningful field is not a risk worth taking without confirming the theory first.

**What was added instead**: a clear diagnostic in both `acceptService()` and `assignTechnician()` — the moment either fires with `currentVendorRecord` still null, it now logs the exact `LOGIN_EMAIL` that failed to match and whether `currentAvailabilityRecord` resolved instead, right at accept time (not just `loadAvailability()`'s own easy-to-miss boot-time warning).

**Next step to actually resolve this**: reproduce on a vendor's device with console access (a desktop browser logged in as that same vendor is far easier to check than a phone) and look for `"[GR-Vendor] acceptService — currentVendorRecord is NULL"` in the console right after tapping Accept. That single line will confirm or rule out this theory with real data — if it fires, the `LOGIN_EMAIL` and `currentAvailabilityRecord` details it logs pinpoint exactly what to fix in `Vendors_Report`'s own data (e.g. a missing/mismatched email) or in `CONFIG.vendorEmailField`'s own guess.

**Tested** against the real shipped code (Node-VM harness, stubbing `apiUpdate()`) — 6 checks: confirmed the diagnostic fires with the exact right details when `currentVendorRecord` is null, confirmed it does NOT fire (no false alarm) when the match succeeds normally, and confirmed `Assigned_Vendor` genuinely comes through as `null` (not silently omitted) in the null-match case, exactly matching the reported symptom. All passed.

Syntax-checked and packed (`vendorTicket.zip`).

## 2026-08-19 (later still) — Real fix shipped for the above investigation: a second live ticket (RSID396829) confirmed the same symptom — `acceptService()`/`assignTechnician()` now retry the real match once, and fall back to `LOGIN_EMAIL` (never a guess) for the plain email fields

User's second report, same day: on yet another ticket (RSID396829), a vendor accepted directly on their own phone — Expected Arrival Time (6:40 PM) was correctly populated, proving the accept genuinely went through, but Vendor Email/Vendor Email (Assignment)/Assigned Vendor were all still blank. This is the SECOND live occurrence of the exact symptom the diagnostic-only entry above was watching for — enough to justify a real fix rather than waiting on console access from a vendor's device.

**What changed**: the vendor-identity match ("`Vendors_Report` record whose `Email` equals `LOGIN_EMAIL`", previously only ever run once, at boot, inside `loadAvailability()`) is now its own reusable function, `resolveVendorRecord(logContext)`. Both `acceptService()` and `assignTechnician()` call it again, right at accept/hand-off time, if `currentVendorRecord` is still null — a real re-check of the same real query, not a guess, so it directly fixes a slow-mobile-network/boot-timing race outright (the most likely explanation for why this reproduces on a phone specifically: `loadAvailability()`'s own one-shot fetch losing a race is far more plausible on a flaky mobile connection than on a desktop).

**If the retry still finds nothing** (a genuine `Vendors_Report` data gap for that vendor's login email, not a timing issue): `Assigned_Vendor` — a real Lookup targeting `Vendors_Report` — is still deliberately left blank, never fabricated, for exactly the reason the diagnostic-only entry above already gave. But the plain `Vendor_Email`/`Vendor_Email1`/`Vendor_Email2` fields no longer have to stay blank too — they now fall back to `LOGIN_EMAIL` itself, the actual authenticated login that just tapped Accept. That's a real, verified fact (not a guess) and safe to write into a plain email field, unlike fabricating a Lookup id. Applied identically in `assignTechnician()` for the fleet-vendor's own email.

**Not resolved by this fix, still worth knowing**: if the underlying cause turns out to be the permanent data-gap case rather than a timing race, `Assigned_Vendor` will still show blank on the agent's screen even after this — the diagnostic warnings from the entry above still fire in that case and still point at exactly which `Vendors_Report` record/email to fix.

**Tested** against the real shipped code (Node-VM harness, stubbing `apiGetReport()`/`apiUpdate()`) — 8 checks: retry succeeding fully resolves `Assigned_Vendor` + `Vendor_Email` from the re-fetched record; retry still failing correctly leaves `Assigned_Vendor` null while `Vendor_Email`/`Vendor_Email1` fall back to `LOGIN_EMAIL`; already-resolved case shows no regression and never even attempts the retry. All passed. Also re-ran the 2026-08-19 fleet hand-off test above to confirm the `resolveVendorRecord()` extraction didn't change that flow — still passes.

Syntax-checked and packed (`vendorTicket.zip`).

## 2026-08-19 (yet later) — Follow-up on the above: user confirmed a matching Vendors_Report record genuinely exists (RSID396869) yet Assigned Vendor still showed blank — 200-record fetch cap removed, diagnostic enriched to pinpoint the real cause next time

User's follow-up, same day: on a THIRD ticket (RSID396869), Vendor Email/Vendor Email (Assignment) now correctly showed the vendor's real email (confirming the LOGIN_EMAIL fallback above IS working) — but Assigned Vendor still showed "— select —". Asked directly: confirmed the build was redeployed, AND confirmed a Vendors_Report record with that exact email genuinely exists. That rules out the "genuine data gap" explanation the previous entry left open — something else is preventing the match.

**Two real, safe improvements made without guessing at a fix that could go wrong**:
1. `resolveVendorRecord()`'s own `apiGetReport(CONFIG.vendorReport, ...)` call now fetches up to **1000** records instead of 200 (the next allowed tier — Zoho's own `getRecords` only accepts 200/500/1000). If Vendors_Report has quietly grown past 200 records, this exact symptom — a real record existing but never being *fetched* in the first place — would be silently invisible to the old 200-cap. Costs nothing, pure upside.
2. The "no match found" diagnostic now dumps **every fetched record's own value under `CONFIG.vendorEmailField`** (as an ID+email pairing), not just the first record's raw keys. If every single one comes back `undefined`, that proves `CONFIG.vendorEmailField` ("Email") isn't this report's real API name for that column (a display-label-vs-API-name mismatch, common in Zoho after a schema edit) — if real emails show up but genuinely never match, it's a whitespace/casing/typo issue instead. Either way, the very next time this reproduces, the console tells us exactly which one it is instead of needing another round of screenshots back and forth.

**Deliberately NOT done**: no field-name-candidate fallback (the kind `fleetTechFirstMatch()` already uses for Technicians_Report elsewhere in this file) added here — trying several guessed field names and taking whichever one matches risks matching the WRONG vendor's record on some unrelated column, and writing that into `Assigned_Vendor` (a real Lookup) would be a worse outcome than the current blank. Confirming the real field name first, then hardcoding it correctly, is the safe order of operations — this entry's diagnostic is what that confirmation now depends on.

**Tested** against the real shipped code — re-ran the full 2026-08-19 vendor-identity test suite (retry/fallback scenarios + the fleet hand-off suite) to confirm the 200→1000 change and the enriched warning don't alter any actual behavior, only the diagnostic's own verbosity and the fetch ceiling. All still pass.

Syntax-checked and packed (`vendorTicket.zip`).


## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
