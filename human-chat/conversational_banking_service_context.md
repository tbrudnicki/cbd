# Context: Conversational Banking Service (AI chat layer for the mobile banking app)

## How to use this document

This is design context from an earlier architecture discussion. It describes a **new service** to be added to an **existing mobile banking application** (a Polish retail bank, PLN as the main currency, users writing in Polish, English or a mix, often without diacritics).

- It describes the target capabilities, architecture, responsibilities and principles of the service.
- It does **not** describe the existing application's code, APIs or screens. Where this document refers to "existing" services (beneficiaries, payments, SCA, content, contact center), assume they already exist in the bank's landscape and must be **reused, not rebuilt**. Inspect or ask about the actual implementation before designing integrations.
- Example payloads are illustrative contracts, not final API specs.
- The invoice payment flow in section 8 is **one example** of a transactional flow, used to show the generic pattern. It is not the core of the service.

---

## 1. Purpose

Add a chat interface to the mobile banking app where users can type (or attach documents) and get one of four kinds of outcome:

1. **Inform**: answers from the bank's own approved content (FAQ, help center, fee tables, terms), with citations.
2. **Navigate**: a deep link to the right existing screen in the app (e.g. card settings, transaction history).
3. **Transact**: a guided, multi-turn flow that prepares an operation (transfer, bill/invoice payment, BLIK, standing order, top-up…) and hands it to the existing native confirmation + SCA flow for execution.
4. **Escalate**: hand-off to a human (contact center, fraud hotline) or a polite decline.

A single user message can contain several intents (e.g. a question plus a payment request); they are handled in order.

---

## 2. Core principles (non-negotiable)

1. **The bank's backend owns the orchestration.** A deterministic **Conversation Orchestrator** service decides everything that matters. Not the mobile app (tamperable, duplicated logic), not the LLM (probabilistic, injectable, not auditable).
2. **LLMs only propose; code decides.** Models classify, extract and phrase. Every model output is treated as untrusted input: validated against a schema, checked for consistency with the source text, and subject to business rules.
3. **No model can move money or change data.** LLMs have no tools that write to core banking. Even the orchestrator only *prepares* operations (drafts). **The user's SCA on a native screen authorizes**, and **existing bank services execute**.
4. **What the user confirms comes from bank systems, not chat text.** Confirmation screens load data from the payment/beneficiary APIs by ID, never from model-generated text. Amounts and statuses in replies come from data.
5. **Escalation and safety rules live in code, outside any agent loop.** Fraud keywords, thresholds, timeouts and fallbacks are deterministic and cannot be talked out of by a prompt. An agent may *request* escalation; it can never *suppress* it.
6. **Reuse existing flows.** Native screens (add beneficiary, confirm payment, card settings) and their security stay as they are; the chat prefills and launches them.
7. **Everything is auditable.** Every routing decision, model call, validation result, state transition and confirmation is logged with inputs and reasons.

---

## 3. Architecture and responsibilities

```
Mobile app ──► Conversation Orchestrator (bank backend) ──► LLM service(s) via model gateway
                 │  ├─ Intent Router (code)
                 │  ├─ Conversation & flow state
                 │  ├─ Intent registry (config)
                 │  └─ Handlers: Inform | Navigate | Transact (state machines) | Escalate
                 ▼
     Existing bank systems: content/FAQ index, app screen catalogue,
     beneficiary API, payment API, SCA, fraud engine, contact center
```

| Component | Responsibility | Owns decisions? |
|---|---|---|
| **Mobile app** | Chat UI; renders structured responses (text, citations, chips, deep links, prefilled forms, confirmation cards); runs native screens and SCA; uploads attachments to the document store | No (renders and authenticates) |
| **Conversation Orchestrator** | Single entry point for chat; conversation state; routing; flow execution; validation; calls to models and bank APIs; audit log | **Yes** |
| **Intent Router** (inside orchestrator, code) | Decides which handler/flow gets each message (layered: rules → active flow → model cascade → clarify) | **Yes** |
| **Intent registry** (config) | Single source of truth per intent: kind, description, required slots, confidence threshold, target screens, confirmation rules, example phrases. Classifier prompts, validation and tests are generated from it | Defines the rules |
| **Handlers** (inside orchestrator) | Inform (RAG), Navigate (screen catalogue), Transact (one deterministic state machine per flow), Escalate (handoff) | **Yes**, within their scope |
| **Model gateway** (inside orchestrator) | One internal interface (`classify()`, `extract()`, `extract_document()`, `answer()`, `phrase()`); model choice per task in config; timeouts; structured output enforcement | No |
| **LLM services** | Classification, slot extraction, document extraction, grounded answer writing, reply phrasing | **No** (advisory only) |
| **Existing bank systems** | Content index, beneficiaries, payments (drafts + execution), SCA, fraud scoring, contact center | Execute; enforce their own rules |

