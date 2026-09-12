# Get-Rescue — Call Center (`callCenter`) — Working Log

A standalone Zoho Creator widget for placing voice-callback calls and checking their status, via two Custom APIs the user built directly in Zoho (Ozonetel-backed). Split out of `getRescueTicket` on 2026-09-10 — it briefly lived inline there first (see `getRescueTicket/README.md`'s own 2026-09-10 entries), then was moved here per explicit user decision ("create separate widget for these function and detailing" → "Replace the inline button" when asked whether to keep both).

Not embedded on any ticket. A free-standing page — the agent types in a phone number (and, optionally, a reference/Case ID) directly, rather than this widget reading a `Create_Case` record.

See also **`../ACCESS.md`** (repo root) — this widget's own section there lists exactly what it needs.

---

## Custom APIs this widget needs (already built by the user, not by this session)

Two confirmed-live URLs, given directly:
```
https://www.zohoapis.in/creator/custom/getrescued/getVoiceCallbackDetails
https://www.zohoapis.in/creator/custom/getrescued/getCallbackStatus
```
`api_name` in `app/widget.html` matches each URL's own last path segment; `workspace_name:"getrescued"` matches the URL's own path segment. Both are called with `http_method:"POST"` — confirmed elsewhere in this project (`getRescueTicket`'s `checkOperationsManagerPermission()` comment, 2026-09-01) that Custom APIs in this Zoho account only ever accept POST, including for what's conceptually a read-only status check.

**`getVoiceCallbackDetails(customerId, phoneNumber, callbackMessage)`** — places the call. Expected response: `{code:"SUCCESS", ucid:"...", ...}` on success. `customerId` is sent as whatever the agent typed into the Reference / Case ID box (falls back to the phone number itself if left blank) — this widget has no ticket context to pull a real Case_ID from, unlike `getRescueTicket`'s own earlier (now-reverted) use of this same API.

**`getCallbackStatus(callbackId)`** — checks a call's status. Expected response: `{code:"SUCCESS", status:"...", call_duration:"...", ...}`.

**⚠ Response-shape not independently confirmed.** During the work that led here, two draft versions of `getCallbackStatus` were shown, with different field names for the call's outcome — one returned `call_result`, the other `disconnect_reason`. The user then said they'd created their own finished Custom API without confirming which shape it actually returns. `checkVoiceCallbackStatus()` in `app/widget.html` reads **both** defensively (`data.call_result || data.disconnect_reason`) rather than guessing. If the on-screen "Outcome" column/field ever shows blank while Status/Duration populate fine, that field-name mismatch is the first thing to check — open this Custom API's own execution log in Zoho and see what it actually named that field.

---

## Zoho Form/Report needed for call history (does NOT exist yet)

Placing a call and checking its status both work with **zero** Zoho schema changes — those two Custom APIs already exist. **Call history is the one feature that needs something new**: a place to persist each call so it survives a page refresh and is visible to every agent, not just whoever placed it.

This session cannot create Zoho Forms/Reports directly — create these in Zoho Studio before call history will work:

**Form: `Call_Log`** (Link Name must match exactly — this widget references it by that name)

| Field (Link Name) | Type | Notes |
|---|---|---|
| `Phone_Number` | Single Line Text | The number called (bare 10 digits, no country code — matches this widget's own `normalizePhone()`) |
| `Reference_ID` | Single Line Text | Whatever the agent typed (often a ticket's Case_ID, but free text — no validation) |
| `Callback_Message` | Single Line Text | Optional message sent with the call request |
| `UCID` | Single Line Text | The call ID `getVoiceCallbackDetails` returns — used to re-check status later |
| `Call_Status` | Single Line Text | Latest known status (updated in place on every re-check, not a new row) |
| `Call_Duration` | Single Line Text | Stored as text deliberately — avoids guessing whether Ozonetel's duration is seconds or `HH:MM:SS` |
| `Call_Outcome` | Single Line Text | `call_result`/`disconnect_reason`, whichever the real API returns (see the ambiguity note above) |
| `Placed_By` | Single Line Text | The logged-in agent's email (`getInitParams().loginUser`) |

`Added_Time` is automatic on every Zoho form — used for the history table's "Placed At" column, nothing extra needed for that.

**Report: `Call_Log_Report`** — expose every `Call_Log` record (same "one form, one all-records report" pattern every other widget in this project already uses, e.g. `Create_Case`/`Agent_Ticket_Report`).

**Until this form/report exists**: the widget degrades honestly — an on-screen banner reads *"Call history isn't set up in Zoho yet... Calls can still be placed and their status checked — nothing here is broken."* Placing/checking a call never blocks on this; only the history list and the failed-attempt audit log do. `apiGetCallLogs()` returns `null` (distinct from `[]`, a genuinely empty-but-existing report) on any read failure, which is what triggers the banner.

---

## Decisions confirmed before building (asked, not guessed)

Three explicit questions asked via `AskUserQuestion` before this widget was built:

1. **Relationship to `getRescueTicket`'s inline button**: *"Replace the inline button"* — not "keep both". The earlier inline "Call via System" button + status line added to `Phone_Number1`/`Alternate_Phone_Number` there was fully reverted (see `getRescueTicket/README.md`'s matching 2026-09-10 entry) once this widget existed to replace it.
2. **Context source**: *"Standalone — agent enters phone number manually"* — not embedded on a ticket, no automatic Case_ID/phone pull.
3. **Detail level**: user selected **all three** offered options — full response detail, call history across calls, and a clearer/bigger presentation than the old inline version. All three are built: the "Current Call" panel shows every field both APIs return (including a collapsible raw-JSON block), the "Call History" table lists every past call with a per-row refresh, and the whole page is a dedicated, uncluttered layout instead of a compact inline status line.

Earlier, separately, the exact API contract was also confirmed rather than guessed: `customerId` = `Case_ID` (asked via `AskUserQuestion`, 2026-09-10, before this widget existed — inherited here as "whatever reference the agent typed", the closest equivalent without ticket context), and the two exact live Custom API URLs (given directly by the user, not assumed).

---

## What's real vs. what's a known gap

**Real and working** (pending a live test — not yet confirmed by the user):
- Placing a call via `getVoiceCallbackDetails`, showing the returned UCID.
- Checking a call's status via `getCallbackStatus`, both automatically (4s after placing) and manually (Refresh button).
- Full detail display (every response field, plus raw JSON) for the current call.
- Graceful degradation when `Call_Log` doesn't exist — clear on-screen banner, nothing crashes.

**Known gaps**:
- **Call history won't actually persist until `Call_Log`/`Call_Log_Report` are created in Zoho** (see above) — this is the one piece that needs Zoho-side setup, not more code.
- **`Call_Outcome` may show blank** if the real `getCallbackStatus` Custom API uses a field name other than `call_result`/`disconnect_reason` — see the response-shape note above.
- No pagination on the history table (`max_records:200`) — fine for now, would need addressing if call volume grows well past that.
- No phone-number format validation beyond "at least 10 digits" — a malformed number is sent straight to `getVoiceCallbackDetails`, which will presumably reject it on the Ozonetel side (not independently confirmed).

---

## Tested

`node -e "new Function(...)"` syntax check on the extracted `<script>` block passed. `zet pack` run, `dist/callCenter.zip` produced. **Not yet live-tested** — this widget has never been opened inside Zoho Creator. Before relying on it:
1. Create `Call_Log`/`Call_Log_Report` in Zoho Studio (exact fields above) and grant this widget's profile Add/View+Edit access (see `../ACCESS.md`).
2. Register this widget in Zoho Creator (upload/connect via `zet`, same as every other widget in this repo).
3. Place a real test call and confirm it actually reaches a phone, a UCID comes back, and the status check returns real (non-blank) data — pay particular attention to the Outcome field given the schema ambiguity flagged above.

---

## Running locally

Same as every other widget in this repo: `npm install && npm start` inside this folder, then open the HTTPS URL it prints (self-signed cert — click through the browser's "unsafe" warning). `zet pack` produces `dist/callCenter.zip` for uploading into Zoho Creator.

---

## 2026-09-10 — Widget created

Full writeup above. Scaffolded from `agentPerformance`'s minimal structure (smallest existing widget in this repo — `plugin-manifest.json` with no `cspDomains`, since this widget makes no direct external-domain calls, only `invokeCustomApi`/`getRecords`/`addRecords`/`updateRecordById` against Zoho itself). `placeVoiceCallback()`/`checkVoiceCallbackStatus()` ported over near-verbatim from their brief stay in `getRescueTicket` (only the `customerId` source changed, from `caseLabel(record)` to the agent's free-typed Reference field, since there's no ticket record here to read a Case_ID from).
