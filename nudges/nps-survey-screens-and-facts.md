# 1. Introduction

This document covers the `nps_survey` nudge: the Feature-1 NPS survey. **The signal to display the survey comes from MoEngage**, using the same Self-Handled In-App pattern as `First-founding-moengage-sdk-delivery-solution-design.md`. `customer-management-service` (Kevlar) keeps exactly two jobs: feeding MoEngage what it needs to target correctly, and storing the submitted result. It is scoped to one question only — **which screens must carry the survey, and what customer-mgmt-service and the data platform must feed MoEngage so its targeting makes the right call on each one.** Reporting (Power BI) is out of scope here.

**Why this is a smaller lift than `first_funding`'s hybrid.** `first_funding` needed a live backend check before every render because a stale "unfunded" signal would show a nudge that's flatly wrong (and costs a wasted middleware round-trip to boot). A stale NPS-survey trigger has no equivalent failure mode — at worst it asks someone slightly too early, too often, or on the wrong screen, all recoverable by a human closing the sheet. That's why this design has no live eligibility check in the render path: MoEngage's own targeting is trusted as the whole decision, not just a coarse pre-filter.

**Ownership.**

| Area | Owner |
|---|---|
| Whether to *offer* the survey at all (targeting, periodic schedule, per-campaign frequency cap/minimum delay, "already responded" exclusion) | MoEngage, evaluated server-side, cached on-device by the SDK (see `first_funding`'s Decisions item 2 for the cache-cadence caveat — applies identically here) |
| Firing `International Transfer Completed`/`Local Transfer Completed` events on every outgoing transfer, so MoEngage's own segmentation does the threshold gating | Mobile app |
| Rendering the survey UI, reporting shown/clicked/dismissed directly to MoEngage — `customer-management-service` is not in this path at all | Mobile app |
| Storing the submitted result (`customer_id`, `score`, `submitted_at`) — the **only** thing `customer-management-service` does for this feature | `customer-management-service` (nudge sub-service), in `customer-mgmt-db` |
| Pushing the result back into MoEngage as a customer attribute, so targeting can exclude recent responders | `customer-management-service` |

Companion to `First-founding-moengage-sdk-delivery-solution-design.md`, which this document follows for the delivery pattern.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
flowchart LR
  subgraph MOBILE["UP by CBD Mobile App"]
    SCR["Screen onResume()<br>(any of the 7 placements)"]
    XFER["Outgoing transfer completes<br>(international/local)"]
    CB_APP["onSelfHandledAvailable(payload)"]
    RENDER["Render survey UI"]
  end

  SDK["MoEngage SDK"]
  MOE[MoEngage]

  subgraph CMS["customer-management-service"]
    API["POST /v1/nps-responses<br>(storage only)"]
  end

  subgraph DB["customer-mgmt-db"]
    LF[("nps.last_response_at<br>nps.last_score")]
  end

  SCR -->|"setInAppContext([screen])<br>+ getSelfHandledInApp()"| SDK
  XFER -->|"trackEvent: International/Local<br>Transfer Completed"| SDK
  SDK -->|"targeting: role + context<br>+ event/attribute filters + frequency cap"| MOE
  MOE -->|"payload, if offered"| SDK --> CB_APP
  CB_APP --> RENDER
  RENDER -->|"selfHandledShown/Clicked/Dismissed<br>(shown, success, skip, dismiss — all of it)"| SDK --> MOE
  RENDER -->|"POST /v1/nps-responses<br>{ customer_id, screen, score, submitted_at }<br>(only on actual submission)"| API
  API --> LF
  API -->|"POST /v1/customer —<br>nps.last_score, nps.last_response_at"| MOE
:::

`customer-management-service` has exactly **one** inbound call for this feature: `POST /v1/nps-responses`, fired only when the user actually submits a score. Shown/clicked/dismissed/skipped all go straight from the mobile app to the MoEngage SDK — `customer-management-service` never sees them, and isn't consulted before rendering either way.

## Sequence diagram

**Path A — MoEngage offers, app renders, result or outcome flows back.**

::: mermaid
sequenceDiagram
autonumber
participant front as Mobile App
participant sdk as MoEngage SDK
participant moe as MoEngage
participant api as customer-management-service
participant db as customer-mgmt-db

Note left of front: Screen resumes (any of the 7 placements)
front ->> sdk: setInAppContext([screen_name])
front ->> sdk: getSelfHandledInApp()
Note right of sdk: local eligible-campaigns cache,<br>refreshed at most once every 15 min per active session<br>(same mechanic as first_funding's Decisions item 2)

alt offered
    sdk -->> front: onSelfHandledAvailable(payload)
    front ->> front: render survey UI
    front ->> sdk: selfHandledShown(payload)
    sdk ->> moe: report impression — counts against MoEngage's own frequency cap

    alt user submits a score
        front ->> sdk: selfHandledClicked(payload)
        sdk ->> moe: report click
        front ->> api: POST /v1/nps-responses<br>{ customer_id, screen, score, submitted_at }<br>(the only call this flow ever makes to customer-management-service)
        api ->> db: upsert nps.last_response_at, nps.last_score
        api ->> moe: POST /v1/customer — nps.last_score, nps.last_response_at
    else user skips or dismisses
        front ->> sdk: selfHandledDismissed(payload)
        sdk ->> moe: report dismissal — counts against MoEngage's own frequency cap.<br>customer-management-service is never called, so no backend-side<br>skip count exists (see Decisions item 3)
    end
else not offered (targeting or MoEngage's own frequency cap excluded)
    sdk -->> front: (no callback fires)
end

Note left of front: Screen stops
front ->> sdk: resetInAppContext()
:::

International and local transfers are outgoing payments triggered from the mobile app, so there's no second path for them: the app fires `International Transfer Completed`/`Local Transfer Completed` directly to MoEngage the moment a transfer completes (see the C2 diagram), and MoEngage's own segmentation — "has executed this event at least N times" — does the threshold gating with no backend involvement at all. Invoice creation doesn't have this guarantee yet; see Decisions item 2 for what's still open there.

# 3. Design & Implementation

## User Journey

_(link to Figma — not yet available)_. Interim reference: `Feature-1.txt` lines 19–51 describe the two-step journey (rating capture, then the promoter/detractor-specific branch) that the mobile app owns; this document only covers how the opportunity to show it reaches the app and what happens to the result.

## Screens and MoEngage triggers

**Design choice.** Each screen below is a distinct **context** for `setInAppContext()`/`getSelfHandledInApp()` — following `first_funding`'s Decisions item 3 (context is the screen, not the nudge). `customer-management-service` only ever learns about an actual submission, never a shown/skip/dismiss, so it has nothing to track and nothing to push beyond "has this customer already responded." Cross-screen repeat-prompt control is entirely MoEngage's own frequency capping and per-campaign delivery controls — see Decisions item 3 for the limits of that.

| Screen | Trigger journey | MoEngage trigger mechanism | What feeds MoEngage |
|---|---|---|---|
| `debit_card_activation_confirmation` | Debit card activation | Event-triggered campaign on "Debit Card Activated" | Backend/data platform fires this event when activation happens |
| `card_limit_increase_confirmation` | Card limit increase, post-consent | Event-triggered campaign on "Card Limit Increased" | Same pattern |
| `international_transfer_confirmation` | First international transfer | Event-triggered campaign on "International Transfer Completed", audience filter "has executed this event at least 1 time" | Mobile app fires this via `trackEvent()` the moment the transfer completes |
| `local_transfer_confirmation` | Third local transfer | Event-triggered campaign on "Local Transfer Completed", audience filter "has executed this event at least 3 times" | Same mechanism; MoEngage's own occurrence count does the threshold check |
| `wps_enrolment_confirmation` | WPS enrolment | Event-triggered campaign on "WPS Enrolled" | Backend/data platform fires this event on enrolment |
| `invoice_creation_confirmation` | Invoice creation | Event-triggered campaign on "Invoice Created" (or "First Invoice Created" — see Decisions item 1) | Backend/data platform fires this event on (first) invoice creation — exact sourcing still open, see Decisions item 2 |
| `home` | Periodic, every 3 months | MoEngage's native periodic/recurring campaign type, context = `home_screen` | Nothing — periodicity is configured directly in MoEngage |

Every campaign above, regardless of trigger type, must additionally target on `role == OWNER` (or whatever role Feature-1 requires) and exclude customers where `nps.last_score`/`nps.last_response_at` is already set — i.e. already responded. That's the only exclusion customer-mgmt-service feeds in; any within-campaign repeat-prompt control (how many times, how far apart) is configured directly on each MoEngage campaign's own delivery controls — see Decisions item 3 and item 5.

**Attributes/events feeding into MoEngage.**

| Name | Kind | Source | Purpose |
|---|---|---|---|
| `nps.last_score`, `nps.last_response_at` | Customer attribute | `customer-management-service`, written synchronously on `POST /v1/nps-responses` | Still the system of record is `customer-mgmt-db` (per the earlier discussion in this conversation — MoEngage attribute writes are asynchronous and there's no sync read-back API suited to this); pushed to MoEngage **write-only** so targeting can exclude customers who already responded |
| `International Transfer Completed`, `Local Transfer Completed` | Event | Mobile app, via `trackEvent()` on every completed outgoing transfer | Drives the two transfer-triggered campaigns; MoEngage's own occurrence-count segmentation does the threshold gating, no backend counting needed |
| `Debit Card Activated`, `Card Limit Increased`, `WPS Enrolled`, `Invoice Created` | Event | Backend/data platform, fired on occurrence | Drives the placement-only campaigns |

**Rule shape for the two transfer-triggered campaigns.** Each campaign's audience filter uses MoEngage's event-occurrence operator — `atleast`, not `exactly` — so a user who racks up several transfers in quick succession still qualifies once they cross the threshold, with no risk of skipping past the exact count. This is evaluated entirely inside MoEngage; nothing on the backend counts transfers.

## Backlog

_(Provide the feature backlog)_

### Features

#TBD — add the Jira/ADO ticket(s) covering `nps_survey` (MoEngage campaign configuration per screen, mobile-app transfer-completion tracking, response storage)

### Decisions

_(Provide decision link — until a formal decision log exists, tracked here)_

1. **Invoice creation: first-ever, or every invoice?** Decides whether the "Invoice Created" event should be filtered to the first occurrence only (data-platform-side) or fired on every invoice (letting MoEngage's campaign fire every time, subject to the "already responded" exclusion and the campaign's own frequency cap/minimum delay).
2. **How does "Invoice Created" actually get fired?** Unlike the two transfer triggers, invoice creation isn't confirmed as something the mobile app can fire directly the moment it happens — whatever creates the invoice (presumably a backend invoicing service) would need to know synchronously whether it's that customer's first-ever invoice, firing the event itself the same way the mobile app now does for transfers. If that's not available, this still needs a batch (T-1) or stream-sourced detection step feeding MoEngage instead, with the usual one-day-lag tradeoff attached to the batch option.
3. **The 3-skip-then-park suppression policy (Feature-1 line 58's "prevent overlapping or repeated prompts") is not replicated here.** Shown/clicked/dismissed/skipped are reported by the mobile app straight to the MoEngage SDK; `customer-management-service` is never called for any of them, so it has no skip count to track and nothing to push beyond "has already responded." The only remaining repeat-prompt control is MoEngage's own per-campaign frequency cap, which is **device-level, not customer-level** (confirmed in `first_funding`'s research) — so a multi-device OWNER who dismisses on one device can still be re-offered on another, and there is no customer-level "N skips, then pause" behavior at all, only "at most N shows per device per period." Confirm this gap is acceptable before launch; if not, the only fix is routing outcome events back through `customer-management-service`, which means building a suppression tracker there after all.
4. **Attribute push latency.** MoEngage's own docs confirm customer attribute writes are processed asynchronously with no guaranteed freshness window. A user who just submitted a response could, in principle, still be re-offered the survey on a near-simultaneous second screen visit before `nps.last_response_at` propagates. Low-stakes here (unlike `first_funding`'s equivalent risk) — worst case is one redundant prompt, not an incorrect financial decision — but worth confirming is acceptable rather than assumed.
5. **`min_interval` value.** Configured directly on each campaign's MoEngage delivery controls ("add a minimum delay between two messages of this campaign"; confirmed available for the Self-Handled template type, unlike MoEngage's separate "Nudges" template type where that setting is disabled). Still device-level, same caveat as Decisions item 3; still needs an actual number, not `none`.
6. **Does `customer_id` for this feature match the identifier MoEngage already uses elsewhere?** The payload in §API definition assumes `customer_id` is whatever identifier `customer-management-service` already uses to address the customer in MoEngage (see `first_funding`'s Decisions item 9 on `company_rim`/`personal_rim` — not re-litigated here, but the same identifier choice must apply to keep `nps.last_score` etc. attached to the same MoEngage profile the campaigns target).

### Tests

_(Provide tests link — not yet created)_

- MoEngage offers on `local_transfer_confirmation` (3rd-transfer event already received, not frequency-capped) → `onSelfHandledAvailable` fires, app renders.
- MoEngage does not offer (`nps.last_response_at` already set, or frequency capped) → no callback fires, `customer-management-service` is never called.
- User submits a score → `POST /v1/nps-responses` is the only call made to `customer-management-service`; it stores `customer_id`, `score`, `submitted_at`, and `nps.last_score`/`nps.last_response_at` are pushed to MoEngage.
- User skips or dismisses → `selfHandledDismissed` is reported to the MoEngage SDK only; `customer-management-service` receives no call and its DB is unchanged.
- User dismisses the survey repeatedly across different screens on the same device → MoEngage's own frequency cap eventually stops offering it on that device; confirm this is accepted in place of a customer-level skip budget (Decisions item 3).
- User completes their 1st international transfer → mobile app fires `International Transfer Completed` via `trackEvent()`; MoEngage's "at least 1 time" segmentation now matches, so `international_transfer_confirmation`'s campaign becomes eligible.
- User completes their 2nd local transfer → `trackEvent()` fires, but MoEngage's "at least 3 times" segmentation still doesn't match — campaign stays ineligible.
- User completes their 3rd local transfer → segmentation now matches; the campaign becomes eligible on the next visit to `local_transfer_confirmation`.
- Race: user responds on one device while a second device is mid-render of the same campaign → acceptable per Decisions item 4; confirm the second device's render doesn't itself error, only potentially duplicates the prompt.

## API definition

_(link to Swagger — not yet published)_

**`customer-management-service`'s only inbound call for this feature:**
- `POST /v1/nps-responses` — `customer_id` is required in the payload alongside the score, so the result can be stored against the right customer:

```json
{
  "customer_id": "99812345",
  "screen": "local_transfer_confirmation",
  "score": 9,
  "submitted_at": "2026-11-12T10:15:00Z"
}
```

Stores `nps.last_response_at`/`nps.last_score` in `customer-mgmt-db` (system of record — not MoEngage, per the write-only rationale in §1). Nothing else is tracked server-side for this feature.

**`customer-management-service` → MoEngage attribute push**, same shape and endpoint as `first_funding`'s customer-attribute call:

```
POST https://api-0X.moengage.com/v1/customer/{{appId}}?app_id={{appId}}
```
```json
{
  "type": "customer",
  "customer_id": "99812345",
  "attributes": {
    "nps_last_score": 9,
    "nps_last_response_at": "2026-11-12T10:15:00Z"
  }
}
```

**Mobile → MoEngage SDK calls:**

- `trackEvent("International Transfer Completed", { ... })` / `trackEvent("Local Transfer Completed", { ... })` — called by the mobile app the moment an outgoing transfer completes, independent of any of the 7 placement screens. MoEngage's own segmentation gates `international_transfer_confirmation`'s and `local_transfer_confirmation`'s campaigns on this event's occurrence count (`atleast 1` / `atleast 3`) — see §3's rule-shape note. No backend call involved.
- `setInAppContext([screen_name])` then `getSelfHandledInApp()` — called from each placement screen's `onResume()`.
- `onSelfHandledAvailable(payload)` — callback; triggers rendering directly, with no backend call in between (contrast with `first_funding`, where this callback triggers a live check instead).
- `selfHandledShown(payload)` / `selfHandledClicked(payload)` / `selfHandledDismissed(payload)` — reported to MoEngage on every render, regardless of outcome. This is the **only** place shown/clicked/dismissed/skipped are recorded — there is no equivalent backend call (see Decisions item 3).
- `resetInAppContext()` — called from each placement screen's `onStop()` (or platform equivalent), per `first_funding`'s Decisions item 12.
