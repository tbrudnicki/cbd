# 1. Introduction

**Feature.** Third iteration of the `first_funding` nudge design (see `First-founding-solution-design.md` for the original lifecycle-engine baseline, and `First-founding-moengage-solution-design.md` for the Kevlar-owned-decision variant). This version is a hybrid of the previous two:

- **MoEngage decides *when to offer* the nudge** — via the native **Self-Handled In-App ("InApp NATIV")** mechanism described in the previous revision of this document. MoEngage's targeting is coarse (role, context, frequency capping) and is allowed to be stale.
- **The mobile app does not render on MoEngage's say alone.** When `getSelfHandledInApp()` hands it the `first_funding` payload, the app calls a **BFF (`customer-management-service` / Kevlar)** endpoint to perform a **live eligibility check** against core banking before rendering anything. The eligibility decision — "is this company actually still unfunded, right now" — lives entirely in that BFF call, not in MoEngage's targeting rule and not in the mobile app's own logic beyond "did the BFF say yes."
- That same BFF call is also what feeds MoEngage the stop signal: if the live check finds a credit transaction, it tells the app not to render, **and** sends the `Transaction Completed` event to MoEngage. If it finds nothing, it tells the app to render, **and** schedules a background check that will send that same event later, whenever funding eventually happens — covering the case where the user never triggers another live check by reopening the app.

So MoEngage is the delivery/trigger mechanism and the destination for the stop signal; Kevlar is the sole source of truth for whether the company is actually funded, checked live, every time MoEngage offers the nudge.

**Ownership.**

| Concern | Owner |
|---|---|
| Whether to *offer* the nudge opportunity at all (coarse targeting, context, frequency capping) | MoEngage, evaluated server-side on each SDK call — allowed to be stale |
| Whether the nudge is *actually* eligible right now (the real, authoritative decision) | Kevlar, via a live middleware check, called synchronously by the mobile app |
| Rendering the nudge UI, and reporting shown/clicked/dismissed back to MoEngage | UP by CBD mobile app, only after Kevlar confirms eligibility |
| Live account/transaction data (balance, credit transactions) | Middleware (core banking facade) |
| Catching funding that happens while the app is never reopened, and reporting it to MoEngage | Kevlar scheduled poller |
| Pushing the funded signal and the `role` attribute into MoEngage | Kevlar (funded signal), Identity/session (role attribute) |
| MoEngage campaign configuration itself (coarse targeting rule, creative, frequency cap) | TBD — see Decisions |

Kevlar is back to owning the real decision — same as the original baseline design — but it's no longer reached via a generic `GET /v1/nudges` call on every screen load. It's reached **only when MoEngage has already decided the opportunity is worth offering**, which keeps Kevlar/middleware traffic bounded by MoEngage's own frequency capping instead of by how often the user opens the app.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
flowchart LR
  subgraph MOBILE["UP by CBD Mobile App"]
    HOME["Home screen<br>onResume()"]
    CB_APP["onSelfHandledAvailable(payload)"]
    RENDER["Render nudge UI"]
  end

  subgraph KV["Kevlar (customer-management-service)"]
    BFF["BFF: live eligibility check<br>(POST /v1/first-funding/check)"]
    SCHED["Scheduled Poller"]
    DB[("FUNDING_CHECK<br>status table")]
  end

  HOME -->|"setInAppContext(['home_screen'])<br>+ getSelfHandledInApp()"| SDK["MoEngage SDK"]
  SDK -->|"coarse eligibility:<br>role + context='home_screen' + frequency cap"| MOE[MoEngage]
  MOE -->|"payload, if offered"| SDK --> CB_APP
  CB_APP -->|"POST /v1/first-funding/check"| BFF

  BFF --> DB
  SCHED --> DB
  BFF -->|"GetAccountSummaryByIrim,<br>PartyAcctRelInq,<br>SearchAccountTransactions"| MW["Middleware<br>(core banking facade)"]
  SCHED -->|"same calls, on schedule"| MW
  MW --> CBK[Core Banking]

  BFF -->|"eligible / not eligible"| CB_APP
  CB_APP -->|"if eligible"| RENDER
  RENDER -->|"selfHandledShown"| SDK --> MOE

  BFF -->|"POST /v1/event — Transaction Completed<br>(if funded now)"| MOE
  SCHED -->|"POST /v1/event — Transaction Completed<br>(if funded later)"| MOE

  ID["Identity / Session"] -->|"POST /v1/customer — role attribute"| MOE
:::

Kevlar now exposes exactly **one** inbound call for this feature — `POST /v1/first-funding/check` — callable only by the mobile app, and only reached after MoEngage has already decided to offer the opportunity.

## Sequence diagram

**Path A — mobile delivery, MoEngage offers, Kevlar decides live.**

