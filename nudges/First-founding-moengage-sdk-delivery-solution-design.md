# 1. Introduction

**Feature.** The `first_funding` nudge: a prompt shown to OWNER-role users of a newly onboarded, unfunded company, encouraging them to fund their account.

- **MoEngage decides *when to offer* the nudge** — via the native **Self-Handled In-App ("InApp NATIV")** mechanism. MoEngage's targeting is coarse (role, context, frequency capping) and is allowed to be stale.
- **The mobile app does not render on MoEngage's say alone.** When `getSelfHandledInApp()` hands it the `first_funding` payload, the app calls a **BFF (`customer-mgmt-service`)** endpoint to perform a **live eligibility check** against core banking before rendering anything. The eligibility decision — "is this company actually still unfunded, right now" — lives entirely in that BFF call, not in MoEngage's targeting rule and not in the mobile app's own logic beyond "did the BFF say yes."
- That same BFF call is also what feeds MoEngage the stop signal: if the live check finds a credit transaction, it tells the app not to render, **and** sends the `Transaction Completed` event to MoEngage. If it finds nothing, it tells the app to render, **and** schedules a background check that will send that same event later, whenever funding eventually happens — covering the case where the user never triggers another live check by reopening the app.

So MoEngage is the delivery/trigger mechanism and the destination for the stop signal; customer-mgmt-service is the sole source of truth for whether the company is actually funded, checked live, every time MoEngage offers the nudge.

**Ownership.**

| Concern | Owner |
|---|---|
| Whether to *offer* the nudge opportunity at all (coarse targeting, context, frequency capping) | MoEngage, evaluated server-side and cached on-device by the SDK via a meta API call — refreshed at most once every 15 minutes during an active session, immediately on a new session start or app relaunch (confirmed in MoEngage's SDK docs); allowed to be stale within that window |
| Whether the nudge is *actually* eligible right now (the real, authoritative decision) | customer-mgmt-service, via a live middleware check, called synchronously by the mobile app |
| Rendering the nudge UI, and reporting shown/clicked/dismissed back to MoEngage | UP by CBD mobile app, only after customer-mgmt-service confirms eligibility |
| Live account/transaction data (balance, credit transactions) | Middleware (core banking facade) |
| Catching funding that happens while the app is never reopened, and reporting it to MoEngage | customer-mgmt-service Scheduled Job |
| Pushing the funded signal and the `role` attribute into MoEngage | customer-mgmt-service (funded signal), Identity/session (role attribute) |
| MoEngage campaign configuration itself (coarse targeting rule, creative, frequency cap) | TBD — see Decisions |

customer-mgmt-service owns the real decision, and is reached **only when MoEngage has already decided the opportunity is worth offering**, which keeps customer-mgmt-service/middleware traffic bounded by MoEngage's own frequency capping instead of by how often the user opens the app.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
flowchart LR
  MOBILE["UP by CBD Mobile App"]
  CMS["customer-mgmt-service"]
  MW["Middleware<br>(core banking facade)"]

  MOBILE -->|"context + eligibility check,<br>payload if offered"| MOE[MoEngage]
  MOBILE -->|"POST /v1/first-funding/check,<br>eligible / not eligible"| CMS
  CMS -->|"account/transaction lookups"| MW
  MW --> CBK[Core Banking]
  CMS -->|"Transaction Completed event"| MOE
:::

customer-mgmt-service now exposes exactly **one** inbound call for this feature — `POST /v1/first-funding/check` — callable only by the mobile app, and only reached after MoEngage has already decided to offer the opportunity.

## Sequence diagram

**Path A — mobile delivery, MoEngage offers, customer-mgmt-service decides live.**

::: mermaid
sequenceDiagram
autonumber
participant front as Mobile App
participant sdk as MoEngage SDK
participant moe as MoEngage
participant bff as customer-mgmt-service
participant db as customer-mgmt-db
participant mw as Middleware

Note left of front: Home screen resumes

front ->> sdk: setInAppContext(["home_screen"])
front ->> sdk: getSelfHandledInApp()
Note right of sdk: checked against the SDK's local eligible-campaigns cache,<br>refreshed via a meta API call to MoEngage at most once every 15 min<br>during an active session, immediately on new session start/app relaunch<br>(confirmed in MoEngage's SDK docs) — repeated visits within that window<br>replay the same cached decision, see Decisions item 2
sdk ->> sdk: check cache: role == OWNER<br>AND context contains "home_screen"<br>AND within frequency cap<br>(optionally also "has not tracked Transaction Completed" —<br>see Decisions item 1)

