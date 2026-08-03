# Get-Rescue — Zoho Creator Access Requirements (single source of truth)

This file lists, **per profile/widget**, exactly which Zoho Creator **Forms** and **Reports** that user needs permission to — so whoever sets up Zoho user roles/permissions can grant the right access without guessing from the code.

- **Whenever a widget starts reading or writing a new report/form, add it here in the same change** — this file should never drift behind what the code actually calls.
- **Add** = needs to create new records in that Form (Zoho Creator's `addRecords`/`ZOHO.CREATOR.API` add call).
- **View** = needs to read records from that Report (`getRecords`).
- **Edit** = needs to update existing records in that Report, and/or upload files to it (`updateRecordById`, `uploadFile`) — Zoho Creator ties file uploads to the same report used for the update.
- See **`FIELDS.md`** (same folder) for what each report/form's individual fields are used for.

---

## Quick summary

| Profile / Widget | Forms needing **Add** | Reports needing **View** | Reports needing **View + Edit** |
|---|---|---|---|
| **Agent** (`getRescueTicket`) | `Create_Case` | `vehicle_master_Report`, `Vehicle_Issue_Report`, `vendors_Report`, `technicians_Report`, `Client_Report`, `Rate_Master_Report` | `Agent_Ticket_Report` |
| **Kanban / Ops board** (`ticketKanbanGetRescue`) | `Create_Case` (only if a ticket is somehow opened before it has an ID — see its own code comment) | same option reports as Agent | `Agent_Ticket_Report` |
| **Vendor** (`vendorTicket`) | `Task_Rejections` | — | `Agent_Ticket_Report`, `My_Availability_Vendor`, `technicians_Report` (read for fleet hand-off list, but see note below) |
| **Technician** (`technicianTicket`) | `Task_Rejections` | — | `Agent_Ticket_Report`, `technicians_Report` |
| **Driver** (`driverTicket`) | `Task_Rejections` | — | `Agent_Ticket_Report`, `technicians_Report` |
| **Toggle** (`toggleGetRescue` — superseded, see root `README.md`) | — | — | `technicians_Report` **or** `My_Availability_Vendor` (whichever matches the logged-in user) |

---

## Agent (`getRescueTicket`)

The back-office agent creates and drives a ticket through its entire lifecycle, and assigns vendors/technicians.

- **`Create_Case` — Add.** Required to create a brand-new ticket (`addTicket()` on the very first "Save & Continue"). Every subsequent save on the same ticket goes through `Agent_Ticket_Report` instead (see below), never `Create_Case` directly again.
- **`Agent_Ticket_Report` — View + Edit.** The single report this widget reads from and writes to for literally everything else: the dashboard's 4 status-bucket lists, phone/Case-ID/reg-no. search, opening/resuming a ticket at any step, every stage's save, the Final Closure screen's summary + click-to-edit popups, and file uploads (receipt photos, arrival/pre/post-service photos, etc. — all go through this same report).
- **`vehicle_master_Report` — View.** Vehicle picker option list (Create step).
- **`Vehicle_Issue_Report` — View.** Vehicle Issue picker, filtered by the selected vehicle's category (Create step).
- **`vendors_Report` — View.** Vendor picker option list + phone/priority/online-status/location lookups (Assignment step).
- **`technicians_Report` — View.** Assigned Technician picker option list (Assignment step).
- **`Client_Report` — View.** Client picker option list (Create step).
- **`Rate_Master_Report` — View.** Fee calculation rules (Quote step).

Not needed: `My_Availability_Vendor` (agent never reads/writes vendor availability directly), `Task_Rejections` (agent never writes a rejection log — only the field-facing widgets do, since only they ever reject a job).

## Kanban / Ops board (`ticketKanbanGetRescue`)

Same underlying access as Agent — it's a different view (Kanban columns) over the same `Create_Case` data, reusing the same edit wizard to resume a ticket. Grant it identically to Agent's own list above.

## Vendor (`vendorTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (filtered client-side by the logged-in email matching one of the Vendor-Email candidate fields — see `FIELDS.md`), and writes every field across Accept/Reject, Reach, WIP/Loading/Reached-Drop/Unloading (if Individual), and Final Payment.
- **`My_Availability_Vendor` — View + Edit.** The online/offline toggle folded into this widget's topbar (same report `toggleGetRescue` used).
- **`technicians_Report` — View** (at minimum). Used in two ways: (1) fleet hand-off — listing the fleet's own technicians to assign a job to; (2) **not currently used**, but if `Vendor_Type`/`Fleet_Vendor` end up living on this report instead of `vendors_Report`/`My_Availability_Vendor` (see `getRescueTicket/QUESTIONS_TO_ASK.md`), this may need to become View + Edit too. Currently only ever read, never written, by this widget.
- **`Task_Rejections` — Add.** Best-effort rejection log (non-blocking) on Reject.

Not needed: `vehicle_master_Report`, `Vehicle_Issue_Report`, `Client_Report`, `Rate_Master_Report` — this widget never touches vehicle/issue/client/rate master data, only the ticket record itself.

## Technician (`technicianTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (RSR only), writes every field across Accept/Reject, Reach, WIP, and Final Payment.
- **`technicians_Report` — View + Edit.** Both the "assigned to me" dashboard match (View) and this technician's own online/offline toggle (Edit).
- **`Task_Rejections` — Add.** Best-effort rejection log.

## Driver (`driverTicket`)

- **`Agent_Ticket_Report` — View + Edit.** Reads "my tickets" (TOW only), writes every field across Accept/Reject, Reach, Loading, Reached Drop Location, Unloading & Handover, and Final Payment.
- **`technicians_Report` — View + Edit.** Same dual purpose as Technician's own entry above.
- **`Task_Rejections` — Add.** Best-effort rejection log.

## Toggle (`toggleGetRescue`) — superseded 2026-07-31

Kept for reference/rollback only; the toggle now lives inside Vendor/Technician/Driver directly (see root `README.md`). If it's ever run standalone again: **`technicians_Report`** or **`My_Availability_Vendor` — View + Edit**, whichever report actually has a record matching the logged-in user's email (it checks `technicians_Report` first, falls back to `My_Availability_Vendor`).

---

## Open questions this file doesn't resolve on its own

- **`Task_Rejections`** — form name itself is unconfirmed (see `getRescueTicket/QUESTIONS_TO_ASK.md`); grant Add access once the real form name is confirmed, not necessarily this exact one.
- **`vendors_Report` vs. `My_Availability_Vendor` vs. `technicians_Report`** for the vendor Individual/Fleet type and the fleet→technician link — depending on where these guessed fields actually live (see `QUESTIONS_TO_ASK.md`), Vendor's own access list above may need `vendors_Report` added, or `technicians_Report` upgraded from View to View + Edit.
