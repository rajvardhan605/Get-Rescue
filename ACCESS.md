# Get-Rescue — Zoho Creator Access Requirements (single source of truth)

This file lists, **per profile/widget**, exactly which Zoho Creator **Forms** and **Reports** that user needs permission to — so whoever sets up Zoho user roles/permissions can grant the right access without guessing from the code.

- **Whenever a widget starts reading or writing a new report/form, add it here in the same change** — this file should never drift behind what the code actually calls.
- **Add** = needs to create new records in that Form (Zoho Creator's `addRecords`/`ZOHO.CREATOR.API` add call).
- **View** = needs to read records from that Report (`getRecords`).
- **Edit** = needs to update existing records in that Report, and/or upload files to it (`updateRecordById`, `uploadFile`) — Zoho Creator ties file uploads to the same report used for the update.
- See **`FIELDS.md`** (same folder) for what each report/form's individual fields are used for, and the full build spec for every form.

**2026-08-03 — all Zoho forms/reports were deleted and rebuilt from scratch.** Report names below reflect the rebuilt schema: `vehicle_master_Report` → `Vehicle_Master_Report`, `vendors_Report` → `Vendors_Report`, `technicians_Report` → `Technicians_Report`. The old separate `My_Availability_Vendor` form was folded into `Vendors_Report` at the time.

**2026-08-05 — `My_Availability_Vendor` brought back, toggle only (user request).** The 2026-08-03 unification above didn't hold up for the vendor portal specifically — see `vendorTicket`'s own entry below. The online/offline toggle now targets `My_Availability_Vendor` again, independent of `Vendors_Report`; everything else about the 2026-08-03 rebuild (report/field names elsewhere) is unaffected.

---

## Quick summary

| Profile / Widget | Forms needing **Add** | Reports needing **View** | Reports needing **View + Edit** |
|---|---|---|---|
| **Agent** (`getRescueTicket`) | `Create_Case`, `Invites` | `Vehicle_Master_Report`, `Vehicle_Issue_Report`, `Vendors_Report`, `Technicians_Report`, `Client_Report`, `Rate_Master_Report`, `Invites_Report` (duplicate check before creating) | `Agent_Ticket_Report` |
| **Kanban / Ops board** (`ticketKanbanGetRescue`) | `Create_Case` (only if a ticket is somehow opened before it has an ID — see its own code comment) | same option reports as Agent | `Agent_Ticket_Report` |
| **Vendor** (`vendorTicket`) | `Task_Rejections`, `Invites` (fleet hand-off only) | `Vehicle_Master_Report`, `Vehicle_Issue_Report` (corrected 2026-09-13 — see own section) | `Agent_Ticket_Report`, `Vendors_Report`, `Technicians_Report` (read for fleet hand-off list, but see note below), `Invites_Report` |
| **Technician** (`technicianTicket`) | `Task_Rejections` | `Vehicle_Master_Report`, `Vehicle_Issue_Report` (added 2026-09-13 — see own section) | `Agent_Ticket_Report`, `Technicians_Report`, `Invites_Report` |
| **Driver** (`driverTicket`) | `Task_Rejections` | `Vehicle_Master_Report`, `Vehicle_Issue_Report` (added 2026-09-13 — see own section) | `Agent_Ticket_Report`, `Technicians_Report`, `Invites_Report` |
| **Customer** (`customerPage`) | — | — | — (no direct report/form access at all — see its own section below) |
| **Vendor Performance** (`vendorPerformance`) | — | `Agent_Ticket_Report`, `Task_Rejections`, `Vendors_Report` | — (read-only, aggregate stats only) |
| **Agent Performance** (`agentPerformance`) | — | `Agent_Ticket_Report` | — (read-only, aggregate stats only) |
| **Toggle** (`toggleGetRescue` — superseded, see root `README.md`) | — | — | `Technicians_Report` (this widget was never updated for the `Vendors_Report` unification — see its own section below) |
| **Operations Manager — Fleet Map** (`opsMap`) | — | `Agent_Ticket_Report`, `Vendors_Report`, `Technicians_Report`, `Agent_Report` | — |
| **Call Center** (`callCenter`) | `Call_Log` (⚠ not yet created — see its own section below) | — | `Call_Log_Report` (⚠ not yet created) |

---

## Agent (`getRescueTicket`)

The back-office agent creates and drives a ticket through its entire lifecycle, and assigns vendors/technicians.

- **`Create_Case` — Add.** Required to create a brand-new ticket (`addTicket()` on the very first "Save & Continue"). Every subsequent save on the same ticket goes through `Agent_Ticket_Report` instead (see below), never `Create_Case` directly again.
- **`Agent_Ticket_Report` — View + Edit.** The single report this widget reads from and writes to for literally everything else: the dashboard's 4 status-bucket lists, phone/Case-ID/reg-no. search, opening/resuming a ticket at any step, every stage's save, the Final Closure screen's summary + click-to-edit popups, and file uploads (receipt photos, arrival/pre/post-service photos, etc. — all go through this same report).
- **`Vehicle_Master_Report` — View.** Vehicle picker option list (Create step).
- **`Vehicle_Issue_Report` — View.** Vehicle Issue picker, filtered by the selected vehicle's category (Create step).
- **`Vendors_Report` — View.** Vendor picker option list + phone/priority/online-status/location lookups (Assignment step).
- **`Technicians_Report` — View.** Assigned Technician picker option list (Assignment step).
- **`Client_Report` — View.** Client picker option list (Create step).
- **`Rate_Master_Report` — View.** Fee calculation rules (Quote step).
- **`Invites` (form) — Add.** `syncInvites()` creates one `Invites_Report` record per invited vendor when the Assignment step is saved.
- **`Invites_Report` — View.** Read before adding, so `syncInvites()` never creates a duplicate record for the same ticket/vendor pair on a re-save.
- **Custom APIs `getVoiceCallbackDetails`/`getCallbackStatus`** — added 2026-09-14. Same two Ozonetel-backed Custom APIs `callCenter` already uses (see that widget's own section below) — this widget now also calls them directly, for the Create step's "📲 Call via System" button. Not a Form/Report permission (Custom APIs aren't governed by this file's usual View/Add/Edit model) — noted here only for completeness/consistency, since this is a new dependency this widget didn't have before.

Not needed: `Task_Rejections` (agent never writes a rejection log — only the field-facing widgets do, since only they ever reject a job).

## Kanban / Ops board (`ticketKanbanGetRescue`)

Same underlying access as Agent — it's a different view (Kanban columns) over the same `Create_Case` data, reusing the same edit wizard to resume a ticket. Grant it identically to Agent's own list above. Note: this widget still lags behind `getRescueTicket` on some features (no Invites integration yet, no `Assigned_Vendor` field) — it doesn't need `Invites`/`Invites_Report` access today, but will once that gap is closed.

## Vendor (`vendorTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (filtered client-side by the logged-in email matching one of the Vendor-Email candidate fields — see `FIELDS.md`), and writes every field across Accept/Reject, Reach, WIP/Loading/Reached-Drop/Unloading (if Individual), and Final Payment.
- **`Vendors_Report` — View + Edit.** This vendor's own identity for ticket matching (Assigned_Vendor/Vendors1/Invites, real Lookups into this report) and the agent's own Assignment-step picker. **Un-unified 2026-08-05 (user request)**: the 2026-08-03 merge that folded the online/offline toggle into this same report didn't work for the vendor portal — reverted the toggle specifically back to its own separate report below; `Vendors_Report` itself is unchanged otherwise.
- **`My_Availability_Vendor` — View + Edit.** Re-added 2026-08-05 — the vendor's own online/offline toggle (`Availability_Status`, matched by `Email`) lives here again, independent of `Vendors_Report`. Deliberately never used for ID matching (Assigned_Vendor/Vendors1/Invites) — its own record id is not a `Vendors_Report` id.
- **`Technicians_Report` — View** (at minimum). Used for the fleet hand-off — listing the fleet's own technicians to assign a job to. Currently only ever read, never written, by this widget.
- **`Task_Rejections` — Add.** Best-effort rejection log (non-blocking) on Reject.
- **`Invites` (form) — Add.** `assignTechnician()` creates an `Invites_Report` record (`RSID`/`Technician`) when a fleet vendor hands a job off to one of their own technicians. (Fixed 2026-08-03 — this used to incorrectly write into the `Vendor` field; see `Invites`' own field list in `FIELDS.md`.)
- **`Invites_Report` — View + Edit.** `loadMyInvites()` reads this report to match "my tickets" (by the vendor's own `Vendors_Report` record ID — fixed 2026-08-05, was comparing against a name and never matched anything — falling back to the older `Vendor_Emails` email match). `updateMyInvite()` writes this vendor's own invite record on Accept/Reject (`Service_Acceptance_Next`, `Status`, `Reject_Reason`).
- **`Vehicle_Master_Report` — View.** **Corrected 2026-09-13 — this row previously said "Not needed" below, which was already stale the day it was written.** `loadVehicleNames()`/`vehicleDisplayFor()` (added 2026-08-21) reads this report to resolve the ticket's `Vehicle` id to a real name for the dashboard cards and the Vehicle row on every action screen; without View access here, `loadVehicleNames()` fails closed (empty map) and every ticket falls back to showing the raw `Vehicle_Master_Report` record id instead of a name — **user-reported live 2026-09-13** (dashboard cards showing e.g. `4488810000000057692` where the vehicle name belongs). Grant this in Zoho, then re-check: if the raw id still shows after the grant, the remaining possibility is that specific vehicle's own `Vehicle_Master` record has a blank `Name` field — a data-entry gap, not a permissions one — check the console's `loadVehicleNames` warning (says "0 records" for a permission/report-name problem, vs. "N of M records had no name" for specific blank records) to tell the two apart.
- **`Vehicle_Issue_Report` — View.** Also previously listed as "not needed" below, also already stale — `loadIssueNames()` (2026-08-05, i.e. *older* than that stale note) reads this report to resolve issue ids to names for the same ticket cards/screens. Apparently already working live today (issue names do render correctly), so this grant is likely already in place in Zoho even though it was never reflected here — noted for consistency with the `Vehicle_Master_Report` correction above, not because it's newly broken.

Not needed: `Client_Report`, `Rate_Master_Report` — this widget never touches client/rate master data, only the ticket record itself.

## Technician (`technicianTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (RSR only), writes every field across Accept/Reject, Reach, WIP, and Final Payment.
- **`Technicians_Report` — View + Edit.** Both the "assigned to me" dashboard match (View) and this technician's own online/offline toggle (Edit).
- **`Task_Rejections` — Add.** Best-effort rejection log.
- **`Invites_Report` — View + Edit.** `loadMyInvites()` reads this report first to match "my tickets" (by technician name against the `Technician` field, falling back to `Technician_Emails`); `updateMyInvite()` writes this technician's own invite record on Accept/Reject. (Fixed 2026-08-03 — this used to match against the wrong field, `Vendor`, which a fleet hand-off could never actually have populated correctly; see `FIELDS.md`.)
- **`Vehicle_Master_Report` — View** and **`Vehicle_Issue_Report` — View.** **Added 2026-09-13 — this section never listed either, even though this widget runs the identical `loadVehicleNames()`/`loadIssueNames()` calls as `vendorTicket` (see that widget's own matching, corrected entries above for the full rationale and the live symptom this causes when missing).**

## Driver (`driverTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (TOW only), writes every field across Accept/Reject, Reach, Loading, Reached Drop Location, Unloading & Handover, and Final Payment.
- **`Technicians_Report` — View + Edit.** Same dual purpose as Technician's own entry above.
- **`Task_Rejections` — Add.** Best-effort rejection log.
- **`Invites_Report` — View + Edit.** Same as Technician's own entry above (same fix applies).
- **`Vehicle_Master_Report` — View** and **`Vehicle_Issue_Report` — View.** Same addition and same reason as Technician's own entry just above.

## Toggle (`toggleGetRescue`) — superseded 2026-07-31

Kept for reference/rollback only; the toggle now lives inside Vendor/Technician/Driver directly (see root `README.md`). This file was **not** updated for the 2026-08-03 `Vendors_Report`/`My_Availability_Vendor` unification (it's dead code, not part of the rebuilt schema) — if it's ever run standalone again, its own `CONFIG` needs `My_Availability_Vendor` repointed to `Vendors_Report` first. As shipped, it checks `Technicians_Report` first, falls back to whatever its own `My_Availability_Vendor` config still points at — **View + Edit** on whichever actually matches the logged-in user's email.

---

## Customer (`customerPage`)

Public, no-login page, added 2026-09-02 — see its own `README.md` for the full architecture writeup. **Deliberately has zero direct report/form access** — it never calls `ZOHO.CREATOR.DATA`/`.PUBLISH` at all. A real security finding drove this: the standard public-page data-access mechanism (`ZOHO.CREATOR.PUBLISH.getRecordById`) requires publishing the whole report behind one shared token that isn't scoped per-record, which would let any customer enumerate any other customer's data on `Create_Case`. Everything instead routes through 3 narrow Custom APIs (server-side Deluge, its own service-level permissions, not tied to the anonymous visitor at all):

- `getCustomerTicketInfo` — reads a handful of fields off `Create_Case` by record ID.
- `submitCustomerLocation` — writes `Latitude`/`Longitude` or `DropLocationLat`/`DropLocationLong`.
- `submitCustomerFeedback` — writes `Cx_Feedback_Score`/`Cx_Feedback_Text`.

Whatever Zoho permission profile governs this public page needs to be able to invoke these 3 Custom APIs — confirm live whether that's automatic (Custom APIs commonly run at the app's own service level regardless of visitor auth) or needs an explicit grant.

## Vendor Performance (`vendorPerformance`)

Standalone widget, added 2026-09-03, for a logged-in vendor's own performance stats. **Read-only** and deliberately never shows individual tickets — only aggregate counts/percentages, per an explicit hand-drawn wireframe rule ("Vendor & mechanic will not be shown case HISTORY").

**Settlement statement added 2026-09-14** — the "vendor statement view in Vendor Portal" piece of the Vendor Accounting scope. Four aggregate cards (Awaiting Payment before TDS, Paid to You, TDS Deducted, Cash You Collected) read from `Settlement_Status` / `Vendor_Payment_Status` / `Vendor_Balance_Before_TDS` / `Vendor_Amount_Paid` / `TDS_Amount` / `Cash_Collected` on this vendor's own already-fetched tickets. **No new report or permission** — the existing `Agent_Ticket_Report` View grant below already covers it, since that report is fetched with `field_config:"all"`. Aggregate totals only, per the explicit 2026-09-04 decision ("aggregate totals only, **not** a full case-level statement"): no case list, no Case IDs, no per-job breakdown — do not add them without re-confirming that decision. Deliberately all-time and labelled as such on screen, ignoring the page's date filter, so money owed is never hidden by a month boundary.

- **`Vendors_Report` — View.** Resolves the logged-in vendor's own identity (matched by login email), same pattern `vendorTicket` already uses.
- **`Agent_Ticket_Report` — View.** This vendor's own tickets (`Assigned_Vendor` match) for Completed/Cancelled counts and the ETA/Issue/Feedback percentages.
- **`Task_Rejections` — View.** This vendor's own rejected/timed-out count (matched by name, same as `vendorTicket`'s own reject-log write).

## Agent Performance (`agentPerformance`)

Standalone widget, added 2026-09-03, for a logged-in agent's own performance stats. **Read-only.**

- **`Agent_Ticket_Report` — View.** Filtered by the new `Created_By_Agent_Email` field (added 2026-09-03, stamped at ticket creation in `getRescueTicket`) to scope every metric to this agent's own tickets. Tickets created before this field existed won't be attributed to any agent — a known, accepted gap for historical data.

## Operations Manager — Fleet Map (`opsMap`) — REMOVED 2026-09-04

Standalone widget, added 2026-08-31, retired 2026-09-04 in favor of `getRescueTicket`'s own in-app "Ops Map" button once both ended up gated identically, then deleted entirely the same day. The access requirements below are kept only as a record of what it needed, in case it's ever rebuilt — `getRescueTicket`'s own in-app Ops Map screen needs the same `Agent_Ticket_Report`/`Vendors_Report`/`Technicians_Report`/`Agent_Report` View access already covered by its own section above, so no separate access setup is needed today.

- `Agent_Ticket_Report` — View. Read only, to compute who's currently "on a case" (live, not a stored flag).
- `Vendors_Report` — View. Marker data — location, online/offline status, Engagement Type.
- `Technicians_Report` — View. Same, for technicians/drivers.
- `Agent_Report` — View. Resolves the logged-in user's own `User_Type`, to gate access to Operations Managers only.

## Call Center (`callCenter`)

New standalone widget, added 2026-09-10 — places/checks voice callback calls via two user-built Custom APIs (`getVoiceCallbackDetails`/`getCallbackStatus`, Ozonetel-backed), and logs a history of calls placed. Not embedded on any ticket — a free-standing page, phone number entered manually. See its own `README.md` for the full writeup and exact field list.

- **`Call_Log` (form) — Add.** ⚠ **Does not exist yet** — needs creating in Zoho Studio before call history works at all (placing/checking calls itself doesn't depend on this, only the history list does — the widget degrades honestly with an on-screen banner if this is missing). Fields needed: `Phone_Number`, `Reference_ID`, `Callback_Message`, `UCID`, `Call_Status`, `Call_Duration`, `Call_Outcome`, `Placed_By` (all Single Line Text) — see `callCenter/README.md` for the full spec.
- **`Call_Log_Report` — View + Edit.** ⚠ **Does not exist yet**, same as above — the report exposing `Call_Log`'s records, read for the history table and updated in place each time a call's status is re-checked (rather than adding a new row per check).
- Does **not** need access to `Create_Case`/`Agent_Ticket_Report` or any other existing form/report — deliberately standalone, no ticket lookups.

## Open questions this file doesn't resolve on its own

- **`vendors_Report` vs. `My_Availability_Vendor` vs. `technicians_Report`** — resolved 2026-08-03 by unifying the first two into one `Vendors_Report`. The remaining open question is whether `Fleet_Vendor` (on `Technicians_Report`, linking a technician back to their owning fleet vendor) should become a real Lookup instead of a name-string match — see `getRescueTicket/QUESTIONS_TO_ASK.md`.
