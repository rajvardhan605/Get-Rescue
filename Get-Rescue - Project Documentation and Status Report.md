# Get-Rescue Project Status Report

What was asked for, what has been built, and what is outstanding. Roadside-assistance and towing dispatch CRM on Zoho Creator.

Oct 12, 2026 · @Subash

## Executive summary

Get-Rescue is a complete dispatch CRM for roadside assistance, covering both RSR (on-site repair) and TOW service types. It is built as twelve Zoho Creator widgets supported by twenty server-side Custom APIs.

**The originally scoped application is built and in client testing.** Agent intake wizard, vendor/technician/driver field apps, operations board, quoting, payments and closure are all present in deployed code.

Since the original scope was agreed the client has introduced several substantial additions, and **all of them have been built**:

- a per-vendor invite architecture
- an agent-on-behalf-of-vendor data entry mode
- a detailed locations, distances and timestamps specification
- Ozonetel telephony integration
- a public customer page
- two performance dashboards
- a vendor accounting and operations-approval layer
- inbound WhatsApp, including customer location pins

### What changed since the 8 October edition of this report

This section is the substance of the update. Nine further items were requested and built in the four days to 12 October:

| Item | Source | State |
| --- | --- | --- |
| Vendor accept/reject visible on the Assign page | Client defect report | Fixed — an undefined function was blanking the panel |
| Call log must not take an old case's RSID | Client defect report | Fixed — four phone-guess writes removed |
| Closure page shows linked calls only | Client decision | Built |
| Old Case search in the call pop | Wireframe p1 | Built |
| Closed tickets open view-only in Kanban | Client request | Built |
| Inbound WhatsApp location pin | ETA spec #34 | Built, blocked on WhatsPortal |
| SEND 9 five minutes after SEND 8 | LOGIC-REPAIR / LOGIC-TOW | Built, needs one field created |
| Duplicate call-log records | Client mail, 10 Oct | Fixed 12 Oct, **not yet verified** |
| Random call linked to a ticket | Client mail, 10 Oct | Fixed 12 Oct, **not yet verified** |

**One item was built and then withdrawn.** Territory and pincode zoning was built on 10 October and reverted the same day at the client's instruction. The geo-fence half of that requirement — the 8 km and 20 km radius — has been in force since July and is unaffected.

**Five items are explicitly out of scope** at the client's direction, stated 11 October: the Zoho Books finance dashboard, Airtable sync, the REASSIGN button, structured QA and UAT, and territory/pincode.

### Two corrections to the previous edition

Both were counted from the deployed code for this update.

| Previously stated | Actually |
| --- | --- |
| A fixed **sixteen**-value Status picklist | **Eighteen** values. The earlier list omitted `SCHEDULED` and `CLOSED`, both of which the code writes |
| **Twelve** wizard stages | **Fourteen** are defined. A single ticket traverses nine (RSR) or eleven (TOW) — the branch stages are not shared |

### The honest position on testing

The original scope and every client addition are **built**. Structured QA and UAT were excluded by the client, so most of the application has been proven by live use and client defect reports rather than by a test pass. Section 14 states which areas have been verified and which have not, and does not blur the two.

## Evidence base and its limits

**Stated plainly:** several client documents referenced in the project history were supplied at various points and are not all available for direct re-reading. These include the original Screen-by-Screen Flow Atlas, the Action 1–7 flow specifications, the LOCATIONS-DISTANCES-DATETIME specification of 7 September, the Ozonetel integration document, the vendor-accounting specification and the soft-test feedback lists.

Where this report describes those requirements it does so from the implementation record written at the time each was acted on — contemporaneous and detailed, but a **secondary source**.

### Read directly and reported first-hand

| Source | Date |
| --- | --- |
| The client's edited “Google Distances and Times” specification and covering email | 8 Oct |
| The five-point multiple-vendor requirement list | — |
| The client's mail on incoming-call behaviour and ticket linking | 10 Oct |
| The three-step Ozonetel specification | Oct |
| The deployed source of all twelve widgets and twenty Custom APIs | 12 Oct |

### What changed in this edition's method

The previous edition described the implementation largely from the project record. **For this update the deployed code was read directly**, module by module, which is what produced the two corrections in section 1 and the open-items list in section 12.

Where this report says a field name, a function name, a constant or a formula, that name was read out of the source rather than recalled. Where a name could not be confirmed against live Zoho data it is marked **Needs verification** — a distinction that matters more on this platform than most, because Creator fails silently when a field is missing from a report layout.

## How the build approach changed

The project started on standard Zoho forms and moved to custom widgets after the client's visit to our office.

| Phase | Approach | What it meant |
| --- | --- | --- |
| Phase 1 | Form-based | Built with Zoho Creator's built-in forms and functionality |
| Phase 2 (current) | Widget-based | All screens and functionality rebuilt as custom HTML widgets, with the design updated to the client's requirements |

### What that choice bought, and what it costs

Worth stating because it explains a recurring class of defect in section 12.

**Bought:** the staged wizard, conditional field logic, live distance calculations, the vendor picker with its radius filter, and the field apps' offline behaviour. None of these are achievable with Creator's native forms.

**Costs:** every field name is a string in JavaScript, resolved at runtime against a report. Nothing is type-checked. A field renamed in Zoho, or present on a form but missing from the report layout, **reads back as undefined with no error at all**. That single characteristic is behind a majority of the defects this project has found and fixed.

## Project overview and modules

**Scope.** A dispatch and case-management system for a roadside-assistance operator, covering the full life of a case from the customer's first call to vendor payment and closure, for two service types: **RSR** (a mechanic repairs the vehicle on the spot) and **TOW** (a vehicle is recovered and delivered to a drop location).

