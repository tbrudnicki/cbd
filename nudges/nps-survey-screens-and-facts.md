# 1. Introduction

This document covers the `nps_survey` nudge: the Feature-1 NPS survey, modeled as a single nudge within the **nudge sub-service** of `customer-management-service` (Kevlar). It is scoped to one question only — **which screens must carry the nudge, and which facts decide whether it is eligible on each of them.** Reporting (Power BI) and the MoEngage event stream the app itself emits are out of scope here
**Ownership.**

| Area | Owner |
|---|---|
| Nudge decision (`nps_survey` eligibility, lifecycle) | Nudge sub-service, `customer-management-service` |
| Customer facts (`CUSTOMER_FACT`: counters, `first_invoice_at`) | Data platform |
| Live facts (`CUSTOMER_LIVE_FACT`: NPS result, live lookups) | Nudge sub-service |
| Presentation (rating UI, feedback form, app-store redirect) | Mobile app |

Companion to `nudges-model.md` §7 (use cases) — the general data model, lifecycle engine, and API are defined there; this document applies them to the NPS feature.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
flowchart LR
  FRONT[SME Mobile Application<br/>Up by CBD]

  subgraph CMS["Customer Management Service — Kevlar"]
    API[Nudge sub-service API<br/>+ lifecycle engine + rules]
  end

  subgraph DB["Customer Management DB — shared instance"]
    MART[("nudge_mart schema<br/>CUSTOMER_FACT")]
    NSCHEMA[("nudge schema<br/>NUDGE, CUSTOMER_LIVE_FACT,<br/>CUSTOMER_NUDGE, CUSTOMER_NUDGE_HISTORY")]
  end

  subgraph DP["Data Platform"]
    ETL[ETL pipeline]
    STREAM[Stream ingestion]
  end

  CB[Core Banking]
  MOE[MoEngage]
  DS[Downstream APIs<br/>accounts, profile]

  FRONT -->|"GET /v1/nudges<br/>POST outcomes, nps-responses"| API
  API --> NSCHEMA
  API -. read-only .-> MART
  API -->|live lookup, request time| DS
  CB --> ETL
  FRONT -->|app events| MOE
  MOE --> STREAM
  ETL -->|daily, T-1| MART
  STREAM -->|near real time| MART
:::

## Sequence diagram

One `GET /v1/nudges?screen=...` call, evaluating only `nps_survey`, following `nudges-model.md` §5.1's order exactly.

::: mermaid
sequenceDiagram
autonumber
participant front as SME Mobile Application
box Customer Management Service
    participant api as Nudge Sub-service API
    participant engine as Lifecycle Engine
    participant rule as NpsSurveyRule
end
box Customer Management DB
    participant cfg as NUDGE / NUDGE_PLACEMENT
    participant state as CUSTOMER_NUDGE
    participant cf as CUSTOMER_FACT
    participant lf as CUSTOMER_LIVE_FACT
end
participant ds as Downstream API

Note left of front: Request nudges for a screen
front ->> api: GET /v1/nudges?screen=local_transfer_confirmation
api ->> cfg: active nudges placed on this screen
cfg -->> api: [nps_survey, ...]

Note left of front: Config and state check
api ->> engine: evaluate(customer, nps_survey, now)
engine ->> cfg: nps_survey.is_active? rule class registered?
cfg -->> engine: yes
engine ->> state: get CUSTOMER_NUDGE(customer, nps_survey)
state -->> engine: row (status, valid_until, attempt_count, last_shown_at)

alt row.status is terminal
    engine -->> api: return row — not served
else row.status == ELIGIBLE and valid_until > now
    engine -->> api: return row — cached decision, still servable
else row.status == PARKED and valid_until > now
    engine -->> api: return row — suppressed (3rd skip, cooling down)