::: mermaid
sequenceDiagram
autonumber
participant front as Mobile App
participant sdk as MoEngage SDK
participant moe as MoEngage
participant bff as Kevlar BFF
participant db as FUNDING_CHECK
participant mw as Middleware

Note left of front: Home screen resumes

front ->> sdk: setInAppContext(["home_screen"])
front ->> sdk: getSelfHandledInApp()
sdk ->> moe: resolve eligible campaigns for this user
Note right of moe: coarse targeting = role == OWNER<br>AND context contains "home_screen"<br>AND within frequency cap<br>(optionally also "has not tracked Transaction Completed" —<br>see Decisions item 1)

alt offered
    moe -->> sdk: campaign payload (first_funding)
    sdk -->> front: onSelfHandledAvailable(payload)

    front ->> bff: POST /v1/first-funding/check { company_rim }
    bff ->> db: get row(company_rim)

    alt row.status == COMPLETED
        bff -->> front: not eligible (no middleware call)
    else row missing or PENDING
        bff ->> mw: GetAccountSummaryByIrim / PartyAcctRelInq / SearchAccountTransactions
        mw -->> bff: accounts, balances, transactions

        alt credit transaction or non-zero balance found
            bff ->> db: upsert status = COMPLETED
            bff ->> moe: POST /v1/event — Transaction Completed (dedup_id)
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
    moe -->> sdk: no campaign
    sdk -->> front: (no callback fires)
end
:::

**Path B — scheduled poller, catches funding when the app is never reopened to trigger Path A again.**

::: mermaid
sequenceDiagram
autonumber
participant sched as Scheduled Poller
participant db as FUNDING_CHECK
participant mw as Middleware
participant moe as MoEngage

loop every poll interval
    sched ->> db: select rows where status = PENDING
    db -->> sched: pending company_rims
    loop for each pending row
        sched ->> mw: GetAccountSummaryByIrim / PartyAcctRelInq / SearchAccountTransactions
        mw -->> sched: result
        alt credit transaction or non-zero balance found
            sched ->> db: update status = COMPLETED
            sched ->> moe: POST /v1/event — Transaction Completed (dedup_id)
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

Same user-visible outcome as the prior two designs: OWNER-role users of a newly onboarded, unfunded company see a nudge on `home`, gone for good once the company is funded. The difference here is that the nudge is only ever rendered after a fresh, live check — MoEngage's own targeting is never trusted as the final word.

## Backlog

_(link to backlog — TBD)_

### Features

#TBD — Fund the Account nudge (`first_funding`), MoEngage-offer + Kevlar-live-check hybrid

### Decisions

1. **Should MoEngage's coarse targeting also exclude on `Transaction Completed`?** Two options: (a) MoEngage's campaign targets only on `role == OWNER` + context, and relies entirely on Kevlar's live check to filter out already-funded companies on every single offer; or (b) MoEngage's targeting also excludes `has tracked "Transaction Completed"`, fed by the same event Kevlar already sends, as a cheap pre-filter that avoids even offering the campaign — and therefore avoids the BFF/middleware round-trip — once it's already known funded. (b) is recommended purely to cut wasted live checks once the answer is already known, but it is not required for correctness: the BFF check is authoritative either way, so a stale "not yet excluded" state in MoEngage only ever costs an extra round-trip, never an incorrect render.
2. **Does declining to render after a "not eligible" BFF response count against MoEngage's frequency capping?** Same open question as the prior revision — MoEngage's docs don't confirm whether a delivered-but-unshown campaign still counts against frequency cap/priority. Here it matters more: if the BFF says "not eligible" and the app doesn't call `selfHandledShown`, and this *doesn't* count against the cap, the campaign could be offered again on literally the next app open, causing a live check (and a middleware call) on every single home-screen visit for an already-funded company until Decision item 1's pre-filter (if adopted) catches up. Confirm with MoEngage support.
3. **Context is the screen (`home_screen`), not the nudge.** `first_funding`'s campaign is linked to the `home_screen` context — it's the screen that's tagged, not a nudge-specific tag invented just to disambiguate. This matches the context's actual purpose ("where is the user right now") and matches the existing `NUDGE_PLACEMENT`/screen model used elsewhere in the nudges model. The consequence: `home_screen` is a shared context, so if any other nudge is ever also placed on `home` and configured with the same context, `getSelfHandledInApp()` (singular) is no longer guaranteed to return specifically `first_funding` — MoEngage resolves one eligible campaign per call by priority, not by name. Today, with `first_funding` as the only `home_screen`-context campaign, this is moot. The day a second one is added, switch to `getSelfHandledInApps()` (plural, up to 5) and resolve the earlier-flagged open question — whether the native payload exposes a campaign identifier the app can use to tell them apart — before relying on this pattern for more than one nudge per screen.
4. **BFF endpoint ownership and shape.** `POST /v1/first-funding/check` is new surface area on Kevlar, specific to this feature, replacing the generic `GET /v1/nudges?screen=home` contract entirely for `first_funding`. Confirm whether this should be a dedicated endpoint per nudge (as drafted here) or a more general "live-check" endpoint parameterized by nudge name, if more nudges end up needing the same MoEngage-offers/backend-confirms pattern — which becomes more likely once item 3's shared-context scenario is reached.
5. **Payload schema** for the self-handled campaign (copy, CTA, styling tokens) — same open item as the prior revision; needed regardless of where the eligibility check lives.
6. **Role attribute freshness for the coarse MoEngage pre-filter** — same as the prior revision, lower stakes now since role is only used to decide whether to *offer*, not whether to *show*; a stale role on MoEngage's side can at most cause a wasted BFF call (BFF has no independent role check in this design — see Decisions item 8).
7. **Gap: nothing seeds a `PENDING` row until Path A runs once.** If a company is never visited on `home` by an OWNER session after onboarding (e.g. the user only ever uses a different screen, or churns before opening the app again), no `PENDING` row is ever created, the poller has nothing to check, and `Transaction Completed` never reaches MoEngage even if the company does eventually get funded through another channel. If anything outside this mobile-triggered flow depends on that event (e.g. a push/email escalation Flow in MoEngage), it will never stop for these companies. Decide whether an independent seed — e.g. a one-time check fired from onboarding completion, as in the prior revision's "Account Activated" trigger — is still needed alongside this design, specifically to guarantee poller coverage even when the mobile flow never runs.
8. **Should the BFF re-check role, or trust MoEngage already did?** This design's `POST /v1/first-funding/check` call doesn't re-validate OWNER role — it assumes MoEngage's targeting already filtered on that. Confirm this is acceptable, or whether role should be re-checked server-side in the BFF call too (defense in depth against a stale/forged client call), given the BFF endpoint is now a real inbound API surface, unlike the SDK-only prior revision.
9. **Multi-account aggregation semantics, `Irim`/`company_rim` terminology** — same as all prior designs; not re-litigated here.
10. **No expiry** — same as all prior designs; a company that never funds stays targetable indefinitely, `PENDING` rows retried forever.
11. **MoEngage event-send failure handling** — same as both prior alternative designs; decide on at-least-once delivery with `dedup_id` vs. a local outbox/retry if the event send fails after the DB row is already `COMPLETED`.

