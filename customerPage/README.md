# Get-Rescue — Customer Page (`customerPage`) — Working Log

Standalone Zoho Creator widget, `app/widget.html`, for the **public, no-login "Customer" page** — where customers land after clicking a WhatsApp link for one of four requests: **Booking Payment**, **Remaining Payment**, **Share Location**, **Give Feedback**.

## Why this is architecturally different from every other widget in this repo

Every other widget (`getRescueTicket`, `vendorTicket`, `technicianTicket`, `driverTicket`, `opsMap`) runs behind an authenticated Zoho Portal login. A customer clicking a WhatsApp link has **no login at all** — this widget has to work for a completely anonymous visitor.

**A real security finding drove the whole design**, investigated before writing any code: the "obvious" mechanism for public-page data access (`ZOHO.CREATOR.PUBLISH.getRecordById`) requires **publishing the entire report** and sharing one `private_link` token across every customer. That token isn't scoped per-record — anyone holding it (i.e. every customer who ever received a link) could fetch **any other customer's record** just by guessing a different record ID, since Zoho's own official docs confirm access is scoped to the *published report*, not the individual record. Not acceptable for `Create_Case`, which holds every customer's name, phone, and financial data.

**Fix**: this widget never talks to `Create_Case`/`Agent_Ticket_Report` directly — no `ZOHO.CREATOR.DATA`/`.PUBLISH` calls at all. Everything goes through three narrow, purpose-built Custom APIs (server-side Deluge, same `invokeCustomApi()` mechanism already proven for `onDemandInvoicing`/`checkOperationsManagerPermission` elsewhere in this project) that accept a record ID and return **only** the minimal fields needed for the one thing being shown — the client never gets direct query access to the report.

## URL contract

```
<published page URL>?recordId=<Create_Case record ID>&mode=booking_payment|remaining_payment|location|feedback
```
Plus `&locationType=breakdown|drop` for `mode=location` — set by whichever agent-side button generated the link (customer never chooses this).

**Debug mode**: append `&debug=1` to bypass the styled per-mode card and instead dump whatever `getCustomerTicketInfo` actually returned as plain field:value rows — added 2026-09-02 to verify the data pipeline itself while wiring this up live, separate from checking the UI. Opt-in only, doesn't touch default behavior. Previewable in `demo.html` via its own "Debug — raw fields" nav link.

## ⚠ TEMPORARY — design-preview fallback (added 2026-09-02, MUST be reverted before going live)

`boot()` currently falls back to `renderDesignPreview()` — a sample-data view with a mode-switcher nav bar, clearly labeled "⚠ DESIGN PREVIEW" — whenever `recordId`/`mode` are missing, instead of the real "This link looks incomplete" error. This was requested live, while repeated attempts to reach the page with real query params kept showing "incomplete" and it wasn't yet clear why — this fallback lets the card *design* be checked directly on the real published page without depending on that being resolved first.

**This is not safe to ship to real customers as-is**: a genuinely broken/incomplete link should show the real error, not silently render fake data. Before this page goes live:
1. Delete `renderDesignPreview()` in `app/widget.html`.
2. In `boot()`, delete the `renderDesignPreview();` call and its comment, and un-comment the `renderError("This link looks incomplete...")` line directly above it.
3. Remove the `<div id="previewNav"></div>` line from the HTML body (cosmetic only, safe to leave, but it's dead weight without the JS that fills it).
4. Re-run `test_customer_page_widget.js` — the two tests marked "TEMPORARY" plus the safety-net test that checks the real error string still exists in source will need re-reading at that point (the safety-net test will still pass either way, since it only checks the string exists somewhere in the file).
5. `zet pack`.

The underlying question this was standing in for — **why real query params weren't reaching the widget on the live public page** — is still open and needs to be resolved separately (see the `getPageParams()` section above and the "Still needed before this works live" list below).

**Updated 2026-09-07**: the preview card's own `<h1>Hi <name></h1>` header now carries a small "SAMPLE" badge whenever it's rendering `renderDesignPreview()`'s mock data (a new `IS_PREVIEW` flag, set only inside that function). Requested so the preview can never be mistaken for an actual created customer — even from a cropped screenshot that doesn't include the "⚠ DESIGN PREVIEW" nav banner above the card. All 4 preview modes still walk the same single sample persona ("Priya Sharma") through the ticket lifecycle exactly as before — only the header rendering changed, via a new shared `customerHeaderHtml()` helper used by all 5 real+preview header call sites. The real (non-preview) customer flow is byte-for-byte unchanged: `IS_PREVIEW` defaults to `false` and `boot()`'s live-data path never touches it. Covered by a new 11-check Node-VM test (`test_customer_preview_badge.js`).

Zoho's own long, non-sequential record ID format (e.g. `448881000000450014`) is reasonably resistant to guessing in practice, but this is not a cryptographic guarantee — flagged honestly, not oversold.

**Real bug found and fixed 2026-09-02, before this went live**: the widget originally read these params via plain `window.location.search` (`URLSearchParams`). Zoho's own docs confirm an in-app page's parameter URL puts the query string *after* the page's `#hash` — `https://<domain>/<owner>/<appLinkName>/#<pageLinkName>?recordId=...&mode=...` — which `window.location.search` never sees at all (that only covers the part *before* `#`). Fixed by reading params through the widget SDK's own `ZOHO.CREATOR.UTIL.getQueryParams()` instead (`getPageParams()` in `app/widget.html`), with `window.location.search` kept only as a fallback for `demo.html`/standalone preview where there's no Zoho SDK to ask. Covered by 2 new tests in `test_customer_page_widget.js`.

**Resolved 2026-09-02, confirmed against a real published link**: the public permalink (Settings → Publish) looks like `https://creatorapp.zohopublic.in/<owner>/<appLinkName>/page-perma/Customer_Page/<long token>` — a **plain path + token, no `#hash` at all**, unlike the in-app builder URL. So `?recordId=...&mode=...` appended to the end lands as a completely normal query string, exactly where `window.location.search` looks — meaning `getPageParams()`'s fallback branch (not the `getQueryParams()` SDK branch) is what actually fires on the real public page. Confirmed live, not assumed.

## Custom APIs this widget needs (must be created in Zoho before it will work)

- **`getCustomerTicketInfo(string recordId, string mode)`** — returns customer name + whatever the requested mode needs (amount due + paid status + invoice URL for the two payment modes; whether a location/feedback was already submitted for the other two). `custom-apis/getCustomerTicketInfo.deluge`.
- **`submitCustomerLocation(string recordId, string locationType, string lat, string lon)`** — writes `Latitude`/`Longitude` (breakdown) or `DropLocationLat`/`DropLocationLong` (drop). `custom-apis/submitCustomerLocation.deluge`.
- **`submitCustomerFeedback(string recordId, string score, string feedbackText)`** — writes `Cx_Feedback_Score`/`Cx_Feedback_Text` (these fields already existed, previously filled only by an external process — this is the first real in-app mechanism to actually populate them). `custom-apis/submitCustomerFeedback.deluge`.

All three: whole body wrapped in one try/catch, real fetch by record ID (`Create_Case[ID == recordId.toLong()]`), no ternary operators (confirmed unsupported in Deluge earlier this session), response built via a Deluge `Map().toString()` (confirmed to serialize as valid JSON, same technique already proven in `onDemandInvoicing.deluge`).

## Decisions confirmed before building (asked, not guessed)

1. **Access model**: a public, no-login Zoho Creator page (not a Portal login) — confirmed.
2. **Payment mechanism**: the widget shows the amount due + a "Pay Now" button linking out to the *existing* Zoho Books invoice — it does **not** embed a payment gateway itself. That's the separate, larger, explicitly-excluded Zoho Payments integration.
3. **Location capture**: a live browser GPS picker (`navigator.geolocation`) with a Share button — not a text/address box.

## What's real vs. what's a known gap

- **Booking Payment**: real invoice URL, reconstructed server-side from `Zoho_Books_Invoice_ID` (already written back onto the ticket by the existing `onDemandInvoicing` flow) — a genuine, working "Pay Now" link.
- **Remaining Payment**: shows the amount due, but **no payment link** — the existing remaining-fee flow (`Send_Payment_Link1`) never creates a Zoho Books invoice at all, so there's nothing to link to yet. The widget is honest about this (shows a plain note instead of a broken/missing button) rather than pretending a link exists. Building a real invoice for this flow is separate scope, not attempted here.
- **Location** and **Feedback**: fully real — write straight to the same fields the agent-facing app already reads/displays.

## Wiring into the existing agent app (`getRescueTicket`)

- `Send_Payment_Link` (booking fee) and `Send_Payment_Link1` (remaining fee): now fold this widget's own link into the *existing* WhatsApp message (in place of/alongside the raw Books URL) — backward compatible, falls back to exactly today's behavior if `CUSTOMER_PAGE_BASE_URL` isn't configured yet.
- `Send_Link_For_The_Breakdown_Location` (both copies): now includes the customer-page link in the *same single* WhatsApp template parameter that already just held the customer's name — deliberately does **not** add a second parameter, since a param-count mismatch against a real WhatsApp Business template is a real failure mode this project already hit once (`Send_Payment_Link`'s own `(#132018)` error history).
- `Send_Link_For_The_Drop_Location`: **real bug fixed as a side effect** — this button had no `action` at all before this change (clicking it did nothing beyond flipping its own checkbox field). Now wired the same safe way as the breakdown-location button.
- New `Customer_Feedback_Requested` checkbox on the Final Closure screen (mirrors the existing `Google_Feedback_Requested` pattern exactly) — sends a new `customer_feedback_link` WhatsApp template.

## Still needed before this works live (Zoho-side setup, not code)

1. Create the 3 Custom APIs above in Zoho, paste in the `.deluge` scripts.
2. Create a public Page in Zoho Creator, upload `dist/customerPage.zip` as its widget, publish it.
3. Set `CUSTOMER_PAGE_BASE_URL` in `getRescueTicket/app/widget.html` to the real published page URL.
4. WhatsApp Business template names used here (`location`, `booking_fees_link`, `customer_feedback_link`) are — same confidence level as every other template name in this app — not confirmed against the real approved template text. `customer_feedback_link` is genuinely new and needs approval.
5. Grant whatever permission the public page's own visitors need to invoke these 3 Custom APIs (Custom APIs typically run with the app's own service-level permissions regardless of visitor login state, but confirm this live).

## Tested