**Architecture.** Zoho Creator is the backend. Each ticket is one `Create_Case` record, created on first save and updated at every subsequent step. `Create_Case.Status` carries one of eighteen values; every stage writes one of those exact values. The user interface is a set of self-contained HTML widgets embedded in Creator, talking to it through the widget SDK Data API, plus twenty Deluge Custom APIs for work that must run server-side — payments, invoicing, webhooks, telephony and cross-report reads.

### The twelve modules

| Module | Audience | Purpose |
| --- | --- | --- |
| `getRescueTicket` | Agent / back office | The core step wizard. Creates and drives a ticket through its full lifecycle: Create → Quote & Booking Fee → Capture Locations → Vendor Assignment → RSR or TOW path → Final Payment → Final Closure. Also hosts the dashboard, search, Ops Map, Accept/Reject audit, vendor accounting and operations approval |
| `vendorTicket` | Vendor | A vendor's full lifecycle across both service types, with an Individual path (the vendor does the job) and a Fleet path (accept, then hand off to one of their own technicians) |
| `technicianTicket` | Technician | The RSR on-road flow: Accept/Reject, Reach, Work In Progress, Final Payment |
| `driverTicket` | Driver | The TOW on-road flow: Accept/Reject, Reach, Loading, Reached Drop, Unloading & Handover, Final Payment |
| `Ticket Kanban` | Operations | A board view over existing cases, one column per Status, reusing the same edit wizard. A closed ticket now opens view-only |
| `customerPage` | Customer (public, no login) | Booking-fee payment, remaining payment, location sharing and feedback. Routed through narrow Custom APIs rather than direct report access, to avoid a per-record data-exposure risk in Creator's public-page mechanism |
| `vendorPerformance` | Vendor | Read-only aggregate statistics for the logged-in vendor. Never lists individual tickets |
| `agentPerformance` | Agent | Read-only aggregate statistics for the logged-in agent, filterable by date range |
| `callCenter` / `inboundCallPop` | Agent | Voice callbacks and inbound-call handling through Ozonetel, with a call log and the Old Case search |
| `serviceRequest` | Agent | Service-request intake. This page was created for a showcase to the client; they later said they are simply using the Zoho form, and we make changes to that form such as hiding fields and styling |
| `toggleGetRescue` | — | The original standalone availability switch. Superseded 31 July by a toggle built into each field app; retained for reference only |

The two performance pages were designed to show agent and vendor performance.

### The fourteen wizard stages

Fourteen stages are defined. A single ticket traverses **nine** on the RSR path or **eleven** on the TOW path — the branch stages are separate stages writing separate fields, not one stage reused.

| Shared | RSR branch | TOW branch | Shared tail |
| --- | --- | --- | --- |
| Create | Service Acceptance | Service Acceptance (Tow) | Final Payment |
| Quote & Booking Fee | Reach Breakdown | Reach Breakdown (Tow) | Agent Closure |
| Capture Locations | Work In Progress | Loading |  |
| Vendor Assignment |  | Reached Drop Location |  |
|  |  | Unloading & Handover |  |

## Original requirements

The original scope, as recorded at the time it was built.

| Area | What was required |
| --- | --- |
| **Agent ticket wizard** | A guided, step-by-step screen set that creates one case and carries it through its life. Create Service Ticket; Service Fee Quote & Booking Fee; Capture Locations; Vendor Assignment; then a service-type branch into the RSR path (Service Acceptance, Reach Breakdown Location, Work In Progress) or the TOW path (Service Acceptance, Reach Breakdown, Loading, Reached Drop Location, Unloading & Handover); then Final Payment Collection; then Final Closure |
| **Status model** | A fixed Status picklist driving both the case's own state and where the wizard resumes when a case is reopened |
| **Quoting** | A fee quote driven by a Rate Master, keyed on vehicle type, vehicle segment and the selected vehicle issue, producing a suggested rate and a rate breakup. Flat-rate and distance-formula rate rules. An agent-entered total service fee, a booking fee collected up front, and a remaining fee due at completion |
| **Locations** | Capture of the breakdown location, and for TOW also a drop location, as coordinates on the ticket. A link sent to the customer to share their location. A round-trip distance calculated for quoting |
| **Vendor assignment** | A vendor picker filtered by distance from the breakdown location — 8 km for RSR, 20 km for TOW — sorted by priority then distance, showing each vendor's priority, distance, phone and online/offline state. A preferred-vendor override that shows every vendor |
| **Field applications** | Separate applications for the vendor, technician and driver, each showing only their own assigned cases, with an online/offline availability toggle and the on-road actions for their service type, including photo capture at defined points |
| **Final Closure** | A dedicated closure screen, separate from the wizard and reachable at any point, recording closure status and reason, refund details, customer feedback score and text, B2B GST invoice requirement and GST number, and a closure photo. It also surfaces every field from every stage as an individually editable row |
| **Operations board** | A Kanban view over existing cases, one column per Status, reusing the same edit wizard |
| **Distance calculation** | Road distance and travel time between defined points, using the Google Maps Distance Matrix API |

### The status model, corrected

The previous edition described this as sixteen values. **Eighteen** are written by the deployed code; the earlier list omitted `SCHEDULED` and `CLOSED`.

`SERVICE FEE QUOTE` · `BOOKING FEE PENDING` · `LOCATION REQUEST` · `LOCATION PENDING` · `SCHEDULED` · `READY FOR ASSIGNMENT` · `RSP IRA (INVITE RESPONSE AWAITED)` · `RSP ON THE WAY` · `WORK IN PROGRESS` · `LOADING` · `TO DROP LOCATION` · `UNLOADING` · `REMAINING FEE DUE` · `RSP CLOSED` · `CLOSED` · `RSP CANCELLED` · `RSP REJECT` · `CX REJECT`

