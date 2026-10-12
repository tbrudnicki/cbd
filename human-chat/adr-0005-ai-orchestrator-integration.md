# ADR-0005: AI conversation phase — chat-orchestrator-service extraction and REST/text-event-stream integration

Status: **resolved**
Relates to: [solution-design.md](./solution-design.md) §1.2/§1.4/§1.6/§2/§3, [conversational_banking_service_context.md](./conversational_banking_service_context.md), [ADR-0001](./adr-0001-realtime-transport.md), [ADR-0003](./adr-0003-reconciliation-job.md)
Decision owner: TBD — needs a counterpart owner on whichever team builds `chat-orchestrator-service`, not just `smb-chat-service`'s team

## 1. Context

The original design ran the bot-first phase of the conversation (`FAQ`) as an in-process, synchronous call inside `smb-chat-service` — a confirmed no-external-dependency SDK call, answered in the same HTTP response. `conversational_banking_service_context.md` describes a much larger, separately-owned **Conversation Orchestrator** service — intent routing, RAG answers, multi-turn transactional flows, a model gateway to LLM providers — that is explicitly out of scope for `smb-chat-service` to own or rebuild.

This ADR covers moving the bot-first phase out of `smb-chat-service` entirely, onto that new service (named here **`chat-orchestrator-service`**), and defining the boundary between the two: what crosses it, how, and what each side still owns. It does **not** cover the orchestrator's own internals (routing logic, RAG, flow state machines, model gateway/LLM hosting) — those belong to `conversational_banking_service_context.md` and that service's own team.

## 2. Revision note

This ADR originally specified **asynchronous integration over Azure Service Bus** (two queues, `chat-ai-inbound`/`chat-ai-outbound`). That transport decision has been **superseded** — the final decision (§3 below) is a direct, synchronous REST call with a streamed (`text/event-stream`) response instead. Keeping this note rather than quietly rewriting history: the Service Bus design was fully worked through (queue topology, session-based ordering options, idempotency-key generalization, DLQ posture) before being replaced, not abandoned half-finished. Most of that work carries over unchanged (§5's idempotency-key generalization, the one-way `AI → HUMAN` rule, the C2 diagram shape) because those decisions didn't actually depend on which transport sat between the two services. What changed is narrower than it might look: §3 and the open items in §7.

## 3. Decision: two phases, AI delegated over synchronous REST with a streamed response

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

Proceed with: `AI`/`HUMAN`/`CLOSED` phase model (one-way `AI → HUMAN`); `smb-chat-service` as pure router; direct synchronous REST call to `chat-orchestrator-service` (`POST /converse`) with a `text/event-stream` response for long-running generation, no broker; `delta` chunks relayed live over Web PubSub, `done`/`escalate` driving the same persist-and-push or escalate logic as before; `activityId` generalized to `externalMessageId` (load-bearing dedupe for `HUMAN`, informational for `AI`); attachment access via facade-issued SAS URL in the request, no new C2 edge; single-boundary C2 shape with `chat-orchestrator-service` as an external-to-the-boundary `Container`. Nothing architecturally significant remains open (§8 is confirmation/operational work, not a design fork) — this ADR is closed.
