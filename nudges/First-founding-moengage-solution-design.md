# 1. Introduction

**Feature.** Alternative design for the `first_funding` nudge (see `First-founding-solution-design.md` for the baseline). This version removes the generic nudge lifecycle engine (`CUSTOMER_NUDGE`, `CUSTOMER_NUDGE_HISTORY`, `CUSTOMER_LIVE_FACT`, rule classes, `facts.get`/`facts.live`) and replaces it with a purpose-built mechanism: a synchronous check at request time, a small per-feature status table, a scheduled poller for the case where the customer never reopens the app, and MoEngage as the system of record for cross-channel (push/email) escalation and stop-propagation.

The in-app eligibility decision stays entirely owned by Kevlar (`customer-management-service`) — the mobile app's home-screen nudge is not rendered by a MoEngage in-app campaign. MoEngage is used only as the destination for the "funded" signal, so that any push/email escalation built on top of it (e.g. MoEngage's own documented "First Transaction Activation Flow" pattern) starts and stops in sync with the in-app decision, without Kevlar having to own multi-channel delivery itself.

**Ownership.**

| Concern | Owner |
|---|---|
| In-app `first_funding` eligibility decision (sync check + poller) | Nudge sub-service (`customer-management-service` / Kevlar) |
| `FIRST_FUNDING_CHECK` status table | Kevlar |
| Live account/transaction data (balance, credit transactions) | Middleware (core banking facade) |
| User role (OWNER / non-OWNER) within the company | Identity/session — authorization, not a customer fact |
| Cross-channel (push/email) escalation once funded signal is known | MoEngage, fed by Kevlar's event |
| Presentation of the in-app nudge | UP by CBD mobile app, driven directly by Kevlar's API response |

Kevlar remains the only writer of `FIRST_FUNDING_CHECK`. Middleware is a read-only input. MoEngage is a one-way downstream consumer of the "funded" signal — nothing flows back from MoEngage into the in-app eligibility decision.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
flowchart LR
  APP["UP by CBD<br>Mobile App"] -->|"GET /v1/nudges?screen=home"| API
  subgraph KV["Nudge sub-service (customer-management-service / Kevlar)"]
    API[Nudge API]
    CHK["FirstFundingCheck<br>(sync, per request)"]
    SCHED["Scheduled Poller<br>(async, catches never-reopened app)"]
    DB[("FIRST_FUNDING_CHECK<br>status table")]
  end
  API --> CHK
  CHK --> DB
  SCHED --> DB
  CHK -->|"GetAccountSummaryByIrim,<br>PartyAcctRelInq,<br>SearchAccountTransactions"| MW["Middleware<br>(core banking facade)"]
  SCHED -->|"same calls, on schedule"| MW
  MW --> CB[Core Banking]
  CHK -->|"POST /v1/event<br>(funded signal)"| MOE[MoEngage]
  SCHED -->|"POST /v1/event<br>(funded signal)"| MOE
  MOE -->|"Flow: push + email escalation<br>exits on same event"| CH["Push / Email channels"]
:::

No shared nudge-model schema is used here — this is a single-purpose table and two code paths (sync check, scheduled poller), not an instance of the generic rule engine in `nudges-model.md`.

## Sequence diagram

**Path A — mobile app open, synchronous check.**

::: mermaid
sequenceDiagram
autonumber
participant front as UP by CBD<br>Mobile App
participant api as Kevlar Nudge API
participant db as FIRST_FUNDING_CHECK
participant mw as Middleware
participant moe as MoEngage

front ->> api: GET /v1/nudges?screen=home

opt session role is not OWNER
    api -->> front: nudges: []
end

api ->> db: get row(company_rim)

alt row.status == COMPLETED
    api -->> front: nudges: [] (no middleware call)
else row missing or status == PENDING
    api ->> mw: GetAccountSummaryByIrim(company_rim)
    mw -->> api: accounts for company_rim

    loop for each account, until one qualifies
        api ->> mw: PartyAcctRelInq (balance)
        mw -->> api: available balance
        api ->> mw: SearchAccountTransactions [filter=CREDIT]
        mw -->> api: credit transactions
    end

    alt credit transaction or non-zero balance found
        api ->> db: upsert status = COMPLETED
        api ->> moe: POST /v1/event — funded signal (dedup_id = company_rim)
        api -->> front: nudges: []
    else none found
        api ->> db: upsert status = PENDING (if row absent)
        api -->> front: nudges: [{ "name": "first_funding" }]
    end
end
:::

**Path B — scheduled poller, catches customers who never reopen the app.**

::: mermaid
sequenceDiagram
autonumber
participant sched as Scheduled Poller
participant db as FIRST_FUNDING_CHECK
participant mw as Middleware
participant moe as MoEngage

loop every poll interval
    sched ->> db: select rows where status = PENDING
    db -->> sched: pending company_rims

    loop for each pending row
        sched ->> mw: GetAccountSummaryByIrim(company_rim)
        mw -->> sched: accounts
        loop for each account, until one qualifies
            sched ->> mw: PartyAcctRelInq / SearchAccountTransactions [CREDIT]
            mw -->> sched: balance / credit transactions
        end
        alt credit transaction or non-zero balance found
            sched ->> db: update status = COMPLETED
            sched ->> moe: POST /v1/event — funded signal (dedup_id = company_rim)
        else none found
            sched ->> db: update last_checked_at
        end
    end
end
:::

