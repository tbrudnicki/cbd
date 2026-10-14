# ADR-0005: AI conversation phase — chat-orchestrator-service extraction and REST/text-event-stream integration

Status: **open** — the transport decision in §3 is under architecture-forum discussion, see §10. §3 remains the **working assumption** the solution design is drawn against until the forum decides; §4–§7 are not in question.
Relates to: [solution-design.md](./solution-design.md) §1.2/§1.4/§1.6/§2/§3, [conversational_banking_service_context.md](./conversational_banking_service_context.md), [ADR-0001](./adr-0001-realtime-transport.md), [ADR-0003](./adr-0003-reconciliation-job.md)
Decision owner: TBD — needs a counterpart owner on whichever team builds `chat-orchestrator-service`, not just `smb-chat-service`'s team

## 1. Context

The original design ran the bot-first phase of the conversation (`FAQ`) as an in-process, synchronous call inside `smb-chat-service` — a confirmed no-external-dependency SDK call, answered in the same HTTP response. `conversational_banking_service_context.md` describes a much larger, separately-owned **Conversation Orchestrator** service — intent routing, RAG answers, multi-turn transactional flows, a model gateway to LLM providers — that is explicitly out of scope for `smb-chat-service` to own or rebuild.

This ADR covers moving the bot-first phase out of `smb-chat-service` entirely, onto that new service (named here **`chat-orchestrator-service`**), and defining the boundary between the two: what crosses it, how, and what each side still owns. It does **not** cover the orchestrator's own internals (routing logic, RAG, flow state machines, model gateway/LLM hosting) — those belong to `conversational_banking_service_context.md` and that service's own team.

## 2. Revision note