else
    opt row.status == PARKED and valid_until <= now
        engine ->> state: attempt_count = 0 — pause is over
    end
    alt min_interval set and last_shown_at > now - min_interval
        engine -->> api: NOT_ELIGIBLE — asked too recently
    else
        Note left of front: Fact evaluation
        engine ->> rule: evaluate(facts, state, now)
        rule ->> cf: get international_transfer_count, local_transfer_count, first_invoice_at
        cf -->> rule: values + as_of
        rule ->> lf: get nps.last_response_at, nps.last_score
        lf -->> rule: values, or missing
        opt same-day firing needed and dwh count is stale
            rule ->> ds: live lookup (accounts API)
            ds -->> rule: current count
            rule ->> lf: facts.live(...) stores the result
        end
        rule -->> engine: Decision — Eligible / NotEligible / Park / ...
        engine ->> state: apply decision — status, valid_until, attempt_count
        engine ->> state: write CUSTOMER_NUDGE_HISTORY (reason, facts_used)
        engine -->> api: updated row
    end
end

Note left of front: Response
alt row is servable
    api -->> front: { "nudges": [{ "name": "nps_survey" }] }
else not servable
    api -->> front: { "nudges": [] }
end
:::

Two things this makes visible that are easy to miss reading prose alone:
- The **suppressed** branch (3rd skip, `PARKED`) never reaches the rule or the facts at all — it short-circuits on `CUSTOMER_NUDGE` state alone, confirming that suppression isn't fact-driven.
- The live lookup is the *only* optional step — everything else runs on every request; it only fires when the rule decides a `dwh` counter might be too stale to trust for a same-day trigger.

# 3. Design & Implementation

## User Journey

_(link to Figma — not yet available)_. Interim reference: `Feature-1.txt` lines 19–51 describe the two-step journey (rating capture, then the promoter/detractor-specific branch) that the mobile app owns; this document only covers the backend decision that precedes it.

## Screens and facts

**Design choice.** One `NUDGE` row, `nps_survey`, placed on all seven screens below. Per `nudges-model.md` principle #8 (state is per customer and nudge, not per screen), this gives shared skip/suppression state across every trigger for free — which is what Feature-1 line 58 ("transaction-triggered and periodic rules must prevent overlapping or repeated prompts") asks for, without extra engine work.

**Screens to support (`NUDGE_PLACEMENT`).**

| Screen | Trigger journey | Trigger type | What gates eligibility |
|---|---|---|---|
| `debit_card_activation_confirmation` | Debit card activation | journey | Placement only — reaching the screen implies success (same pattern as `csat_survey`, §7.2) |
| `card_limit_increase_confirmation` | Card limit increase, post-consent | journey | Placement only |
| `international_transfer_confirmation` | First international transfer | journey | Needs `international_transfer_count` — see below |
| `local_transfer_confirmation` | Third local transfer | journey | Needs `local_transfer_count` — see below |
| `wps_enrolment_confirmation` | WPS enrolment | journey | Placement only — reaching the screen is assumed to indicate success |
| `invoice_creation_confirmation` | Invoice creation | journey | Placement only, **or** reuse `first_invoice_at` if the intended trigger is "first invoice ever" — see Decisions |
| `home` | Periodic, every 3 months | periodic | Uses `CUSTOMER_NUDGE.last_shown_at` / `nps.last_response_at`; no new data-platform fact |

**Facts required.** Working assumption (per `nudges-model.md` §3–4.1): data platform output (ETL/`dwh` and stream/`moengage_stream`) lands in `CUSTOMER_FACT`. Anything the nudge sub-service detects itself while handling a request — live lookups and app-submitted results — lands in `CUSTOMER_LIVE_FACT`.

| `fact_key` | `owner` | `source` | `kind` | Lands in | Why |
|---|---|---|---|---|---|
| `international_transfer_count` | `data_platform` | `dwh` | `snapshot` | `CUSTOMER_FACT` | Running count of international transfers; the rule compares it against the configured nth-transaction threshold (Feature-1 line 116) instead of checking a single once-fact |
| `local_transfer_count` | `data_platform` | `dwh` | `snapshot` | `CUSTOMER_FACT` | Running count of local transfers |
| `first_invoice_at` | `data_platform` | `dwh` | `once` | `CUSTOMER_FACT` | Already exists in the model (§4.1, used by `try_invoicing`) — reuse rather than redefine, if "invoice creation" means first invoice |
| `nps.last_response_at` | `nudge_sub_service` | `app` | `snapshot` | `CUSTOMER_LIVE_FACT` | Written synchronously when the app posts the NPS result — mirrors `csat.last_response_at` (§4.1, §5.3) |
| `nps.last_score` | `nudge_sub_service` | `app` | `snapshot` | `CUSTOMER_LIVE_FACT` | Written synchronously, same call |
| transfer count via live lookup (international/local) | `nudge_sub_service` | `live_lookup` | — | `CUSTOMER_LIVE_FACT` | Fallback only — fixes `dwh`'s T-1 lag for same-day firing |

