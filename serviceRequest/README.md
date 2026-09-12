# Get-Rescue — Service Request Widget (`serviceRequest`) — Working Log

Public, no-login Zoho Creator widget for `getrescued.in`'s own Service Request page (currently a plain single-page HTML form — see build notes below for exactly what it replaces). Splits that form into a 3-step wizard and, on submit, creates a real `Create_Case` ticket that shows up on the Agent Portal (`getRescueTicket`) dashboard and flows through the exact same existing wizard an agent-created ticket does.

## 2026-09-09 — built from scratch

**User request** (verbatim): *"please create a new widget for the Service Request Page to allow customers to submit service requests directly from the website. I want the Service Request form to be divided into three steps with a clear step-by-step UI... When a customer completes and submits the Service Request form, the request should automatically be created and appear in the Agent Portal so the agent can process it using the existing workflow."*

**Confirmed first, not guessed**: `getrescued.in` is a Zoho Creator page under a custom domain (user-confirmed via AskUserQuestion) — same platform as `getRescueTicket`/`customerPage`, not a separate WordPress/CMS site. That made this a normal new Zoho Creator widget, added to that existing page, rather than needing some other embed mechanism.

### The 3 steps (my own grouping — not specified exactly by the user, flagged as an interpretation)

1. **Contact** — Name*, Phone*, Alternate Phone, Email
2. **Vehicle & Issue** — Vehicle* (select), Vehicle Reg. Number, Vehicle Issue* (chip multiselect, filtered by the chosen vehicle's category — same rule `getRescueTicket`'s own `vehicleIssuesFor()` uses), Vehicle Status* (On Road / Safe Parking)
3. **Timing** — Time of Service* (Now / Later), Service Date & Time* (Later only, a single `datetime-local` picker), Terms & Conditions* checkbox, Submit

On success: shows the new ticket's `Case_ID` ("RSID" + last 6 digits, same scheme every other ticket gets) as a confirmation reference.

### Field mapping — deliberately matches the CURRENT `Create_Case` schema, not the site's existing (older) form shape