Suggested ownership: payments/digital channels domain team owns the orchestrator and flows (they own payment rules); AI/platform team owns the model gateway, prompts, model selection and evals.

---

## 4. Request lifecycle

### 4.1 Inbound event (app → orchestrator)

Attachments are uploaded first to the bank's document store (malware scan, size/page limits); chat events carry only references. Chips/buttons send structured events, not text.

```json
{
  "event": "message",
  "conversation_id": "conv_77", "user_id": "usr_1203",
  "text": "…user text…",
  "attachments": [{"file_id": "doc_81f2", "mime": "application/pdf", "pages": 1, "sha256": "…", "malware_scan": "clean"}],
  "context": {"screen": "chat", "active_flow": null, "locale": "pl-PL"}
}
```

Other inbound event types: `chip_selected`, `ui_result` (e.g. user finished a native screen), and system events from bank services (e.g. `beneficiary_added`, `payment_status`).

### 4.2 Routing layers (Intent Router, in order)

1. **Deterministic fast paths**: safety keywords (fraud/theft stems: "ukrad", "kradzie", "oszust", "zastrze", "stolen", "fraud"…) → Escalate immediately, no model call. Clear structured signals (e.g. a PDF with "zapłać" or no text) can route without a model too.
2. **Active flow**: if a flow is waiting for input, the message goes to that flow (slot-filling: "300", "the second one", "tak"), unless it clearly breaks out ("anuluj", "cancel", or an unrelated question → park the flow, offer to resume).
3. **Model cascade (escalating classification)**: small/fast model first; escalate to a larger model if the call fails or times out, the output is invalid or has an unknown intent, confidence is below the intent's threshold, the message has multiple intents, or a **deterministic consistency check** fails (e.g. the message contains an amount but no transactional intent was returned; an extracted amount does not appear in the text).
4. **Fallback**: if no tier is confident → clarifying question with chip options; if risky → Escalate; after repeated failure → search results / menu.

Thresholds are per intent and stricter for risky kinds (e.g. transact ≥ 0.85, others ≥ 0.70, fraud = any signal). Self-reported confidence is a weak signal and is always combined with the deterministic checks.

### 4.3 Classifier contract

Input: user text, attachment **metadata only** (not the document; file names treated as untrusted), recent turns, and context. Output (schema-constrained, names only from the registry):

```json
{"intents": [
  {"name": "faq_question", "confidence": 0.92, "slots": {}},
  {"name": "domestic_transfer", "confidence": 0.88, "slots": {"amount": 50, "currency": "PLN", "payee_hint": "Tomek"}}
]}
```

### 4.4 Router decision (logged)

```json
{"route": "handle|clarify|escalate|continue_flow", "intents": ["…"],
 "model_used": "small-8b", "tiers_tried": ["small-8b"], "reasons": [], "latency_ms": 180}
```

### 4.5 Handing over to a flow

```json
{"command": "start", "flow": "<flow_name>", "flow_id": "flw_42",
 "user_id": "usr_1203", "conversation_id": "conv_77",
 "inputs": {"…": "…"}, "hints": {"…": "…"}, "classification_ref": "dec_5531"}
```

`hints` from models are not facts; flows resolve them with code and show them to the user.

### 4.6 Outbound response types (orchestrator → app)

| Type | Used by | Notes |
|---|---|---|
| `text` (+ `citations`) | Inform, all | Free text never carries values the code acts on |
| `chips` | Clarify, suggestions | Selecting a chip sends a structured event |
| `deep_link` | Navigate | `screen_id` + params from a fixed screen catalogue; the model never builds URLs |
| `form_prefill` | Transact | Opens an existing native screen prefilled and editable (e.g. add beneficiary) |
| `confirmation_card` | Transact | References a `draft_id`; the app loads details from the bank API; SCA happens there |
| `handoff` | Escalate | Conversation + summary passed to contact center / fraud line |
| `receipt` | Transact | Result built from the bank system's status data |

---

## 5. Handlers

### Inform
RAG over the bank's curated content index (FAQ, help center, fee tables, T&Cs). Answers only from retrieved passages, with citations; "I don't know" when nothing relevant is found. Fees, rates and legal text come from structured data or verbatim snippets, never from model memory. Can suggest a related action chip. An agent loop with read-only tools (re-search, refine query, follow-up question) is acceptable here.

### Navigate
Picks a target from a fixed **screen catalogue** (`screen_id` + params). The app opens the screen; the screen's own security applies. Prefer this over rebuilding features in chat (e.g. card blocking, PIN change).