`RSP CLOSED` and `CLOSED` both mean finished. Which one a ticket ends on depends on whether the vendor closed it from the field or the agent closed it from the closure page.

## Completed requirements

Grouped by module. **Completed** here means built and present in the deployed code; section 14 separates that from independently tested and confirmed.

### Agent ticket wizard

| Requirement | What was built |
| --- | --- |
| The step wizard | All fourteen stages implemented as a single `STAGES` table driving one shared renderer. The path branches on `Service_Type` after Vendor Assignment. A stepper shows progress and locks steps ahead of unsaved work |
| Status writing and resume routing | Every stage computes and writes one of the eighteen values. Each stage declares the statuses it resumes from, so reopening a case lands on the right step. A guard, `statusStillEarly`, prevents any stage recomputing a Status backwards once the case has genuinely moved past it |
| Quoting | Rate Master lookup by vehicle type, segment and issue; flat and distance-formula rules; suggested rate; auto-generated rate breakup that re-seeds when the issue or quote distance changes but preserves a manual edit otherwise; remaining fee computed as total minus booking fee |
| Locations | Coordinate capture by paste-and-split into separate latitude/longitude fields, for both breakdown and drop. A customer location-request link sent by WhatsApp. Round-trip distance calculated for quoting. `Location_Received_Time` stamped once, on first capture |
| Vendor assignment | Distance- and priority-sorted picker with the RSR/TOW radius split and the preferred-vendor override; each option shows priority, distance, phone and availability |
| Final Closure | The dedicated closure screen with all ten closure fields, the full per-stage editable summary, collapsible groups, and a view-only lock once the case is closed |
| Dashboard and search | Four status-grouped report lists, per-row paid/location flags, and search by phone, Case ID, vehicle registration number and client tracking ID |

### Field applications

| Requirement | What was built |
| --- | --- |
| Per-user case lists | Each app resolves the logged-in user's own record and shows only their cases. The vendor app covers both service types and both the Individual and Fleet paths, including hand-off to one of the vendor's own technicians |
| Availability toggle | An online/offline switch built into each of the three field apps, writing to the user's own availability record. This replaced the original standalone toggle widget |
| On-road flows | RSR: Accept/Reject, Reach, Work In Progress, Final Payment. TOW: Accept/Reject, Reach, Loading, Reached Drop, Unloading & Handover, Final Payment. Each step captures the photographs its stage requires, with enforced minimum counts |
| GPS capture | Position recorded at Accept, Reach, Cancel, Picked-Up and Reached Drop, each written to its own field on the case |
| New-case alerts | A background poll while the case list is on screen detects a newly arrived invite or a status change on a known case and raises an audible alert. Each of the three apps uses a deliberately distinct sound |

### Distances, ETA and timestamps

Locations A–H from the client's specification are all captured and saved: drop location, vendor reach-drop position, breakdown location, vendor start position, vendor reach position, vendor cancel position, nearest vendor fleet location, and the static office coordinate.

| Measurement | Trigger | Status |
| --- | --- | --- |
| Start-to-breakdown distance and ETA (RSR and TOW) | Vendor taps Accept | Completed, on Google |
| Breakdown-to-drop distance and ETA (TOW) | Vendor taps Picked-Up | Completed, on Google |
| Rescue round-trip distance (`Roundtrip_Distance`) | Task Confirmation Time, as of 8 Oct | Completed |
| Vendor distance (`Vendor_Distance`, RSR) | Agent Closure save | Completed — one branch outstanding, section 12 |
| Vendor round-trip distance (`Vendor_Round_Trip_Distance`, TOW) | Agent Closure save | Completed — one branch outstanding, section 12 |
| Cancel distance and cancel-to-breakdown distance | Vendor cancels in their app | Completed; cancel-to-breakdown uses straight-line by design |
| Travel mode | — | Completed — TOW routes as four-wheeler, RSR as two-wheeler |

### Payments and invoicing

| Requirement | What was built |
| --- | --- |
| Booking fee collection | A payment link generated through Zoho Payments and sent to the customer on an agent click, with a public customer page to pay on. A webhook records the outcome, writes the transaction reference, the payment method and the receipt time, and advances the case status |
| Remaining fee collection | The same mechanism at the Final Payment stage, against its own fields |
| On-demand invoicing | Invoice creation in Zoho Books for on-demand (walk-in) cases, with the GST split applied, payment recorded against the invoice, and the invoice PDF captured back onto the case and openable from the ticket |
| Vendor payment | Vendor fee calculation from the vendor's own rate card — a KM slab formula applied to the actual distance, plus night-rate handling and extras — producing gross fee, TDS and net payable. TDS resolved from the vendor's own Zoho Books record with a rate cross-check. Payment recorded back to Books |
| Operations approval | A review stage where an operations manager can adjust the TOW round-trip distance and extras, see the fee recalculate live, and approve — individually or in bulk — with the computed fee written back and the settlement status advanced |

### Messaging, telephony and the customer page

| Requirement | What was built |
| --- | --- |
| WhatsApp messaging | Templated messages through WhatsPortal at defined points: the opener pair, booking-fee confirmation, rescuer details and ETA, an extra-charges advisory, status milestones and a Google feedback request. Customer-facing sends are deliberately placed on an agent click rather than firing automatically |
| WhatsApp inbound webhook | An endpoint accepting three payload shapes, matching an inbound message to a case on the last ten digits of the sender, and writing an inbound log. **Extended 10 Oct** to parse a shared location pin and fill an empty breakdown location |
| Ozonetel telephony | Click-to-call from a ticket, inbound screen-pop handing the call identifier to the ticket, call-to-case linking and a per-ticket call history. **Substantially revised 9–12 Oct** — see section 9 |
| Customer page | A public, no-login page for booking payment, remaining payment, location sharing and feedback, routed entirely through narrow Custom APIs |