**Rule shape for the two counter-based triggers.** Fire on `count >= threshold`, not `count == threshold`. A plain equality check breaks the moment two transfers land in the same T-1 cycle — the count can jump straight from 0 to 2, skipping the exact value 1 (or 2 to 4, skipping 3) entirely. `>=` still catches it the first time the count reaches or passes the threshold.

**Why no per-trigger "already prompted" fact is needed.** `nps_survey` is one nudge with one shared decision, skip budget, and suppression window across every trigger — journey or periodic. `>=` makes the counter conditions permanently true once reached, but that's fine: the engine's existing `min_interval` (§4.3, §5.1) already blocks re-asking before it elapses, and repeated skips already drive the shared `attempt_count` to `PARKED` for the configured suppression period (§5.2), regardless of which trigger caused each skip. A third local transfer, a WPS enrolment, and the periodic clock are just three different ways of making the same one decision eligible — they don't need separate bookkeeping, and adding it would fragment a skip budget that Feature-1 (line 58) explicitly wants unified.

**No fact needed** — placement alone decides it, following §7.2's rule ("reaching the screen means the action has just happened"), extended to every journey trigger whose success can't be told apart from a pending or interrupted attempt purely by other means — accepted as a uniform assumption:
- Debit card activation
- Card limit increase, post-consent
- WPS enrolment
- Invoice creation, if every creation (not just the first) should trigger the survey

## Backlog

_(Provide the feature backlog)_

### Features

#TBD — add the Jira/ADO ticket(s) covering `nps_survey` (eligibility, lifecycle, fact definitions)

### Decisions

_(Provide decision link — until a formal decision log exists, tracked here)_

1. **Invoice creation: first-ever, or every invoice?** Changes whether this trigger needs `first_invoice_at` or just placement. Feature-1 states it without an ordinal qualifier, unlike the transfer triggers, so this needs confirming rather than assuming.
2. **Are international/local transfers app-exclusive for this product?** If customers can also transfer via another channel, `dwh` must stay authoritative for the counters; if app-only, `moengage_stream` could carry them near-real-time instead, removing the live-lookup fallback entirely.
3. **`moengage_stream` end-to-end latency** (already open decision #9 in `nudges-model.md`) — needed before deciding whether stream facts are fast enough for any of the above same-day firings.
4. **`min_interval` for `nps_survey`** (same open point as `nudges-model.md` open decision #5, now load-bearing here too). Since the transfer-count conditions stay permanently true once reached, `min_interval` is what stops every subsequent qualifying screen visit from re-asking before the configured suppression period (Feature-1 line 58) has passed — it needs an actual value, not `none`.

### Tests

_(Provide tests link — not yet created)_. Per `nudges-model.md` §4.3 guideline 3, `NpsSurveyRule` needs unit tests built from fact fixtures, for example: "no counters at threshold, no recent response → `NotEligible`"; "`local_transfer_count = 3`, no prior prompt → `Eligible`"; "`csat`-style `nps.last_response_at` within 3 months → `Park`".

## API definition

_(link to Swagger — not yet published)_. Relevant endpoints, defined in `nudges-model.md` §6:
- `GET /v1/nudges?screen={screenName}` — existing, unchanged
- `POST /v1/nudges/outcomes` — existing, unchanged (`shown` / `skipped` / `dismissed` for `nps_survey`)
- `POST /v1/nps-responses` — new, same shape as `POST /v1/csat-responses` (§6.3): `response_id`, `screen`, `score`, `submitted_at`; stores `nps.last_response_at` / `nps.last_score` as live facts and forces re-evaluation