alt offered
    sdk -->> sdk: match found in cache (first_funding)
    sdk -->> front: onSelfHandledAvailable(payload)

    front ->> bff: POST /v1/first-funding/check { company_rim }
    bff ->> db: get row(company_rim)

    alt row.status == COMPLETED
        bff -->> front: not eligible (no middleware call)
    else row missing or PENDING
        bff ->> mw: GetAccountSummaryByIrim / PartyAcctRelInq / SearchAccountTransactions
        mw -->> bff: accounts, balances, transactions

        alt credit transaction or non-zero balance found
            bff ->> db: conditional upsert status = COMPLETED<br>(WHERE status != COMPLETED)
            alt upsert changed the row
                bff ->> moe: POST /v1/event — Transaction Completed
            end
            bff -->> front: not eligible
        else none found
            bff ->> db: upsert status = PENDING (if absent)
            bff -->> front: eligible
        end
    end

    alt front received "eligible"
        front ->> front: render nudge UI
        front ->> sdk: selfHandledShown(payload)
        sdk ->> moe: report impression
    else front received "not eligible"
        front ->> front: discard payload, nothing rendered,<br>nothing reported (see Decisions item 2)
    end
else not offered (role mismatch, context miss, or frequency capped)
    sdk -->> sdk: no match in cache
    sdk -->> front: (no callback fires)
end

Note left of front: Home screen stops (onStop)
front ->> sdk: resetInAppContext()
:::

**Path B — customer-mgmt-service Scheduled Job, catches funding when the app is never reopened to trigger Path A again.**

::: mermaid
sequenceDiagram
autonumber
participant sched as customer-mgmt-service Scheduled Job
participant db as customer-mgmt-db
participant mw as Middleware
participant moe as MoEngage

loop every poll interval
    sched ->> db: select rows where status = PENDING
    db -->> sched: pending company_rims
    loop for each pending row
        sched ->> mw: GetAccountSummaryByIrim / PartyAcctRelInq / SearchAccountTransactions
        mw -->> sched: result
        alt credit transaction or non-zero balance found
            sched ->> db: conditional update status = COMPLETED<br>(WHERE status != COMPLETED)
            alt update changed the row
                sched ->> moe: POST /v1/event — Transaction Completed
            end
        else none found
            sched ->> db: update last_checked_at
        end
    end
end
:::

Path B only has anything to do once at least one `PENDING` row exists — and the only thing that creates a `PENDING` row in this design is Path A actually running once. See Decisions item 6 for the gap that follows from that.

# 3. Design & Implementation

## User Journey

_(link to figma — TBD)_

OWNER-role users of a newly onboarded, unfunded company see a nudge on `home`, gone for good once the company is funded. The difference here is that the nudge is only ever rendered after a fresh, live check — MoEngage's own targeting is never trusted as the final word.

## Backlog

_(link to backlog — TBD)_

### Features

#TBD — Fund the Account nudge (`first_funding`), MoEngage-offer + customer-mgmt-service-live-check hybrid

### Decisions