## New requirements the client introduced

Everything below was introduced after the original scope was agreed. All are built unless the state column says otherwise.

| Date | What the client introduced | State |
| --- | --- | --- |
| 3 Aug | **Per-vendor invite architecture.** One `Invites` record per invited vendor per case, created when the Assignment step is saved, carrying that vendor's own response. Replaced reading assignment state off the shared case record | Built |
| 7 Sep | **LOCATIONS-DISTANCES-DATETIME specification.** A formal definition of every location to capture, every distance and ETA to calculate, each one's trigger point, and a set of named timestamps | Built |
| Early Sep | **Customer page, vendor performance dashboard, agent performance dashboard.** Three new user-facing surfaces | Built |
| Sep | **Ozonetel telephony integration.** Click-to-call, inbound screen-pop, call logging and call-to-case linking | Built, revised Oct |
| Sep | **Vendor accounting and operations approval.** Vendor fee calculation, TDS, settlement status and an approval stage | Built |
| Oct | **WhatsApp inbound webhook.** Receiving customer replies rather than only sending | Built, blocked on WhatsPortal |
| Oct | **Invoice PDF display.** The Zoho Books invoice PDF itself shown from the ticket, rather than a link to a Books page | Built |
| Oct | **The three-step Ozonetel specification.** Create a call row at ring time; attach the RSID at Create Case or Assign; write call details at hangup | Steps 2–3 built; step 1 needs an Ozonetel setting |
| 9 Oct | **Closure page shows linked calls only.** Unlinked calls sharing the number are counted, not listed | Built |
| 10 Oct | **Old Case search in the call pop** by mobile number, client tracking ID, vehicle registration or Case ID | Built |
| 10 Oct | **Closed tickets open view-only** in the Kanban board | Built |
| 10 Oct | **Inbound location pin** parsed from WhatsApp and written to an empty breakdown location | Built, blocked on WhatsPortal |
| 10 Oct | **SEND 9 five minutes after SEND 8**, rather than immediately | Built, needs one field created |

### Withdrawn after being built

**Territory and pincode zoning**, 10 October. A `Territory_Master` form, a `Breakdown_Pincode` field on the ticket, a vendor `Territory` field, and territory-aware ordering in the assignment picker were built and then reverted the same day at the client's instruction.

The revert was made by reversing each edit individually rather than by discarding the file, because other uncommitted work shared it. Two incidental faults were caught during that revert and corrected.

### Explicitly out of scope

Stated by the client on 11 October: *“right now we are not doing these point”*.

- Zoho Books finance dashboard
- Airtable sync
- REASSIGN button
- Structured QA and UAT
- Territory and pincode

## Changes to requirements already built

| Date | Originally | Changed to |
| --- | --- | --- |
| 7 Aug | Eight vendor/technician/driver stages were entirely view-only for the agent | Every field editable at every step, so an agent can enter a case on a vendor's behalf. Implemented through an `Acceptance_Mode` setting that switches each field between agent-editable and locked, rather than a blanket unlock |
| 3 Oct | A rejected case stayed on the vendor's list for 24 hours so they could see what they had turned down | It must disappear immediately. The window was removed rather than shortened |
| 3 Oct | A reject wrote `RSP REJECT` onto the shared case record | A reject goes on that vendor's own invite row only. The shared case is marked rejected only when no other invited vendor is still live — so one vendor's reject no longer blocks the others |
| 3 Oct | The invite timeout was a 15-minute placeholder | Two minutes, client-set |
| 8 Sep, then 7 Oct | Google Distance Matrix was used for every distance calculation in every app | First restricted to closure calculations only, then re-enabled for four named calculations and no others. Implemented as an explicit opt-in per call site, defaulting to straight-line, so no unlisted calculation can silently return to billed API usage |
| 7 Oct | The booking-fee payment method defaulted to Cash and a receipt photo was mandatory whenever a method was set | A zero booking fee requires no receipt and leaves the method blank. Extended 8 Oct: a fee already paid through the gateway no longer pre-selects Cash either |
| 8 Oct | `Roundtrip_Distance` was calculated at Agent Closure | Calculated at Task Confirmation Time. Implemented as an additional, earlier write; the closure calculation is retained as a fallback and can never overwrite the earlier value |
| 8 Oct | The customer's ETA was sent as a number of hours | Sent as a clock time: calculated ETA plus ten minutes for vendor-app cases, or acceptance time plus the agent's ETA override for agent-on-behalf cases |
| 9 Oct | The closure page listed calls matching the ticket by phone number or call id as well as by link | Linked calls only. The others are counted and the count reported, so an empty table is never mistaken for a broken read |
| 12 Oct | A call could be linked to a ticket from any unlinked call sharing the number within four hours | Only the agent's own last unlinked call, attributed through `Placed_By`. If their last call is already linked, nothing is linked — the search never walks further back |

### Why the last change links less than before

Worth stating because it is a deliberate reduction in behaviour, not an oversight.

A call the agent never popped carries nothing attributing it to them, so it is now **left alone rather than guessed at**. That is what removes the randomness the client reported. A switch, `CALL_LINK_REQUIRE_OWN_CALL`, restores the previous phone-and-recency search in one character should the narrower rule prove too restrictive in practice.

## Client requirement lists, point by point

Two lists were raised as a set. Both are answered here in the client's own numbering.

### The five-point multiple-vendor list

All five are implemented. Points 1 and 3 were blocked until 9 October by a single defect in the invite-response read path, described in section 12.

