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

## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