Node-VM suite (`test_customer_page_widget.js`, 33 checks) extracting the real shipped functions from both this widget and `getRescueTicket`'s new link-building helpers: URL construction (configured/unconfigured), the link-folding-into-one-parameter technique (no newlines, correct fallback), `getPageParams()`'s own SDK-vs-fallback paths, all four render modes (paid/unpaid, already-shared/not-yet, already-submitted/not-yet), the full geolocation-permission-denied path, debug-mode's raw field dump, and `boot()`'s error handling (missing URL params, a real ERROR response, all 4 happy paths). Full existing `getRescueTicket` regression suite (102 checks across 5 files) re-run — no interaction. Both widgets syntax-checked and packed.

**Not tested**: anything requiring a live browser/Zoho environment — the actual public page rendering, the 3 Custom APIs' real Deluge execution, and the WhatsApp sends with the new link text. Same caveat every other Deluge script in this project carries.

## 2026-09-07 — `getCustomerTicketInfo.deluge`: Remaining Payment mode now returns a real invoice link too

User decision ("split into two invoices," see `getRescueTicket/README.md`'s matching entry) resolved a confirmed mismatch where the Booking Payment mode's own "Pay Now" link pointed at an invoice for the ticket's FULL fee, not the Booking Fee actually shown as due. As part of that fix, `getRescueTicket` now also creates a genuine, separate Zoho Books invoice for the Remaining Fee (previously: none existed at all for this mode). `remaining_payment` mode here now reads a new field, `Zoho_Books_Invoice_ID_Remaining` (needs creating on `Create_Case` — Single Line Text, same type as the existing `Zoho_Books_Invoice_ID`), and returns `invoiceUrl` the exact same way `booking_payment` already does. **No widget-side change was needed** — `renderBookingOrRemainingPayment()` already handles the `invoiceUrl`-present-vs-absent cases generically for both modes.

**Needs deploying**: the updated Deluge script content needs pasting into this Custom API in Zoho Creator (same manual step every Deluge change in this project requires) before this takes effect live.

**Tested** — covered by `getRescueTicket`'s own `test_split_invoice_2026_09_07.js`, which includes checks directly against this file's own source confirming the new field read and the `invoiceUrl` construction mirror `booking_payment`'s pattern exactly.

## 2026-09-07 (later same day) — Real Deluge type error fixed: "left expression is of type DECIMAL and right expression is of type TEXT and the operator != is not valid"

User hit this live immediately after the fix above. Root cause: `Zoho_Books_Invoice_ID` (and now `Zoho_Books_Invoice_ID_Remaining`) read back as **DECIMAL** in Deluge, not the Single Line Text this project's own documentation assumed — comparing that value directly against the TEXT literal `""` (`invoiceId != ""`) is exactly what Deluge's type checker rejects outright. Same anti-pattern (`if(X != null && X != "")`, combining a null-check and a type-sensitive comparison into one expression) turned out to be pre-existing in **three other places in this same file** — `Cx_Feedback_Score` (feedback mode) and, confirmed genuinely broken since `Latitude`/`DropLocationLat` really are Decimal fields, both checks in `location` mode. All five fixed uniformly: split into nested `if` blocks (matching the already-proven-safe pattern this project's other Deluge scripts — `onDemandInvoicing.deluge`, `recordVendorPayment.deluge` — already use instead of combining checks with `&&`), with the value normalized via `.toString()` before the empty-string comparison so it works regardless of whether the underlying field is Decimal, Number, or genuinely Text.

**Tested** — extended `test_split_invoice_2026_09_07.js` to 26 checks: confirms the risky combined-expression pattern is gone from actual code (not just from comments describing the fix — an early version of this exact check false-positived on its own explanatory text, caught and corrected), and that all five affected checks (2 invoice IDs, feedback score, 2 location fields) now normalize via `.toString()` before comparing. Full suite re-run — no regressions. Needs redeploying into the Custom API in Zoho Creator, same as every Deluge change in this project.

## 2026-09-07 (later same day) — Real Deluge compile error fixed: "Missing return statement: Provide STRING expression to return"

User hit this immediately after trying to save the type-error fix above. This function's original shape used ONE shared `return result.toString();` placed after the entire `if/else if/else` mode-dispatch chain, relying on every branch falling through to it — Deluge's own static return-path checker couldn't prove every branch actually reached that point (the nested `if` blocks the DECIMAL/TEXT fix just above introduced made this harder for it to follow) and refused to save the script at all.

**Fixed the only fully robust way**: every branch (`booking_payment`, `remaining_payment`, `feedback`, `location`, and the `else`/unknown-mode fallback) now ends with its own explicit `return result.toString();` — there's no shared fall-through return left anywhere for Deluge to need to prove reachability for. Purely structural; no field, formula, or output-key logic changed at all.

**Tested** — extended `test_split_invoice_2026_09_07.js` to 29 checks: confirms exactly 7 explicit returns exist in the function (not-found guard, 4 modes, unknown-mode fallback, catch block), and that the booking_payment branch's own closing brace is immediately preceded by its own return rather than falling through into the next branch. Full suite re-run — no regressions. Still needs redeploying into the Custom API in Zoho Creator before either fix takes effect live.

## 2026-09-07 (later still) — 4 new "first thoughts" requirements added

User provided a short list of initial requirements to add. Implemented all 4, each flagged where genuinely ambiguous rather than guessed silently.

**1. Drop Location was missing from Design Preview.** The real URL contract already fully supports it (`LOCATION_TYPE=drop`, written by `getRescueTicket`'s own `Send_Link_For_The_Drop_Location` button) — the gap was specifically that `renderDesignPreview()`'s own mode-switcher nav only ever exercised the breakdown variant, with no way to preview what a drop-location request looks like. Fixed by splitting the preview nav's "Location" button into two — "Location (Breakdown)" and "Location (Drop)" — both still map to the same real `MODE="location"`, differing only in `LOCATION_TYPE`, exactly mirroring the real 4-mode/2-location-type contract rather than inventing a 5th mode.

**2. Vehicle/Issue/Total Service Fee context, "at the beginning."** New shared `ticketContextHtml()` helper, shown right under the "Hi `<name>`" heading on **all 4 modes** (not just one) — interpreted "at the beginning" as "whichever page the customer first lands on," since a customer might only ever receive one of the 4 link types. `getCustomerTicketInfo.deluge` now returns `vehicle`/`issue`/`totalServiceFee` in its shared (pre-mode-dispatch) section. **Flagged, not guessed silently**: `Vehicle`/`Vehicle_Issue` are Lookup fields, and this project has no prior confirmed pattern for resolving a Lookup's display name inside a Deluge script (every other place that needs Vehicle Issue names resolves them client-side via a separate options-report fetch, not inside Deluge) — used the safest possible `.toString()` fallback (cannot itself throw), but it may show a raw ID rather than a friendly name until confirmed live. Test this specifically and report back exactly what displays so the field-access syntax can be corrected if needed.

**3. T&C checkbox, "at one place... for customer to submit."** Placed on **Booking Payment specifically** — the customer's first payment interaction, the most standard place to gate on terms acceptance; by Remaining Payment they've already agreed once. **Flagged as an interpretation, not a literal instruction** — the request didn't specify which page; straightforward to move or duplicate elsewhere if a different page was actually meant. Pay Now renders visually disabled (dimmed, non-clickable) until the checkbox is ticked; Remaining Payment's own Pay Now is completely unaffected — same eligible-for-a-real-invoice logic either way, just no gating on that page. T&C link text/URL is a placeholder (`#`) — no real Terms page exists in this project yet.

**4. Feedback split into 3 separate 1-10 scores** (Service Provider / Call Agent / Overall Experience) — was one shared score. Implemented, then **reverted later the same day** — see the changelog entry directly below. Feedback stays as the original single `Cx_Feedback_Score`.

**Needs redeploying**: `getCustomerTicketInfo.deluge` changed (point 2's shared vehicle/issue/fee section) — paste into its Custom API in Zoho Creator. `submitCustomerFeedback.deluge`'s own logic reverted back to single-score, but see the correction below — it needed a further real fix before it would actually save in Zoho.

**Tested** — new `test_customer_page_additions_2026_09_07.js`, a real execution test using a minimal hand-rolled DOM (no jsdom in this environment): the context strip's exact output for full/partial/empty info, the Design Preview's Drop Location variant correctly setting `MODE`/`LOCATION_TYPE`, and the T&C checkbox's initial gated state and its live un-gating on check (booking payment) versus its total absence (remaining payment, both with and without an invoice). Extended `test_customer_page_widget.js` (the original, broader suite) to inject the new shared helpers (`customerHeaderHtml`/`ticketContextHtml`/`tncCheckboxHtml`) that `renderBookingOrRemainingPayment`/`renderLocation`/`renderFeedback` now depend on, and fixed one stale assertion (the Design Preview nav-order check referenced the old single `location` button key, now split into `location_breakdown`/`location_drop`). Also re-ran and fixed one stale assertion in `test_split_invoice_2026_09_07.js` (checked for a variable name that no longer exists by design). See the revert entry below for feedback-specific test changes and final counts.

## 2026-09-07 (later still) — Feedback reverted back to a single score

Direct user feedback right after the 4-point addition above shipped: *"Single feedback as you have now is good enough.. rest 3 points will be needed."* — i.e. keep points 1–3 (Drop Location preview, context strip, T&C checkbox), undo point 4 only.

Reverted `renderFeedback()` back to its original single `#scoreGrid` of 10 options (removed the `FEEDBACK_GROUPS` const and the 3-independent-grid rendering/click-handling logic entirely) — **kept** the new `ticketContextHtml(info)` call on this page (that's point 2, which stays). `submitCustomerFeedback.deluge` reverted back to its original signature/body (`string submitCustomerFeedback(string recordId, string score, string feedbackText)`, writing only `rec.Cx_Feedback_Score`). `getCustomerTicketInfo.deluge`'s `feedback` mode reverted back to checking the single `Cx_Feedback_Score` for `alreadySubmitted` — **kept** the `.toString()` defensive normalization from the earlier DECIMAL/TEXT fix (that fix stands regardless of 1-score-vs-3-score). No new fields needed — `Cx_Feedback_Score_Agent`/`Cx_Feedback_Score_Overall` proposed earlier today were never created in Zoho and are no longer needed; removed from `FIELDS.md`.

**Tested** — `test_customer_page_additions_2026_09_07.js`'s feedback section rewritten to lock in the reverted shape instead (single `#scoreGrid`, 10 options, no `scoreGrid_vendor`/`agent`/`overall`, `FEEDBACK_GROUPS` confirmed absent from source, context strip still present on this page) — 27 checks. `test_customer_page_widget.js` re-run after dropping the now-nonexistent `FEEDBACK_GROUPS` from its injected source list — 41 checks. `test_split_invoice_2026_09_07.js` re-run after reverting its one feedback-related assertion back to checking the single `score` variable — 29 checks. All three suites plus the syntax check: 97 checks total, zero regressions. Packed.

## 2026-09-07 (later still) — Auto-advance to the next step after a payment succeeds

Direct user request: *"redirect-flow after booking fee or remaining fee success will be needed."* Clarified via question: auto-advance (no click needed), not just a "Continue" button or an external redirect URL.

**The constraint that shaped this**: Pay Now opens the Zoho Books hosted invoice page in a **new tab** (`target="_blank"`) — payment happens there, entirely outside this widget, so there's no in-page "payment succeeded" event to hook into. Zoho Books may support a post-payment redirect URL at the payment-gateway/organization level, but that is **not confirmed** as something this project's Deluge code can set per-invoice — not relied on. Instead: this widget now **polls its own existing `getCustomerTicketInfo` Custom API every 8 seconds** (`pollForPaymentAndAdvance()`) once a payment page renders in its "awaiting payment" state (unpaid + has a real `invoiceUrl` — nothing starts if there's no invoice link to pay, or if it's already paid), and reacts the instant `paid` flips to `true`. No new backend endpoint needed. Caps at 90 attempts (~12 minutes) then gives up quietly — if a customer leaves the tab open far longer than that, reopening the same link still shows the correct paid state via the normal `boot()` path regardless.

**Next-step mapping** (`advanceAfterPayment()`) — **flagged as an interpretation**, since the request didn't name the exact next screen: Booking Fee is paid early, so its natural next step is sharing the **breakdown (pickup) location**; Remaining Fee is paid at the end of the service, so its natural next step is **feedback**. This matches the ticket lifecycle order this app already models elsewhere (see `renderDesignPreview()`'s own nav order: Booking → Location → Remaining → Feedback). On success, shows a brief "✓ Payment received! Taking you to the next step…" card for 1.5s, then fetches and renders the next step via the same existing render functions (`renderLocation`/`renderFeedback`) — reuses their own already-correct "already shared"/"already submitted" handling, so nothing extra was needed there. If the next-step fetch itself fails, degrades to a plain "✓ Payment received! Thank you." confirmation rather than an error screen — the payment already succeeded and must never look broken because of an unrelated follow-up fetch.

**Scope, deliberately minimal**: only wired into `renderBookingOrRemainingPayment()`'s own live-poll path — the *already-paid-on-first-load* case (a customer revisiting an old paid link) is untouched and still shows the existing static confirmation, exactly as before. Widening auto-advance to that case too is straightforward if wanted, but wasn't part of what was asked.

**Tested** — new `test_redirect_flow_2026_09_07.js` (19 checks), a real execution test using fake, manually-stepped timers (no real 8-second waits) covering: the poll starts only when unpaid + has an invoice (not when already paid, not when there's no invoice link yet); it correctly detects `paid` flipping true mid-poll and stops recurring; both next-step mappings (Booking Fee → Location/breakdown, Remaining Fee → Feedback) render via the real, unmodified `renderLocation`/`renderFeedback` functions; a failed next-step fetch degrades to a plain confirmation, not an error page; and re-rendering the same payment page never stacks two pollers. Re-ran `test_customer_page_additions_2026_09_07.js` and `test_customer_page_widget.js` after adding no-op `setInterval`/`clearInterval` stubs to their vm contexts (needed once `renderBookingOrRemainingPayment` started calling the new polling code — always present in a real browser, just not in these hand-rolled test contexts) — 27 and 41 checks respectively, no regressions. All four suites plus the syntax check: 116 checks total. Packed.

## 2026-09-07 (later still) — Real Deluge compile error, again: `submitCustomerFeedback.deluge` — "Missing return statement: Provide STRING expression to return"

User hit this live right after the feedback revert above. Same underlying Deluge quirk as `getCustomerTicketInfo.deluge`'s matching fix earlier today (see that entry) — its static return-path checker doesn't reliably prove every path returns when a function is written as a **sequence of independent early-return guard clauses** (`if(A){return;} if(B){return;} ...plain code...; return;`), even though every one of those paths genuinely does return. Restructured into the exact `if/else-if/else` shape already proven live to satisfy Zoho's compiler in `getCustomerTicketInfo.deluge` — same logic, same fields, same two failure messages, just reshaped so every branch keeps its own explicit return with nothing shared left to prove reachability for.

**Needs redeploying**: `submitCustomerFeedback.deluge` — paste the current file into its Custom API in Zoho Creator.

**Tested** — re-ran `test_customer_page_additions_2026_09_07.js` (its Deluge wiring checks for this file don't depend on the if-chain shape, only on the signature/field-writes/absence of the old 3-score fields, all unaffected by this restructuring) — 27 checks, no regressions.

## 2026-09-08 — TOW tickets now auto-chain from Breakdown to Drop Location

User-requested flow integration: *"if the service type is TOW, then proceed to the Drop Location step"* after breakdown location is shared. Previously `submitLocation()` always ended with a single static "Location shared — thank you!" regardless of service type — `Service_Type` was already returned by `getCustomerTicketInfo.deluge` on every mode (added earlier for the ticket-context strip), but the widget's own JS never read it at all.

**Fix**: new `TICKET_SERVICE_TYPE` global, set from `info.serviceType` inside `renderLocation()`. `submitLocation()` now checks, right after a successful save: if this was a **breakdown** submission (not drop) **and** `TICKET_SERVICE_TYPE==="TOW"`, shows a brief "✓ Breakdown location shared! Now let's get your drop-off location…" transition, waits 1.5s (same pattern as the existing payment-success auto-advance), then switches `LOCATION_TYPE="drop"`, re-fetches `getCustomerTicketInfo` for fresh drop-location status, and re-renders the location step for drop. Every other case — RSR/non-TOW breakdown, or the drop submission itself (never chains a second time) — keeps the exact original single-step "thank you" behavior. If the re-fetch itself fails, degrades to the plain thank-you rather than hanging.

**Needs redeploying**: none — client-side JS only (the `serviceType` field was already being returned server-side).

**Tested** — new `test_customer_flow_integration_2026_09_08.js` (14 checks): confirms `renderLocation()` correctly captures `serviceType`, confirms the TOW chain fires only after breakdown (not drop) and only for TOW, confirms RSR never chains, confirms a failed re-fetch degrades gracefully, and confirms geolocation-denied error handling is completely unaffected. Full existing suite re-run — no regressions.

## 2026-09-08 (later) — diagnosing: real Customer Page links (with `?recordId=&mode=` confirmed present on the sending side) still land on the sample-data design preview

Traced a live report from the getRescueTicket side ("customer page link not attach" → eventually "showing dummy data not ticket data") all the way to this widget's own `boot()`. Found `boot()` already had an existing, **pre-dating-this-session "TEMPORARY"** fallback: whenever `RECORD_ID`/`MODE` come back empty, it silently shows `renderDesignPreview()` (the sample "Priya Sharma" data) instead of a real error — the actual `renderError(...)` line for this case is commented out with a note to restore it "before this goes live to real customers." That's why a broken link never surfaced as an obvious error; it just quietly showed sample data instead.

The real open question this doesn't yet answer: **why are `RECORD_ID`/`MODE` empty at all**, given the sending side (`getRescueTicket`'s `buildCustomerPageUrl()`) is confirmed to append `?recordId=<id>&mode=<mode>` to the link. This file's own header comment already documents a known Zoho-specific gotcha — real Zoho Creator page URLs route the query string after the page's own `#hash`, which plain `window.location.search` never sees, which is exactly why `getPageParams()` prefers `ZOHO.CREATOR.UTIL.getQueryParams()` instead. Added a temporary `alert()` right where `boot()` currently falls back to the design preview, showing: whether the `getQueryParams` SDK function was even found, the raw params object it returned, and the raw `window.location.href`/`search`/`hash` — this will show definitively whether the SDK call itself is failing/unavailable here, or whether the params are landing somewhere this widget isn't looking. REMOVE this alert() once the real cause is confirmed, and remember to also swap the commented-out `renderError(...)` back in per its own note (separate, pre-existing TODO, not something this session introduced).

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs a Portal/Development "Relaunch to update" click after publishing, per the exact same caching behavior confirmed on the `getRescueTicket` side today.

**Tested** — syntax-checked only; this is a live-debugging diagnostic addition, not a fix, so no behavior to regression-test yet.

## 2026-09-08 (later still) — Real Deluge compile error, a third time: `submitCustomerLocation.deluge` — "Missing return statement: Provide STRING expression to return"

Same exact quirk as `getCustomerTicketInfo.deluge` and `submitCustomerFeedback.deluge` before it (see both entries above) — the `"drop"`/`"breakdown"` branches of the `locationType` if/else-if/else fell through to one shared `result.put("code","SUCCESS"); return result.toString();` after the block, while the `else` (unknown-locationType) branch returned early inside the block itself. Deluge's static return-path checker can't reliably prove every path reaches the shared trailing return once branches are mixed like this.

**Fixed the same proven way**: `"drop"` and `"breakdown"` now each end with their own explicit `result.put("code","SUCCESS"); return result.toString();` right where they set the ticket's lat/lon fields, instead of falling through to a shared one. The old shared trailing return (now unreachable/redundant) is removed. Purely structural — same two field-writes, same success/error codes and messages, nothing else changed.

**Needs redeploying**: `submitCustomerLocation.deluge` — paste the current file into its Custom API in Zoho Creator.

**Tested** — existing test files (`test_customer_flow_integration_2026_09_08.js`, `test_customer_page_widget.js`) mock `submitCustomerLocation` at the JS/API-call layer, not the Deluge file's internal structure, so they're unaffected and don't need updating for this change.

## 2026-09-09 — `getPageParams()` hardened with a hash-based fallback; real root cause of recordId/mode never arriving found on the getRescueTicket side

Companion fix to `getRescueTicket`'s own README entry of the same date (see that one for the full investigation — reading the actual Zoho widget SDK source plus Zoho's own documentation on page parameters). Short version: `ZOHO.CREATOR.UTIL.getQueryParams()` doesn't read this widget's own URL at all — it asks the parent Zoho Creator page, whose own param-resolution logic this project has no visibility into, and real Zoho page parameters are documented as living after the page's own `#hash`, not a plain `?query` on the raw path (which is what `getRescueTicket` was sending).

**Fix**: `getPageParams()` now tries, in order: (1) the SDK's `getQueryParams()` as before — still authoritative if it returns something; (2) if that comes back without a `recordId`, parses `window.location.hash` directly (`...#Customer_Page?recordId=X&mode=Y`) — since `getRescueTicket`'s `buildCustomerPageUrl()` now sends the params in this form too; (3) the existing plain `window.location.search` fallback, for standalone/demo use and in case the plain-query form is reachable after all in some context. Each step only fills in keys the previous one didn't already provide — never overrides a real SDK-provided value.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs a Portal/Development "Relaunch to update" click after publishing, same as always.

**Tested** — syntax-checked. Not yet confirmed live — the existing `alert()` debug in `boot()` (still in place) will simply stop firing once real ticket data loads instead of the sample-data fallback, which is the clearest possible signal this actually worked.

## 2026-09-09 (later) — real fix for the `9370` "invalid public key" / 401 errors: added `public_key` support to `callCustomApi()`

Live console confirmed the actual blocker, separate from the recordId issue above: every `getCustomerTicketInfo` call failed with `POST .../getCustomerTicketInfo 401 (Unauthorized)` and `{code: 9370, message: "The request contains an invalid public key for this Custom API."}`. Root cause, confirmed via Zoho's own documentation: this page runs with **no logged-in Zoho session at all** (a real customer just opens a WhatsApp link), so Custom API calls can't ride on session auth the way every other widget in this project's calls do. Zoho Creator's own mechanism for exactly this case is a per-Custom-API "PublicKey" authentication mode — when enabled on a Custom API, Zoho shows a Public Key value that must be sent as `public_key` in the `invokeCustomApi()` config for an anonymous call to be accepted.

**Fix**: new `CUSTOM_API_PUBLIC_KEYS` map (currently blank placeholders for `getCustomerTicketInfo`, `submitCustomerLocation`, `submitCustomerFeedback` — all three are called from this same public page, so all three need this). `callCustomApi()` now includes `public_key` in its config whenever a key is filled in for that API name, and logs a clear warning (not a silent failure) when one isn't. Purely additive — doesn't change any existing payload/response handling.

**Needs action in Zoho Creator, not just redeploying**: for each of the 3 Custom APIs above — open it in Zoho Creator, set its Authentication to "PublicKey", copy the Public Key value it displays, and paste it into `CUSTOM_API_PUBLIC_KEYS` in this file. Then redeploy the widget as usual (upload, publish, Relaunch).

**Tested** — syntax-checked. Cannot be live-tested until real public keys are filled in — this is infrastructure for the actual fix, not the fix's live confirmation.

## 2026-09-09 (later still) — real Public Keys filled in for all 3 Custom APIs

User enabled PublicKey authentication for `getCustomerTicketInfo`, `submitCustomerLocation`, and `submitCustomerFeedback` in Zoho Creator and provided each one's real key (from their PublicKey-auth endpoint URLs — confirms the raw-REST query param is `publickey`, consistent with the SDK's own `public_key` config field this project already uses). `CUSTOM_API_PUBLIC_KEYS` now has all 3 real values instead of blank placeholders. Only `getRescueTicket`'s own Custom APIs (`onDemandInvoicing`, `checkOperationsManagerPermission`, `recordVendorPayment`) are out of scope here — those are only ever called from an authenticated agent session, never from this public page, so they don't need PublicKey auth at all.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs the usual publish + Relaunch to update.

**Tested** — syntax-checked. Not yet confirmed live — next test should show `getCustomerTicketInfo`/`submitCustomerLocation`/`submitCustomerFeedback` calls succeeding (no more 401/9370 in the console) once this is redeployed.

## 2026-09-09 (later) — real secondary bug found and fixed: design preview was triggering live API polling with an empty recordId

Live error showed `getCustomerTicketInfo` called with `{recordId: "", mode: "booking_payment"}` — traced to `renderDesignPreview()`'s own sample data (`paid:false` + a fake `invoiceUrl`, by design so the preview looks realistic) triggering `renderBookingOrRemainingPayment()`'s real payment-polling (`pollForPaymentAndAdvance()`), which then calls the real `getCustomerTicketInfo` Custom API every 8s using the global `RECORD_ID` — empty, since design preview never sets it. This was firing real (failing) API calls purely as a side effect of viewing the sample-data preview, muddying live debugging with errors that have nothing to do with an actual customer's real link.

**Fix**: `renderBookingOrRemainingPayment()`'s polling trigger now also checks the existing `IS_PREVIEW` flag (already set by `renderDesignPreview()`, previously unused anywhere) — `if(!IS_PREVIEW && !info.paid && info.invoiceUrl)`. Design preview never starts live polling now; real ticket rendering (`IS_PREVIEW` stays `false`) is completely unaffected.

**Separately confirms**: the underlying recordId/mode issue (see the two entries above from earlier today) is still not resolved live — this empty-recordId call only happens via the design-preview fallback, meaning `boot()` is still landing there, meaning `RECORD_ID`/`MODE` are still coming back empty from a real link. Points to `customerPage` itself not yet running the code with the hash-fallback fix (or the public-key fix) — needs its own separate upload+publish+Relaunch cycle, distinct from `getRescueTicket`'s.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked.

**Tested** — syntax-checked only.

## 2026-09-09 (later still) — added a pre-call payload log to `callCustomApi()`

User request: "before call please logs the payload so that we can debug." `callCustomApi()` now logs, right before every actual API call: the api name, the payload being sent, whether a `public_key` is attached for it, and the current global `RECORD_ID`/`MODE` values — one line covering everything needed to see at a glance whether a call is going out with real data or (like the design-preview polling bug just fixed) empty/garbage values. Purely additive, no behavior changed.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked.

**Tested** — syntax-checked only; this is a console-only diagnostic addition.

## 2026-09-09 (later yet) — pre-call debug log elevated from `console.log` to `console.error`

Live screenshot showed the same `401`/`9370` errors but the new pre-call log wasn't visible anywhere, not even truncated — most likely explanation: the console's own log-level filter (a "Default levels" dropdown, visible in an earlier screenshot) was hiding Info-level `console.log()` output while still showing warnings/errors/network failures. Changed the single line from `console.log(...)` to `console.error(...)` — same content, but error-level output isn't hidden by that filter. Purely cosmetic (log severity only); nothing about program behavior changed.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked, verified directly inside the zip.

**Tested** — syntax-checked, and directly confirmed present inside `dist/customerPage.zip` by unzipping and grepping it (not just checking file timestamps).

## 2026-09-09 (yet again) — always-on boot() log added: complete URL + resolved recordId/mode, every load

User asked why `recordId`/`mode` weren't showing up in the log, and to log the complete URL — the existing `alert()` in `boot()` only ever fired when `recordId`/`mode` came back EMPTY, so a load that resolved them correctly (or partially) never logged anything at all. Added a `console.error()` (not `.log()` — same reasoning as the entry above) right after `getPageParams()` resolves, firing on every single `boot()` call: the complete `window.location.href`/`search`/`hash`, the raw object `getPageParams()` returned, and what `RECORD_ID`/`MODE` actually resolved to.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs a Portal/Development "Relaunch to update" click, per every prior lesson in this file.

**Tested**: syntax-checked only; console-only diagnostic addition, no behavior changed.

## 2026-09-09 (final) — step-by-step logging added across the whole flow

User request: "add logs in customer page so that every step we can check." Added a `console.error()` (same log-level reasoning as every other entry above — plain `.log()` was confirmed invisible under some filter settings) at the entry point of every major step, all prefixed `"[Get-Rescue Customer] STEP:"` for easy console filtering:

- `boot()` — `getCustomerTicketInfo` result and which mode is about to render (in addition to the existing URL/params log)
- `renderBookingOrRemainingPayment()` — amount, paid status, invoice URL presence, preview flag
- `pollForPaymentAndAdvance()` — start, every 8s tick's paid status, and giving-up after max attempts
- `advanceAfterPayment()` — which next step it's advancing to, and the fetch result for that step
- `renderLocation()` — location type, service type, already-have-location flags
- `submitLocation()` — captured geolocation, save result, TOW-chain trigger decision, permission-denied case
- `renderFeedback()` — already-submitted flag, submit result

Purely additive — no logic, control flow, or rendered output changed anywhere.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs Relaunch to update, as always.

**Tested**: syntax-checked only.

## 2026-09-09 (real fix, finally live-testable) — Booking Fee showed ₹0.00 on the Customer Page while the portal showed 200 for the same ticket

First live confirmation that the recordId/mode fix (Page Variables, see this same day's earlier entries) AND the public-key fix both actually work — this session's console showed the full step-by-step log trail for the first time, no more 401/9370, no more design-preview fallback. Real remaining bug found: `renderBookingOrRemainingPayment` showed **₹0.00** for a ticket whose portal `Booking Fee` field clearly read `200`.

**Root cause**: `getCustomerTicketInfo.deluge`'s `amount = rec.Booking_Fee; if(amount == null) { amount = 0; }` — the same class of bug this file already fixed once for `Zoho_Books_Invoice_ID` (2026-09-07 entry, "DECIMAL and TEXT" error): a Number/Decimal field doesn't always read back as a clean `null` when blank-ish, and a plain `== null` check can silently fall through, but more importantly can also fail to catch the real reason correctly depending on the exact value shape Zoho hands back.

**Fixed**: both `booking_payment` (`Booking_Fee`, a `Number` field per `FIELDS.md`) and `remaining_payment` (`Remaining_Fee`, a `Decimal` field) now normalize via `.toString()` first — same established, already-proven pattern this file already uses for `Zoho_Books_Invoice_ID`/`Zoho_Books_Invoice_ID_Remaining` — before deciding whether to fall back to `0`. Only the blank-detection changed; a real fee value still flows straight through as the actual number.

**Separately flagged, not fixed yet**: this same live screenshot also confirmed a previously-only-suspected issue — Vehicle/Issue show as raw Lookup IDs (e.g. `4488810000000057692`) instead of friendly names (e.g. "Maruti Suzuki Brezza"), exactly the risk flagged back on 2026-09-07 ("may show a raw ID rather than a friendly name until confirmed live"). Not addressed in this pass — say the word and this is next.

**Needs re-pasting**: `getCustomerTicketInfo.deluge` into its Custom API in Zoho Creator.

**Tested**: structurally reviewed against the file's own established, already-working pattern for the identical class of bug — not yet independently re-confirmed live (waiting on the next test after re-pasting).

## 2026-09-09 (later) — booking fee amount now sent directly in the URL, bypassing the Deluge fee-read entirely

User's own idea: "can we do that when we are sending the customer page also attach in the url bookingfee amount" — since `getCustomerTicketInfo.deluge`'s own fee-reading had already shown one real live bug (the ₹0.00 entry above), avoid depending on it being correct at all for this one value. `getRescueTicket`'s `buildCustomerPageUrl()` now sends `amount` as an extra query param (see that project's own README) — the exact `Booking_Fee`/`Remaining_Fee` value already in hand at send time.

**Fix here**: `boot()` now checks for `p.amount` right after fetching `info` — if present, numeric, and the ticket isn't already `paid`, it overwrites `info.amountDue` before rendering. Falls back to the Deluge-derived value for any older link sent before this existed, or if the param is ever missing. `paid` tickets are untouched — that status always comes from the real record, never the URL.

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs Relaunch to update.

**Tested**: syntax-checked. Not yet independently live-tested — needs a fresh link from the updated `getRescueTicket` to actually carry the `amount` param.

## 2026-09-09 (later) — param renamed `amount` → `bookingamount`, matching the real Zoho Page Variable name

User declared the actual Page Variable in Zoho Creator as `bookingamount` (Text), not `amount` — renamed to match exactly (see `getRescueTicket/README.md`'s matching entry for the sending side). `boot()` now reads `p.bookingamount` instead of `p.amount`; same override logic (preferred over Deluge's `amountDue` when present, numeric, and not already paid).

**Needs redeploying**: this widget itself (client-side JS) — freshly repacked. Also needs Relaunch to update.

**Tested**: syntax-checked. Not yet independently live-tested.

## 2026-09-09 (later) — real bugs fixed: raw Lookup IDs shown instead of names; Total_Service_Fee blank-value bug; Vehicle Issue ID type mismatch

User-confirmed live: Customer Page showed raw record IDs ("448881000000057692 - 448881000000056038") instead of vehicle/issue names. `.toString()` on a Lookup field returns its raw ID, not its display value — `FIELDS.md` confirms `Vehicle` is a single Lookup (display field `Name`, on `vehicle_master`) and `Vehicle_Issue` is a multi-select Lookup (display field `Issue_Name`). Also found: `Total_Service_Fee` never got the same DECIMAL-blank-value `.toString()`-normalization fix already applied to `Booking_Fee`/`Remaining_Fee` the same day, so it could show `0` for the same reason those did before their own fix.

**Fixed**: `Vehicle` resolved via `vehicle_master[ID == rec.Vehicle.toLong()].Name` (a dot-chain like `rec.Vehicle.Name` hit a real "Invalid collection object found" compile error — bracket lookup is required instead); `Vehicle_Issue` resolved by iterating its ID list and looking up each `Vehicle_Issue[ID == issId]`, joined with ", ". Both `rec.Vehicle`/issue IDs read back as TEXT rather than NUMBER (a real "left expression is of type NUMBER and right expression is of type TEXT" save-time error), fixed with `.toLong()` before the `[ID == ...]` comparison, same as every other Number/Decimal type-quirk fix already in this file. `Total_Service_Fee` now uses the same `.toString()`-normalization pattern as `Booking_Fee`/`Remaining_Fee`. Every attempt keeps the raw-ID string as its own fallback (set first, in the same outer try) and only overwrites it on success — a wrong assumption about either field's shape can only fall back to the old raw-ID display, never break the response.

**Needs re-pasting**: `getCustomerTicketInfo.deluge` into its Custom API.

**Tested**: user-confirmed live — vehicle/issue names now resolve correctly ("Ampere Magnus Neo", "CABLE CUT, BATTERY JUMPSTART, AIR FILL").

## 2026-09-09 (yet later) — real payment page instead of a hand-built invoice-view URL guess

Same fix as `getRescueTicket/README.md`'s matching entry (user request: "open the Zoho Books payment page directly instead of opening the invoice separately") — `getCustomerTicketInfo.deluge` now prefers a REAL, payable link (`Zoho_Books_Payment_URL` / `Zoho_Books_Payment_URL_Remaining`, written by `onDemandInvoicing()` on the `getRescueTicket` side) as the `invoiceUrl` it returns, falling back to the old hand-built `books.zoho.in/invoices/{id}?organization_id=...` format whenever that new field is blank (older tickets, or if payment-link generation ever fails). No widget-side change needed here — the existing `payNowBtn` `<a href="...">` mechanism and the existing `pollForPaymentAndAdvance()` auto-advance-after-payment polling both already work off whatever `invoiceUrl` this function returns.

**Needs re-pasting**: `getCustomerTicketInfo.deluge` into its Custom API (same file as the fix just above — re-paste once, covers both).

**Tested**: not yet independently live-tested — depends on `Zoho_Books_Payment_URL`(`_Remaining`) actually being created in Zoho and a real invoice being created after that (see `getRescueTicket/README.md`'s own entry for the full chain and its genuinely-uncertain API response field name).

## 2026-09-09 (later still) — real bug fixed: writing via a fetched record variable failed with "has no matching records"

User-reported live, submitting a breakdown location: `{"code":"ERROR","message":"Error at line : 28, 'rec' has no matching records. Unable to update the value rec.Latitude."}` — genuinely surprising since the identical fetch (`Create_Case[ID == recordId.toLong()]`) reads this exact same record fine via `getCustomerTicketInfo.deluge`. The difference: that file only ever READS `rec`; `submitCustomerLocation.deluge` WRITES to it (`rec.Latitude = lat;`) — writing via a field assignment on a previously-fetched record variable isn't reliable for an update in a Custom API, even when the record definitely exists and the same criteria found it moments earlier for reading.

**Fixed**: `submitCustomerLocation.deluge` and `submitCustomerFeedback.deluge` both switched to Zoho's own more canonical single-statement update form — `Create_Case[ID == recordId.toLong()].Field = value;` — which re-resolves the criteria directly as part of the write itself, instead of relying on an earlier-fetched `rec` reference still being valid for a write. `rec` is kept in both files only for the initial "Ticket not found" existence check; unchanged otherwise. Also applied proactively to `serviceRequest/custom-apis/createServiceRequest.deluge`'s own Case_ID update (same exact pattern, same Custom API execution context, never live-tested yet).

**Flagged, not changed**: `getRescueTicket/workflows/CreateServiceTicket.deluge` uses the same two-step fetch-then-write shape (`newRec.Case_ID=caseId;`) but is a Zoho **workflow** (triggered on record create, not called via `invokeCustomApi()`) — a different execution context this bug hasn't been confirmed to affect, and this file was earlier confirmed working for other reasons. Left as-is per "minimal changes" — worth applying the same fix if it's ever seen to fail the same way.

**Needs re-pasting**: `submitCustomerLocation.deluge` and `submitCustomerFeedback.deluge` into their Custom APIs.

**Tested**: not yet independently re-verified live — next location/feedback submission should confirm.

## 2026-09-09 (correction, same day) — the combined-statement "fix" above was wrong; reverted to the two-step form

User re-tested with their own hand-edit (converting to the `returnstr` pattern) and still hit a NEW error: "Improper Statement — Error might be due to missing ';'..." — a genuine SAVE-time error, unlike the original bug report (a RUNTIME error from a real `invokeCustomApi()` call, meaning the ORIGINAL two-step `rec.Field = value;` form actually compiled/saved fine all along). The entry above's "fix" — switching to a combined `Form[criteria].Field = value;` single statement — was based on general Deluge knowledge, not on anything confirmed working in this project, and this new save-time error is strong evidence that combined form isn't valid syntax in this Zoho instance at all.

**Reverted**: `submitCustomerLocation.deluge`, `submitCustomerFeedback.deluge`, and `serviceRequest/custom-apis/createServiceRequest.deluge` all back to the two-step fetch-then-write shape — matching `CreateServiceTicket.deluge`'s own confirmed-working pattern exactly (fetch into a variable, then a plain `variable.Field = value;` statement) — rather than the invalid combined form. `submitCustomerLocation.deluge` also picked up a real logic fix along the way: the user's own hand-edit had converted its early-`return` guards to `returnstr =` assignments without adding the matching `else`/`else if` chaining, so a "Ticket not found" or "missing lat/lon" error would have been silently overwritten by the location-type branch running anyway; restructured into one clean `if/else if/else if/else` chain (one `returnstr` assignment per branch, nothing left to fall through).

**Still unresolved**: the ORIGINAL runtime error ("'rec' has no matching records. Unable to update the value rec.Latitude.") — its real root cause is still unknown. Possibilities not yet ruled out: a Custom API permissions gap (read access to Create_Case but not write), or something specific to this exact combination of guard-clause structure before the write. Each write now uses a FRESH fetch (its own record variable, e.g. `dropRec`/`breakdownRec`/`feedbackRec`) taken immediately before the write, rather than reusing the earlier `rec` — closer to `CreateServiceTicket.deluge`'s own shape (fetch right before use) in case that mattered, but this is not confirmed to be the actual cause.

**Needs re-pasting**: `submitCustomerLocation.deluge`, `submitCustomerFeedback.deluge`.

**Tested**: not yet independently re-verified live — if the ORIGINAL runtime error resurfaces even with a fresh fetch, the next thing to check is this Custom API's own write/update permissions on Create_Case in Zoho Creator's setup screen.

## 2026-09-10 — map-based location picker for Breakdown/Drop Location

Explicit user request: "When collecting the Breakdown Location and Drop Location from the customer on the Customer Page, add a map so the customer can select or set the exact location. After the customer selects the location, capture and save the corresponding latitude and longitude along with the location details."

**Before**: `mode=location`'s "Share My Location"/"Share Again" button called `navigator.geolocation.getCurrentPosition()` once and submitted whatever raw fix the device returned, with no way for the customer to see or correct it (GPS drift is common indoors/multi-storey buildings, and this project's own Agent Portal already has multiple comments about exactly that problem).

**Added**: `renderLocationPicker()` — a drag/zoom map card with a fixed center pin (the customer pans the map under the pin, the standard Uber/Ola/Swiggy "confirm pickup location" pattern) plus a "Confirm This Location" button that submits the map's current center. Geolocation is still used, just demoted to a convenience: on open, and via an explicit "Use My Current Location" button, it recenters the map — a denial/timeout/unsupported browser is no longer a dead-end error, it just leaves the map on a Bengaluru fallback center (this project's own established service area) for the customer to pan to manually. Same "already have it -> Share Again" first screen is unchanged, it now opens the picker instead of firing geolocation directly.

The map itself (`PIN_TILE_SIZE`/`pmLonToX`/`pmMount`/`pmRedraw`/`pmSetView` etc., placed right before `renderLocation()`) is the same dependency-free Web Mercator tile renderer + OpenStreetMap raster tiles approach already proven live in `getRescueTicket`'s own Ops Fleet Map (`SIMPLE_MAP`/`smMount()`) — no paid Google Maps API, no third-party map library (Leaflet etc.), consistent with every widget in this project staying 100% vanilla JS. Each widget.html here is its own self-contained file with no shared JS module, so this is its own copy of that same math/rendering code, not an import — trimmed down for a single fixed pin instead of N colored markers, so there's no marker-click/popup logic to port.

**Unchanged, deliberately**: `submitCustomerLocation.deluge`'s contract (`recordId`, `locationType`, `lat`, `lon` -> `Latitude`/`Longitude` or `DropLocationLat`/`DropLocationLong` on `Create_Case`) — the map picker still ends by calling this exact same Custom API with a lat/lon pair, just a customer-confirmed one instead of a raw GPS fix. The Agent Portal (`getRescueTicket`) already reads and displays both coordinate pairs (readonly "Breakdown/Drop Latitude/Longitude" fields, a Google Maps link, distance calculations, and its own Ops Fleet Map) — nothing there needed to change for these coordinates to "reflect in the Agent Portal", since the fields being written are exactly the same ones it already consumes. `demo.html` (already stale relative to `app/widget.html` in other spots — missing several earlier features too) was **also updated the same day** (user follow-up, screenshotting `demo.html`'s own old geolocation-button flow and asking for the map there too) — this one section is kept in sync since `demo.html`'s Location nav entries are actively used to preview this exact flow; the other pre-existing drift (TOW auto-chain, `ticketContextHtml`, T&C checkbox, etc., all added to `widget.html` after `demo.html` was first written) is unchanged, still out of scope.

**Also changed**: `plugin-manifest.json` — added `https://a/b/c.tile.openstreetmap.org` to `connect-src` and a new `img-src` entry (exact same 3 domains `getRescueTicket/plugin-manifest.json` already uses for its own OSM tiles) so the map's tile `<img>` requests aren't blocked by this widget's CSP. Pre-existing `maps.googleapis.com` entries left untouched (unused by this widget either before or after this change, likely copied from another widget's manifest originally — not this session's concern).

**Needs redeploying**: re-upload `dist/customerPage.zip` and Relaunch/Deploy to Production, whichever this environment needs (see this project's own established Portal-caching / Dev-Stage-Production notes elsewhere in this README).

**Tested**: not yet independently live-tested — tile loading inside the real Zoho Creator public-page sandbox, and drag/zoom on an actual touch device, should both be confirmed on first real use. If tiles don't appear, `plugin-manifest.json`'s new CSP domains are the first thing to check (exact same note this project already has for the Ops Fleet Map).

## 2026-09-10 (later) — coordinate validation + precision rounding, both customer and agent paths

Follow-up to the map-picker entry above. User clarified requirements after seeing the Agent Portal's own "Capture Locations" step (paste-lat-long field) alongside the new customer map:

1. Customer map-selected location -> saved (done above).
2. Agent can also manually enter/paste lat,long (already existed — `LatLongPaste`/`DropLatLongPaste` in `getRescueTicket/app/widget.html`, unchanged structurally).
3. Either path's save must land in `Create_Case` (already true for both — customer via `submitCustomerLocation.deluge`, agent via the normal step `Save & Continue` -> `buildPayload()`/`saveWithCoordFallback()` -> `updateTicket()`).
4. Stay synchronized with the Agent Portal (already true — `getRescueTicket`'s own 30s dashboard poll, `hasCoordPair()`/`updateLocationSnapshot()`/`notifyLocationShared()`, already detects a blank->populated `Latitude`/`Longitude`/`DropLocationLat`/`DropLocationLong` transition from either path and shows a notification card).
5. **Validate the coordinates, both paths** — this was the actual gap. `submitCustomerLocation.deluge` only ever checked for blank lat/lon, never a garbage/out-of-range value, and never protected against the real "Decimal field exceeded maximum digits" error this project already hit on these exact fields on the agent side (`getRescueTicket`'s own `roundCoord()`/`saveWithCoordFallback()`, needed because a raw GPS fix or map pan/zoom math can carry 15-17+ decimal digits).

**Fixed**: `submitCustomerLocation.deluge` now converts lat/lon via `.toDecimal().round(6)` (both confirmed-working Deluge syntax already used the same way in `recordVendorPayment.deluge`) — 6 decimal places is ~11cm of real-world precision, comfortably under Zoho's digit limit — and range-checks the result (-90..90 / -180..180), returning `{code:"ERROR", message:"Invalid coordinates"}` rather than writing garbage or risking the digit-limit error. `app/widget.html`'s `submitLocation()` (and `demo.html`'s copy) also rounds to 6 decimals client-side before sending, for a cleaner payload — the Deluge-side check is the authoritative one, this is just tidiness. The map's own drag math already guarantees a valid range via `pmSetView()`'s existing lat clamp/lon wrap, so this mainly matters for the geolocation-sourced auto-center value.

**Also fixed, agent side**: `getRescueTicket/app/widget.html`'s `wireField()` `parseTarget` handler (the `LatLongPaste`/`DropLatLongPaste` fields) previously accepted any two parseable numbers with zero bounds checking — a typo like "500, 999" would have been written straight into `Latitude`/`Longitude` as if it were real GPS coordinates. Now range-checked the same way, and — matching this same handler's own existing precedent for unparsable text — an out-of-range pair is left alone (doesn't overwrite a still-valid previous value) with a `toast()` telling the agent why nothing changed.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

**Needs redeploying**: `dist/customerPage.zip` and `dist/getRescueTicket.zip` (both re-packed).

**Tested**: not yet independently live-tested.

## 2026-09-10 (later still) — location search added to the map picker

Explicit user request, after confirming the map picker itself worked in `demo.html`: "add in customer page break down and drop location also add search so that customer search location then confirm location."

**Added**: a search box (`#pinSearchInput`) above the map in `renderLocationPicker()`, backed by free-text search against **OpenStreetMap Nominatim** (`nominatim.openstreetmap.org/search`) — the same OSM service this widget already pulls map tiles from. Typing (3+ characters, 450ms debounce) shows a dropdown of matching places; clicking one recenters the map there (zoom 17) exactly the way "Use My Current Location" already does. Search is purely a faster way to get the pin close — the pin is still always the map's center, and **Confirm This Location** is still the one action that actually saves, so this doesn't add a second save path.

This project had this exact search (debounced Nominatim fetch, suggestion dropdown) in `serviceRequest`'s own location step before — removed 2026-09-09 only because that whole field was dropped from that widget's flow entirely, not for any technical failure of the search itself, so this is a fresh implementation of the same already-proven approach, not an untested one.

Query is scoped with `countrycodes=in` (this business's only service country) and a `viewbox` biasing ambiguous matches toward greater Bengaluru (this business's established service area) — **not** `bounded=1`, so a genuinely different address a customer might legitimately enter still returns results rather than being silently excluded. A search failure (network hiccup, Nominatim rate-limiting) degrades to a plain message telling the customer to drag the map instead — never blocks the core flow.

**Also changed**: `plugin-manifest.json` — added `https://nominatim.openstreetmap.org` to `connect-src` (the exact same domain this project's `serviceRequest` widget's manifest already used for this same feature, before it and its manifest entry were both removed).

**Ported to `demo.html`** the same day, same reasoning as the map picker itself.

**Needs redeploying**: `dist/customerPage.zip` (re-packed).

**Tested**: confirmed live in `demo.html` — the map itself renders and drags correctly (user screenshot). Search itself not yet independently tested (needs a live network path to `nominatim.openstreetmap.org`, which `demo.html` has being a plain local file, but the real Zoho Creator widget needs the new CSP entry deployed first).

## 2026-09-10 (yet later) — real bug fixed: "Unknown locationType: breakdown." on a live test

User-reported live, with the browser console and the raw API response: `{"code":"ERROR","message":"Unknown locationType: breakdown."}` — note the trailing period. `submitCustomerLocation`'s own console logging showed the value arriving at `getPageParams()` was already `locationType: 'breakdown.'` (with the period) before this widget ever touched it.

**Checked first, confirmed clean**: `getRescueTicket/app/widget.html`'s own `buildCustomerPageUrl()` — the only place in this whole project that constructs this URL — builds a plain `locationType=breakdown` with `encodeURIComponent()`, no punctuation added anywhere. So the stray trailing period isn't coming from this project's own link-building code; whatever specific link/navigation produced this particular test (the screenshot showed a `page-perma` URL with no visible query string at all, meaning `recordId`/`mode`/`locationType` came back through `ZOHO.CREATOR.UTIL.getQueryParams()`, not a URL query string this repo built) is the thing to check — most likely a Zoho Creator Page Link parameter value with a typo, configured directly in Zoho Studio, outside this repo. Worth checking there if this recurs.

**Fixed anyway, defensively**: `submitCustomerLocation.deluge` now trims `locationType` and tolerates a single stray trailing "." as the same value ("breakdown." → "breakdown", "drop." → "drop") rather than hard-failing the whole save on it — a small transcription typo shouldn't block a customer from sharing their location. Kept to `.trim()` and plain `==` only (both already used everywhere in this project) — deliberately avoided any Deluge string method not already confirmed working elsewhere in this codebase (no existing use of `.endsWith()`/`.replaceAll()` found anywhere in this repo's `.deluge` files).

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

**Tested**: not yet independently re-verified live — next Breakdown/Drop location submission should confirm both that this specific "breakdown."/"drop." case now succeeds, and that the underlying Zoho-side parameter typo (if that's really the source) doesn't recur on a real WhatsApp-delivered link.

## 2026-09-10 (correction, same day) — the locationType-normalization fix above had a real save-time syntax error

User pasted the fix above into Zoho and hit: `Error at line number: 5 Improper Statement — Error might be due to missing ';' at end of the line or incomplete expression.` The file compiled and ran fine before that edit (the earlier "Unknown locationType" was a runtime response, not a compile failure), so the new error had to be in what was just added.

**Root cause**: the fix used brace-less single-statement `if`/`else if` lines (`if(locTypeNorm == "breakdown.") locTypeNorm = "breakdown";`) — a construct not used anywhere else in this entire project's `.deluge` files (every single `if`/`else` in every Custom API and workflow in this repo always braces its body). Never independently confirmed as valid Deluge syntax, and evidently isn't.

**Fixed**: wrapped both branches in `{ }`, matching this codebase's own exclusive convention exactly.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API (again — same file as the entry just above).

**Tested**: not yet independently re-verified live.

## 2026-09-10 (widened) — locationType matching generalized to cover Drop too, not just the one exact "breakdown." case seen live

User request: "fix this function for all use cases and also fix for the drop location." The previous fix only hardcoded the exact two strings `"breakdown."`/`"drop."` — real, but narrow: any OTHER stray character/case variant (e.g. `"Breakdown."`, `"drop  "`, `"BREAKDOWN.."`) would still have fallen through to "Unknown locationType", and the Drop side of that fix was never actually exercised live (only Breakdown was seen failing so far).

**Fixed, more generally**: `locTypeNorm = ifnull(locationType,"").trim().toLowerCase();` then matched by **substring** (`.contains("drop")` / `.contains("breakdown")`) instead of a growing list of exact strings — covers any punctuation/whitespace/case variant on either side symmetrically, using only Deluge methods already confirmed working elsewhere in this project (`.trim()`/`.toLowerCase()`/`.contains()` — see `createServiceRequest.deluge`'s own `.toLowerCase().contains("tow")`). Every branch stays fully braced (no repeat of the brace-less-`if` mistake from the entry just above).

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API (again).

**Tested**: not yet independently re-verified live — next Breakdown AND Drop location submission should both confirm this.

## 2026-09-10 (later) — real field names: Zoho_Payment_URL (was Zoho_Books_Payment_URL)

User created the real fields in Zoho (`Zoho_Payment_URL`/`Zoho_Payment_URL_Remaining`). `getCustomerTicketInfo.deluge` now reads these instead of the earlier placeholder names (`Zoho_Books_Payment_URL`/`Zoho_Books_Payment_URL_Remaining`), which were never confirmed to actually exist in Zoho. Same fallback behavior as before (falls back to the hand-built invoice-view URL if blank) — see `getRescueTicket/README.md`'s own matching entry for the write side of this same rename.

**Needs re-pasting**: `getCustomerTicketInfo.deluge` into its Custom API.

**Tested**: syntax-checked only (braces balanced).

## 2026-09-10 (later) — real root cause of the long-unresolved "'X' has no matching records" bug confirmed and fixed

This project hit `'rec'`/`'breakdownRec'`/`'dropRec'` "has no matching records" errors on `submitCustomerLocation.deluge` several times this project, each time fixed with a guess that didn't hold (a combined single-statement form that turned out to be invalid syntax; a fresh re-fetch that turned out to still fail). The real cause only became clear after the exact same error appeared in a brand-new, unrelated Custom API (`getRescueTicket/custom-apis/generateZohoPaymentLink.deluge`) with an identical shape — enough data points to actually compare against every write that's ever worked in this project.

**Root cause**: fetching the SAME record TWICE with identical criteria within one Custom API execution fails — once for an existence/null-check, then again later for the actual write. Every write that's ever worked in this project (`createServiceRequest.deluge`, `CreateServiceTicket.deluge`) fetches a record exactly ONCE, immediately before writing to it. `submitCustomerLocation.deluge` fetched `rec` for the null-check, then separately re-fetched as `dropRec`/`breakdownRec` for the write; `submitCustomerFeedback.deluge` did the same with `feedbackRec`. That second, redundant fetch is what was failing, the whole time.

**Fixed, both files**: write directly onto `rec` (already fetched, already confirmed non-null) instead of re-fetching a second variable — no second query at all.

**Needs re-pasting**: `submitCustomerLocation.deluge` and `submitCustomerFeedback.deluge` into their Custom APIs.

**Tested**: braces verified balanced on both files — not yet independently re-tested live. If this is confirmed live, it resolves a bug this project has carried, unexplained, for several days.

## 2026-09-10 (correction, same day) — "double-fetch" theory disproven by a live re-test; trying a different, honestly-flagged hypothesis

User re-tested the "reuse `rec`, no re-fetch" fix from the entry above and got the **exact same error** — `'rec' has no matching records. Unable to update the value rec.Latitude.` — on the single, already-fetched, already-null-checked `rec` variable, no second query anywhere in the function. That directly disproves the double-fetch theory as the (sole) root cause.

**Searched Zoho's own documentation and community discussions for this exact error text — found nothing definitive.** Rather than guess again without any basis, applied the one concrete, testable difference between this failing write and the confirmed-working one in `generateZohoPaymentLink.deluge`: that file assigns a **string**; this one was assigning a raw **Decimal** (`latVal`/`lonVal`, straight from `.round(6)`) directly to `Latitude`/`Longitude`/`DropLocationLat`/`DropLocationLong`. Every other numeric write anywhere else in this project's own Deluge files sends the value as a string first — this was the one exception.

**Changed**: `rec.Latitude = latVal;` → `rec.Latitude = latVal.toString();` (and the same for `Longitude`/`DropLocationLat`/`DropLocationLong`). **Flagged honestly as a hypothesis, not a confirmed fix** — low-risk (a working write can't be broken by stringifying the value first), but not verified against any documented Zoho behavior. Also added the same defensive `recordId.trim()` fix already confirmed real in `generateZohoPaymentLink.deluge` the same day, in case a stray space is also part of what's going on here.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

**Tested**: braces verified balanced. Genuinely unconfirmed live — if this doesn't work either, the next thing to actually check (rather than guess further) is this Custom API's own write/update permission on `Create_Case` in Zoho's setup screen, which has been flagged as a possibility since the very first time this error appeared and still hasn't been ruled in or out.

## 2026-09-10 (later) — defensive retry added, in case the field's Decimal Points setting is never widened

Explicit user request: "if i dont do what happen please manage submitCustomerLocation function accordingly" — i.e. make the function itself resilient to the Max Digits=16/Decimal Points=15 field configuration issue found live this same day, rather than depending entirely on that Zoho-side change actually happening.

**Added**: a retry loop (merging the `isDrop`/`isBreakdown` branches, since both now share it) that tries the write at progressively fewer decimal places — 6, 4, 2, 0 — mirroring `getRescueTicket`'s own already-proven `COORD_PRECISION_FALLBACKS`/`saveWithCoordFallback()` pattern for this exact class of Zoho Decimal-field digit-limit error. If every level fails, returns a genuinely specific, actionable error message (naming the actual field configuration problem and what to change) instead of Zoho's own confusing generic "has no matching records" text.

**Flagged honestly**: if the real constraint is specifically on the field's *integer* part (which the Max Digits=16/Decimal Points=15 configuration suggests), reducing decimal places won't free up room there — `28` stays 2 digits no matter how many decimals follow — so this retry may not actually rescue a failing write. Its value either way: (1) it's free/harmless to try, (2) if the real constraint turns out to be more lenient than the strict precision/scale reading, this could genuinely fix it, and (3) even on total failure, the error message is now honest and specific instead of misleading.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

## 2026-09-10 (later still) — retry loop confirmed dead-on-arrival by the math; removed and replaced with a direct fix message

The honest flag in the entry above ("if the real constraint is specifically on the field's integer part... this retry may not actually rescue a failing write") is now confirmed true, not just a possibility. Zoho's Decimal field type works like SQL `DECIMAL(precision, scale)`: the whole-number part can only ever hold `Max Digits - Decimal Points` digits. With the confirmed live configuration (Max Digits=16, Decimal Points=15), that's **1 digit** — and every real latitude/longitude has a 2-3 digit whole-number part (`28` in `28.613977`, `77` in `77.209023`). Reducing decimal places (the retry's whole strategy) never touches the whole-number part at all, so all 4 of its precision levels (6/4/2/0) were mathematically guaranteed to fail identically, every single time — this was never going to rescue a write, regardless of how many levels were tried.

**Removed**: the entire `precisionLevels` retry loop; write logic simplified back to a single clean attempt (no more dead-weight retry iterations).

**Changed**: the error message now names the exact fix instead of a vague "may need widening" — *"FIX: in Zoho Studio, open Create_Case's field list, edit each of those 4 fields, and change Decimal Points from 15 to 6 (leave Max Digits at 16 — that leaves 10 whole-number digits, far more than the 2-3 ever needed)."* Confirmed safe to be this specific/technical: `customerPage/app/widget.html`'s own `submitLocation()` never shows this raw message to the customer — on any non-`SUCCESS` response it calls `renderError("Couldn't save your location — please try again, or contact your agent directly.")`, a fixed generic string; the detailed message is only ever visible in this Custom API's own execution log and the browser console (`console.error`).

**The only real fix remains Zoho-side** — widening `Decimal Points` on `Latitude`/`Longitude`/`DropLocationLat`/`DropLocationLong` from 15 to 6 in Zoho Studio (Create_Case's own field list → each field → Edit Properties). No code-side workaround can exist given the math above.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

**Tested**: braces/parens verified balanced (`node` scan). Not yet live-tested — but the write path itself is unchanged from the confirmed-working `generateZohoPaymentLink.deluge` pattern (string-cast values, single fetch, no re-query), so it should behave identically to that file once the Zoho fix is applied — the only thing genuinely gating this now is the field configuration, not the code.

## 2026-09-10 (yet later) — Zoho fix confirmed applied (Max Digits=15/Decimal Points=6)

User widened `Latitude`'s field config live in Zoho Studio (screenshot confirmed: Max Digits=15, Decimal Points=6) — same change applied to the other 3 fields per the same instruction. 9 whole-number digits of headroom (15-6), far more than the 2-3 any real coordinate needs — this resolves the root cause directly.

This widget's own write path (`submitCustomerLocation.deluge`, simplified in the entry above) was already safe under this new config without any further change — it rounds to exactly 6 decimal places server-side (`.round(6)`) before writing, both client-side (`customerPage/app/widget.html`'s own `.toFixed(6)`) and server-side, so nothing here needed touching.

**Cross-check run across the whole app suite** (at the user's request, to confirm nothing else breaks under the new *stricter* 6-decimal-place limit — the old config nominally allowed 15 decimals, even though the 1-digit whole-number cap made it unusable regardless): `getRescueTicket` needed a small fix (its own `roundCoord()` defaulted to 17 decimals on the first save attempt, relying on a retry loop to converge to 6 — see `getRescueTicket/README.md`'s own matching entry). A separate, older widget (`Ticket Kanban`) was found to have zero rounding/retry protection at all for these same fields — a real, pre-existing gap, flagged to the user but not fixed (outside this session's authorized scope so far).

**Tested**: braces verified balanced. Not yet independently live-tested — and this doesn't replace actually widening the field's Decimal Points setting in Zoho if that's feasible; that remains the more certain fix.

## 2026-09-11 — `submitCustomerLocation.deluge`: stamps its own received-time now, instead of relying on the agent to catch up

Part of a wider "capture location+time at every relevant step" request — see `getRescueTicket/README.md`'s own matching entry for the full lifecycle audit and field list. `Location_Received_Time` (breakdown) already existed but was only ever stamped later, whenever the agent's own `locations` wizard step next happened to save (`getRescueTicket/app/widget.html`'s `extra()` for that stage) — not the actual moment the customer shared it. User created a new `Drop_Location_Received_Time` field (drop side had no timestamp at all before this).

**Changed**: right after the existing `Latitude`/`Longitude` or `DropLocationLat`/`DropLocationLong` write (unchanged), a new nested block stamps `Location_Received_Time` or `Drop_Location_Received_Time` (whichever applies) via Deluge's own `zoho.currenttime` system variable. Wrapped in its **own** try/catch, separate from the coordinate write — the timestamp is new, unverified-live territory (Deluge's documented behavior for `zoho.currenttime` is confirmed for form-submit inserts; behavior in this file's own fetch-then-update pattern isn't independently confirmed yet), and it must never be allowed to fail the actual coordinate save, which remains this function's one genuinely critical job and was already working correctly before this change. On a timestamp-write failure, the function still returns `SUCCESS` (coordinate saved) and just logs the timestamp failure via `info`.

`getRescueTicket/app/widget.html`'s own `locations`-stage `Location_Received_Time` stamp is left completely unchanged — now effectively a harmless no-op for any ticket where this Custom API already set it, since that code only fires when the field is still blank.

Also updated the stale error message in the coordinate-write catch block — it used to cite the OLD field config (Max Digits=16/Decimal Points=15), which the user has since corrected; that text would have been actively misleading if this branch ever fired again post-fix.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API.

**Tested**: braces/parens verified balanced. Not yet live-tested — the `zoho.currenttime` assignment specifically needs a real coordinate save (both breakdown and drop) to confirm it's accepted by Zoho in this exact context; if it isn't, the try/catch means the coordinate still saves correctly and only the timestamp silently stays blank (visible in this Custom API's own execution log).

## 2026-09-11 (later) — Stale Decimal-field comments/error message replaced, now that `Latitude`/`Longitude`/`DropLocationLat`/`DropLocationLong` are Single Line Text

User converted all 4 of these fields (plus `RSP_Start_*`/`RSP_Drop_*`/`Cancel_Location_Lat`/`Cancel_Location_Lon` elsewhere in the app) from Decimal to Single Line Text, per `getRescueTicket/README.md`'s own matching entries — eliminates the "exceeded maximum digits" error class entirely (no digit limit on Text). This file's write logic needed **no change** — it already wrote a Deluge string (`latVal.toString()`) either way — but its comments and the catch block's error message still described the old Decimal(precision,scale) math and the specific Max Digits/Decimal Points values from the previous (now superseded) fix, which would have been actively misleading if read after this conversion.

- The root-cause comment block (above the write) now explains the fields were converted to Text instead of continuing to describe stale Decimal config numbers.
- The catch block's error message dropped its Decimal-specific explanation (`"...were reconfigured 2026-09-11 to Max Digits=15/Decimal Points=6..."`) since it no longer applies to anything — now just surfaces Zoho's own error text directly.

**Needs re-pasting**: `submitCustomerLocation.deluge` into its Custom API (comment/message-only change, but the file still needs to be re-synced with Zoho — same as every other pending edit to this file).

## 2026-09-12 — Full-application audit: 2 real bugs fixed, plus the "must revert before going live" items actually reverted

Part of a 5-parallel-audit full-application verification pass (see `getRescueTicket/README.md`'s own matching entry for the full scope).

**Bug 1 — Remaining Payment's "real payment link" preference read the wrong field name, so it never worked.** `getCustomerTicketInfo.deluge`'s `remaining_payment` mode read `rec.Zoho_Books_Payment_URL_Remaining` — the OLD placeholder name from before the 2026-09-10 rename. Nothing in this project has ever written to that name (confirmed via a repo-wide search); the real field `getRescueTicket` writes/reads is `Zoho_Payment_URL_Remaining`. This branch could never succeed, so every customer's Remaining Fee "Pay Now" link has always silently fallen back to the older hand-built Books invoice-view URL, even on tickets where a real, directly-payable Zoho Payments link existed. **Fixed**: now reads `rec.Zoho_Payment_URL_Remaining`, matching the Booking Fee side's already-correct pattern exactly.

**Bug 2 (documentation)**: the `location` mode's own comment still claimed `Latitude`/`DropLocationLat` "are Decimal fields" — stale since the 2026-09-11 Text conversion (the sibling `submitCustomerLocation.deluge` was corrected at the time; this file was missed). Corrected — functionally harmless either way (`.toString()` on an already-Text value is a no-op), but worth being accurate.

**"Must revert before going live" items, actually reverted**: this widget's own header comments had said since 2026-09-02/08 that two things needed removing before real customers used this page — and the page is confirmed to already be handling real traffic. (1) The debug `alert()` popup shown whenever `recordId`/`mode` come back empty — removed; its diagnostic purpose was already resolved on 2026-09-08, and the always-on `console.error()` log right above it covers the same information without interrupting a real customer with a popup. (2) `renderDesignPreview()` (fake "Priya Sharma" sample data) as `boot()`'s own fallback for a broken/incomplete link — replaced with the real `renderError("This link looks incomplete — please contact your agent for a new one.")` that had been commented out this whole time. A genuinely broken link now tells the customer to contact their agent instead of silently showing someone else's fake data as if it were their own ticket. `renderDesignPreview()` itself is left in the file (smaller diff) but is now orphaned — it was called from nowhere else.

**Also fixed**: the Terms & Conditions checkbox still linked to a `#` placeholder — the sibling `serviceRequest` widget got the real URL on 2026-09-09 and this page was never updated to match. Now points to `https://www.getrescued.in/terms-and-conditions`, same as `serviceRequest`.

**Needs redeploying**: `dist/customerPage.zip` (re-packed). **Needs re-pasting**: `getCustomerTicketInfo.deluge` into its Custom API (Bug 1's field-name fix). **Not yet live-tested** — worth confirming a real Remaining Fee link now shows the actual Zoho Payments checkout page instead of the Books invoice view, and confirming a genuinely broken/incomplete link now shows the real error message.

**Tested**: braces/parens verified balanced (0/0). No logic changed, so no new live-testing need beyond what was already pending above.

## 2026-09-13 — Real bug fixed: `submitCustomerLocation.deluge` — "'rec' has no matching records. Unable to update the value rec.Latitude."

User-reported live, exact Zoho error above, on a real payload (`recordId`/`locationType=breakdown`/real lat/lon). The failure happened at the WRITE step (`rec.Latitude = ...`), not the read — `rec` had already passed the "Ticket not found" null-check moments earlier using the identical `ID == recordId.trim().toLong()` criteria, so the record genuinely exists and genuinely matched a SELECT; the dot-notation write specifically then failed to re-match it.

This exact `rec = Form[criteria]; rec.Field = value;` pattern is independently proven working elsewhere in this project against this same `Create_Case` form (`generateZohoPaymentLink.deluge`'s own `rec.Zoho_Payment_URL = ...`), so this isn't a flaw in the pattern itself — the true root cause for why it fails specifically here is **not confirmed**. Leading candidate, not verified: this Custom API's own "Execute the script as" permission configuration in Zoho may differ from `generateZohoPaymentLink`'s (each Custom API can be configured with its own execution user/permission scope) — worth comparing the two directly in Zoho if this recurs after the fix below.

**Fix applied**: switched the actual write (both `Latitude`/`Longitude` and `DropLocationLat`/`DropLocationLong`, plus the separate `Location_Received_Time`/`Drop_Location_Received_Time` stamp) from the dot-notation record write to Deluge's own documented bulk `update <form> set ... where ...;` statement, which targets `Create_Case` directly by criteria instead of depending on a previously-fetched record's own write-back. `updateCount` is checked explicitly — an update matching 0 records still surfaces the same kind of `"ERROR"` response as before, never a silent false `"SUCCESS"`, so this fix cannot quietly start reporting success while saving nothing.

**⚠ This exact `update ... set ... where ...;` statement form is new to this project** — not used or independently confirmed anywhere else in this repo's `.deluge` files before now. Braces/parens verified balanced via a script, but Deluge's own script editor in Zoho is the real authority on whether this syntax is valid — paste this file's updated content into the `submitCustomerLocation` Custom API and watch for a save-time syntax error first, before assuming the logic is correct. If Zoho rejects it, that rules this approach out cleanly with zero live/customer-facing risk, since nothing was published yet.

**Needs**: re-pasting into the `submitCustomerLocation` Custom API in Zoho, then a real test — same payload shape as the bug report (`recordId`, `locationType=breakdown`, real lat/lon) — confirming `code:"SUCCESS"` and that `Latitude`/`Longitude` actually land on the ticket this time.

## 2026-09-13 (later) — Correction: the `update ... set ... where ...;` attempt above was rejected by Zoho's own script editor

User pasted the fix above into the real `submitCustomerLocation` Custom API and got a save-time error: `Improper Statement — Error might be due to missing ';' at end of the line or incomplete expression`. Confirms what the entry above already flagged as unverified — that exact bulk-`update` statement form is **not valid Deluge syntax in this account**, at least not in the shape attempted. No live/customer-facing impact either way, since it never got past Save.

**Reverted** to the dot-notation write (`rec.Field = value`) — the one pattern independently proven working elsewhere in this project against this same `Create_Case` form (`generateZohoPaymentLink.deluge`). The one thing changed versus the ORIGINAL (pre-fix) code: the record is now re-fetched immediately before the write (`recForWrite = Create_Case[ID == recordId.trim().toLong()];`, right at the write site) rather than reusing the `rec` fetched much earlier at the top of the function — a zero-new-syntax hedge against the original runtime error being caused by that reference going stale, using only the exact same `Create_Case[ID == ...]` pattern already proven to compile and run. **Not confirmed to fix the underlying issue** — if the exact same `"'rec' has no matching records"` error recurs even with a fresh fetch immediately before the write, that rules out staleness and points at something Zoho-side: compare `submitCustomerLocation`'s own "Execute the script as" permission/user configuration against `generateZohoPaymentLink`'s directly in Zoho — that's the next thing to check, not another code guess.

**Needs**: re-pasting this corrected version into the `submitCustomerLocation` Custom API, confirming it saves clean this time (no syntax error), then the same real-payload test as before.