| # | Requirement | State |
| --- | --- | --- |
| 1 | When the first vendor accepts, the case disappears from every other invited vendor's app and they are told they missed it | Completed |
| 2 | A vendor who does not respond within two minutes times out, the case disappears, and they are told it was missed | Completed |
| 3 | Multiple-vendor assignment status must be visible on the ticket | Completed — the Vendor Invite Status panel on the Assignment step, plus a dedicated Accept/Reject audit screen |
| 4 | A rejected case must disappear from the vendor's app | Completed |
| 5 | One vendor's reject must not mark the case rejected for the others | Completed |

### The incoming-call mail of 10 October

The client sent detailed findings on call behaviour and ticket linking. Each point is answered below.

| Point | Our position |
| --- | --- |
| **1A — the call row should be created when the call lands, not at hangup** | **Agreed, and already built.** `recordRingingCall` exists and writes a row with status `RINGING`. It is not firing because Ozonetel's ScreenPop has not been pointed at it. This is a setting on their side, not a code change |
| **1A — one row for an unanswered call** | Confirmed working; untouched |
| **1A — two records created when a ticket is made during a call** | **Confirmed as a defect in our code, and fixed.** The screen-pop wrote a row carrying whichever id the ScreenPop URL supplied; the end-of-call handler could not match it, so it created a second row |
| **1A — “can this be resolved using UCID and Call\_status?”** | **Yes, and that is what was built.** A row for the caller's number still carrying `RINGING`, `Incoming` or `Placed` is claimed and completed in place. Those three words are written only by row creators and never by a callback, so such a row is by definition waiting for that call — whatever id it holds. No Ozonetel support call was needed |
| **1B — a random pre-existing call gets attached** | **Confirmed and fixed.** Two code paths searched for any unlinked call sharing the number within four hours; a repeat caller always has older rows in that window |
| **1B — Option 1, link only during the call** | **Not viable as written, and we said so.** The complete record does not exist until hangup, so linking during the call can only ever attach the placeholder. It becomes viable once the duplicate is fixed, because then the live row and the final row are the same row |
| **1B — Option 2, only the agent's last call** | **Built.** Attributed through `Placed_By`, and restricted to exactly one candidate |
| **Key concern — could several open windows leave several UCIDs active?** | **No.** The call id is a plain page variable read from that window's own URL. Nothing is stored in `sessionStorage` or `localStorage`, so windows cannot contaminate each other |
| **“Will a last-received-call constraint resolve mismatches?”** | **Yes, and it addresses the real risk.** The mechanism they feared is not present, but a related one is: the variable is never cleared, so a window opened for an earlier call still links *that* call if a ticket is created in it much later. Option 2 fixes exactly that case |

**Both 10 October fixes are deployed but not yet verified by a test call.** The `getCallbackStatus` execution log names which match stage fired, and that line settles what identifier the ScreenPop URL actually carries.

## Original vs changed vs implemented

The single table answering what was asked for, what the client changed, and what we built.

| Module / feature | Original requirement | Client's new requirement or change | What we implemented | Status | Remarks / pending |
| --- | --- | --- | --- | --- | --- |
| **Vendor assignment** | A picker filtered by distance and priority; assignment written to the case | Per-vendor invites; first-accept wins; two-minute timeout; per-vendor reject; visible status | An `Invites` record per vendor per case; invite status panel on the Assignment step; a separate Accept/Reject audit screen with response time and reason | Completed | — |
| **Agent-on-behalf entry** | Vendor stages were view-only for the agent | Every field editable so an agent can run the case for the vendor | `Acceptance_Mode` switches each field between agent-editable and locked; distance calculations branch on the same setting | Completed | — |
| **Distances and ETA** | Google Distance Matrix for distance and travel time | A formal nine-measurement specification with named trigger points; then Google restricted to four named calculations; then revised branch rules and trigger points | All nine measurements with their trigger points; Google opt-in per call site; branch logic per the revised specification | Completed | Two billing branches rest on an assumption — section 15 |
| **Customer ETA message** | ETA shown to the customer | Clock time, +10 minutes on vendor-app cases; acceptance time + override on agent-on-behalf cases | Both rules built, 24-hour format, with a switch to revert | Completed | The approved WhatsApp template has the word “Hrs” in its body; it must be edited and re-approved by Meta |
| **Booking fee** | A booking fee collected up front, by link or in person, with a receipt photo | Zero fee requires no receipt and no method; a gateway-paid fee must not demand a cash receipt | The receipt requirement and the method default are both conditional on the fee and on whether it was already paid; on-screen text states the rule in force | Completed | Add **Payment Gateway** as an option on `Payment_Method` in Zoho |
| **Invoicing** | Not in the original scope | On-demand invoicing in Zoho Books; then the invoice PDF shown from the ticket | Invoice creation with GST split, payment recorded, PDF captured and openable from the ticket. Automatic on closure since 14 Sep | Completed | — |
| **Vendor payment** | Not in the original scope | Vendor fee from a rate card, TDS, settlement status, operations approval | KM slab formula, night rate, extras, gross/TDS/net, Books payment, approval stage with bulk approval | Completed | `Ops_Approval_Comments` must change from Number to Multi Line in Zoho |
| **Telephony** | Not in the original scope | Click-to-call, screen-pop, call logging; then the three-step specification; then the duplicate and random-link defects | All of it, plus the five-stage call-row match and agent's-own-last-call linking | Completed | Ring-time rows need ScreenPop pointed at `recordRingingCall`. Both Oct fixes unverified |
| **WhatsApp** | Customer notifications | Inbound replies received as well as sent; booking-fee message after the customer taps Continue; location pins parsed | Templated sends at defined points; an inbound webhook writing a log and filling an empty breakdown location | Partially completed | The send after Continue is recorded but deliberately left on an agent click — the client expects it automatic, needs confirmation. Inbound is blocked on WhatsPortal webhook support |
| **SEND 9 timing** | Not in the original scope | Five minutes after SEND 8, not immediately | A due field stamped at acceptance and swept by the agent dashboard poll | Completed | Needs `Send_9_Due_At` created. **The dashboard is the only sweeper** — see section 12 |
| **Closure** | A dedicated closure screen with refund, feedback and GST fields | Closure-time distance calculations; validation rules; a view-only lock once closed; linked calls only | All of it, plus a per-stage editable summary of every field in the case | Completed | Ten closure field names were originally guesses; several have since been confirmed. Needs verification for the remainder |
| **Operations board** | A Kanban view, one column per Status | Closed tickets must open view-only | Banner, inert field area, Save and Close hidden, with two guards on the write paths | Completed | — |
| **Territory & pincode** | Geo-fence radius only | Pincode zones mapped to vendor pools | Built 10 Oct, **reverted the same day** at the client's instruction | Withdrawn | The 8 km / 20 km geo-fence is unaffected and in force |

