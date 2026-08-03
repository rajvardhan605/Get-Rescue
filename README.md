# Get-Rescue

Zoho Creator custom widgets for the Get-Rescue (RSR/TOW dispatch) app.

## Field names & access

- **[FIELDS.md](FIELDS.md) is the single source of truth for every Zoho field API/link name used by any widget in this repo**, organized step-by-step through the ticket's real lifecycle (Create → Quote → Locations → Assignment → Acceptance → Reach → WIP → Payment → Final Closure). Check there before using a field name in code; if a name is ever wrong, it gets corrected there first, not scattered across individual widgets' comments.
- **[ACCESS.md](ACCESS.md) is the single source of truth for which Zoho Creator Forms/Reports each profile (Agent, Vendor, Technician, Driver, Kanban) needs permission to** — check there when setting up Zoho user/role access. Whenever a widget starts reading/writing a new report or form, that gets added here in the same change.

## Projects

- **getRescueTicket** — agent-facing ticket intake: a step wizard that creates and drives a single `Create_Case` record through its full lifecycle. See [getRescueTicket/README.md](getRescueTicket/README.md) for the full schema/status/field documentation, and [getRescueTicket/QUESTIONS_TO_ASK.md](getRescueTicket/QUESTIONS_TO_ASK.md) for the current "needs a human answer" list spanning all of these projects.
- **ticketKanbanGetRescue** — operations board: a Kanban-style view over existing `Create_Case` records (one column per Status value), reusing the same edit wizard to resume/update a ticket.
- **toggleGetRescue** — a small standalone "Online/Offline" availability switch (superseded 2026-07-31 by the toggle now built directly into each of the three field-facing widgets below — kept here for reference/rollback, no longer the intended integration point for new work).
- **technicianTicket** — field-facing widget for the RSR (repair) on-road flow: dashboard of assigned tickets, online/offline toggle, Accept/Reject, Reach, Work In Progress, Final Payment. See [technicianTicket/README.md](technicianTicket/README.md).
- **driverTicket** — field-facing widget for the TOW on-road flow: Accept/Reject, Reach, Loading, Reached Drop Location, Unloading & Handover, Final Payment. See [driverTicket/README.md](driverTicket/README.md).
- **vendorTicket** — field-facing widget covering a vendor's full lifecycle across both RSR and TOW tickets, with an Individual-vendor path (same on-road flow as Technician/Driver) and a Fleet-vendor path (accept, then hand off to one of their own technicians). See [vendorTicket/README.md](vendorTicket/README.md).

Each project is a standalone Zoho Creator widget (Express dev server + `app/widget.html` + `plugin-manifest.json`). Run `npm install && npm start` inside a project folder to serve it locally over HTTPS for widget preview/development. All six share the same visual theme/design tokens by design — not factored into a shared file, since each widget is deployed to Zoho as one self-contained HTML file.