### Tests

_(link to test suite — TBD)_

- MoEngage offers, row missing, no credit transaction found → BFF returns eligible, row created `PENDING`, app renders and reports `selfHandledShown`.
- MoEngage offers, row missing or `PENDING`, credit transaction found → BFF returns not eligible, row updated `COMPLETED`, MoEngage event sent exactly once, app does not render.
- MoEngage offers, row already `COMPLETED` → BFF returns not eligible without a middleware call.
- MoEngage does not offer (role mismatch, context miss, frequency capped) → no callback fires, BFF is never called.
- Poller: `PENDING` row, credit transaction found on a later cycle → row updated `COMPLETED`, event sent exactly once, independent of whether the app is ever reopened.
- Race: Path A and the poller both detect funding for the same company near-simultaneously → MoEngage receives at most one effective event (`dedup_id`), DB update is idempotent.
- Gap case (Decisions item 6): company funded via another channel, no OWNER session ever opens `home` after onboarding → confirm whether this is accepted as a known limitation or whether a seed mechanism is required before launch.
- Middleware failure during the BFF call → no incorrect `COMPLETED` write, response to mobile indicates a retryable state rather than a false "not eligible."

## API definition

_(link to swagger — TBD)_

**Retired for this feature:** `GET /v1/nudges?screen=home` is not used by `first_funding` in this design.

**New: mobile → Kevlar BFF call**, made only after `onSelfHandledAvailable` fires for the `first_funding` campaign:

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

**Backend → MoEngage event**, sent by both the BFF call and the scheduled poller, unchanged from the prior revision:

```
POST https://api-{dc}.moengage.com/v1/event/{Workspace_ID}
```
```json
{
  "type": "event",
  "customer_id": "company_rim:{company_rim}",
  "dedup_id": "funded_{company_rim}",
  "actions": [
    {
      "action": "Transaction Completed",
      "attributes": { "detected_via": "bff_check | scheduled_poll" },
      "current_time": "{timestamp of the detected transaction}"
    }
  ]
}
```

**Mobile → MoEngage SDK calls**, unchanged from the prior revision:

- `setInAppContext(["home_screen"])` then `getSelfHandledInApp()` — called from `home` screen's `onResume()`. The `first_funding` campaign is linked to the `home_screen` context in the MoEngage dashboard (Decisions item 3).
- `onSelfHandledAvailable(payload)` — callback; triggers the `POST /v1/first-funding/check` call before any rendering decision.
- `selfHandledShown(payload)` / `selfHandledClicked(payload)` / `selfHandledDismissed(payload)` — reported back only when the BFF confirmed eligibility and the app actually rendered.