## Additional features built beyond the requirements

Each of these was built in response to a real failure or risk observed during the project, not to a stated requirement.

| Enhancement | Purpose and benefit |
| --- | --- |
| **Record locking** | A case being edited is locked to that agent, with a heartbeat and a stale-lock timeout of fifteen minutes. Added after the client reported two agents editing the same new ticket simultaneously. Prevents one agent's work silently overwriting another's |
| **Offline photo queue** | Photographs taken without a connection are queued and flushed when connectivity returns, with the submit path waiting on the flush. Added after photographs were reported lost when a vendor came back online |
| **Two-tap confirmation** | Destructive actions in the field apps use an arm-then-confirm button instead of a browser confirmation dialog, because such dialogs silently return “no” inside mobile web views — which made photo removal appear broken on phones |
| **On-screen diagnostics** | Where a failure would otherwise be invisible — an unresolvable login, a case with no linked invite, an invite with no readable ticket reference — the cause is rendered into the screen in plain language rather than only written to a console that does not exist on a phone |
| **Write verification** | Several writes are read back and confirmed rather than trusted. Creator can accept an update that silently stores nothing, most often when a field exists on a form but not on the report layout. This pattern has caught real failures on the call link, the invoice PDF and the payment webhook |
| **Invite de-duplication** | Re-saving the Assignment step no longer creates duplicate invite records |
| **Call-link mismatch warning** | Linking a call whose number does not match the ticket is still possible — a customer may call from another phone — but is marked on the button and spelled out in the confirmation, after a mismatched call was linked unnoticed |
| **Vendor base fallback** | Where a vendor's measured distance is missing, the accounting screen falls back to the recorded breakdown distance so a case with a good measurement is never priced at zero kilometres |
| **Audible new-case alerts** | Each field app raises a distinct sound when a new case arrives, generated in the browser with no audio file to host |
| **Search breadth** | Cases are findable by phone, Case ID, vehicle registration number and client tracking ID, including closed cases hidden from the dashboard |
| **The empty-panel diagnostic** | An empty Vendor Invite Status panel used to render as nothing at all, indistinguishable from “no vendors invited” and from a failed read. It now states which it is, with the row counts and whether `RSID` was among the fields the report returned |
| **Near-miss call counting** | When the closure page shows no linked call it reports how many unlinked calls share the ticket's number, so an empty table is never mistaken for a broken one |
| **Alternate call-id matching** | The end-of-call handler tries four identifiers and a status-based rule before creating a row, so a mismatch between Ozonetel's ring-time and hangup identifiers cannot produce a duplicate |

## Open items and known defects

This section was referenced in the previous edition but was missing from it. It is the honest counterpart to the completion tables above.

### Defects found and fixed

All of these were live. Each is recorded because it illustrates a trap specific to this platform.

| Defect | Cause | Fixed |
| --- | --- | --- |
| Vendor accepted but the ticket still showed PENDING | `lookupId()` was called in the Assign panel's renderer but never defined, throwing exactly when there were invites to display — so the panel rendered blank precisely when it had something to show | 9 Oct |
| Multi-vendor invites never matched | The vendor app compared `inv.Vendor.display_value`, but Creator returns a Lookup as a **bare id**, not an enriched object | Oct |
| Every re-save created duplicate invites | A server-side criteria on a Lookup field is rejected by Zoho — code 3330. The failure was caught and treated as “no existing invites” | Oct |
| New calls attached to old closed cases | Four separate places guessed the ticket from the caller's phone number | Oct |
| A random older call attached to a ticket | Two paths searched any unlinked call sharing the number within four hours | 12 Oct |
| Two call rows per call | The screen-pop's identifier and the hangup identifier differ, so the match missed and a second row was created | 12 Oct |
| Gateway payments relabelled as cash | The `Payment_Method` default guessed Cash when a webhook retry dropped the field | Oct |
| 55 call rows stored a malformed id | Order-dependent string stripping ran each cleanup pass once, in a fixed order | Sep |
| Vendors past the 200th were unresolvable | A read capped at 200 records — blank in the picker, “(no vendor)” in accounting, no rate card, so those cases could not be approved | Aug |
| A payment webhook that silently did nothing for weeks | The wrapper key is `event_object`, not `data`, and a Deluge parameter merely *named* in a signature binds null | Sep |

### Structural limitations

These are not defects to fix but characteristics to understand.

**The agent dashboard is the only scheduler.** Anything time-delayed — the SEND 9 delay, scheduled-ticket promotion, picking up field-side changes — runs on the dashboard's 30-second poll, which only runs while somebody has that page open. Nobody on the dashboard all evening means SEND 9 does not go at all. A Creator widget cannot hold a timer across a closed tab, so this is a platform constraint rather than a bug. `SEND9_DELAY_MINUTES = 0` restores immediate sending.

