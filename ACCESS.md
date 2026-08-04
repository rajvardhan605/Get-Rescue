# Get-Rescue — Zoho Creator Access Requirements (single source of truth)

This file lists, **per profile/widget**, exactly which Zoho Creator **Forms** and **Reports** that user needs permission to — so whoever sets up Zoho user roles/permissions can grant the right access without guessing from the code.

- **Whenever a widget starts reading or writing a new report/form, add it here in the same change** — this file should never drift behind what the code actually calls.
- **Add** = needs to create new records in that Form (Zoho Creator's `addRecords`/`ZOHO.CREATOR.API` add call).
- **View** = needs to read records from that Report (`getRecords`).
- **Edit** = needs to update existing records in that Report, and/or upload files to it (`updateRecordById`, `uploadFile`) — Zoho Creator ties file uploads to the same report used for the update.
- See **`FIELDS.md`** (same folder) for what each report/form's individual fields are used for, and the full build spec for every form.

**2026-08-03 — all Zoho forms/reports were deleted and rebuilt from scratch.** Report names below reflect the rebuilt schema: `vehicle_master_Report` → `Vehicle_Master_Report`, `vendors_Report` → `Vendors_Report`, `technicians_Report` → `Technicians_Report`. The old separate `My_Availability_Vendor` form is gone entirely — it's now unified into `Vendors_Report` (one vendor master, not two).

---

## Quick summary

| Profile / Widget | Forms needing **Add** | Reports needing **View** | Reports needing **View + Edit** |
|---|---|---|---|
| **Agent** (`getRescueTicket`) | `Create_Case`, `Invites` | `Vehicle_Master_Report`, `Vehicle_Issue_Report`, `Vendors_Report`, `Technicians_Report`, `Client_Report`, `Rate_Master_Report`, `Invites_Report` (duplicate check before creating) | `Agent_Ticket_Report` |
| **Kanban / Ops board** (`ticketKanbanGetRescue`) | `Create_Case` (only if a ticket is somehow opened before it has an ID — see its own code comment) | same option reports as Agent | `Agent_Ticket_Report` |
| **Vendor** (`vendorTicket`) | `Task_Rejections`, `Invites` (fleet hand-off only) | — | `Agent_Ticket_Report`, `Vendors_Report`, `Technicians_Report` (read for fleet hand-off list, but see note below), `Invites_Report` |
| **Technician** (`technicianTicket`) | `Task_Rejections` | — | `Agent_Ticket_Report`, `Technicians_Report`, `Invites_Report` |
| **Driver** (`driverTicket`) | `Task_Rejections` | — | `Agent_Ticket_Report`, `Technicians_Report`, `Invites_Report` |
| **Toggle** (`toggleGetRescue` — superseded, see root `README.md`) | — | — | `Technicians_Report` (this widget was never updated for the `Vendors_Report` unification — see its own section below) |

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

Not needed: `Task_Rejections` (agent never writes a rejection log — only the field-facing widgets do, since only they ever reject a job).

## Kanban / Ops board (`ticketKanbanGetRescue`)

Same underlying access as Agent — it's a different view (Kanban columns) over the same `Create_Case` data, reusing the same edit wizard to resume a ticket. Grant it identically to Agent's own list above. Note: this widget still lags behind `getRescueTicket` on some features (no Invites integration yet, no `Assigned_Vendor` field) — it doesn't need `Invites`/`Invites_Report` access today, but will once that gap is closed.

## Vendor (`vendorTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (filtered client-side by the logged-in email matching one of the Vendor-Email candidate fields — see `FIELDS.md`), and writes every field across Accept/Reject, Reach, WIP/Loading/Reached-Drop/Unloading (if Individual), and Final Payment.
- **`Vendors_Report` — View + Edit.** New 2026-08-03: this single report now covers both the agent's Assignment-step picker AND this vendor's own login/toggle profile (Email match, `Availability_Status` online/offline toggle, `Vendor_Type` Individual/Fleet check) — previously two separate forms (`vendors_Report` + `My_Availability_Vendor`), now unified.
- **`Technicians_Report` — View** (at minimum). Used for the fleet hand-off — listing the fleet's own technicians to assign a job to. Currently only ever read, never written, by this widget.
- **`Task_Rejections` — Add.** Best-effort rejection log (non-blocking) on Reject.
- **`Invites` (form) — Add.** `assignTechnician()` creates an `Invites_Report` record (`RSID`/`Technician`) when a fleet vendor hands a job off to one of their own technicians. (Fixed 2026-08-03 — this used to incorrectly write into the `Vendor` field; see `Invites`' own field list in `FIELDS.md`.)
- **`Invites_Report` — View + Edit.** `loadMyInvites()` reads this report to match "my tickets" (by vendor name, falling back to the older `Vendor_Emails`/name match); `updateMyInvite()` writes this vendor's own invite record on Accept/Reject (`Service_Acceptance_Next`, `Status`, `Reject_Reason`).

Not needed: `Vehicle_Master_Report`, `Vehicle_Issue_Report`, `Client_Report`, `Rate_Master_Report` — this widget never touches vehicle/issue/client/rate master data, only the ticket record itself.

## Technician (`technicianTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (RSR only), writes every field across Accept/Reject, Reach, WIP, and Final Payment.
- **`Technicians_Report` — View + Edit.** Both the "assigned to me" dashboard match (View) and this technician's own online/offline toggle (Edit).
- **`Task_Rejections` — Add.** Best-effort rejection log.
- **`Invites_Report` — View + Edit.** `loadMyInvites()` reads this report first to match "my tickets" (by technician name against the `Technician` field, falling back to `Technician_Emails`); `updateMyInvite()` writes this technician's own invite record on Accept/Reject. (Fixed 2026-08-03 — this used to match against the wrong field, `Vendor`, which a fleet hand-off could never actually have populated correctly; see `FIELDS.md`.)

## Driver (`driverTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (TOW only), writes every field across Accept/Reject, Reach, Loading, Reached Drop Location, Unloading & Handover, and Final Payment.
- **`Technicians_Report` — View + Edit.** Same dual purpose as Technician's own entry above.
- **`Task_Rejections` — Add.** Best-effort rejection log.
- **`Invites_Report` — View + Edit.** Same as Technician's own entry above (same fix applies).

## Toggle (`toggleGetRescue`) — superseded 2026-07-31

Kept for reference/rollback only; the toggle now lives inside Vendor/Technician/Driver directly (see root `README.md`). This file was **not** updated for the 2026-08-03 `Vendors_Report`/`My_Availability_Vendor` unification (it's dead code, not part of the rebuilt schema) — if it's ever run standalone again, its own `CONFIG` needs `My_Availability_Vendor` repointed to `Vendors_Report` first. As shipped, it checks `Technicians_Report` first, falls back to whatever its own `My_Availability_Vendor` config still points at — **View + Edit** on whichever actually matches the logged-in user's email.

---

## Open questions this file doesn't resolve on its own

- **`vendors_Report` vs. `My_Availability_Vendor` vs. `technicians_Report`** — resolved 2026-08-03 by unifying the first two into one `Vendors_Report`. The remaining open question is whether `Fleet_Vendor` (on `Technicians_Report`, linking a technician back to their owning fleet vendor) should become a real Lookup instead of a name-string match — see `getRescueTicket/QUESTIONS_TO_ASK.md`.
