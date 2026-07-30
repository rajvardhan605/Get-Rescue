# Get-Rescue

Zoho Creator custom widgets for the Get-Rescue (RSR/TOW dispatch) app.

## Projects

- **getRescueTicket** — agent-facing ticket intake: a step wizard that creates and drives a single `Create_Case` record through its full lifecycle. See [getRescueTicket/README.md](getRescueTicket/README.md) for the full schema/status/field documentation.
- **ticketKanbanGetRescue** — operations board: a Kanban-style view over existing `Create_Case` records (one column per Status value), reusing the same edit wizard to resume/update a ticket.
- **toggleGetRescue** — a small "Online/Offline" availability switch for the logged-in technician/vendor.

Each project is a standalone Zoho Creator widget (Express dev server + `app/widget.html` + `plugin-manifest.json`). Run `npm install && npm start` inside a project folder to serve it locally over HTTPS for widget preview/development.
