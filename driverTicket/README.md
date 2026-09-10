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

## 2026-08-11 — Reach now captures a live GPS fix; two brand-new distance fields added (`Distance_Reach_To_Breakdown_TOW`, `Distance_Cancel_To_Breakdown`)

Per the user's "Missing Fields & Field Behavior" spec: Reach Breakdown (`markReached()`) never captured a live GPS fix at all before this, unlike Accept (`RSP_Start_Latitude`/`RSP_Start_Longitude`) and Cancel (`Cancel_Location_Lat`/`Cancel_Location_Lon`), both of which already do. Added a best-effort `getPositionSafe()` call right before the existing `apiUpdate()` in `markReached()`, writing `Reach_Location_Lat`/`Reach_Location_Lon` (both brand-new fields, shared across RSR/TOW — not split into `_TOW`/plain variants like some other fields in this codebase) and, whenever a fix is actually available, a new `Distance_Reach_To_Breakdown_TOW` field (TOW-specific — this widget is TOW-only, so the `_TOW` variant is the right one here, not a plain RSR-named field) computed via the existing `calcDistanceETA()` from that live position to the ticket's static `Latitude`/`Longitude` breakdown point. Same denied/unavailable-location tolerance Accept's/Cancel's own GPS fixes already have — a skipped fix just means these two fields are silently omitted from the payload, nothing blocks the "Reached Location" action itself.

Also added `Distance_Cancel_To_Breakdown` (brand-new, shared field) to `confirmCancel()`, inserted right after its existing `Cancel_Distance` calc. Deliberately a different measurement from that neighboring field: `Cancel_Distance` is Accept-location → Cancel-location (already shipped), while this new one is Cancel-location → the ticket's own static breakdown location, which nothing previously computed. This widget only has the one cancel path (`confirmCancel()`, sets `Status:"RSP CANCELLED"`) — the Loading screen's "Customer Rejected" branch (`confirmCxRejectLoading()`) is a different outcome entirely (routes to `REMAINING FEE DUE`, not a cancellation) and never computed `Cancel_Distance` either, so it was correctly left untouched.