This ADR originally specified **asynchronous integration over Azure Service Bus** (two queues, `chat-ai-inbound`/`chat-ai-outbound`). That transport decision has been **superseded** — the final decision (§3 below) is a direct, synchronous REST call with a streamed (`text/event-stream`) response instead. Keeping this note rather than quietly rewriting history: the Service Bus design was fully worked through (queue topology, session-based ordering options, idempotency-key generalization, DLQ posture) before being replaced, not abandoned half-finished. Most of that work carries over unchanged (§5's idempotency-key generalization, the one-way `AI → HUMAN` rule, the C2 diagram shape) because those decisions didn't actually depend on which transport sat between the two services. What changed is narrower than it might look: §3 and the open items in §7.

## 3. Decision: two phases, AI delegated over synchronous REST with a streamed response

> **Open — working assumption only.** The phase model and the router role below stand. The transport choice (direct REST + `text/event-stream`, no broker) is under forum discussion — see §10 for why, the alternatives, and the questions the forum needs to answer.

- `Conversation.state` is renamed `FAQ` → **`AI`**. The two-phase model stays `AI → HUMAN → CLOSED`; only the first phase's name and implementation change.
- `smb-chat-service` becomes a pure router/facade: no conversational logic of its own. Per message it branches on `Conversation.state` — `AI` calls `chat-orchestrator-service` directly, `HUMAN` forwards to D365 exactly as before.
- Integration between `smb-chat-service` and `chat-orchestrator-service` is a **direct, synchronous REST call — no broker, no queue**. `smb-chat-service` calls a single endpoint on the orchestrator (e.g. `POST /converse`) with the message, context, and any attachment SAS references. The response is `Content-Type: text/event-stream` and is consumed as a stream rather than a single blocking JSON body — this is what lets a long-running model generation proceed without an all-or-nothing request timeout. Event types on the stream: `delta` (partial text, zero or more), `done` (final answer — text/citations/chips/deep_link/confirmation_card/receipt, per `conversational_banking_service_context.md` §4.6), `escalate` (hand-off request + reason, mutually exclusive with `done`).
- **`smb-chat-service` does not hold the mobile app's original request open for this.** It acks the mobile app immediately (202) before calling the orchestrator, then consumes the SSE stream server-side: `delta` chunks are relayed live to the device over Web PubSub as a best-effort "typing" effect (never persisted, same ephemeral treatment as `TYPING`), and the final `done` event is what gets persisted as a `ConversationEvent` and pushed through the same dedupe-then-deliver mechanism a D365 agent message already uses. This keeps the mobile-facing contract identical to what it was under the Service Bus design — only the `smb-chat-service` ↔ `chat-orchestrator-service` leg changed.
- Why this over the original Service Bus design: it's the more conventional shape for a backend-to-backend call (plain REST, no broker to stand up, operate, or reason about for this one integration), and `text/event-stream` is a standard, well-understood way to carry a long-running generation without either blocking the caller indefinitely or needing a broker's durability guarantees for what is, after all, a single request's answer — not an event other systems need to subscribe to.

## 4. Decision: `AI → HUMAN` is one-way and terminal

Unchanged by the transport revision. Once `Conversation.state` moves to `HUMAN`, no code path ever moves it back to `AI`. This was already implicit in the original design (nothing modeled a reverse transition) — this ADR makes it an explicit, load-bearing rule rather than an accident of what wasn't built:

- `chat-orchestrator-service` has no further role in a conversation once it's escalated — no resume path, no need to keep or rehydrate flow state for an escalated conversation "just in case."
- Escalation has **two trigger paths**, both converging on the same internal `smb-chat-service` operation: an explicit user action (unchanged REST call, `POST /chat/sessions/{id}/escalate`) or an orchestrator decision, signaled as an `event: escalate` SSE event mid-stream rather than a separate call (per `conversational_banking_service_context.md`'s principle that escalation is deterministic-router-owned and model-requestable but never model-suppressible). There is exactly one escalate implementation in `smb-chat-service`, not two parallel ones to keep in sync.

## 5. Decision: idempotency key generalizes, doesn't duplicate

`activityId` was D365-specific: the dedupe key for any D365-sourced event, nullable, unique — necessary because D365's webhook can redeliver the same activity. Under the (now superseded) Service Bus design, the orchestrator was a second at-least-once channel with the identical redelivery problem, which motivated generalizing the column. That specific justification weakens under a direct synchronous call — there's no broker redelivering the same logical message — but the generalization is kept anyway, for a narrower reason:

- `activityId` generalizes to **`externalMessageId`** on `ConversationEvent`. For `HUMAN`-phase rows it's still the load-bearing dedupe key, populated from D365's own `activityId`. For `AI`-phase rows it's populated from the orchestrator's `done`-event `messageId` when present, but is **informational/correlation only** — there is no redelivery path on this leg to dedupe against, since a dropped or retried call is a fresh request, not a replay of an already-processed one. One column, one unique constraint, used for actual dedupe on one phase and as a plain reference on the other.
- `ConversationEvent.origin` gains `AI_BOT` (replacing `FAQ_BOT`).
- Rich response types (chips, deep links, confirmation cards, receipts) don't get new columns — they use `contentType = RICH` with the structured payload in the existing `metadata` JSON field, the same field already used for `ESCALATED`/`CLOSED` metadata.

## 6. Decision: attachment access mirrors D365's existing (undrawn) pattern

`chat-orchestrator-service` needs attachment bytes for document-extraction flows (`conversational_banking_service_context.md` §5/§8). Two options were considered:

| Option | Description | Assessment |
|---|---|---|
| A. New facade-proxied download edge | Orchestrator calls a new/reused `smb-chat-service` endpoint (e.g. the already-proposed `GET /chat/attachments/{attachmentId}`, solution-design.md §3 row 10) to fetch bytes | Keeps every external read behind the facade, but adds a second HTTP dependency between the two services beyond the main `/converse` call, for no real safety gain over option B |
| **B. Facade-issued SAS URL carried in the request (chosen)** | The `POST /converse` request body carries a `contentUrl` (SAS, facade-issued, same shape as the existing `Attachment.contentUrl`); the orchestrator fetches bytes directly from Blob Storage via that URL | Identical to how D365 already fetches attachment bytes today (solution-design.md scenario 5/7) — a downstream consumer reading via a scoped, facade-minted SAS URL, not a mobile-client-style direct upload. Not a violation of the single-ingress-path principle (that principle is about the *untrusted original uploader*, not a trusted downstream reader) |

Option B is the decision, unchanged by the transport revision — it never depended on Service Bus being in the picture. No new edge is drawn in the C2 diagram for this — consistent with D365's existing attachment fetch also not being drawn as a separate relationship.

## 7. C2 diagram shape: why not two boundaries

A two-`Container_Boundary` layout (one per owning team: "Bank Chat Platform" vs. "Conversational AI Platform") was tried first, since it would have signaled the team/ownership split more explicitly. It was dropped after rendering: Mermaid's C4 auto-layout collapses *every* boundary's internal wrapping down to a single column the moment a second top-level boundary exists, regardless of diagram size — turning a 5-container boundary that otherwise wraps cleanly in two columns into a single tall column, which produced materially more crossing arrows for no clearer a signal than the shape distinction already gives. The adopted shape keeps one boundary ("Bank Chat Platform," unchanged) and draws `chat-orchestrator-service` as a plain `Container` (not `System_Ext` — it's still bank-owned, just not provisioned by this project) outside it, with exactly one relationship into the boundary (`smb-chat-service` ↔ `chat-orchestrator-service`, REST). Declaration order was chosen specifically to avoid box-crossing arrows; see the C2 diagram's own notes in solution-design.md for the detail. This reasoning is unaffected by the transport revision — the Service Bus container has simply been removed, which if anything made the graph easier to keep clean.

## 8. Open items

- [ ] **Timeout, retry, and failure-handling policy for the `/converse` call.** No broker behind it means `smb-chat-service` owns this outright: how long to wait, whether to retry (and how many times/backoff) before surfacing a failure, and what the user sees on failure (there's nothing to silently backfill, unlike the D365 leg's reconciliation sweep).
- [ ] **Streaming-connection infrastructure.** Anything sitting between the two services (load balancer, API gateway, proxy) needs confirmed support for long-lived streamed HTTP responses — idle-timeout settings in particular — plus a heartbeat/keep-alive convention so an intermediary doesn't kill an in-progress generation as idle.
- [ ] **`POST /converse` request/response schema** — the shapes sketched in §3 (`delta`/`done`/`escalate` event types) are illustrative, not confirmed; need joint sign-off with the orchestrator's own team.
- [ ] **Per-conversation send serialization** — whether a second `AI`-phase send for the same conversation is rejected or queued while a previous one's stream is still open, and what timeout governs "in-flight" so a stuck stream can't permanently block a conversation. Much narrower than the broker-ordering question this replaces (a single HTTP exchange is inherently ordered; this is only about concurrent sends, not message reordering), but still needs an answer.
- [ ] Mobile "thinking" state for `AI`-phase sends, ideally rendering `delta` chunks progressively — a UX consequence of the async-to-mobile design, not a backend item.
- [ ] Terminal state for a conversation that stays in `AI` phase and never escalates — `CLOSED` is currently only reachable via D365 agent closure. Pre-existing gap in the old `FAQ`-phase design too, not introduced here, but worth a real answer now that `AI`-phase conversations are expected to be the common case rather than a brief first exchange.
- [ ] Concrete technology/hosting for `chat-orchestrator-service` itself (self-hosted vs. hosted LLM providers, language/framework) — owned by that service's own team per `conversational_banking_service_context.md` §10, not by this ADR.

## 9. Decision summary

Proceed with: `AI`/`HUMAN`/`CLOSED` phase model (one-way `AI → HUMAN`); `smb-chat-service` as pure router; direct synchronous REST call to `chat-orchestrator-service` (`POST /converse`) with a `text/event-stream` response for long-running generation, no broker; `delta` chunks relayed live over Web PubSub, `done`/`escalate` driving the same persist-and-push or escalate logic as before; `activityId` generalized to `externalMessageId` (load-bearing dedupe for `HUMAN`, informational for `AI`); attachment access via facade-issued SAS URL in the request, no new C2 edge; single-boundary C2 shape with `chat-orchestrator-service` as an external-to-the-boundary `Container`. The transport choice (direct REST + `text/event-stream`) is the working assumption only and remains open for forum decision — see §10. Everything else in this summary stands.

## 10. Open: transport for the `smb-chat-service` ↔ `chat-orchestrator-service` leg

Status: **open — for architecture forum discussion.** Until the forum decides, §3 (REST + `text/event-stream`) remains the working assumption, and the solution design stays drawn against it so the document is coherent.

### 10.1 Why this is open

§3 replaced the original Service Bus design on the grounds that plain REST is "the more conventional shape … no broker to stand up, operate, or reason about." That reasoning never named **failure recovery** as a driver. Walking through what happens to a long-running generation when either service is redeployed or crashes shows that:

- §3 as written has **no recovery path** — a dropped stream is a user-visible failure and the customer has to resend (solution-design.md scenario 2, "call fails or times out" branch).
- Adding a recovery path to the synchronous design means building, by hand and split across two teams, the guarantees a message broker already provides: a durable request record, a durable result, redelivery, and dedupe.

The question for the forum is therefore not "sync vs. async" in the abstract, but: **given that recovery from redeploys and crashes is a requirement, do we build those guarantees ourselves over HTTP, or take them from a broker?**

### 10.2 Scope

- **In question:** only the transport of §3, i.e. the `smb-chat-service` ↔ `chat-orchestrator-service` leg.
- **Not in question:** the mobile ↔ `smb-chat-service` contract (202 Accepted + answer pushed over Web PubSub — unchanged under every option below), the phase model and one-way `AI → HUMAN` rule (§4), attachment access via SAS (§6).
- **Affected by the outcome:** §5 — under any option that retries or redelivers, `externalMessageId` becomes a load-bearing dedupe key for `AI`-phase rows again, not informational. §7 — a broker option adds one container to the C2 diagram.

### 10.3 Confirmed context (inputs to the decision)

| # | Fact | Effect |
|---|---|---|
| C1 | The interaction is strict request/response: one prompt in, one answer out. The orchestrator never initiates a message and never sends several outputs for one input (per current Figma designs). | Async is not *forced* by the interaction shape — this is a choice on reliability grounds |
| C2 | No transactional flows (SCA, payment execution, beneficiary changes) run through this chat's `AI` phase today. | If such flows are added later, the orchestrator will need to send messages nobody asked for (receipts, status updates) — request/response cannot carry these, which would strengthen the async case further |
| C3 | **Requirement:** a lost connection or a dead replica on either side, including during a routine redeploy, must be detected and recovered **without the customer resending**. | Primary driver. Option 1 below does not meet it |

### 10.4 Failure modes

| | Failure | What is lost | §3 as written today |
|---|---|---|---|
| F1 | `chat-orchestrator-service` pod dies mid-generation (crash or redeploy) | The work — the answer never existed | Stream drops; nothing re-runs the work; customer must resend |
| F2 | `smb-chat-service` pod dies mid-stream, orchestrator finishes | A finished answer nobody receives | Lost — and nothing even records that a reply is still owed, so the conversation silently goes quiet |
| F3 | Connection drops between two live services (network, proxy idle timeout) | Either of the above — neither side knows which | Ambiguous; treated as a failure |

### 10.5 Options

**Option 1 — REST + SSE as in §3 (status quo).** Keeps live token streaming, no broker. Fails C3 on F1, F2 and F3. Listed for completeness; not viable if C3 stands.

**Option 2 — REST + SSE with idempotent replay.** `smb-chat-service` calls `POST /converse` with a stable per-turn idempotency key (e.g. the prompt's `ConversationEvent` id). If the stream breaks, the orchestrator keeps generating and stores the result; a surviving or restarted `smb-chat-service` replica retries with the same key and receives the stored result instead of a fresh generation. Requires:

- *Orchestrator team:* a shared (not pod-local) result store with TTL; generation decoupled from the HTTP connection (frameworks often cancel on client disconnect by default); request state `IN_PROGRESS` (owner + lease) → `DONE`/`FAILED`; defined retry behaviour per state (replay / reattach / regenerate on expired lease).
- *`smb-chat-service`:* a durable "reply owed" record per `AI` turn (`AWAITING_REPLY`, attempts, deadline); a restart-safe sweeper that retries stale turns and finally surfaces a failure; dedupe on `externalMessageId`; a distributed lock or equivalent for per-conversation serialization (§8).
- *Infrastructure:* long-lived-connection support and heartbeats on every intermediary (§8).

**Option 3 — Azure Service Bus request/reply queues with sessions.** Essentially the original design this ADR started with (§2: `chat-ai-inbound` / `chat-ai-outbound`), with `sessionId = conversationId`. The platform supplies: redelivery on lock expiry (F1), a durable reply any facade replica can consume (F2), no held connection (F3 disappears), one in-flight turn per conversation (sessions), and load levelling so a slow orchestrator never ties up facade capacity that `HUMAN`-phase traffic shares. Still requires:

- a **transactional outbox** in `smb-chat-service` (the prompt's SQL insert and the Service Bus send are not one transaction);
- **dedupe on both sides** (at-least-once delivery): facade on `externalMessageId`, orchestrator on the turn key;
- a **customer-visible deadline** (message TTL + dead-letter handling → failure state shown to the customer);
- **DLQ monitoring** and an operational runbook;
- **giving up live token streaming** — the customer sees a "thinking" state, then the whole answer.

**Option 4 — Hybrid.** Option 3 for the durable request and final answer, plus a separate best-effort channel for live `delta` chunks. Keeps streaming, but brings back part of the connection-handling complexity Option 3 removes. Only worth it if streaming is a hard requirement *and* Option 2's cross-team commitment isn't available.

### 10.6 Comparison

| Driver | Option 1 | Option 2 | Option 3 | Option 4 |
|---|---|---|---|---|
| Recovers F1 (orchestrator dies) | ❌ | ✅ regenerate on expired lease | ✅ redelivery | ✅ redelivery |
| Recovers F2 (facade dies) | ❌ | ✅ retry + replay | ✅ durable reply | ✅ durable reply |
| F3 (connection drop) | ❌ | ✅ retry + replay / reattach | n/a — no held connection | Streaming channel only — deltas lost, answer safe |
| Live token streaming | ✅ | ✅ | ❌ | ✅ |
| Hand-built reliability mechanisms | none | many, across both teams | outbox, dedupe, DLQ handling (standard patterns) | Option 3's + a streaming channel |
| Per-conversation serialization (§8) | distributed lock | distributed lock | Service Bus sessions | Service Bus sessions |
| Slow orchestrator affects `HUMAN`-phase capacity | yes | yes | no — queue absorbs backlog | partly |
| New infrastructure to operate | none | none (but connection-tuned intermediaries) | Service Bus | Service Bus + streaming path |
| Orchestrator team commitment | low | high (result store, leases, replay API) | medium (queue consumer, dedupe) | medium-high |

### 10.7 Preliminary recommendation

**Option 3 (Service Bus with sessions)**, on condition that:

1. live token streaming is **not** a hard product requirement (a "thinking" indicator followed by the full answer is acceptable), and
2. Azure Service Bus is already an operated platform service at the bank, or is acceptable to introduce.

Reasoning: given C1 (pure request/response) and C3 (recovery is required), Option 2 rebuilds a broker's guarantees by hand across two teams' codebases, while Option 3 reduces the custom work to well-known patterns (outbox, dedupe, DLQ) and also resolves §8's serialization and streaming-infrastructure items. If condition 1 fails, the choice is between Option 2 and Option 4. If condition 2 fails, Option 2.

### 10.8 Questions for the forum

1. **Streaming (decisive).** Is progressive token streaming a product/UX requirement in the Figma designs, or is a "thinking" indicator acceptable?
2. **Platform (decisive).** Is Azure Service Bus an existing, operated platform service? If not, is introducing it acceptable for this integration?
3. **Orchestrator team.** Do they prefer consuming a queue or exposing a streamed endpoint? Will they commit to their half of Option 2 (result store, leases, replay) if that's the outcome?
4. **Regeneration.** When the orchestrator dies mid-generation (F1), every option regenerates the answer. Is that acceptable (LLM cost, a delay, possibly different wording)?
5. **Orchestrator state.** Does the orchestrator store conversation turns itself, or is it stateless with the history sent per call? If it stores them, retries and redeliveries must be idempotent on its side under every option.
6. **Latency.** Expected time to the full answer, typical and p99. Seconds favour Option 2; minutes strongly favour Option 3.
7. **Decision owner.** Who owns this decision jointly on the `smb-chat-service` and `chat-orchestrator-service` sides (still TBD at the top of this ADR)?

### 10.9 Solution design impact if Option 3 is adopted

- C2 diagram: add a Service Bus container; the `smb-chat-service` ↔ `chat-orchestrator-service` relationship becomes two queue relationships.
- Scenario 2 redrawn: outbox → request queue → orchestrator → reply queue → facade consumer → dedupe → persist → push. The `delta` loop is removed (or moves to the side channel under Option 4).
- §1.6 "AI conversation" bullets rewritten; `externalMessageId` becomes load-bearing dedupe for `AI`-phase rows.
- §4 open items: 10 (timeout/streaming infrastructure) and 13 (per-conversation serialization) are replaced by outbox, message TTL/deadline, and DLQ-handling items; item 11 becomes a "thinking" indicator only.
- §8 of this ADR is replaced accordingly.