### Transact (generic pattern, one state machine per flow)
Every money- or data-changing operation follows the same skeleton:

1. **Collect slots**: required slots defined in the registry; ask only for what is missing; fill defaults from context (main account, last payee).
2. **Extract** (if a document/text needs reading): model returns structured fields with evidence; no tools.
3. **Validate (code)**: formats/checksums, grounding (extracted values must appear in the source), business rules, limits, balance, duplicates, fraud scoring.
4. **Resolve entities** against bank systems (payees, accounts, cards), with mismatch warnings.
5. **Prepare prerequisites via existing native screens** if needed (e.g. add beneficiary), with their own SCA; re-verify the result from the bank API, never trust the event payload.
6. **Create a draft** in the bank service from validated data only → `draft_id`.
7. **Native confirmation + SCA** (PSD2 dynamic linking to amount and payee) → **existing service executes** (idempotent).
8. **Result**: status from the bank system → receipt; clear the active flow.

Example states: `RECEIVED → COLLECTING/EXTRACTING → VALIDATING → RESOLVING → [AWAITING_PREREQUISITE] → DRAFT_CREATED → AWAITING_CONFIRMATION → EXECUTING → DONE`, plus `NEEDS_USER_INPUT`, `CANCELLED`, `EXPIRED`, `FAILED`. Every transition is logged.

Example transactional flows: domestic transfer, invoice/bill payment from an attachment, BLIK, standing order, phone top-up, SEPA/foreign transfers (separate flow with its own fields).

### Escalate
Immediate handoff with a conversation summary for fraud/theft, complaints, legal threats, vulnerability signals, or out-of-policy requests. Triggered deterministically by the router and optionally requested by models; never suppressible by a model.

---

## 6. Models

- **Tiering per task**: use the smallest model that passes evals for each task. Typical: small fast model for classification (possibly fine-tuned, or embeddings + classifier); small/medium for slot extraction; medium for grounded RAG answers; document-capable (vision or OCR + LLM) model for document extraction; larger model only as an escalation tier for ambiguous/multi-intent turns.
- **Open-weight / frontier models**: the user plans to use open frontier models. All are called through the model gateway (e.g. OpenAI-compatible API exposed by vLLM/TGI/Ollama or hosted providers). Self-hosting in EU infrastructure helps with GDPR data residency and DORA third-party risk; a hybrid (self-hosted small models seeing every message, hosted large model for rare hard turns with pseudonymized input) is an option.
- **Structured output**: enforce with schema-constrained decoding (provider "strict JSON schema"/structured outputs, or vLLM `guided_json`/Outlines/XGrammar) or forced tool calls; parse-and-retry only as a fallback. Schemas are generated from the same Pydantic models used for validation. Schema design: enums wherever possible, always an escape value (`unclear`/`needs_clarification`), small flat schemas, required-but-nullable slots, control fields separate from user-facing text.
- **Streaming**: routing/extraction calls are structured and non-streamed; only user-facing answer text (Inform) is streamed.
- **Pin model versions**; a model change is a release that goes through the full eval suite.
- **Polish language quality** varies strongly between models (slang, missing diacritics, code-switching); choose models on the bank's own data, not public benchmarks.

---

## 7. Security, risk and compliance

- **Untrusted content**: user text, attachments (hidden text, white-on-white, metadata), file names, payee names, transfer titles and transaction history are all prompt-injection vectors. Models that read them have no tools; outputs are schema-validated, grounded and confirmed by the user on native screens.
- **Fraud patterns**: e.g. new payee + payment from an uploaded document (invoice-swap fraud); known payee name with a different IBAN is a hard warning. Feed signals to the existing fraud engine.
- **PSD2**: SCA with dynamic linking for payments; SCA for adding trusted beneficiaries. Whether to combine prerequisite + payment into one SCA, and cooling-off/limits for first payments to new payees, are risk/compliance decisions.
- **Polish specifics** (where relevant): VAT White List check (NIP vs account) and split payment (MPP) for business customers; deterministic parsing first when structured sources exist (ZBP payment QR codes, KSeF structured e-invoices).
- **Regulation to involve early**: GDPR (data residency, minimization), DORA (ICT third-party risk incl. LLM providers), EU AI Act transparency obligations, PSD2.
- **Audit**: full chain logged (input, routing decision, model outputs, validation results, confirmations, execution) for disputes and regulators.

---

## 8. Example flow: invoice payment from an attached PDF (illustrative only)

Shows how the generic Transact pattern works with a document and a prerequisite step. Business rule in this bank: **money can only be sent to saved beneficiaries.**