None of these four fields (`Reach_Location_Lat`, `Reach_Location_Lon`, `Distance_Reach_To_Breakdown_TOW`, `Distance_Cancel_To_Breakdown`) exist in `Create_Case` in Zoho Studio yet, same as every other guessed/new field name already in this codebase — they need to be created there before these writes actually persist. See `../FIELDS.md` for the field-name ledger (per this file's own header, corrections happen there first).

**Real caveat worth flagging** (see `vendorTicket/README.md`'s own matching entry for the full reasoning): these four are bundled into the SAME `apiUpdate()` payload as `Reach_Time`/`Status`/`Cancellation_Time` — the fields Reach/Cancel actually depend on to advance the ticket at all. This repo's own precedent (new fields added straight into an existing save, dozens of times this session, no reported failures) suggests Zoho's `updateRecordById` likely tolerates an unrecognized key rather than rejecting the whole request — but that's inferred, not independently confirmed for this exact case. If Reach or Cancel stops working before these fields are created in Zoho, that's the signal this inference was wrong.

## 2026-08-11 (later still) — User confirmed: all four fields above now created in Zoho Studio

See `getRescueTicket/README.md`'s own matching entry for the full picture (all 11 fields flagged as pending across this session, not just this widget's four) — `FIELDS.md` updated accordingly. The payload-rejection caveat above is now moot for these four; worth a real Reach/Cancel test on a live ticket to confirm the values actually land correctly.

## 2026-08-17 — Live location heartbeat while Online: `Current_Latitude`/`Current_Longitude`

User request: "when vendor is online it is supposed to show the distance from the vendor's current location (latest location from app); when vendor is offline it is supposed to show the distance from the vendor's static location (as in vendor table)" — user confirmed this applies to Technicians/Drivers too, not just Vendors, and that the refresh should happen roughly every 2-3 minutes while online. Until now, `Technicians_Report` only ever held a driver's static/home location (whatever it was set to once, off-app) — there was no way for `getRescueTicket`'s own Assignment-step vendor/technician picker to show a distance based on where an Online driver actually is *right now*.

This widget's own toggle already targets the SAME `Technicians_Report` (`CONFIG.techReport`) that `technicianTicket` writes to — confirmed by reading this file's own `CONFIG` block before making this change, not assumed — so the identical fix from `technicianTicket`'s own matching entry (see its README) applies here verbatim, just re-homed to this widget's own variable/function names:

- **Two new `CONFIG` fields** (widget.html, right after `techNameField`, ~widget.html:241-242): `techCurrentLatField: "Current_Latitude"`, `techCurrentLonField: "Current_Longitude"` — same field names `getRescueTicket`'s own picker expects, written directly to this driver's own `Technicians_Report` record.
- **`pingCurrentLocation()`** (widget.html:1384) — takes a fresh `getPositionSafe()` fix (the same helper already used throughout this file's Accept/Reach/Cancel/Vehicle-Picked/Reached-Drop screens) and writes it to `Current_Latitude`/`Current_Longitude` via `updateRecordById`, guarded on `currentTechRecord` being set; best-effort throughout (a denied/unavailable fix, or a failed write, just silently skips that one tick, logged via `console.warn("[Get-Rescue] pingCurrentLocation failed...")`).
- **`startLocationHeartbeat()`/`stopLocationHeartbeat()`** (widget.html:1396/1401) — `start` fires an immediate ping (no waiting for the first interval tick) then runs `pingCurrentLocation()` on a `setInterval` every `LOCATION_HEARTBEAT_MS` (widget.html:1382, 150000ms / 2.5 min — the midpoint of the user's own 2-3 min range); `stop` clears the timer. `start` always calls `stop` first so repeated toggling never stacks two timers.
- **Wired into the toggle**: `onToggleClick()` (widget.html:1413) now calls `startLocationHeartbeat()` when going Online and `stopLocationHeartbeat()` when going Offline. `loadAvailability()` (widget.html:1367, the boot-time loader) also calls `startLocationHeartbeat()` immediately after setting the toggle's initial on/off UI state, if the driver is already Online when the widget loads — so a page refresh/reopen while Online doesn't require toggling off and back on to resume fresh pings.

Neither `Current_Latitude` nor `Current_Longitude` exists on `Technicians_Report` in Zoho Studio yet — same "ships safely ahead of the field existing" pattern used throughout this app (per this file's own header, corrections/pending-field tracking happen in `../FIELDS.md` first). See `vendorTicket/README.md`'s and `technicianTicket/README.md`'s own matching 2026-08-17 entries for the fuller cross-widget picture (vendorTicket's own toggle writes through a different intermediate report, so its version of this fix looks structurally different even though the end result — `Current_Latitude`/`Current_Longitude` kept fresh while Online — is the same).

## 2026-08-20 — Real bug fixed: after rejecting a ticket, it silently vanished from "My Tickets" with no lasting sign it was ever rejected (user-reported live)

User's exact report, confirmed after a round of clarifying questions: "if rejected... vendor acceptance status is not being displayed" — turned out to mean this widget's OWN "My Tickets" list, not the agent's `getRescueTicket` dashboard or the per-vendor Invite Status panel there (both already handle Rejected correctly, see `getRescueTicket/README.md`). Confirmed root cause: `rejectService()` writes `Status:"RSP REJECT"`, but `RSP REJECT` was never in `MY_STATUSES` — so the instant that save landed, the ticket dropped out of `fetchMyTickets()`'s own filter entirely. The reject toast fades a couple seconds later and the ticket is just gone, with nothing left anywhere on this widget showing "you rejected this."

**Confirmed fix** (user's own choice between two options — keep it in the main list vs. a separate "recent activity" section): stays in the main **"My Tickets" list** with a clearly-styled **Rejected** pill, rather than a separate screen/tab.

**Fix**:
- `MY_STATUSES` now includes `"RSP REJECT"`.
- `STATUS_PILL["RSP REJECT"]` added — `{cls:"rej", label:"Rejected"}`, styled with the same `--stop`/`--stop-soft` red already used for Reject Reason/Cancel Reason required-field markers elsewhere in this file (new `.st.rej` CSS rule).
- `renderActionScreen()` gained a dedicated `RSP REJECT` branch — tapping back into a rejected ticket now shows a clear "You Rejected This Ticket" panel with the real reject reason (`Reject_Reason1` — this widget is TOW-only), instead of falling through to the generic "Nothing to do on this ticket right now" filler (which used to be unreachable for a rejected ticket anyway, since it never stayed in the list long enough to tap back open).

**Worth knowing — a real consequence of this change, not yet separately reported as a problem**: `Status` is a single ticket-level field. If a ticket was invited to **multiple** vendors and only ONE of them rejects, the ticket now shows "Rejected" in **every** invited vendor's own "My Tickets" too — including ones who haven't responded at all yet — since there's no per-vendor status at this widget's own dashboard level (only `getRescueTicket`'s separate Invites-based panel tracks that distinction). Flagging rather than silently deciding it away: if this turns out to be confusing in practice with multi-vendor invites, the fix would need to check this vendor's own Invites-report row instead of the ticket-level `Status` before showing the Rejected pill — not implemented here since the user's own confirmed scope was the single-vendor case.

**Tested** against the real shipped code (Node-VM harness, all three widgets — `vendorTicket`/`technicianTicket`/`driverTicket`) — 10 checks: `MY_STATUSES`/`STATUS_PILL` updated correctly in each, and a live `renderActionScreen()` call confirms the new "You Rejected This Ticket" panel actually renders the real reject reason (checking both `Reject_Reason`/`Reject_Reason1` where driverTicket branches by `Service_Type`). Re-ran every other same-day vendorTicket test suite (vendor-identity retry/fallback, fleet hand-off, live-location heartbeat) to confirm no regression — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-21 — Real bug fixed: agent's own "Vendor Invite Status" panel kept showing "Awaiting response" even after a genuine accept (user-reported live: "acceptance status is not being displayed")

User's exact report: on the Assignment page in the agent portal, invited vendors' own acceptance status wasn't showing correctly even after they'd genuinely accepted.

**Root cause, traced through the same identity-resolution gap found earlier this session**: `state.inviteByTicketId` (the cache `updateMyInvite()` uses to find which Invites row belongs to this ticket) is built exactly ONCE, at `loadDashboard()`'s own `fetchMyTickets()`/`loadMyInvites()` call, by matching against whatever currentTechRecord held AT THAT MOMENT. If currentTechRecord hadn't resolved yet then (the exact gap `resolveVendorRecord()`'s own 2026-08-19 retry was built for), this cache stayed empty for that vendor for the rest of the session — even after a LATER retry inside `acceptService()` itself successfully resolved currentTechRecord, `updateMyInvite()` kept trusting the stale, empty cache and silently no-op'd on every write, since it never re-checked. `Service_Acceptance_Next`/`Status` on the real Invites row never got written — and since the agent's own "Vendor Invite Status" panel (`assignInviteStatusHtml()` in `getRescueTicket`) reads directly off that same Invites record, it kept showing "Awaiting response" forever, even though the vendor genuinely accepted and the ticket's own `Status` field (written directly, not through Invites) updated correctly the whole time.

**Fix**: `updateMyInvite()` now retries a real, direct `CONFIG.invitesReport` lookup (matched on the Technician Lookup field, same "read the report, filter client-side by id" pattern used everywhere else in this app for this exact report) whenever the cached invite isn't found — instead of silently giving up. A successful retry also backfills `state.inviteByTicketId` so later calls in the same session don't need to retry again. If currentTechRecord is STILL null even at retry time (a genuine identity gap, not just a stale cache), this still safely no-ops with a clear diagnostic — never guesses at which Invites row to write to.

**Tested** against the real shipped code (Node-VM harness, stubbing `apiGetReport()`/`apiUpdateReport()`) — confirmed the retry finds and writes to the real Invites row despite a stale/empty cache, confirmed it backfills the cache for next time, confirmed no write is attempted (and nothing throws) when the identity genuinely never resolved, and confirmed the already-correct-cache case still writes directly with no regression. Re-ran the vendor-identity retry/fallback and fleet hand-off suites from earlier this session to confirm no interaction — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-21 — Real bug fixed: reopening the app after a photo was already uploaded showed the field as empty and re-blocked progress (user-reported live)

User's exact report: "when photo is clicked, it is being sent immediately and is visible on the agent portal. but if vendor closes app and opens his app again and comes back to the same page, the fields display as 'empty'. so vendor has to click all photos again in order to be able to move forward."

**Root cause**: every photo field's own display (`renderPhotoGrid()`) and every "N photos required" validation check across this widget only ever looked at `state.photos[field]` — a plain array of browser `File` objects picked/captured THIS session. That starts completely empty on every fresh app load, regardless of what's genuinely already saved on the ticket record itself (the upload really did succeed — that's exactly why it was already visible on the agent portal). So the grid rendered nothing, and the "at least N required" gate blocked the vendor from proceeding until they re-took photos that were never actually missing.

**Fix**:
- `fileCountFor(rawValue)`/`zohoFileDownloadUrl(recordId, fieldName, rawValue, index)` — ported directly from `getRescueTicket`'s own already-working versions (same account/app/report, so this isn't a new guess).
- `effectivePhotoCount(fieldName)` — the real count now used by every "N photos required" check: whatever's already on the ticket record (from a PREVIOUS session — `state.current` is only ever set fresh by `openTicket()`, never mutated after an upload, so this can never double-count) plus whatever's been picked THIS session.
- `renderPhotoGrid()` now renders a tile for each already-uploaded photo too — a real thumbnail via `zohoFileDownloadUrl()` where a usable URL can be built, hydrated asynchronously by the new `hydrateAlreadyUploadedThumbs()` (same onerror-recovery idea as `getRescueTicket`'s own file previews — reverts cleanly to a plain "✓ Saved" badge instead of leaving a broken image icon if the URL fails to load), otherwise falling straight to that plain badge. These tiles have no ✕ remove button — there's no "delete an already-uploaded photo" flow, only newly-picked-but-not-yet-final files can be removed before upload.
- Every "N photos required" check (Arrival, Cancel, Pre/Post-service, on-truck, VCRF, drop-location, unloaded, handover, receipt — every photo field in this widget) now uses `effectivePhotoCount()` instead of the raw session-only count.

**Tested** against the real shipped code (Node-VM harness with a richer fake DOM tracking appended children) — confirmed `effectivePhotoCount()` correctly counts an already-saved photo from a previous session even with zero picked this session (the exact reported scenario), correctly still blocks when genuinely nothing exists either way, and correctly avoids double-counting when both an already-saved AND a freshly-picked photo exist together; confirmed `renderPhotoGrid()` actually renders the already-saved placeholder tile and the count label reflects it instead of showing empty. Re-ran the vendor-identity retry/fallback, invite-status retry, and "Rejected stays visible" suites from earlier this session to confirm no regression — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-21 (later) — Two real bugs fixed: "Vehicle" showed a raw record id instead of a name, and "Location" showed raw decimal Latitude/Longitude digits (user-reported live, screenshot)

User's exact report/screenshot: on the Service Acceptance screen, `VEHICLE` showed `448881000000058404` (a raw `Vehicle_Master_Report` id) instead of a real vehicle name, and `LOCATION` showed `28.53000000000000, 77.17000000000000` (raw coordinates) with an explicit follow-up request to hide the lat/long fields.

**Vehicle name root cause**: `CONFIG.vehicleNameField` (`"Name"`) was a single, never-independently-reconfirmed guess for `Vehicle_Master_Report`'s real name column. If the real field is named something else on this account, `loadVehicleNames()` would build an entirely empty `VEHICLE_NAMES` map — every single vehicle, not just one — and `vehicleDisplayFor()`'s own fallback would show the raw id instead, exactly matching the screenshot.

**Fix**: `loadVehicleNames()` now tries several plausible field-name candidates (`Vehicle_Name`, `vehicle_name`, `Name`, `name`, plus the original config value) instead of one guess — same pattern this file's own `FLEET_TECH_NAME_CANDIDATES` already uses for an identical class of gap. If every candidate still comes up empty for every record, it now falls back to the raw id (unchanged, still correct — never guesses a wrong name) but with a clear console diagnostic explaining why, instead of silently doing nothing.

**Location fix**: the raw `Latitude`/`Longitude` fallback in `renderTicketInfo()` is removed — `Location` now only shows when there's a real `Break_Down_Location1` address string; otherwise the row is hidden entirely (the existing row-filter already drops any falsy value, so this required no new logic, just removing the fallback).

**Tested** against the real shipped code (Node-VM harness) — 20 checks across all three widgets: vehicle name resolves correctly via a fallback candidate when the primary guess is wrong (the exact reported case); a genuine total miss still safely falls back to the raw id with a diagnostic, not silently; `renderTicketInfo` no longer shows raw coordinates; a real address string still displays correctly (no regression). Re-ran the photo-persistence, vendor-identity retry, and invite-status retry suites from earlier this session — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-21 (later still) — New: "Navigate to Breakdown" added to the Accept screen (user request: "shows based on the service type like RSR and TOW")

User's exact ask: "NAVIGATE TO BREAKDOWN & NAVIGATE TO DROP LOCATION Buttons to be available in vendor app and these buttons shows based on the service type like RSR and TOW." Checking every screen turned up a real, consistent gap: the Reach screen already has "Navigate to Breakdown" (and, for TOW, "Navigate to Drop Location" already exists at the appropriate later step too), but the Accept screen — one step earlier — never had a breakdown nav button at all, in any of the three field widgets. `driverTicket`'s own Accept screen already had "Navigate to Drop Location" but no breakdown button; `technicianTicket`'s had neither.

**Fix**: `renderAcceptScreen()` now also shows the breakdown-nav link, placed ahead of the pre-existing (unchanged) Drop Location link — this widget is TOW-only, so both links are always relevant here.

**Tested** against the real shipped code (Node-VM harness) — 8 checks across all three widgets: Accept screen now shows "Navigate to Breakdown" for both RSR and TOW (where applicable); RSR correctly never shows "Navigate to Drop Location" (no drop leg exists for that service type); TOW/driverTicket's pre-existing "Navigate to Drop Location" is confirmed unchanged (no regression), with the new Breakdown link correctly appearing first, matching the real order of the flow (pick up, then drop). Re-ran the vehicle-name/hide-latlong, photo-persistence, and vendor-identity-retry suites from earlier this session to confirm no interaction — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-24 — Real bug fixed: Online/Offline toggle silently did nothing when `currentTechRecord` hadn't resolved (user-reported live: "online/offline toggle is not working")

Same fix as technicianTicket's own matching entry, applied here proactively — this widget shares the identical toggle architecture (currentTechRecord, loadAvailability(), onToggleClick()), so the same latent bug almost certainly exists here too, even without a separate report.

**Root cause**: `onToggleClick()` used to just `return` immediately whenever `currentTechRecord` hadn't resolved — no error, no toast, nothing. Every other identity-resolution bug found this session (vendorTicket's `currentVendorRecord`, the Invites-status write) traced back to the exact same shape of gap: a one-time match at boot time that can fail to resolve (a slow connection, a login-email mismatch) and then never gets retried, even though the underlying data might be perfectly fine. A silently-dead button reads exactly like "not working" from the driver's side, with zero clue why.

**Fix**: the boot-time match (previously inline in `loadAvailability()`) is now its own `resolveTechRecord(logContext)`, reused as a real retry inside `onToggleClick()` — if `currentTechRecord` is still null right when the driver taps the toggle, it retries the actual `CONFIG.techReport` query once before giving up. If the identity genuinely still can't be resolved after that, the driver now sees a clear error toast instead of a dead, unresponsive switch. Also bumped the fetch from 200 to 1000 records, same reasoning as vendorTicket's own `resolveVendorRecord()` fix (rules out `Technicians_Report` simply exceeding the old cap).

**Tested** against the real shipped code (Node-VM harness, stubbing `apiGetReport()`/`updateRecordById`) — confirmed a retry that finds the real record lets the toggle go through normally (a success toast, not an error); confirmed a genuine failure (no matching record at all) now shows a clear error toast instead of silently doing nothing. Re-ran the invite-status-retry, Accept-screen-nav-buttons, vehicle-name, and photo-persistence suites from earlier this session to confirm no regression — all still pass.

Syntax-checked and packed (`driverTicket.zip`).


## 2026-08-27 — Google Distance Matrix travelMode made explicit (mirrored from vendorTicket/technicianTicket) — no behavior change

User's exact ask (originally against `vendorTicket`, then: "fixed mirrored there" for this app and `technicianTicket`): "for all distance calculations that use google api - ensure that travel type has to be four wheeler for TOW cases and distance type should be two wheeler for REPAIR/RSR cases." This app only ever handles TOW tickets — `loadDashboard()`'s own query is hardcoded `Service_Type == "TOW"`, no RSR branch exists anywhere in the file — and Google's `"DRIVING"` mode (four-wheeler/car routing) was already the hardcoded default everywhere, which is already correct for a tow vehicle. Nothing was actually broken here.

**Change**: `googleDistanceMatrix`/`calcDistanceETA`/`calc4PointRoundtrip` all take a new, optional trailing `travelMode` param — omitted, it still defaults to `"DRIVING"` (backward compatible). Added `travelModeFor(record)`, mirroring vendorTicket's/technicianTicket's own function name/shape — since this app can never see an RSR ticket, it always resolves to `"DRIVING"`. All 8 real call sites (`acceptService`, `markReachedTow`/reach flow, `confirmCancel` ×3, `vehiclePicked`, `confirmCxRejectLoading`, `reachedDrop`) now pass it through explicitly rather than relying on the implicit module default — purely for consistency/self-documentation across all three Get-Rescue field apps, not a functional fix.

**Tested** against the real shipped code (Node-VM harness, functions extracted directly from `widget.html` via brace-matching) — confirmed `travelModeFor` always returns `"DRIVING"` regardless of input, and that `calcDistanceETA`/`calc4PointRoundtrip` both correctly resolve to `"DRIVING"` on the real Google API call shape (unchanged from before). Syntax-checked and packed (`driverTicket.zip`).


### 2026-09-01 — Real gap fixed: dashboard auto-refresh only handled brand-new tickets, not status changes/removals on already-known ones

Found while fixing the identical gap in `getRescueTicket`'s own dashboard poll (user-reported live: a new ticket wasn't appearing in a second open tab) — audited every widget's own auto-refresh for the same class of issue. `pollForNewCases()` here only ever re-rendered when a genuinely new ticket arrived; an already-known ticket's status silently changing elsewhere (another vendor took a shared invite first, the agent cancelled it) or a known ticket dropping out of "my tickets" entirely (assigned to someone else) updated `state.tickets` in the background but never refreshed what's actually on screen.

**Fix**: added `KNOWN_TICKET_STATUSES` (companion to the existing `KNOWN_TICKET_IDS`) to track each known ticket's own last-seen status. `pollForNewCases()` now also re-renders on a status change or a removal — but deliberately doesn't play the new-case sound/notification for either (that's reserved for a genuinely new arrival), so this only fixes staleness, not notification behavior.

**Tested** — new suite (`test_field_dashboard_auto_refresh.js`, 24 checks across vendorTicket/technicianTicket/driverTicket) extracting the real shipped functions: brand-new ticket still notifies+renders; a status change on a known ticket renders but doesn't notify; a known ticket disappearing renders but doesn't notify; nothing changed renders/notifies nothing; the status snapshot itself stays current. Syntax-checked and packed (`driverTicket.zip`).

### 2026-09-06 — New: WhatsApp integration added (was zero before this) — Reach message wired, more to follow

User provided three PDFs ("RESCUE - WhatsApp Message Templates," "RESCUE - Repair Service Logic Flow," "RESCUE - Towing Service Logic Flow") specifying 13 numbered WhatsApp templates and their trigger logic across the whole ticket lifecycle. A full audit found this widget (and `technicianTicket`/`vendorTicket`) had **zero** WhatsApp code at all — every send in the app lived only in `getRescueTicket`, covering just 4 of the 13 templates. Proceeding incrementally, safest piece first, per explicit user instruction to not risk breaking existing functionality.

**Built**: `formatIndianPhone()`/`sendWhatsAppTemplate()` copied verbatim from `getRescueTicket`'s own already-proven implementation (same `sendWhatsAppMessage` Custom API, same param shape) — this file had neither before. Wired into `markReached()`: after the existing Reach save/photo-upload/toast completes, fires the `reach_time` template (spec item 10 — "Mechanic/Driver app - Reach Breakdown Location page, on click REACH button, Send 10") to the customer's `Phone_Number`. Deliberately **not awaited** — a failed/slow send can never block or delay the real Reach action, matching this app's existing tolerance for best-effort GPS capture.

**Deliberately not built yet, blocked on real answers rather than guessed**:
- **Accept-time message (template 8, "vendor/mechanic name and number")** — user confirmed live that the phone number IS a real template parameter, but `Technicians_Report` has no confirmed phone field anywhere in this codebase (unlike `Vendors_Report`'s confirmed `Mobile_Number_01`) — user is checking the real field name before this gets built, to avoid sending a broken/blank parameter to real customers.
- **5-minutes-later follow-up (template 9)** and the **multi-step CTA-triggered thread (templates 1→2→3)** — both need an architecture decision (a delayed/scheduled send survives an agent closing their tab; the CTA case needs a webhook receiving the customer's own button-tap) before building either.
- **Complete-message with Remaining-Fee branching (templates 11/12)** — the field apps have no payment-link-generation capability at all today (that lives only in `getRescueTicket`); needs a design decision on whether to add it here or route through the agent side.

**Template name flagged, not confirmed**: `reach_time` is a best-effort guess (no numbered/named catalog exists anywhere, matching every other template name in this app) — the spec's own row title is "Share rescuer Reach Time." Also has no visible parameter slots in the spec's own template text, so it's sent with zero params — correct this once the real approved WhatsApp Business template name/shape is confirmed.

**Tested** — new suite (`test_field_apps_whatsapp.js`, 12 checks across technicianTicket/driverTicket/vendorTicket): `formatIndianPhone()` correctness, and that the Reach handler in each app actually calls `sendWhatsAppTemplate` with the `reach_time` template, unawaited (confirming it truly can't block navigation). Full existing suite re-run — no regressions. Syntax-checked and packed (`driverTicket.zip`).

### 2026-09-07 — Location/distance fixes from the user-provided LOCATIONS-DISTANCES-DATETIME spec

Full gap analysis + fix writeup lives in `getRescueTicket/README.md`'s own matching entry — this app's own changes:
- `OFFICE_LAT`/`OFFICE_LON` corrected to `12.967945122837245, 77.6110507612277` (previous value was off by ~13-20m).
- **Real bug fixed**: `reachedDrop()`'s round trip used static breakdown/drop coordinates; now uses the stored `Reach_Location_Lat/Lon` (from the earlier Reach click) plus this click's own live GPS fix.
- `confirmCxRejectLoading()`'s round trip now uses the stored `Reach_Location_Lat/Lon` too — its original 2026-08-04 comment already said the *intended* input was "driver location @ click of Reach breakdown location," just approximated with the static breakdown coordinate since there was no dedicated field for the real value at the time. Same intent, more precise now that the field exists.
- **Deliberately left unchanged**: `vehiclePicked()` — its own prior comment ("Mock says Drop distance/ETA are 'based on current GPS location of Driver'") conflicts with the new spec's "use the stored Reach-time position" definition for this same leg. Flagged to the user rather than silently picking a side.

**Tested** — new `test_locations_distances_2026_09_07.js` (32 checks, shared with `vendorTicket`/`technicianTicket`/`getRescueTicket`). Full existing `test_field_apps_whatsapp.js` suite re-run — no regressions from touching the same Reach/Cancel/Pickup functions. Syntax-checked (`node --check` on the extracted script block) and packed.

### 2026-09-07 (later) — New: `Remaining_Fee_Receipt_Time`/`RSP_Closure_Time` wired into `confirmPaymentReceived()`

User created all 5 new timestamp fields from the same spec (see `getRescueTicket/README.md`'s own matching entry for the full field list/rationale, including the other 3 which are `getRescueTicket`-only). This app's own change: `confirmPaymentReceived()`'s payload now also includes `Remaining_Fee_Receipt_Time` and `RSP_Closure_Time` (both stamped at this same click, kept as two separate fields per the spec's own literal numbering) — no extra guard needed, since this function only ever runs on a genuine "Payment Received" click.

**Tested** — new `test_new_timestamps_2026_09_07.js` (15 checks, shared with `vendorTicket`/`technicianTicket`/`getRescueTicket`). Full existing suites re-run — no regressions. Syntax-checked and packed.

### 2026-09-07 (later still) — Real bug fixed: photo capture appeared frozen during the GPS wait, inviting repeated taps

Found during a full 24-point feature-coverage audit. `capturePhoto()`'s `getPositionSafe()` call can take up to 8 seconds for a real GPS fix, and nothing disabled the shutter or showed progress during that wait — it just looked frozen, and each repeated tap started a fully separate concurrent capture (duplicate photo + duplicate upload). Fixed with a simple `capturingPhoto` in-flight flag (extra taps become a no-op) plus a visibly disabled/dimmed shutter button for the duration — the actual capture/watermark/upload logic is completely unchanged, only wrapped. The two camera/gallery buttons themselves were confirmed intentional (2026-08-03 user request), not the cause.

**Tested** — new `test_photo_capture_debounce_2026_09_07.js` (36 checks, shared with `vendorTicket`/`technicianTicket`), a real execution test with a controllable-delay GPS mock proving a double-tap produces exactly one photo, not two. Full existing suite re-run — no regressions. Syntax-checked and packed.

### 2026-09-08 — Google Distance Matrix API scope-restricted to only getRescueTicket's Final Closure calculation

Explicit user decision: *"use Google api's only for final distance calculations.. round trip, vendor travel distances and vendor round trip"* — confirmed via AskUserQuestion to mean Google's billed Distance Matrix API should be used ONLY inside `getRescueTicket`'s Final Closure calculation (`Roundtrip_Distance`/`Vendor_Distance`/`Vendor_Round_Trip_Distance`), and reverted everywhere else. See `vendorTicket/README.md`'s matching entry for the full rationale — this app's own 8 call sites (Accept, Reach, Cancel, Vehicle Picked, Reached Drop, Cx Reject-during-Loading) are fixed identically: `loadGoogleMaps()` short-circuits to `return Promise.resolve(false);` at the top, so the Google Maps script is never requested (zero network calls, zero billing risk) and every call site falls straight into its existing Haversine fallback. `travelModeFor()` (always `DRIVING` in this TOW-only app) is unchanged, just inert now.

**Needs redeploying**: none — client-side JS only.

**Tested** — `test_technician_driver_travelmode.js` rewritten (same "Google never reached, Haversine still works" pattern). New `test_google_api_scope_2026_09_08.js` (29 checks, shared across all 4 widgets). Full existing suite re-run — no new regressions (pre-existing, unrelated failures only, none touching distance/API code). Syntax-checked and packed.

## 2026-09-10 — quick-win fixes from the client's soft-test audit

Two fixes from the full soft-test gap audit (see the published findings), picked as the lowest-effort items involving this widget:

**Notification sound made distinct.** vendorTicket/technicianTicket/driverTicket previously played the byte-identical 2-beep 880Hz sine tone. This app now plays a rising two-note "whoop" sweep (523Hz → 784Hz, triangle wave) instead of a flat beep — a sweep reads very differently even half-heard, distinct from vendorTicket's 2 sine beeps and technicianTicket's 3 square beeps — and louder (`gain` 0.28 → 0.5).

**Customer phone number in the ticket-info summary is now click-to-dial.** Same fix as vendorTicket's own matching entry — `renderTicketInfo()`'s "Phone" row is now a `tel:` link via `formatIndianPhone()` (already proven elsewhere in this file for WhatsApp sending), visible text unchanged.

**Needs redeploying**: none — client-side JS only.

**Tested**: syntax-checked and packed — not yet independently live-tested.

## Running locally

Same as every other project in this repo: `npm install && npm start` inside this folder serves `app/widget.html` over HTTPS for Zoho widget preview/development.