**Role gating is cosmetic.** Operations-manager sections are hidden with CSS and their data is still fetched. Developer tools reveal them. If vendor payout figures must be genuinely restricted, that needs enforcing in Zoho's own permission model.

**Silent failure is the platform's normal behaviour.** A field missing from a report layout reads `undefined`; a wrong report name loads an empty list; an unregistered Deluge parameter binds null. None of these raise an error. Any change touching a field or report name must be verified against live data.

**Three near-duplicate field apps.** The vendor, technician and driver apps share most of their logic by copy rather than by a shared module. Every fix must be applied three times.

### Outstanding within completed areas

| Item | Area |
| --- | --- |
| `Vendor_Distance` branch 3 and `Vendor_Round_Trip_Distance` branch 4 rest on an unconfirmed reading of “Cancelled” | Billing |
| Ring-time call rows do not exist until ScreenPop is configured | Telephony |
| Inbound WhatsApp cannot work until WhatsPortal supports outbound webhooks | Messaging |
| The booking-fee message after Continue is recorded but not sent automatically | Messaging |
| Ten closure field names were originally guesses; some remain unconfirmed | Closure |

## Integrations

| System | Used for |
| --- | --- |
| **Zoho Creator** | The entire backend: forms, reports, records and the Data API the widgets run on |
| **Zoho Payments** | Booking-fee and remaining-fee payment links, with a webhook recording the outcome. The return URL lands the customer on the location-sharing page after a booking fee and the feedback page after a remaining fee |
| **Zoho Books** | On-demand customer invoicing with GST, invoice PDF retrieval, vendor contacts, TDS resolution and vendor payment recording |
| **WhatsPortal** (WhatsApp Business) | All templated customer messaging, and an inbound webhook for customer replies and location pins |
| **Ozonetel** | Outbound click-to-call, callback status, inbound caller information and screen-pop |
| **Google Maps Distance Matrix** | Road distance and travel time, restricted to four named calculations, with a straight-line fallback whenever a route cannot be obtained |
| **OpenStreetMap** | Map display (Leaflet tiles) and reverse geocoding via Nominatim — used deliberately in place of Google to stay inside the client's API scope |

### Twenty Custom APIs

Server-side work that cannot run in a widget: payments, invoicing, webhooks, telephony and cross-report reads.

| Group | APIs |
| --- | --- |
| Telephony | `recordRingingCall` · `getCallbackStatus` · `getCallbackStatusJson` · `getInboundCallerInfo` · `getVoiceCallbackDetails` · `cleanCallLogUcids` · `testCallLogReport` |
| Messaging | `sendWhatsAppMessage` · `whatsappWebhook` |
| Money | `generateZohoPaymentLink` · `paymentWebhook` · `onDemandInvoicing` · `fetchInvoicePdf` · `recordVendorPayment` |
| Data and access | `createTicket` · `searchTickets` · `getVendorTickets` · `getTaskRejections` · `getVehicleNames` · `checkOperationsManagerPermission` |

## Testing and verification status

This section was referenced in the previous edition but was missing from it. It is the most important section for judging readiness.

### The distinction this report draws

| Label | Means |
| --- | --- |
| **Built** | The code exists and has been read |
| **Verified** | It has been exercised against live Zoho data and behaved correctly |

Structured QA and UAT were **excluded by the client** on 11 October. There is no test suite, no UAT record and no regression pack. What exists instead: syntax and brace-balance checks before each deployment, and defects found in live use and reported by the client. Every fix in section 12 was discovered that way.

### Verified by live use

Mostly because something went wrong first and was then confirmed fixed.

- Ticket creation, the wizard stages, status transitions
- Booking-fee and remaining-fee collection, including the gateway
- Vendor assignment and multi-vendor invites
- WhatsApp outbound templates
- Ozonetel call logging and recordings
- Zoho Books invoice generation
- The payment webhook, after the `event_object` discovery

### Built but not verified

| Area | Why |
| --- | --- |
| Duplicate call-log fix | Deployed 12 Oct, no test call made yet |
| Agent's-own-last-call linking | Same |
| Inbound WhatsApp, including location pins | Blocked: WhatsPortal outbound webhook support unconfirmed |
| SEND 9 five-minute delay | Needs `Send_9_Due_At` created |
| Ticket locking | Needs both lock fields on the report layout |
| Agent and Vendor Performance widgets | Never exercised against live data |
| Closed-ticket export | Column set and CSV escaping unconfirmed |
| `Rate_Master_Report` name | Pattern-matched from the form name, never confirmed |
| Technician and driver invite matching | Fixed alongside the vendor app, never tested on a real multi-vendor ticket |
| Vendor accounting batch payments | Calculations read correct; a real batch not traced end to end |

### What should be tested first

In priority order, because each has a client report behind it:

1. One call log row per call, not two — and read the `getCallbackStatus` execution log to see which match stage fired.
2. Linking takes the agent's own last call only.
3. Vendor accept and reject showing on the Assign page.
4. Closure page showing linked calls only.
5. TOW booking fee editable, BF receipt conditional.

A separate run sheet covers these and six further areas, with shared pass/fail recording.

### Deployment practice

| Check | How |
| --- | --- |
| JavaScript parses | A script-block syntax checker before each pack |
| Deluge braces balance | A comment-and-string-aware brace counter |
| Widget packaging | `zet pack`, per widget, immediately after editing |
| Change isolation | `git diff` reviewed hunk by hunk before packing |

There is no CI, no automated deployment and no staging environment. Changes go from a local edit to production in one step.

## Outstanding actions

Two lists. Neither can be resolved by writing code.

### Zoho and Ozonetel configuration — ours to do