1. **Should MoEngage's coarse targeting also exclude on `Transaction Completed`?** Two options: (a) MoEngage's campaign targets only on `role == OWNER` + context, and relies entirely on customer-mgmt-service's live check to filter out already-funded companies on every single offer; or (b) MoEngage's targeting also excludes `has tracked "Transaction Completed"`, fed by the same event customer-mgmt-service already sends, as a cheap pre-filter that avoids even offering the campaign — and therefore avoids the BFF/middleware round-trip — once it's already known funded. (b) is recommended purely to cut wasted live checks once the answer is already known, but it is not required for correctness: the BFF check is authoritative either way, so a stale "not yet excluded" state in MoEngage only ever costs an extra round-trip, never an incorrect render.
2. **Does declining to render after a "not eligible" BFF response count against MoEngage's frequency capping?** MoEngage's docs don't confirm whether a delivered-but-unshown campaign still counts against frequency cap/priority. Here it matters more: if the BFF says "not eligible" and the app doesn't call `selfHandledShown`, and this *doesn't* count against the cap, the campaign could be offered again on every single home-screen revisit for an already-funded company, causing a live check (and a middleware call) each time, until Decision item 1's pre-filter (if adopted) catches up. Confirmed detail from MoEngage's SDK docs (Android/iOS/React Native/Flutter all state this identically): `getSelfHandledInApp()` doesn't make a live call per invocation — it reads the SDK's local eligible-campaigns cache, which is refreshed via a meta API call at most once every 15 minutes during an active session, and immediately on a new session start or app relaunch. So in the worst case, every home-screen revisit *within that 15-minute window* replays the exact same cached decision and triggers another BFF/middleware round-trip — this is bounded to once per 15 minutes per session, not unbounded per visit, but still worth confirming the frequency-cap behavior with MoEngage support.
3. **Context is the screen (`home_screen`), not the nudge.** `first_funding`'s campaign is linked to the `home_screen` context — it's the screen that's tagged, not a nudge-specific tag invented just to disambiguate. This matches the context's actual purpose ("where is the user right now") and matches the existing `NUDGE_PLACEMENT`/screen model used elsewhere in the nudges model. The consequence: `home_screen` is a shared context, so if any other nudge is ever also placed on `home` and configured with the same context, `getSelfHandledInApp()` (singular) is no longer guaranteed to return specifically `first_funding` — MoEngage resolves one eligible campaign per call by priority, not by name. Today, with `first_funding` as the only `home_screen`-context campaign, this is moot. The day a second one is added, switch to `getSelfHandledInApps()` (plural, up to 5) and resolve whether the native payload exposes a campaign identifier the app can use to tell them apart, before relying on this pattern for more than one nudge per screen.
4. **BFF endpoint ownership and shape.** `POST /v1/first-funding/check` is surface area on customer-mgmt-service, specific to this feature. Confirm whether this should be a dedicated endpoint per nudge (as drafted here) or a more general "live-check" endpoint parameterized by nudge name, if more nudges end up needing the same MoEngage-offers/backend-confirms pattern — which becomes more likely once item 3's shared-context scenario is reached.
5. **Payload schema** for the self-handled campaign (copy, CTA, styling tokens) — needed regardless of where the eligibility check lives.
6. **Role attribute freshness for the coarse MoEngage pre-filter** — lower stakes since role is only used to decide whether to *offer*, not whether to *show*; a stale role on MoEngage's side can at most cause a wasted BFF call (BFF has no independent role check in this design — see Decisions item 8). This staleness compounds with the SDK's own eligible-campaigns cache (see Decision item 2): even once MoEngage has the updated role attribute, a given device won't see its effect until that device's next cache refresh, up to 15 minutes later in an active session.
7. **Gap: nothing seeds a `PENDING` row until Path A runs once.** If a company is never visited on `home` by an OWNER session after onboarding (e.g. the user only ever uses a different screen, or churns before opening the app again), no `PENDING` row is ever created, the scheduled job has nothing to check, and `Transaction Completed` never reaches MoEngage even if the company does eventually get funded through another channel. If anything outside this mobile-triggered flow depends on that event (e.g. a push/email escalation Flow in MoEngage), it will never stop for these companies. Decide whether an independent seed — e.g. a one-time check fired from onboarding completion — is still needed alongside this design, specifically to guarantee scheduled-job coverage even when the mobile flow never runs.
8. **Should the BFF re-check role, or trust MoEngage already did?** This design's `POST /v1/first-funding/check` call doesn't re-validate OWNER role — it assumes MoEngage's targeting already filtered on that. Confirm this is acceptable, or whether role should be re-checked server-side in the BFF call too (defense in depth against a stale/forged client call), given the BFF endpoint is a real inbound API surface.
9. **Multi-account aggregation semantics, `Irim`/`company_rim` terminology** — not re-litigated here.
10. **No expiry** — a company that never funds stays targetable indefinitely, `PENDING` rows retried forever.
11. **MoEngage event-send failure handling.** MoEngage's documented Event API has no `dedup_id` or other request-level deduplication field (confirmed against the official reference, which defines only `type`, `customer_id`, and `actions[]` with `action`/`attributes`/`platform`/`app_version`/`user_time`/`current_time`). Idempotency for the Path A / Scheduled Job race is instead enforced at `customer-mgmt-db`: the `COMPLETED` transition is a conditional write (`UPDATE ... WHERE status != 'COMPLETED'`), and the MoEngage event is sent only when that write actually changes the row, so at most one send happens per company. Still open: if the event POST itself fails *after* that conditional write already succeeded (row is `COMPLETED` but MoEngage never received the event), decide on a retry/outbox to close that gap — a local outbox table keyed on `company_rim` is the natural fit, re-driven until the POST succeeds.
12. **Resetting the In-App context when leaving `home`.** MoEngage's SDK docs state the context must be cleared via `resetInAppContext()` once the user moves out of it, or it stays active for subsequent screens. Without that call, `home_screen` context would remain set after the user navigates away from Home, so `first_funding` (or any other `home_screen`-context campaign) could be considered eligible for offering on a screen other than Home if that screen doesn't set its own context first. This is covered in the sequence diagram and SDK-call list below.