1. App uploads PDF → sends message event with `file_id` (+ optional text like "możesz to ogarnąć do piątku?").
2. Router: fast-path rule or classifier (text + attachment metadata) → `pay_invoice` (TRANSACT, threshold 0.85, required slot `file_id`) → start flow.
3. Flow: check for ZBP QR / KSeF data first; otherwise extraction model returns `is_invoice, seller_name, seller_nip, ibans[], amount_due, currency, invoice_number, due_date, split_payment_required, evidence`.
4. Validate: IBAN/NIP checksums, grounding against the PDF text layer (scans → user reviews all fields), duplicate payment check, White List/MPP where relevant, fraud scoring; multiple IBANs → user chooses.
5. Beneficiary lookup by IBAN: found + name match → continue; IBAN found under another name → soft warning; not found → prefilled native add-beneficiary screen (user edits, SCA, saved by existing beneficiary API; orchestrator re-reads it); known name with different IBAN → hard warning.
6. Create payment draft (beneficiary ID, amount, invoice number as title, execution date resolved from hint vs due date) → `draft_id` → native confirmation card → user SCA → payment service executes.
7. Status → receipt; PDF attached to the transaction; active flow cleared.

Sequence of participants: Mobile app → Router → (Classifier LLM) → Transact state machine → (Extraction LLM) → Bank APIs (beneficiary, payment) ↔ Mobile app (native screens + SCA).

---

## 9. Quality, testing and thresholds

Non-deterministic components are **measured**, not pass/fail tested; deterministic components are tested classically.

| Layer | What | Pass criterion |
|---|---|---|
| Unit tests | Router logic, validators, consistency checks, state machines (with fake models) | 100% |
| Invariant/property tests | "No execution without SCA", "fraud keywords always escalate", "models cannot reach execute APIs", random/adversarial model outputs | 100% |
| Component evals | Classifier, slot extraction, document extraction, RAG answers, each separately | Statistical thresholds per slice |
| Conversation evals | Multi-turn flows with simulated users; check end state, not wording | Statistical thresholds |
| Adversarial evals | Prompt injection (in documents, payee names), jailbreaks, slang, no diacritics | ~0 on critical behaviors |
| Production monitoring | Shadow mode, canary, sampled human review, drift | Alerts on regression |

- **Datasets**: real (anonymized) messages; deliberate hard cases (PL/EN mix, no diacritics, multi-intent, near-misses, very short replies); double labeling to measure human agreement; locked held-out test set; versioned and refreshed monthly.
- **Non-determinism**: run cases multiple times, measure consistency; report confidence intervals (rule of three: 0 errors in n cases ⇒ ≤ ~3/n at 95%).
- **Grading**: exact/structural comparison first (enabled by structured outputs); LLM-as-judge with specific rubrics (validated against humans) for free text; targeted human review.
- **Thresholds**: chosen from coverage-vs-error curves per intent kind against risk-approved error budgets; check calibration; track clarification and escalation rates too.
- **Release gates** in CI on any prompt/model/registry change: hard safety gates, regression gates per critical slice, shadow mode, canary.
- **Validity**: periodically check that offline metrics predict production outcomes (abandonment, fallback to manual forms, complaints, handoffs).

---

## 10. Open decisions / assumptions to confirm

- Concrete model choices per task and hosting (self-hosted vs hosted vs hybrid).
- MVP intent list and which flows go first.
- Exact contracts of the existing beneficiary, payment, SCA, content and contact-center APIs, and how bank systems emit events back to the orchestrator.
- Single vs separate SCA for prerequisite + payment; cooling-off / limits for new payees.
- Supported attachment types (PDF, images/photos) and retention of uploaded documents.
- How chat UI components (cards, chips, forms) are implemented in the existing app.
- Escalation targets and handoff mechanism in the existing contact-center platform.

---

## 11. Reference artifact from the discussion

A Python prototype `escalating_router.py` exists (may be attached separately): intent registry, Pydantic schemas, OpenAI-compatible model gateway with `guided_json`, deterministic fast paths, active-flow handling, two-tier model cascade with per-intent thresholds, multi-intent escalation and consistency checks (unexplained/hallucinated amounts), and a demo with fake models.

## 12. Glossary

- **Orchestrator**: bank-owned backend service that owns conversation state, routing and flows.
- **Router**: code inside the orchestrator choosing the handler/flow per message; calls a classifier model only when rules cannot decide.
- **Intent registry**: config defining all intents, slots, thresholds and targets.
- **Flow / state machine**: deterministic multi-turn process for one transactional intent.
- **Draft**: prepared, not yet authorized operation stored in a bank service, referenced by `draft_id`.
- **SCA**: Strong Customer Authentication (PSD2), done on native screens.
- **Grounding check**: verifying that a model-extracted value literally appears in the source text/document.
- **Escalation (model)**: moving a request to a stronger model tier; **escalation (handler)**: handing the conversation to a human.