Each fails silently: nothing errors, the feature simply never engages.

| Setting | What it blocks |
| --- | --- |
| Point Ozonetel's **ScreenPop** at `recordRingingCall`, Server Side, with `$CallerID$`, `$UCID$`, `$Type$` and `$AgentID$` substituted | Ring-time call rows — step 1 of the three-step specification |
| Create **`Send_9_Due_At`**, Single Line Text, on `Create_Case` | The SEND 9 five-minute delay. Without it SEND 9 sends immediately, which is the old behaviour |
| Register every `whatsappWebhook` parameter with the correct type — `token`, `event`, `from` as String; `data`, `message`, `payload` as Map; `entry` as List | All inbound WhatsApp. An unregistered parameter binds null and the webhook returns `NOTHING_BOUND` |
| Add **Payment Gateway** to `Payment_Method` | Recording a gateway booking-fee payment |
| Change **`Ops_Approval_Comments`** from Number to Multi Line | Typing an approval comment |
| Add **`Locked_By`** and **`Locked_At`** to the `Agent_Ticket_Report` layout | Ticket locking. Both exist on the form; absent from the layout, reads return undefined and the lock silently does nothing |

### Client decisions — theirs to make

Several carry a money consequence and are listed first.

| # | Question | Consequence |
| --- | --- | --- |
| 1 | Is plain `Cancelled` the same as “Cancelled Not Billable”? | Vendor payout on cancelled jobs. Two distance branches depend on it |
| 2 | `Vendor_Distance` branch 3 — measure from the vendor's base? | Billing |
| 3 | `Vendor_Round_Trip_Distance` branch 4 — the same question for tows | Billing |
| 4 | What should happen when a vendor's Final Payable is negative? | It blocks a payment batch today |
| 5 | Should the CONTINUE tap auto-send messages 2 and 3? | Contradicts the standing “agent clicks send” rule, so it needs an explicit decision |
| 6 | Does the `initiate_conversation` template wording need changing? | The template carries “Hrs” and needs Meta re-approval. The full registered text should be read before anyone answers |
| 7 | REASSIGN — appears on nine wireframe pages, never specified | Never built; currently out of scope |
| 8 | What does Login Status mean, and should it gate assignment? | Dispatch behaviour |
| 9 | `Office_To_Mechanic` and `Breakdown_To_Office` — typed or calculated? | Currently calculated |
| 10 | Breakdown-to-drop origin | The client said they would test and revert |
| 11 | Default booking-fee payment method | Currently none — the agent must pick |
| 12 | Is SEND 10 a real message or a gap in the numbering? | One message may be missing from the sequence |

### What WhatsPortal needs to confirm

Whether they support outbound webhooks and can be configured to call our endpoint. **This question is with Chinmay and is unresolved.** Until it is answered, inbound WhatsApp — replies, the CONTINUE tap and location pins — cannot work, however complete the receiving code is.

## Final summary

|  |  |
| --- | --- |
| **Original scope** | An agent ticket wizard across fourteen defined stages for two service types, an eighteen-value status model, rate-master quoting, location capture, a distance- and priority-ranked vendor picker, three field applications, a Kanban operations board, and a dedicated closure screen |
| **Completed work** | The original scope, plus every major addition the client has introduced since: per-vendor invites with accept, reject and timeout handling; agent-on-behalf data entry; the full nine-measurement distance and ETA specification; payments, on-demand invoicing and invoice PDF display; vendor fee calculation, TDS and operations approval; WhatsApp messaging in both directions; Ozonetel telephony including the three-step specification; a public customer page; and two performance dashboards |
| **Additional features** | Record locking, an offline photo queue, mobile-safe confirmations, on-screen diagnostics, write verification, invite de-duplication, call-link mismatch warnings, a vendor-distance fallback, audible new-case alerts, broad search, the empty-panel diagnostic, near-miss call counting and alternate call-id matching. Each built in response to an observed failure rather than a stated requirement |
| **Client-requested changes** | Ten significant changes to already-built behaviour. The largest: the shift from view-only vendor stages to fully agent-editable entry; the move from a shared reject to per-vendor invites; two successive redefinitions of where the Google API may be used; the revised distance branch rules and trigger points of 8 October; and the October rewrite of how a call is matched and linked |
| **Withdrawn** | Territory and pincode zoning, built and reverted on 10 October |
| **Out of scope** | Books finance dashboard, Airtable sync, REASSIGN button, structured QA and UAT, territory and pincode |

### Status at a glance

| Area | Status |
| --- | --- |
| Agent ticket wizard | Completed |
| Field applications (vendor, technician, driver) | Completed |
| Multiple-vendor assignment, accept, reject, timeout | Completed |
| Distances, ETA and timestamps | Completed — two billing branches rest on an assumption |
| Customer ETA message format | **Blocked** on a WhatsApp template change and Meta re-approval |
| Payments, invoicing and vendor accounting | Completed |
| WhatsApp outbound | Completed |
| WhatsApp inbound | Built — **blocked** on WhatsPortal webhook support |
| Telephony | Completed — two October fixes **not yet verified**; ring-time rows need an Ozonetel setting |
| Customer page and performance dashboards | Completed — dashboards unverified |
| Operations board | Completed |
| Territory and pincode | Withdrawn by the client |
| Structured QA and UAT | Excluded by the client |

### The recommendation

**Stop building and start testing.** Nine files were deployed between 10 and 12 October, four of them carrying fixes to defects the client reported, and none of those four has been confirmed working by a test. The six configuration settings in section 15 are small, and three of them are blocking features that are otherwise complete.

The twelve client decisions are the other half. Four of them affect what a vendor gets paid, and they are currently resolved by assumptions recorded in the code rather than by an answer.