### Tests

_(link to test suite — TBD)_

- MoEngage offers, row missing, no credit transaction found → BFF returns eligible, row created `PENDING`, app renders and reports `selfHandledShown`.
- MoEngage offers, row missing or `PENDING`, credit transaction found → BFF returns not eligible, row updated `COMPLETED`, MoEngage event sent exactly once, app does not render.
- MoEngage offers, row already `COMPLETED` → BFF returns not eligible without a middleware call.
- MoEngage does not offer (role mismatch, context miss, frequency capped) → no callback fires, BFF is never called.
- Scheduled job: `PENDING` row, credit transaction found on a later cycle → row updated `COMPLETED`, event sent exactly once, independent of whether the app is ever reopened.
- Race: Path A and the scheduled job both detect funding for the same company near-simultaneously → the `customer-mgmt-db` conditional write (`WHERE status != COMPLETED`) lets only the first of the two actually transition the row, so MoEngage receives exactly one `Transaction Completed` event; the loser's write is a no-op and sends nothing.
- Gap case (Decisions item 6): company funded via another channel, no OWNER session ever opens `home` after onboarding → confirm whether this is accepted as a known limitation or whether a seed mechanism is required before launch.
- Middleware failure during the BFF call → no incorrect `COMPLETED` write, response to mobile indicates a retryable state rather than a false "not eligible."
- Context reset (Decisions item 12): user navigates from `home` to another screen that doesn't set its own context → `resetInAppContext()` has cleared `home_screen`, so `first_funding` is not offered there.

## API definition

_(link to swagger — TBD)_

**Mobile → customer-mgmt-service call**, made only after `onSelfHandledAvailable` fires for the `first_funding` campaign:

```
POST /v1/first-funding/check
```
```json
{ "company_rim": "99812345" }
```
```json
// 200 — eligible
{ "eligible": true }
```
```json
// 200 — not eligible
{ "eligible": false }
```

**Backend → MoEngage event**, sent by both the BFF call and the customer-mgmt-service Scheduled Job. There is no documented `dedup_id` or other request-level deduplication field on this API; see Decisions item 11 for how idempotency is actually achieved instead.

Below is MoEngage's own documented sample for this endpoint, reproduced as-is (unmapped) — mapping its fields (`customer_id`, `action`, `attributes`) to this feature's actual values (`company_rim`, `Transaction Completed`, `detected_via`) is still TBD:

```
POST https://api-0X.moengage.com/v1/event/{{appId}}?app_id={{appId}}&project_code={{projectId}}
```
```
Content-Type: application/json
Authorization: Basic Base64_ENCODED_APPKEY_APIKEY
```
```json
{
  "type": "event",
  "customer_id": "123",
  "actions": [
    {
      "action": "ProductAdded",
      "attributes": {
        "product": "Accessories",
        "color": "Black",
        "Brand": "Adidas"
      },
      "platform": "Android",
      "app_version": "1.2.3",
      "user_time": "1708939453396",
      "current_time": "1708939453396"
    }
  ]
}
```

**Mobile → MoEngage SDK calls**:

- `setInAppContext(["home_screen"])` then `getSelfHandledInApp()` — called from `home` screen's `onResume()`. The `first_funding` campaign is linked to the `home_screen` context in the MoEngage dashboard (Decisions item 3).
- `onSelfHandledAvailable(payload)` — callback; triggers the `POST /v1/first-funding/check` call before any rendering decision.
- `selfHandledShown(payload)` / `selfHandledClicked(payload)` / `selfHandledDismissed(payload)` — reported back only when the BFF confirmed eligibility and the app actually rendered.
- `resetInAppContext()` — called from `home` screen's `onStop()` (or platform equivalent), to clear the `home_screen` context when the user navigates away. MoEngage's SDK docs are explicit that the context must be reset when leaving it; see Decisions item 12 for the gap this closes.