Both paths call the same middleware endpoints and write to the same table, so a customer who gets funded between two app opens is still caught — and still only ever sends the MoEngage event once, since both paths gate on `status != COMPLETED` before acting, and the event carries a `dedup_id` as a second line of defense.

The MoEngage event name should match whatever push/email Flow consumes it. If that Flow is MoEngage's own documented "First Transaction Activation Flow" template, its exit condition is a `Transaction Completed`-named event — naming this signal the same way lets that template be used unmodified instead of configuring a bespoke exit event.

# 3. Design & Implementation

## User Journey

_(link to figma — TBD)_

Same scope as the baseline design: `first_funding` on the `home` screen, OWNER-role users only, newly onboarded company with no funded account. The difference is entirely in how eligibility is computed and cached, not in what the user sees.

## Backlog

_(link to backlog — TBD)_

### Features

#TBD — Fund the Account nudge (`first_funding`), MoEngage-escalation variant

### Decisions

1. **Synchronous check runs on every request while `PENDING`, not just once.** Unlike the baseline design's 15-minute TTL cache, this design re-queries middleware on every `GET /v1/nudges?screen=home` call until the row is `COMPLETED`. For customers who open the app frequently before funding, this is more middleware traffic than the TTL-based design. Confirm this is acceptable, or add a short local throttle (e.g. "skip the sync check if `last_checked_at` was under N minutes ago, just return `PENDING` → eligible") so the sync path doesn't duplicate the poller's job on every single screen load.
2. **Poll interval for the scheduled poller.** Needs to balance "how long can a funded company keep seeing the nudge if they never reopen the app" against middleware call volume across the full `PENDING` backlog. No value proposed yet.
3. **Row lifecycle.** Proposed: keep the row with `status = COMPLETED` indefinitely (never delete) so the sync path can short-circuit without a middleware call forever after. Confirm retention/cleanup policy is acceptable, or whether completed rows should be purged after some period once the MoEngage event has been durably delivered.
4. **Multi-account aggregation semantics** (same as baseline, reused): "funded" = any account under `company_rim` with a non-zero balance or a credit transaction. Confirm for multi-currency/multi-account-type companies.
5. **Role gating is not a customer fact** (same as baseline): enforced at the API/controller layer from session, before `FirstFundingCheck` ever runs — not duplicated per feature.
6. **Terminology reconciliation** (same as baseline): confirm `Irim` used by middleware calls maps 1:1 to `company_rim`.
7. **Event naming and ownership of the push/email Flow.** Decide whether the event this design sends to MoEngage is named to match MoEngage's own "First Transaction Activation Flow" template (`Transaction Completed`) so that template can be reused unmodified for push/email escalation, or whether a differently-named, feature-specific event is preferred. Also confirm who owns configuring and maintaining that Flow in MoEngage — this design only specifies that Kevlar is responsible for firing the event, not for the Flow's content.
8. **No expiry.** As with the baseline design, there is no mechanism here to ever stop showing the nudge to a company that never funds — the row stays `PENDING` and the poller keeps retrying indefinitely. Confirm whether a display cap is still needed.
9. **Failure handling for the MoEngage event send.** If the `POST /v1/event` call to MoEngage fails after the row has been marked `COMPLETED` in the local DB, the funded signal is lost unless retried. Decide whether the event send must succeed before the DB write commits (at-least-once delivery, possible duplicate sends — mitigated by `dedup_id`), or whether a local outbox/retry mechanism is needed so a MoEngage outage can't silently drop the signal.

### Tests

_(link to test suite — TBD)_

- Sync path: row absent, no credit transactions found → row created as `PENDING`, response is `eligible`, no MoEngage event sent.
- Sync path: row absent or `PENDING`, credit transaction found → row updated to `COMPLETED`, MoEngage event sent exactly once, response is `not eligible`.
- Sync path: row already `COMPLETED` → no middleware call made, response is `not eligible`.
- Poller: `PENDING` row, no credit transactions found → row unchanged except `last_checked_at`, no event sent.
- Poller: `PENDING` row, credit transaction found → row updated to `COMPLETED`, MoEngage event sent exactly once.
- Race condition: sync path and poller both detect funding for the same company near-simultaneously → MoEngage receives at most one effective event (verify `dedup_id` behavior), and the DB update is idempotent.
- Middleware failure during sync check → `PENDING_CHECK`-style response, no incorrect `COMPLETED` write, retried on next request.
- Middleware failure during poller run → row left `PENDING`, retried next poll cycle, no partial state.
- MoEngage event send failure after DB marked `COMPLETED` → confirm whatever retry/outbox behavior is decided in Decisions item 9 actually recovers the signal.
- Role filter: non-OWNER session → empty nudges, regardless of row state, no middleware call.

## API definition

_(link to swagger — TBD)_

No new endpoint for the mobile app. Uses the same existing contract as the baseline design:

```
GET /v1/nudges?screen=home
```

```json
{
  "screen": "home",
  "nudges": [
    { "name": "first_funding" }
  ]
}
```

New outbound call, Kevlar → MoEngage, on both the sync and poller paths:

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
      "attributes": { "detected_via": "sync_check | scheduled_poll" },
      "current_time": "{timestamp of the detected transaction, not of the request}"
    }
  ]
}
```

The `GetAccountSummaryByIrim`, `PartyAcctRelInq`, and `SearchAccountTransactions` middleware calls are internal to both `FirstFundingCheck` and the scheduled poller and are not part of the public nudge API.