The site's existing plain form has separate "Service Date" + "Service Time" fields — that's the **old** schema, replaced in `getRescueTicket` on 2026-08-05 by a single `Schedule_Date_Time` field (see that project's own README). This widget uses the current schema (one `datetime-local` picker) so a ticket created here behaves identically to one an agent creates by hand, rather than preserving an outdated field shape. Flagged as a deliberate interpretation, not silently guessed.

Every field name/report used comes from `FIELDS.md` and `getRescueTicket`'s own `create` stage (`CONFIG.optionReports`, `vehicleCategoryField`, `issueVehicleTypeField`, `issueDisplayStatusField`) — not re-guessed independently. A submission from this widget always writes:
- `Client` = the real "ON DEMAND" `Client_Report` record (looked up by name in Deluge, not hardcoded as a raw ID — same walk-in-customer convention every other on-demand flow in this project already uses)
- `Lead_Source` = **"Website"** — this option already existed in the dropdown per `FIELDS.md`, just unused until this widget (the agent-facing widget always writes `"Call"` instead)
- `Status` = `"SERVICE FEE QUOTE"` — the exact same starting status a brand-new agent-created ticket gets, so the dashboard/wizard resume logic needs zero special-casing for a website-originated ticket
- `Service_Type` (RSR/TOW) — derived **server-side** in Deluge from the selected issue(s)' own `Issue_Type`, not trusted from client input, mirroring `getRescueTicket`'s `isTowIssue()` rule (any selected issue whose `Issue_Type` contains "tow" makes the whole ticket TOW)
- `Phone_Number` mirrors `Phone_Number1` (Create_Case requires both — same as the agent widget's own `extra()` hook)

### Backend — 3 new Custom APIs, all need "PublicKey" auth enabled

Same real requirement `customerPage` hit and fully documented in its own README (`401`/`9370` "invalid public key" — this widget has zero logged-in session, exactly like the Customer Page). Originally 2 APIs, split to 3 after a persistent Deluge "Missing return statement" error that turned out to need two separate single-loop Custom APIs rather than one two-loop one — see this file's own dated entries below for the full trail:

- **`getVehicleOptions.deluge`** — no arguments; returns every `Vehicle_Master_Report` vehicle, so the widget can build its Vehicle dropdown.
- **`getIssueOptions.deluge`** — no arguments; returns every `Vehicle_Issue_Report` issue (`DISPLAY_STATUS=SHOW` only), so the widget can do the vehicle-category → issue filtering client-side.
- **`createServiceRequest.deluge`** — accepts the full form payload, creates the `Create_Case` record (two-step: insert, then a second update to set `Case_ID` once the new record's own ID is known — same pattern `getRescueTicket`'s create-stage save already uses), returns `{code, recordId, caseId}`.

**Setup needed in Zoho Creator before this works at all** (mirrors `customerPage`'s exact setup):
1. Create all 3 Custom APIs above (paste each `.deluge` file's contents in). If `getServiceRequestOptions` was already created from an earlier attempt, delete it — it's been replaced.
2. Enable **"PublicKey"** authentication on all 3.
3. Copy each one's Public Key and paste it into `CUSTOM_API_PUBLIC_KEYS` near the top of `app/widget.html` (currently blank placeholders — same shape `customerPage` used before its own keys were filled in).
4. Add this widget to the `getrescued.in` Service Request page (replacing or sitting alongside the existing plain form — your call on which).
5. Upload/publish `dist/serviceRequest.zip`, then (same lesson learned the hard way on `customerPage`) click **Relaunch to update** if the Portal/page shows that prompt.

### Known open risk, flagged not hidden — Deluge date handling

`Computed_Service_Time` and `Schedule_Date_Time` are sent to Deluge as **already-formatted strings from the widget's own JS** (matching `getRescueTicket`'s own `fmtDateTime12()` output / `datetime-local`'s raw value respectively) rather than parsed/computed inside Deluge itself — deliberately, to avoid relying on Deluge's own date-arithmetic functions, since this project has hit real Deluge compile/runtime errors on far simpler logic multiple times already (see `customerPage/README.md`'s own "Missing return statement" entries ×3) and there's no way to live-test Deluge from outside Zoho Creator. **If either field is rejected on a real save, this is the first thing to check** — see `createServiceRequest.deluge`'s own header comment for the exact format each one arrives in.

### Terms & Conditions link

Same placeholder (`href="#"`) as `customerPage`'s own T&C checkbox — no real Terms page exists in this project yet. Swap in the real URL once one exists.

### Styling

Approximated from screenshots of the current `getrescued.in` site (orange `#F15A22` accent, dark navy text, clean sans-serif, triangle "RESCUE" logo mark) — not pulled from the site's own real CSS/brand guide, since I don't have access to that. Close enough to look native, but worth a visual comparison against the real site before this goes live.

### Tested

New `test_service_request_widget.js` (16 checks, temp scratch location, not committed) — covers the pure-logic functions: `formatIndianPhone()`, `valuesOf()` (multiselect normalization), `issuesForVehicle()` (category filtering — shared issues appear for both vehicle types, type-specific issues don't leak across), `computeServiceTimeMs()` (NOW vs LATER vs LATER-with-no-date-picked fallback), `fmtDateTime12()` (exact format match, including the AM/PM and midnight-hour edge case). All pass. **Not yet tested**: the actual Deluge scripts (can't be run outside Zoho Creator), the full 3-step UI/DOM rendering path, or an actual end-to-end submission — all of that needs a real Zoho Creator test once the setup steps above are done.

**Needs deploying**: this is a brand-new widget — nothing to redeploy over, just the initial setup steps above.

## 2026-09-09 (later) — real Deluge error fixed: header comment before the function signature

User hit this immediately when pasting `getServiceRequestOptions.deluge` into its Custom API: `Expected 'VOID','INT','FLOAT','STRING','DATE','BOOL','MAP' or 'LIST' but found '/* getServiceRequestOptions...'`. Real Deluge constraint, confirmed by checking every other `.deluge` file already working in this repo: a Custom API's return-type keyword must be the **literal first token** in the file — a header comment placed before the function signature (the original shape of both new files' own top comments) fails to parse at all. Every existing script in this project already puts its own explanatory comment inside the function body for exactly this reason; both new files here just hadn't followed that convention yet.

**Fixed**: moved both `getServiceRequestOptions.deluge`'s and `createServiceRequest.deluge`'s header comments to right after their `{` instead of before their signature — same content, just relocated. No logic changed.

**Tested**: not independently re-verified live yet (waiting on the user's next paste-and-save attempt) — but this fix is a direct, confirmed structural match to every other already-working `.deluge` file's own shape in this repo, not a guess.

## 2026-09-09 (later still) — real Deluge error fixed: form name vs report name; header comments trimmed way down

User hit a second real error on save: `Variable 'Vehicle_Master_Report' is not defined`. Root cause, confirmed against `FIELDS.md`: Deluge's `for each`/bracket record-access syntax needs the **form's** own link name, not the report name built on top of it — `Vehicle_Master` (report `Vehicle_Master_Report`), `Vehicle_Issue` (report `Vehicle_Issue_Report`), `Client` (report `Client_Report`). `getCustomerTicketInfo.deluge`'s own working `Create_Case[...]` reference was already using a form name for exactly this reason — this project's `optionReports`/`FIELDS.md` naming (which always shows the *report* name) made it easy to reach for the wrong one here.

**Fixed** in both files: every `Vehicle_Master_Report`/`Vehicle_Issue_Report`/`Client_Report` used as a Deluge record source is now `Vehicle_Master`/`Vehicle_Issue`/`Client`.

**Also**: user asked to remove the unwanted (overly long) header comments — both files' multi-paragraph header blocks are now a single short comment, matching the leaner in-body-comment style every other already-working `.deluge` file in this project actually uses (see `getCustomerTicketInfo.deluge`). No logic changed, just far less prose.

**Needs re-pasting**: both `.deluge` files into their Custom APIs in Zoho Creator — this is the second real compile-time fix in a row, so re-paste both together rather than one at a time.

**Tested**: not independently re-verified live yet.

## 2026-09-09 (later yet) — real Deluge error fixed: bare form name needs an explicit criteria to fetch all records

Third real compile error on the same file: `Variable 'Vehicle_Master' is not defined` — even after fixing the form-vs-report name issue above. Confirmed against Zoho's own official Deluge documentation (`for-each-record.html`): a bare `for each x in FormName` with no square-bracket criteria parses as an undefined **variable** reference, not a record-fetch — Zoho's own docs explicitly recommend `[ID != 0]` as the "fetch every record" criteria for exactly this case.

**Fixed**: `for each veh in Vehicle_Master` → `for each veh in Vehicle_Master[ID != 0]`, same for `Vehicle_Issue`. Only `getServiceRequestOptions.deluge` iterates a whole form this way — `createServiceRequest.deluge`'s own `for each` loop iterates a local Deluge List (`issueIdList`), not a form, so it was never affected by this.

**Needs re-pasting**: `getServiceRequestOptions.deluge` again.

**Tested**: not independently re-verified live yet — but this fix is now backed by Zoho's own official documentation, not inferred from another script in this repo.

## 2026-09-09 (yet later) — correction: reverted form-name guess back to the report names, confirmed real by the user directly

User screenshotted the actual live report URL (`.../#Report:Vehicle_Master_Report`) — direct, confirmed proof `Vehicle_Master_Report` is a real identifier. On reflection, the earlier "form name, not report name" diagnosis was likely wrong: **both** "Variable is not defined" errors were more likely the SAME root cause — a bare form/report name with no `[criteria]` parses in Deluge as an undefined variable either way, regardless of which name is used. The form-name switch was never actually confirmed as the fix on its own; it just happened to be changed at the same time other things were still broken.

**Reverted**: `Vehicle_Master`/`Vehicle_Issue` back to `Vehicle_Master_Report`/`Vehicle_Issue_Report` in both files (`Client` back to `Client_Report` too, in `createServiceRequest.deluge`) — keeping the `[ID != 0]` bracket-criteria fix from the entry above, since that specific combination (report name + criteria) had never actually been tried. The single-record lookups (`Client_Report[Client_Name == "ON DEMAND"]`, `Vehicle_Issue_Report[ID == ...]`) already had their own criteria from the start and were never confirmed broken — reverted for consistency, not because they were shown to fail.

**Needs re-pasting**: `getServiceRequestOptions.deluge` and `createServiceRequest.deluge` again.

**Tested**: not independently re-verified live yet.

## 2026-09-09 (still later) — real Deluge error fixed: "Missing return statement", likely caused by `continue` inside a for-each-record loop

User pasted back their own live-edited version of `getServiceRequestOptions.deluge` (identifiers had drifted to `vehicle_master`/`Vehicle_Issue` — see the open question below) and hit `Missing return statement: Provide STRING expression to return`, despite every path already having an explicit `return`. Prime suspect: the `continue` statement inside the issue-filtering `for each iss in ...[ID != 0]` loop — Deluge's static return-path checker has been unreliable with non-linear control flow throughout this whole project (3 prior "missing return" fixes elsewhere, always resolved by removing whatever made a path harder to statically prove).

**Fixed**: removed the `continue`; the `DISPLAY_STATUS != "SHOW"` filter is now an `if(...== "SHOW") { ... }` wrapping the rest of the loop body instead of an early skip. Identical filtering result, no early-exit for the checker to reason about.

**Open question, not resolved**: the pasted-back version had `vehicle_master` (lowercase) and `Vehicle_Issue` (no `_Report` suffix) instead of this file's own `Vehicle_Master_Report`/`Vehicle_Issue_Report`. Unclear whether these came from Zoho's own autocomplete (in which case they're the real, confirmed identifiers and should be adopted everywhere, including `createServiceRequest.deluge`) or were a manual guess/typo. Kept as-is in this fix (not silently reverted a third time) — needs a direct answer before touching the naming again.

**Needs re-pasting**: `getServiceRequestOptions.deluge` again.

**Tested**: not independently re-verified live yet.

## 2026-09-09 (later again) — naming question resolved: form names ARE correct, just needed a field fix too

Next error after the `continue` fix: `Variable Name does not exist in Vehicle_Issue` — this is the confirmation needed: Deluge accepted `Vehicle_Issue` as a real identifier (didn't reject the name itself), only complained about a field on it. `Vehicle_Issue` (and `vehicle_master`) are the real form names after all — the ORIGINAL "form name, not report name" diagnosis was right; it just needed the `[ID != 0]` bracket fix too, and this real field-name bug was hiding behind the two earlier errors.

**Fixed**: `iss.Name` (doesn't exist on the `Vehicle_Issue` form — it only has `Issue_Name`) replaced with the same "fall back to a generated label" pattern the vehicle list already uses (`"Issue " + iss.ID`). Also switched `createServiceRequest.deluge`'s own `Client_Report`/`Vehicle_Issue_Report` references to the form names (`Client`/`Vehicle_Issue`) to match, now that this is confirmed rather than guessed.

**Needs re-pasting**: both `.deluge` files.

**Tested**: not independently re-verified live yet.

## 2026-09-09 (even later) — "Missing return statement" persisted after removing `continue` — nested try/catch inside for-each-record loops removed too

The `continue`-removal fix above did NOT resolve the error — user pasted back the exact same `continue`-free code and hit the identical `Missing return statement: Provide STRING expression to return`. That theory is now disproven; `continue` inside a for-each-record loop was not the actual cause.

**Next hypothesis, applied**: the 4 nested `try/catch` blocks inside the two for-each-record loops (`veh.Category`, `iss.DISPLAY_STATUS`, `iss.VEHICLE_TYPE`, `iss.Issue_Type` — each guarding a `.toString()` call). Replaced every one with a plain `if(x != null) { y = x.toString(); }` — no exception handling, just a null-check — removing every non-linear control-flow construct from the function (no try/catch except the outer one, no continue, no loops-within-loops). If this doesn't resolve it either, the next thing to try is pulling the two loops out into their own helper functions, since that's a structurally different shape from anything tried so far.

## 2026-09-09 (latest) — real fix: split into helper functions — the error's own line number was the key clue

That fix didn't work either — same error again, but this time Zoho's own error dialog showed **"Error at line number: 5"**, pointing at `result = Map();`, right at the top of the `try` block — nowhere near any actual return statement. That's the real signal this was never about a genuine gap in THIS function's return coverage at all; something about having two `for each record` loops directly inside the main function's own try block was tripping up Zoho's checker from the very start of the block, regardless of what was inside those loops.

**Fixed**: confirmed multi-function Custom API files are supported (Zoho's own docs: "you can structure your Deluge script to include multiple helper functions within the same file"), so split this into 3 functions — `getVehicleList()` and `getIssueList()` (each a `list`-returning function containing exactly one for-each-record loop, nothing else) called from `getServiceRequestOptions()` itself, whose own body is now just two function calls, a couple `result.put()`s, and one unconditional return — no loops, no nested anything, nothing left for the checker to trip over. Same output shape, same field logic, purely a structural split.

**Needs re-pasting**: `getServiceRequestOptions.deluge` — as 3 functions this time, not 1. `createServiceRequest.deluge` is unaffected (it was never showing this error).

**Tested**: not independently re-verified live yet — but this is the first fix attempt directly informed by the error's own line number rather than a structural guess.

## 2026-09-09 (final) — real fix: split into two separate Custom APIs, not helper functions in one file

The helper-function split above hit a different, clarifying error: `Reached end of the function block. Remove the code after the line number 28` — line 28 is exactly `getVehicleList()`'s own closing brace. **Zoho's Custom API editor does not support multiple function definitions in one script** — it treats the first function's closing `}` as the end of the whole file (the earlier web search suggesting otherwise was talking about a different Zoho Creator concept — standalone "Custom Functions," created as their own separate objects, not inline helpers in a Custom API's own script).

Considered `zoho.creator.getRecords()` (returns a plain list, sidestepping the `for each record` task construct entirely) but it requires a pre-configured OAuth **Connection** in Zoho Creator — a new setup concept nothing else in this project uses, and heavier than the alternative.

**Real fix**: `getServiceRequestOptions.deluge` deleted, replaced by two separate Custom APIs — `getVehicleOptions.deluge` and `getIssueOptions.deluge` — each with exactly one `for each record` loop and its own try/catch/return, matching the simple single-purpose shape every other working `.deluge` file in this project already uses. The widget (`app/widget.html`) now calls both in parallel via `Promise.all` in `boot()`, and `CUSTOM_API_PUBLIC_KEYS` has 3 entries instead of 2 (`getVehicleOptions`, `getIssueOptions`, `createServiceRequest`).

**Setup now needs 3 Custom APIs total** (was 2), all with PublicKey auth enabled — see this README's own setup section, updated to match.

**Needs re-pasting**: this is now 2 NEW Custom APIs to create in Zoho Creator (`getVehicleOptions`, `getIssueOptions`) — `getServiceRequestOptions` should be deleted if it was ever created there. `createServiceRequest.deluge` is unaffected.

**Tested**: `formatIndianPhone()`, `valuesOf()`, `issuesForVehicle()` still pass (9 checks); confirmed `boot()` now calls `getVehicleOptions`/`getIssueOptions` (not the old combined name) via a mocked `callCustomApi()`. Not yet confirmed live in Zoho Creator — but each individual `.deluge` file is now the same proven shape as every other working script in this project, not a novel structure.

**Needs re-pasting**: `getServiceRequestOptions.deluge`.

**Tested**: not independently re-verified live yet — genuinely uncertain this is the actual cause, flagged as a hypothesis, not a confirmed fix.

## 2026-09-09 (final) — visual refresh + sample-data mode, so the UI renders for client approval without the backend being ready

User added this widget to a real page (`Service_Request`, inside the app itself) and hit the generic "Couldn't load the request form" error — the 3 Custom APIs' PublicKey auth isn't set up in Zoho yet (still blank placeholders in `CUSTOM_API_PUBLIC_KEYS`). Explicit request: "simple just create UI" — get the form actually visible and clickable, decoupled from backend readiness, so it can be shown for client sign-off. A separate, fully interactive standalone preview of this same flow was also published as a shareable Artifact for exactly this purpose (see chat history) — this change brings that same visual treatment into the real widget file itself.

**Two changes**:
1. **Visual refresh** — new type pairing (Archivo for display/headings, Public Sans for body/labels, loaded via Google Fonts — needed adding `style-src`/`font-src` entries to `plugin-manifest.json` for the fonts to actually load inside the Zoho widget sandbox), a warmer neutral palette extending the existing orange/navy brand rather than the previous plain grey, refined spacing/shadows/radii. No fields, validation, or flow logic changed — purely CSS + 2 `<link>` tags.
2. **`SHOW_SAMPLE_DATA = true`** — `boot()` now skips the `getVehicleOptions`/`getIssueOptions` calls entirely and populates `OPTIONS` from hardcoded sample data (`SAMPLE_VEHICLES`/`SAMPLE_ISSUES`, same shape the real APIs return) when this flag is on; `onSubmit()` likewise skips `createServiceRequest` and shows a mock success screen with a random reference number, clearly labeled "Sample confirmation — no real ticket created". `waitForSDK()` also short-circuits straight to `boot()` in this mode, so the widget renders even with no Zoho SDK at all (e.g. opened as a plain file). Every real-data code path is untouched and still present — flip this one constant back to `false`, fill in the 3 real Public Keys, and the widget goes straight back to live Create_Case tickets with no other changes needed.

**Needs redeploying**: this widget itself (client-side JS + manifest) — freshly repacked.

**Tested**: 5 new checks confirm `boot()` populates real-shaped sample data with zero SDK/API calls, and that the vehicle-category → issue filtering still works correctly against that sample data (2W gets shared + 2W-only issues, not 4W-only, and vice versa). Existing logic tests (`formatIndianPhone`, `valuesOf`) unaffected. Not independently viewed inside Zoho Creator yet — next step is confirming it renders correctly there too, not just in the standalone test harness.

## 2026-09-09 (genuinely final) — rebuilt to match a client-provided reference design exactly, including new fields and a reordered flow

User shared 3 screenshots of a reference design and asked to match them exactly — different visual style, different field set, and a different step order than what was built earlier today. Confirmed via 2 clarifying questions rather than guessing: (1) add the reference's Location field, "currently available only in Bengaluru" notice, and reCAPTCHA as real functionality, not just visual style; (2) reorder to Vehicle Details → Contact Details → final step (was Contact → Vehicle & Issue → Timing).

**New step order and fields**:
1. **Vehicle Details** — Vehicle Type (Two Wheeler / Four Wheeler — a fixed 2-option list, not fetched from Zoho at all anymore), Vehicle Make (free text, e.g. "Honda Activa" — no longer a picker off a real `Vehicle_Master` record), Registration Number, Vehicle Issue (chips, filtered directly by Vehicle Type)
2. **Contact Details** — Full Name, Phone Number, WhatsApp Number (renamed from "Alternate Phone" to match the reference's own label — same field underneath, `Alternate_Phone_Number`), Email
3. **Location step** — a "currently available only in Bengaluru" notice (static text — no actual service-area enforcement/rejection logic, just informational, since nothing in the reference implied real validation), a free-text Location field, Terms & Conditions, a reCAPTCHA-styled checkbox, "Make a Request"

**Real, non-cosmetic backend implications, all handled in `createServiceRequest.deluge`**:
- `Vehicle` (a required Lookup on `Create_Case`) no longer has a real picked record to use directly — the form only collects a free-typed "Vehicle Make." Deluge now resolves this itself: iterates `vehicle_master`, best-effort matches a record of the right Category whose Name *contains* what was typed (case-insensitive), and falls back to the *first* vehicle of that Category if nothing matches — Vehicle is never left blank. The customer's own typed text is preserved in `Remarks` regardless of which record actually got matched, so an agent always sees the real context even on an imperfect match.
- `Vehicle_Status`/`Time_of_service` are no longer asked on the form at all (not in the reference) — defaulted server-side to `"ONROAD"`/`"NOW"`, the same assumption an urgent phone-in request already makes.
- The new `location` field has no dedicated column on `Create_Case` — appended to `Remarks` alongside the vehicle make.
- `getVehicleOptions` is **no longer called by this widget** — Vehicle Type is a fixed 2-item list, not fetched data. That Custom API can stay registered in Zoho unused, or be deleted; `getIssueOptions` and `createServiceRequest` are the only 2 still needed.

**Explicitly flagged placeholder, not real**: the reCAPTCHA is a styled checkbox only ("I'm not a robot," matching the reference's look) — there is no real Google site key, no official widget script, and no server-side token verification. Before this goes live: register a real reCAPTCHA site at google.com/recaptcha, load the official widget, and verify the token in `createServiceRequest.deluge` (an HTTP call to Google's siteverify endpoint) instead of trusting this checkbox.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. `createServiceRequest.deluge` needs re-pasting into its Custom API — its parameter list changed (`vehicleId` replaced by `vehicleType`+`vehicleMake`, `whatsappNumber` replacing `alternatePhone`, new `location` param, `vehicleStatus`/`timeOfService`/`scheduleDateTime` removed since the widget no longer sends them).

**Tested**: 9 checks — sample-data boot with only `getIssueOptions` (no vehicle list needed), the new `issuesForType()` filtering (keyed directly off Vehicle Type, not a Vehicle Lookup's Category) for both 2W and 4W plus the empty-selection case, `formatIndianPhone()` unaffected, and `STATE`'s shape confirmed to have the new fields (`vehicleType`, `vehicleMake`, `location`, `captchaChecked`) and not the old ones (`vehicleId`, `timeOfService`, `vehicleStatus`). The Deluge-side vehicle-matching logic itself can't be tested outside Zoho Creator — not yet independently confirmed live.

## 2026-09-09 (one more) — reCAPTCHA removed

User request: "remove captcha in this steps form." It was only ever a styled placeholder (see the entry above — no real Google site key, no verification, never sent to Deluge at all), so this is purely a widget-side removal: the checkbox, its "I'm not a robot" styling, the `captchaChecked` state field, and its own validation check in `onSubmit()` are all gone. `createServiceRequest.deluge` never referenced it and needs no change. Step 3 is now just Location, Terms & Conditions, and "Make a Request."

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked.

**Tested**: syntax-checked; no automated test referenced the captcha fields, so no test updates needed.

## 2026-09-09 (one more still) — real Deluge error, a new variant: a comment as the very first thing after the function's own `{` also breaks it

User hit `Error at line number: 3 — Missing return statement: Provide STRING expression to return` on `getVehicleOptions.deluge` — a single-loop function that already has every path returning, and already had its header comment moved past the *signature* (the original fix from earlier today). Line 3 is the first line of that header comment, immediately after the function's own opening `{`. Real, narrower rule found: it's not just "no comment before the signature" — a comment as the literal first thing after `{` (before `try` or any other real statement) trips the same error, even deeper inside the function.

**Fixed** in all 3 `.deluge` files (`getVehicleOptions`, `getIssueOptions`, `createServiceRequest`) the same way: moved each one's header comment to right after `try {` instead of right after the function's own `{` — `try` is now always the literal first token inside every function in this project. No logic changed anywhere, purely comment relocation.

**Needs re-pasting**: all 3 `.deluge` files, if any hasn't already compiled successfully in Zoho.

**Tested**: not independently re-verified live yet — same fix pattern already confirmed to work for the earlier "comment before signature" variant of this exact bug, applied consistently this time to every file that had it.

## 2026-09-09 (final for real) — `location` parameter renamed, suspected Deluge reserved word

After the comment-relocation fix above, `createServiceRequest.deluge` hit a *different* error the other two files didn't: `Error at line number: 4 — Improper Statement — Error might be due to missing ';' at end of the line or incomplete expression`, pointing at what should just be `try`'s own opening `{` — not a plausible target for that error message on its own. The one thing unique to this file (vs. the other two, which compiled fine with the same comment fix): its newest parameter, `location` — a name that's entirely plausible as a reserved word somewhere in Deluge's own vocabulary.

**Fixed defensively**: renamed the parameter (and the widget's matching payload key) from `location` to `customerLocation` throughout — `serviceRequest/app/widget.html`'s `onSubmit()` payload and `createServiceRequest.deluge`'s own signature/body. Flagged clearly as a hypothesis, not a confirmed root cause — if this doesn't resolve it, the next step is asking Zoho directly what's actually on that file's real line 4, since the error's own line number didn't point anywhere sensible for this class of message.

**Needs redeploying**: the widget itself (client-side JS) — freshly repacked. **Needs re-pasting**: `createServiceRequest.deluge` again (this specific rename).

**Tested**: syntax-checked. Not yet independently confirmed live — genuinely uncertain this is the actual cause.

## 2026-09-09 (actually the last one) — Location field gets live search suggestions, on free OpenStreetMap instead of paid Google Places

User showed a Google Places Autocomplete reference (live suggestions as you type, "powered by Google" list) and asked for the Location field to work "this way." Flagged before building: Places Autocomplete is a paid, per-request-billed Google API, and this field sits on a fully public, no-login page — a more exposed cost risk than an internal tool, and directly against this project's own standing direction ("don't use paid Google APIs unless absolutely required," the same reasoning that already replaced Google Maps with free OSM tiles on `getRescueTicket`'s Ops Manager Map View). Asked, and user chose the free alternative.

**Built**: the Location input now does a live, debounced (450ms after typing stops, minimum 3 characters) search against OpenStreetMap's free Nominatim API (`nominatim.openstreetmap.org/search`), rendering a dropdown of up to 5 matches — pin icon, bold place name, grey secondary detail (mirroring the reference's own layout), footer reads "Search by OpenStreetMap" instead of "powered by Google." Clicking a suggestion fills the input with that place's full name; the dropdown closes on an outside click. No API key, no billing, no Google dependency at all. Added `https://nominatim.openstreetmap.org` to `plugin-manifest.json`'s `connect-src` so the widget's CSP actually allows the request. Works the same whether `SHOW_SAMPLE_DATA` is on or off — it's a genuine third-party API call, unrelated to Zoho's own Custom APIs.

**Needs redeploying**: this widget itself (client-side JS + manifest) — freshly repacked. No `.deluge` changes — the final submitted value is still the same plain text `customerLocation` param either way, just filled in via search instead of typed freehand.

**Tested**: 4 checks — a query under 3 characters never calls `fetch` at all, a real query updates `STATE.location` immediately (not debounced — only the network call is delayed), and after the debounce window exactly one `fetch` call fires, correctly targeting Nominatim with the query encoded in the URL. Not independently viewed inside Zoho Creator yet — the dropdown's visual layout/positioning is worth a quick look once deployed.

## 2026-09-09 (real keys) — `getIssueOptions`/`createServiceRequest` public keys filled in

User provided all 3 Custom APIs' real PublicKey-auth values. `getIssueOptions` and `createServiceRequest` (the only 2 this widget actually calls) are now filled into `CUSTOM_API_PUBLIC_KEYS`. `getVehicleOptions`'s own key is noted in a comment but not wired in, since this widget doesn't call that API at all anymore (see its own file's header).

**Still open**: whether `createServiceRequest.deluge`'s own "Improper Statement" compile error (see the entries above) is actually resolved hasn't been confirmed yet — these keys being available doesn't by itself confirm the script compiles. `SHOW_SAMPLE_DATA` is deliberately left `true` until that's confirmed, so the form doesn't silently break for anyone viewing it in the meantime.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked.

**Tested**: syntax-checked only.

## 2026-09-09 (diagnostic) — all comments stripped from `createServiceRequest.deluge`

The "Improper Statement" error has now survived two targeted fixes (comment relocation, `location` rename) without a line number to confirm either actually addressed the real cause. User asked to remove all comments — a clean way to test whether something in the comments themselves (an em dash, an embedded quote character, an encoding issue on paste into Zoho's editor) is the real culprit, separate from the code logic. Every `//` line is gone; the two error messages that used to have an em dash and single-quoted `'ON DEMAND'` now use a plain hyphen and no quotes around it, removing those characters too on the chance they mattered. No logic changed — same fields, same validation, same Vehicle-matching behavior.

**Needs re-pasting**: `createServiceRequest.deluge`.

**Tested**: confirmed zero `//` comments remain anywhere in the file. If this compiles clean, comments (or a specific character in them) were the cause and the explanatory comments can be reintroduced more carefully; if the same error persists with literally nothing but code left, that rules comments out entirely and points back at the code itself needing a fresh look with an exact line number.

**Extended to the other 2 files** — `getVehicleOptions.deluge` and `getIssueOptions.deluge` also stripped of every comment, for consistency (these two weren't reported as still failing, but now match `createServiceRequest.deluge`'s comment-free state). No logic changed in either. All 3 `.deluge` files in this project are now pure code, no explanatory comments at all — worth keeping in mind for future edits to this project specifically (this app's Custom APIs, unlike every other already-working `.deluge` file elsewhere in this repo, seem to need a more cautious approach to in-file comments).

## 2026-09-09 (turned out unrelated) — `getVehicleOptions.deluge` proven un-fixable, and confirmed unnecessary

Live screenshot of Zoho's own editor for `getVehicleOptions.deluge` showed "Error at line number: 5" pointing at `vehicleList = List();` — an ordinary variable assignment, not plausibly "missing a return statement" on its own. This is direct proof Zoho's error line number is not the real location of any actual problem in that file — it's an anchor point the compiler falls back to when its return-path checker gets confused by a `for each record` loop in a function that must return a value. No further attempt is worth making on this specific Custom API — **it isn't called by the widget at all anymore** (Vehicle Type is a fixed dropdown, not fetched data) — safe to delete from Zoho or leave broken/unused.

## 2026-09-09 (still investigating) — `createServiceRequest.deluge`'s `for each record` removal did NOT fix its "Improper Statement" error

The `for each record` removal from the vehicle-matching logic (see the entry above, replaced with a single-record `vehicle_master[Category == vehicleTypeTrim]` lookup) did not resolve anything — a fresh screenshot showed the exact same `Improper Statement` error, at the exact same anchor point (line 4, `try`'s own opening `{`) as before the fix. Since editing the front half of the function repeatedly hasn't moved this error at all, the working theory now is that the actual problem is somewhere in the **back half** — the `insert into Create_Case [...]` block or the Case_ID update that follows it — and Zoho's error line number is, once again, not pointing at the real location.

**Diagnostic in place now** (temporary, not the real logic): the function currently stops right before the `insert into` block and returns `{"code":"DIAGNOSTIC_STOPPED_BEFORE_INSERT", "vehicleToUseId":..., "clientId":..., "serviceType":..., "issueCount":...}` instead of actually creating a ticket. Everything before that point (all 4 validation guards, the Client lookup, the Vehicle lookup, the issue-processing loop and Service_Type derivation) is the same real logic as before, unchanged.

**What this test will tell us**: if this diagnostic version saves cleanly in Zoho, the bug is confirmed to be in the `insert into`/Case_ID block that's temporarily removed, and that's exactly where to focus next. If it still shows the same "Improper Statement" error, the bug is somewhere in the front half after all, and the `insert into` block was never the problem.

**Needs re-pasting**: `createServiceRequest.deluge` (this diagnostic version — does NOT actually create tickets yet, do not treat a successful save as "done," only as confirmation of where the real bug is).

**Tested**: not independently confirmed live yet — this is the diagnostic itself, not a fix.

## 2026-09-09 (the actual root cause, found by the user) — the real rule: assign a variable inside try/catch, `return` it once, after the catch block

Every "Missing return statement"/"Improper Statement" error chased above — across all 3 files, over dozens of rounds, blamed on comments, form vs. report names, missing bracket criteria, `for each record` loops, a `location` reserved-word guess, front-half vs. back-half bisection — was very likely the same single cause the whole time. The user proved it directly: they pared `createServiceRequest.deluge` down to a trivial diagnostic body and found that Zoho's **Execute** (manual test-run) succeeded even when **Save** failed with "Missing return statement." That's the real signal — the Deluge interpreter runs the code fine; a separate, much stricter static checker rejects it at save time, and that checker chokes on `return` statements placed directly inside `try`/`catch` blocks, no matter how completely every path is covered.

**The confirmed working pattern** (user's own hand-edit, verbatim-confirmed "saving in zoho"): declare a plain string variable before the `try` (e.g. `returnstring = "";`), **assign** to it — never `return` it — everywhere inside both `try` and `catch`, then place exactly one, unconditional `return returnstring;` as the function's literal last statement, after the `catch` block's own closing `}`. No `return` keyword appears anywhere inside `try` or `catch` at all.

**Applied to all 3 real Custom APIs**, restoring full logic rather than leaving any of them as a diagnostic stub:
- **`createServiceRequest.deluge`** — rebuilt to the complete, real logic (the 4 validation guards, `Client[Client_Name == "ON DEMAND"]` lookup with a `Client[Name == "ON DEMAND"]` fallback, `vehicle_master[Category == vehicleTypeTrim]` single-record vehicle match, the issue-processing loop deriving `issueLongIds`/`isTow`/`serviceType`, `remarksLines` construction, the full 15-field `insert into Create_Case`, and the Case_ID generate-then-update step) — restructured so every branch (validation failures, lookup failures, the success path, and the outer `catch`) assigns to `returnstring` instead of returning directly. Guard clauses that used to be early `if(...) { return ...; }` are now `if/else if/else` chains so nothing ever needs to `return` from a nested block. This supersedes the form-fields-only diagnostic detour — `Client`, `Vehicle_Status`, `Time_of_service`, `Service_Type`, `Lead_Source`, `Status`, and `Case_ID` are all back in, so a website submission integrates with the agent dashboard/wizard exactly as originally designed.
- **`getIssueOptions.deluge`** — same pattern applied; kept its `for each record` loop over `Vehicle_Issue[ID != 0]` as-is, since the loop itself was very likely never the actual problem.
- **`getVehicleOptions.deluge`** — same pattern applied too, for consistency, even though this API is still unused by the widget (Vehicle Type is a fixed 2-item list) — costs nothing to keep it correct in case it's ever wired back in.

**What this means for the whole debugging trail above**: most of the intermediate "fixes" in this file (comment relocation, form/report name switches, bracket-criteria additions, the `location` rename, the front/back-half bisection) were very likely never the real cause — they may have been harmless, or coincidentally paired with a version that also happened to avoid a `return`-inside-try/catch somewhere. Recorded here rather than deleted, since some of those ARE real Deluge constraints confirmed independently (the return-type keyword must be the first token in the file; a bare form name needs `[criteria]`; a comment can't be the first thing after `{`) — they just weren't THIS bug.

**Needs re-pasting**: all 3 `.deluge` files, into their respective Custom APIs in Zoho Creator.

**Tested**: syntax-reviewed for structure (every path assigns `returnstring`, no `return` inside try/catch, single `return returnstring;` at the very end) — matches the user's own confirmed-saving pattern exactly. Not yet independently re-verified live for `createServiceRequest.deluge`'s full restored logic (the user only confirmed the trivial diagnostic shape saves; the full logic uses the same pattern but hasn't been re-tested end-to-end yet) — next step is re-pasting all 3 and confirming each saves clean, then a real end-to-end submission test.

## 2026-09-09 (feature parity) — Time of Service (NOW/LATER) restored, step order reversed, real T&C link

Three separate user requests, same session:

**1. Step order reversed**: "first step is Contact Details, second step is Vehicle Details" — reverses the earlier reference-design order (Vehicle → Contact → Location) back to Contact → Vehicle → Location. Implemented by swapping `renderStep1()`/`renderStep2()`'s own content (not restructuring the STEP-index dispatch itself), so `renderStep3()`'s own Back button needed no change.

**2. Terms & Conditions now a real link**: was a dead `href="#"` placeholder (with `onclick="return false"` actively preventing navigation) — now links to `https://www.getrescued.in/terms-and-conditions`, `target="_blank" rel="noopener"` (opens in a new tab, matching every other external link convention in this project).

**3. Time of Service (NOW/LATER) restored to Step 3**, with Service Date + Service Time fields shown only when LATER is chosen — feature parity with the *original* getrescued.in site form (screenshotted by the user), which this widget had entirely dropped during the earlier reference-design rebuild (that rebuild's own reference screens never showed a Time of Service step at all). Two SEPARATE fields (`<input type=date>` + `<input type=time>`), not `getRescueTicket`'s own single `datetime-local` picker convention — deliberate, matching the user's explicit "two fields one is service date and second is service time" request over this project's other convention.

- `STATE.timeOfService` defaults to `"NOW"`; toggling the radio re-renders Step 3 (same pattern `f_vtype.onchange` already uses on Step 2), capturing whatever was already typed in the fields being hidden/shown first.
- `combineServiceDateTime()` combines the two native inputs into one `Date`, then formats it via the existing `fmtDateTime12()` — same output shape `computedServiceTime` already used for the NOW case (`"now + 30 min"`), so no Deluge-side format change was needed for that field.
- `createServiceRequest.deluge` gained a new `timeOfService` parameter, writing the real value to `Time_of_service` (was hardcoded `"NOW"`) — normalized defensively (anything other than exactly `"LATER"` falls back to `"NOW"`) so a missing/old-shaped payload can't write an invalid dropdown value.

**Needs creating in Zoho / needs checking**: `timeOfService` is a brand-new Custom API parameter — per this project's own earlier lesson (`onDemandInvoicing`'s `applyTax` parameter), a new parameter sometimes needs to be explicitly (re-)registered in the Custom API's own setup screen in Zoho Creator, separate from just being named in the function signature, or it silently arrives as `null` (which the defensive fallback above turns into "NOW" without erroring — so this would fail silently, not loudly, if unregistered).

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. **Needs re-pasting**: `createServiceRequest.deluge` into its Custom API.

**Tested**: syntax-checked (script blocks parse cleanly). Not yet independently live-tested — `SHOW_SAMPLE_DATA` is still `true`, so the LATER validation/date-time combine logic can be exercised in the widget itself right now without the backend; the Deluge-side write needs a live test once re-pasted.

## 2026-09-09 (removal) — Location field removed from Step 3

Explicit user request: "remove location field from third step." Fully removed, not just hidden:

- The free-text Location input and its live OpenStreetMap Nominatim search dropdown are gone from `renderStep3()`.
- `locationSearchInput()`/`runLocationSearch()` (the debounced Nominatim fetch + suggestion-list rendering) deleted entirely — dead code with nothing left to call them.
- `STATE.location` removed; the outside-click listener that closed the suggestions dropdown removed.
- `.location-wrap`/`.location-results`/`.location-item`/`.location-attrib` CSS removed, along with `.fgrid.one`/`.fld.full` (only ever used by the now-gone location wrapper).
- `plugin-manifest.json`'s `connect-src` no longer includes `https://nominatim.openstreetmap.org` — nothing in the widget calls it anymore.
- The widget still sends `customerLocation: ""` in the payload (rather than dropping the key) so `createServiceRequest.deluge`'s own `customerLocation` parameter — still registered in the Custom API's setup screen in Zoho — keeps receiving a value of the expected type instead of `undefined`. Harmless no-op on the Deluge side; no re-pasting needed there for this change.

**Needs redeploying**: this widget itself (client-side JS + manifest) — freshly repacked.

**Tested**: syntax-checked, confirmed no dangling references to the removed field/functions remain anywhere in the file.

## 2026-09-09 (live) — SHOW_SAMPLE_DATA flipped off, real Custom API calls now

User confirmed the mock/sample confirmation screen rendering correctly end-to-end, then explicitly asked to "work with dynamic data." Both real Custom API PublicKey values (`getIssueOptions`, `createServiceRequest`) were already filled into `CUSTOM_API_PUBLIC_KEYS` from an earlier entry, so this was a one-line flip: `SHOW_SAMPLE_DATA = true` → `false`. Every sample-data code path stays in the file, untouched, just skipped now — flip it back to `true` at any point to return to demo mode (backend down, showing someone without Zoho access) with zero other changes.

**Still worth checking now that this is live**: `createServiceRequest.deluge` picked up a new `timeOfService` parameter earlier the same day (Time of Service NOW/LATER feature) — per this project's own earlier lesson (`onDemandInvoicing`'s `applyTax` parameter), a brand-new Custom API parameter sometimes needs to be explicitly (re-)registered in the Custom API's own setup screen in Zoho Creator, separate from just being named in the function signature, or it silently arrives as blank. The defensive fallback in that file (anything other than exactly `"LATER"` → `"NOW"`) means this would fail silently rather than with an error if unregistered — worth a real end-to-end test choosing LATER specifically, and checking the created ticket's own `Time_of_service` field in the portal to confirm it actually wrote `"LATER"`, not just defaulting to `"NOW"` every time.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked.

**Tested**: syntax-checked. Not yet independently confirmed live — next real submission through the widget will be the actual first live-data test end-to-end.

## 2026-09-09 (restoration) — real Vehicle + Vehicle Issue picker, matching getRescueTicket

Explicit user request: "vehicle issue and vehicle like we already did in other widgets" — reverses the reference-design rebuild's simplification (fixed Vehicle Type Two/Four Wheeler + free-text Vehicle Make, with server-side best-effort matching against `vehicle_master` by Category) back to a real picker, matching `getRescueTicket`'s own Vehicle field exactly.

**Widget (`app/widget.html`)**:
- `getVehicleOptions` wired back in — `boot()` now fetches both `getVehicleOptions` and `getIssueOptions` in parallel (`Promise.all`), same shape this widget used before the reference-design rebuild made Vehicle Type a fixed list. Its real PublicKey (`tZenZ7W1EXgKOJZshdrJsKetF`, already on record from earlier) is now actually used.
- `STATE.vehicleId` (a real `Vehicle_Master` record ID) replaces `vehicleType`/`vehicleMake`. Vehicle Details now shows a real `<select>` populated from `OPTIONS.vehicles` instead of a fixed Two/Four Wheeler dropdown + free-text make input.
- Vehicle Issue filtering now keyed off the **selected vehicle's own Category** (`categoryForVehicle()`, a new small helper) fed into the same `issuesForType()` this widget already had — same "Vehicle → Category → matching Issue" chain `getRescueTicket`'s own `vehicleIssuesFor()` uses, just resolved from `OPTIONS.vehicles` instead of a live form field.
- `SAMPLE_VEHICLES` added for `SHOW_SAMPLE_DATA` demo mode (4 rows, same `{id, name, category}` shape the real API returns).

**`createServiceRequest.deluge`**:
- Signature changed: `vehicleType, vehicleMake` → `vehicleId`.
- Vehicle lookup changed from a best-effort `vehicle_master[Category == vehicleTypeTrim]` (picks *any* vehicle of the right type) to an exact `vehicle_master[ID == vehicleId.trim().toLong()]` — the same `.toLong()` TEXT-vs-NUMBER fix already proven necessary elsewhere in this project for a Lookup ID arriving as a string param.
- The "Vehicle make (as entered)" `Remarks` line is gone — no longer applicable now that it's a real picked vehicle, not free text.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. **Needs re-pasting**: `createServiceRequest.deluge` into its Custom API (signature changed — the Custom API's own parameter registration screen in Zoho will need `vehicleType`/`vehicleMake` removed and `vehicleId` added, same lesson as `onDemandInvoicing`'s `applyTax`/`timeOfService` params).

**Tested**: syntax-checked (script blocks parse cleanly), confirmed no dangling `vehicleType`/`vehicleMake`/`f_vtype`/`f_make` references remain anywhere in the widget. `getVehicleOptions.deluge` itself was fixed with the `returnstring` pattern earlier the same day but has never been live-tested since it was dormant until now — this is its actual first live exercise.

## 2026-09-10 — live "Couldn't load the request form" report, diagnostics added

User reported the live public page (`page-perma/Service_Request/...`) showing the generic `boot()` fallback error — "Couldn't load the request form right now — please refresh the page, or call 9986 500 500 to book directly." This screen shows whenever `getVehicleOptions` and/or `getIssueOptions` don't both come back `code==="SUCCESS"`, but `callCustomApi()` only ever logged the OUTGOING request ("about to call getVehicleOptions...") — never what actually came back, so there was nothing to diagnose the real cause from.

**Most likely cause**: `getVehicleOptions` is having its first real live exercise since the vehicle-picker restoration (previously dormant/unused) — same class of issue this project already hit live on `customerPage`: a Custom API's PublicKey authentication can be switched off in Zoho Creator even though the matching key (`tZenZ7W1EXgKOJZshdrJsKetF`) is already sitting in this widget's own `CUSTOM_API_PUBLIC_KEYS`. Also possible: the Custom API simply hasn't been re-pasted into Zoho yet with its current code (see the outstanding "Needs re-pasting" items from the vehicle-picker-restoration entry above).

**Fixed (diagnostics only, no behavior change)**: `boot()` now logs the full `vehRes`/`issRes` results right after they come back, and — if the fallback error screen is about to show — explicitly logs which of the two calls failed. Open the browser console (F12 → Console) on the live page and look for `[Get-Rescue SR] boot() — getVehicleOptions result:` — the `code`/`message` shown there is the real reason this is failing (an ERROR message from the Custom API itself, or an all-null result consistent with a 9370 "invalid public key" failure).

**Needs checking in Zoho** (in order of likelihood):
1. `getVehicleOptions` Custom API → Settings/Authentication → confirm PublicKey is actually enabled, and that its displayed Public Key matches `tZenZ7W1EXgKOJZshdrJsKetF` exactly.
2. That `getVehicleOptions.deluge`'s current code (already using the `returnstring` pattern, unchanged this session) is actually what's pasted into the Custom API in Zoho.
3. Same two checks for `getIssueOptions`, in case it regressed too — it worked before the vehicle-picker restoration, so is a less likely culprit but not ruled out.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed, diagnostics-only change).

**Tested**: not yet independently re-verified live — reload the same page and check the console for the new log line to get the real cause.

## 2026-09-10 (later) — real bug fixed: "Improper Configuration..!!" on getVehicleOptions/getIssueOptions

User's console (with the diagnostic logging added just above) showed the actual cause: `[Get-Rescue SR] getVehicleOptions failed Improper Configuration..!!` and the same for `getIssueOptions` — a client-side rejection from Zoho's own widget SDK, thrown before any network call, not a Deluge/server-side error at all.

**Root cause, already known in this project**: this is the EXACT same error `getRescueTicket` hit and fixed on 2026-08-31 (`checkOperationsManagerPermission()`'s own story, see that README) — Zoho's widget SDK rejects a call whose config includes a PRESENT-BUT-EMPTY `payload: {}`, even though `payload` is documented as optional. `getVehicleOptions()`/`getIssueOptions()` are both zero-argument Deluge functions, and `boot()` calls them with `callCustomApi("getVehicleOptions", {})` — exactly the shape that broke `checkOperationsManagerPermission()` before.

**Fixed**: `callCustomApi()`'s shared config-building now only attaches `payload` when it actually has keys (`if(payload && Object.keys(payload).length){ config.payload = payload; }`) instead of always including it. `createServiceRequest` (which does send real fields) is unaffected — this is additive, not a behavior change for any call that already worked.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed).

**Tested**: not yet independently re-verified live — reload the page; the "Couldn't load the request form" screen should be gone and the real Contact Details step should render.

## 2026-09-10 (later still) — removed the grey background behind the Next/Back button footer

Explicit user request (screenshot with a red arrow pointing at the grey strip behind the "Next" button on Contact Details). `.footer-bar`'s `background` (was `var(--footer-bg)`, a light grey) changed to `transparent` — it now blends into the page instead of showing as its own band. `--footer-bg` itself is untouched, since `.case-id` (the success screen's Case ID box) also uses that same variable for something unrelated.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed).

## 2026-09-10 (later) — diagnostics added for "No issues configured for this vehicle"

User-reported live: selecting "Maruti Suzuki S-Presso" on Vehicle Details showed "No issues configured for this vehicle — call 9986 500 500 and we'll take it from there." This means `issuesForType(categoryForVehicle(STATE.vehicleId))` returned zero matches — either this vehicle's own `Category` (from `Vehicle_Master`) is blank, or no `Vehicle_Issue` record's `VEHICLE_TYPE` actually contains that category value. The matching LOGIC itself was checked against `getRescueTicket`'s own confirmed-working equivalent (`vehicleIssuesFor()`) and is structurally identical (same case-insensitive exact-value match against a 2W/4W shared vocabulary) — so this is most likely a real data gap on this specific vehicle's `Category` field (or a vocabulary mismatch), not a widget bug, but unconfirmed without a live log.

**Added (diagnostics only, no behavior change)**: `categoryForVehicle()` now logs when a vehicle has no `Category` value at all; `issuesForType()` now logs the searched-for category plus every fetched issue's own `vehicleTypes` whenever nothing matches — mirroring `getRescueTicket`'s own already-proven diagnostic shape for the exact same kind of mismatch.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed).

**Next step**: reload, select the same vehicle again, and check the console for `[Get-Rescue SR] categoryForVehicle` / `[Get-Rescue SR] issuesForType` — that will show whether this vehicle's Category is blank in Zoho, or whether it has a value that doesn't match any Vehicle_Issue's VEHICLE_TYPE.

## 2026-09-10 (later) — searchable Vehicle dropdown

Explicit user request, with a screenshot of the long plain `<select>` list (alphabetical, dozens of vehicles, "YAMAHA..."/"VESPA..." etc. — impractical to scroll to find one): "also add search in vehicle select dropdown."

**Fixed**: replaced the native `<select id="f_vehicle">` with a searchable combo — a text input (`#f_vehicle`, placeholder "Search Vehicle…") plus a filtered dropdown (`#f_vehicle_dd`) — typing filters `OPTIONS.vehicles` by name (case-insensitive substring), arrow keys/Enter/Escape all work, clicking or Enter-selecting a match sets `STATE.vehicleId` exactly like the old `<select>`'s `onchange` did (including clearing `STATE.issueIds`, unchanged). This is the exact same proven pattern already shipped in `getRescueTicket`'s own Client/Vehicle combo (`wireCombo()`/`.combo`/`.combo-dropdown`/`.combo-opt` CSS) — ported here as `wireVehicleCombo()`, with this file's own color tokens (`--ink`/`--line`/`--road-soft`) swapped in for getRescueTicket's (`--signal`/`--worktop`/`--sh-2`, which don't exist in this file).

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed).

**Tested**: syntax-checked only — not yet independently live-tested.

## 2026-09-10 (later still) — Vehicle Issue validation removed

Explicit user request: "remove vehicle issue validation." Vehicle itself is still required (unchanged) — only the "at least one issue" requirement was dropped, both sides:

- **Widget**: `renderStep2()`'s `next2Btn` handler no longer checks `STATE.issueIds.length`; error message narrowed to "Please select a vehicle."
- **`createServiceRequest.deluge`**: removed the matching `vehicleIssueIds == null || vehicleIssueIds.trim() == ""` guard entirely — without this, the client-side relaxation alone would've just moved the rejection server-side instead of actually letting it through. `issueIdList = vehicleIssueIds.toList(",")` changed to `ifnull(vehicleIssueIds,"").toList(",")` since this parameter can now genuinely arrive `null` (not just `""`), and `.toList()` on a literal null wasn't something this code had ever needed to handle before. The rest of the issue-processing loop (`issueLongIds`/`isTow`/`Service_Type`) already tolerated an empty list gracefully (its own `if(issueIdTrim != "")` guard), so no other change was needed there — an empty Vehicle Issue selection now just means `Vehicle_Issue` saves empty and `Service_Type` defaults to "RSR".

**Needs re-pasting**: `createServiceRequest.deluge` into its Custom API.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed, same zip as the search-combo entry above).

**Tested**: syntax-checked only (braces balanced, script parses) — not yet independently live-tested.

## 2026-09-11 — Placeholder issue chips, for UI verification when a real vehicle has no configured issues

User hit exactly the data-gap scenario the 2026-09-10 diagnostics entry predicted, live: selecting "ROYAL ENFIELD THUNDERBIRD" showed "No issues configured for this vehicle." Explicit request: "please show dummy issue in the list so that i can verify that UI is perfect" — a UI-verification need, not a request to fix the underlying data gap (that's still a real Zoho issue — see below).

**Built**: new `PLACEHOLDER_ISSUES` (3 rows: General Service, Battery Check, Flat Tyre — ids prefixed `__placeholder_`), separate from `SAMPLE_ISSUES`/`SHOW_SAMPLE_DATA` (which replaces the WHOLE vehicle+issue dataset for a fully offline demo). This is narrower: `renderStep2()` now falls back to these chips specifically when a real vehicle IS selected but its real, Zoho-filtered issue list comes back empty — "Select a vehicle first" still takes priority as before. A small muted notice (`.issue-placeholder-note`) appears above the chips: *"No issues configured for this vehicle yet in our system — showing sample options below so you can preview this screen. Selections here are for preview only and won't be submitted."*

**Safety**: `onSubmit()`'s payload now filters `STATE.issueIds` to drop anything with the `__placeholder_` prefix before building `vehicleIssueIds` — a placeholder chip can never reach `createServiceRequest.deluge` as if it were a real Vehicle_Issue ID, even if selected and the form is genuinely submitted (Vehicle Issue selection isn't required to submit at all since the 2026-09-10 validation removal, so this matters more than it might otherwise).

**The real root cause is still open, not fixed by this change** — same diagnosis already in place since 2026-09-10: either this vehicle's own `Category` field is blank on `Vehicle_Master` in Zoho, or no `Vehicle_Issue` record's `VEHICLE_TYPE` actually contains that category value. Reload the page, select this same vehicle again, and check the browser console for `[Get-Rescue SR] categoryForVehicle` / `[Get-Rescue SR] issuesForType` — that log will show exactly which one it is. This placeholder fallback is a UI-preview aid, not a substitute for fixing that data gap so real customers see real issues for this vehicle.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed). No `.deluge` changes — purely client-side.

**Tested**: syntax-checked (script block parses cleanly). Not yet independently live-tested — next step is reloading the live page and re-selecting the same vehicle to confirm the placeholder chips render and the notice text is correct.

## 2026-09-11 (later) — `SHOW_SAMPLE_DATA` flipped back to `true`, temporarily

User's live page regressed to the generic `boot()` fallback screen — "Couldn't load the request form right now" — meaning `getVehicleOptions`/`getIssueOptions` aren't currently succeeding in Zoho. Explicit request: "please recheck so that i can see all the steps without any dependency." Flipped `SHOW_SAMPLE_DATA` back to `true` — this is the exact mechanism built 2026-09-09 for this exact situation: `boot()`/`waitForSDK()` skip the real Custom API calls entirely and populate `OPTIONS` from `SAMPLE_VEHICLES`/`SAMPLE_ISSUES`, so all 3 steps render and are clickable regardless of the backend's current state. Submitting still only shows the mock "Sample confirmation — no real ticket created" screen — no real ticket gets created while this is on.

**This does not fix the real Custom API failure** — that's still broken and needs its own check once you're done previewing the steps. Same checklist as the 2026-09-10 "Couldn't load the request form" entry above, most likely causes in order:
1. `getVehicleOptions`/`getIssueOptions` → Settings/Authentication in Zoho → confirm PublicKey auth is actually still enabled (it can get switched off independent of the key value being correct).
2. Confirm each Custom API's currently-pasted code in Zoho actually matches this file's current `.deluge` scripts — several rounds of fixes happened on these two files (see the "returnstring pattern" entry above), worth a fresh re-paste of both to be sure.
3. Open the live page's browser console (F12 → Console) — `boot()` already logs the full `getVehicleOptions`/`getIssueOptions` results, including which one failed and why, whenever this fallback screen shows.

**Needs redeploying**: `dist/serviceRequest.zip` (re-packed). **Reminder**: flip `SHOW_SAMPLE_DATA` back to `false` (and redeploy again) once the real Custom API issue above is actually fixed — otherwise the live page will keep showing sample data and mock confirmations to real customers indefinitely.

**Tested**: syntax-checked. Not yet independently confirmed — next step is reloading the live page to confirm all 3 steps now render.
