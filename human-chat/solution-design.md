# 1. Introduction

## 1.1 Purpose

This Solution Design describes the architecture for **human chat** support within **Up By CBD** — SME customers get an AI-first support experience (**AI** phase, owned by **chat-orchestrator-service**) that can escalate to a live D365 Omnichannel agent (**HUMAN** phase), fronted by a purpose-built facade, **smb-chat-service**. `smb-chat-service` itself no longer runs any conversational logic — it routes each message to the correct downstream channel (the AI orchestrator or D365) based on `Conversation.state`, owns the compliance log, real-time delivery, and attachment storage. It exists to bring this design to architecture review, security, and compliance (roadmap Phases 5–6) before implementation begins.

## 1.2 Scope

**In scope:**
- Two-phase conversation model: `AI` (delegated asynchronously to **chat-orchestrator-service**) → `HUMAN` (D365 Omnichannel agent) → `CLOSED`. The `AI → HUMAN` transition is **one-way and terminal for the AI leg** — no code path ever moves `Conversation.state` back from `HUMAN` to `AI`; once escalated, the orchestrator has no further role in that conversation (ADR-0005).
- `smb-chat-service` as a pure router/facade: it no longer embeds any conversational logic itself, it dispatches each message to the AI orchestrator or to D365 based on `Conversation.state`.
- Synchronous REST integration with `chat-orchestrator-service` (ADR-0005) — `smb-chat-service` calls it directly; the response streams back over `text/event-stream` to handle long-running generation without blocking the connection. The reply to the mobile app is still delivered asynchronously, pushed over Web PubSub as it streams in, the same as a D365 agent message.
- Text, link, and file (image/PDF) messaging in both directions.
- Real-time delivery via Azure Web PubSub, with push-notification fallback when the device is disconnected.
- Offline agent-response notifications: when the customer is disconnected and an agent responds, notify them via push (every message) and email (throttled — see §1.6), both brokered through the existing `notification-service`.
- A single compliance-source-of-truth event log (Azure SQL Database) covering both phases.
- A fixed-interval reconciliation sweep against D365's message history, as the safety net for lost webhook deliveries (ADR-0003, Option A). This sweep is **D365/HUMAN-phase only** — the AI leg has no equivalent (§1.6, ADR-0005).

**Explicitly out of scope** (confirmed during planning, not an oversight):
- Multi-device fan-out — the org's Identity Provider enforces one session per user, so no cross-device delivery/read-state problem exists here (ADR-0001 §6.2).
- Virus/malware scanning of uploaded attachments — descoped for now (§1.6); flagged as a candidate for a future compliance/security revisit, not silently dropped.
- Pushing unrelated event types (transaction alerts, fraud confirmations) over the same real-time channel — speculative, not a confirmed near-term requirement (ADR-0001 §6.3, §8). If this becomes real it flips the connection-lifecycle design from conversation-scoped to app-session-scoped and would need re-scoping in.
- Direct-to-blob attachment upload (SAS token issued to the client) — all file traffic is proxied through the facade, consistent with the single-ingress-path principle used throughout.
- The orchestrator's own internals (intent routing, RAG/content index, flow state machines for Navigate/Transact, model gateway/LLM provider hosting) — owned and designed separately by `chat-orchestrator-service`'s team; see `conversational_banking_service_context.md`. This document only covers the boundary: how `smb-chat-service` talks to it, not what it does internally.

## 1.3 Related documents

| Document | Role |
|---|---|
| [ADR-0001](./adr-0001-realtime-transport.md) | Real-time transport — Azure Web PubSub |
| [ADR-0002](./adr-0002-channel-system-technology.md) | Compliance store technology — Azure SQL Database only |
| [ADR-0003](./adr-0003-reconciliation-job.md) | Reconciliation job cadence/trigger model |
| [ADR-0004](./adr-0004-webhook-ga-status.md) | D365 webhook GA status |
| [ADR-0005](./adr-0005-ai-orchestrator-integration.md) | AI conversation phase — chat-orchestrator-service extraction & synchronous REST/text-event-stream integration |

## 1.4 Naming & terminology

| Term | Meaning |
|---|---|
| **Up By CBD** | The mobile banking app (iOS/Android) |
| **smb-chat-service** | The Java/Spring Boot facade (BFF) between the mobile app, D365, and `chat-orchestrator-service`. Routes by `Conversation.state`; owns the compliance log, real-time delivery, and attachment storage. Runs no conversational logic itself |
| **chat-orchestrator-service** | New service, separate team/deployable, not built by this project. Owns the `AI` conversation phase: intent routing, FAQ/RAG answers, flow state, the model gateway to LLM providers. Decides when to request escalation. Called synchronously over REST, response streamed via `text/event-stream`. See `conversational_banking_service_context.md` and ADR-0005 |
| **notification-service** | Existing shared platform service (not built by this project) that sends both push (FCM/APNs) and email notifications on our behalf |
| **AI** | Conversation state: every message sent synchronously to `chat-orchestrator-service` over REST, response streamed back, no D365 involvement. Replaces the earlier `FAQ` state name |
| **HUMAN** | Conversation state: messages routed to a live D365 Omnichannel agent. Reached only from `AI`, one-way — there is no transition back to `AI` (§1.6, ADR-0005) |
| **CLOSED** | Terminal conversation state, reached when the agent closes the D365 conversation |
| **ConversationEvent** | The single append-only event log row type spanning both phases — messages, escalation, closure, read receipts, etc. (§1.6) |

## 1.5 Status & ownership

**Status:** Draft. Architecture (C2 diagram, all 12 sequence diagrams), the User Journeys backlog, and the four key ADRs' rationale are complete. Outstanding: Design & UX (blocked on a Figma link — external input), Tests, and the formal OpenAPI/swagger spec (the endpoint catalog itself is done, §3).

**Ownership — still open:**
- Document owner / sign-off authority: **TBD** — not yet assigned.
- Approval chain (architecture review board / security / compliance / D365 platform owner): **TBD** — stakeholders haven't yet been formally confirmed.

This section should be revisited once those stakeholders and an accountable owner are confirmed — the content above reflects what's derivable from the design work done so far, not a substitute for that alignment step.

## 1.6 Key assumptions & confirmed decisions

Facts and decisions the architecture in §2 and §3 is built on. Stated explicitly here so this document is self-contained — the sequence diagrams and API table below reference this section rather than external planning notes.

**Messaging content:**
- A link message is just `contentType: TEXT` with a URL in `body` — no distinct type or preview metadata. This is a working assumption, not a confirmed requirement; see §4 item 1.
- `body` is capped at 6,000 characters — confirmed against the current D365 Messaging API docs, enforced at the facade (not left to D365 to reject) for both `AI` and `HUMAN` phases. The same cap is applied uniformly to `AI`-phase messages even though D365 isn't involved, for consistency of the one validation path.
- On any message, either `text` or `attachments` must be present — a request with neither is rejected. Same D365-confirmed constraint, same facade-side enforcement.

**Attachments:**
- Allowed types: `image/jpeg`, `image/png`, `application/pdf` only. Max size: 10 MB. Both enforced at facade upload validation, not left to the client.
- No virus/malware scanning at this time — files are stored as uploaded. Flagged as a candidate for a future compliance/security revisit, not silently dropped.
- Inbound (agent-sent) attachments are **copied into our own Blob Storage**, not referenced by D365's URL — our store is the retention source of truth, so a customer's later ability to view a file never depends on D365 continuing to host it.
- The `contentUrl` SAS URL uses a **fixed expiry duration, configurable at service deployment** (e.g. an `attachment.sas-token.expiry-minutes` application property) rather than being computed per-request or negotiated dynamically with D365.
- `chat-orchestrator-service` fetches attachment bytes the **same way D365 already does** — the REST request carries a facade-issued `contentUrl` (SAS), and the orchestrator fetches it directly; this is not drawn as a separate edge in the C2 diagram, same convention as D365's existing (undrawn) attachment fetch. It never receives inline bytes and never talks to Blob Storage without going through a facade-issued URL — consistent with the single-ingress-path principle (§1.2).

**Data model / event log:**
- `ConversationEvent` is a single append-only log spanning both `AI` and `HUMAN` phases — one write path, one DB-generated, globally monotonic `sequenceNo` driving watermarks, catch-up, and reconciliation diffing alike.
- `activityId` generalizes to **`externalMessageId`** — the idempotent-dedupe key (unique constraint) for any event sourced from an external async system, now two of them: D365's `activityId` for `HUMAN`-phase rows, and `chat-orchestrator-service`'s own assigned message ID for `AI`-phase rows. It's nullable: briefly null for an in-flight outbound row of either phase between being persisted and the downstream system confirming/assigning an ID.
- `ConversationEvent.origin` gains `AI_BOT` (replacing `FAQ_BOT`) for orchestrator-sourced replies, alongside the existing `CUSTOMER` and `AGENT`.
- Rich AI responses (chips, deep links, confirmation cards, receipts — see ADR-0005) don't get new columns: they use `contentType = RICH` with the structured payload in the existing `metadata` JSON field, the same field already used for `ESCALATED`/`CLOSED` event metadata.
- **The `AI → HUMAN` transition is one-way and terminal for the AI leg** (§1.2): once `Conversation.state` is `HUMAN`, no code path sets it back to `AI`. `chat-orchestrator-service` has no further role in a conversation past that point — it doesn't need to resume or reconcile any state for an escalated conversation.
- `TYPING` events and delivery acks are deliberately **never persisted** — ephemeral, no compliance/audit value; only a denormalized fast-lookup field (e.g. `Conversation.deliveryWatermark`) changes.
- Recommended index: `(conversationId, phase, sequenceNo)` — the shape both the mobile catch-up query and the reconciliation job's scoping query need.
- Single-device enforcement is handled entirely upstream by the **Identity Provider** (one session per IDP profile) — `smb-chat-service` builds no refuse/displace logic, and no `ConnectionRegistry` entity exists in the data model as a result.
- `Conversation.offlineNotifiedAt` (nullable timestamp) — set the first time an offline-notification email is sent for the customer's current continuous-offline stretch; cleared when the device reconnects. Exists purely to throttle email (see Notifications below); has no effect on push.

**AI conversation (chat-orchestrator-service integration, ADR-0005):**
- Interfacing is a **direct, synchronous REST call** from `smb-chat-service` to `chat-orchestrator-service` — no broker, no queue, in between. `smb-chat-service` calls a single endpoint (e.g. `POST /converse` on the orchestrator) with the message, context, and any attachment SAS references; the response has `Content-Type: text/event-stream` and is consumed as a stream, not a single blocking body — this is what lets a long-running model generation avoid tying up the call with an all-or-nothing timeout. (This supersedes ADR-0005's original Azure Service Bus design — see that ADR's revision history for why.)
- `smb-chat-service` does **not** hold the mobile app's original HTTP request open for the duration of the stream. It acks the mobile request immediately (202, same as before) and consumes the orchestrator's SSE stream server-side, relaying chunks to the mobile app live over Web PubSub as they arrive (a progressive "typing" effect) and persisting the complete answer as one `ConversationEvent` once the stream ends. Live chunks themselves are never persisted — same ephemeral treatment as `TYPING` (§1.6) — only the final assembled message is.
- **Escalation has two trigger paths**, both converging on the same internal operation: an explicit user action (e.g. a "talk to a human" control) still calls `POST /chat/sessions/{id}/escalate` directly, unchanged; or the orchestrator decides to escalate (fraud keyword, out-of-policy request, user asked) by emitting a distinct `event: escalate` SSE event within the same stream, instead of a text chunk — `smb-chat-service` watches for this event type while consuming the stream for scenario 2's call.
- **Ordering is inherent, not a separate design problem.** A single HTTP request/response (even a streamed one) is ordered by construction — there's no broker to reorder messages out from under it. The remaining question is narrower: whether `smb-chat-service` allows a second `AI`-phase send for the same conversation while a previous one's stream is still open. Recommendation: no — serialize per conversation (reject or queue a second send until the in-flight stream ends or times out) rather than run two concurrent generations for one conversation. This is a much smaller decision than the broker-ordering question it replaces, and doesn't need its own ADR open item the way the Service Bus version did.
- **No reconciliation-sweep equivalent for the AI leg, same conclusion as before, different reason.** There's no redelivery-prone async channel to reconcile against — it's a synchronous call tied to the original request. A failed or dropped call surfaces as a failure (§4 item 10 covers the retry/failure policy), not a silent gap needing a backfill job.
- **Streaming connections need their own operational care** — idle/keep-alive timeouts on anything sitting between `smb-chat-service` and `chat-orchestrator-service` (load balancers, API gateways, proxies) must tolerate a long-lived open connection, and a heartbeat/comment-line convention is usually needed to stop intermediaries from killing it as idle. Confirm before build (§4 item 10).

**Connectivity check (device connected vs. disconnected):**
- "Device connected" is answered by Azure Web PubSub's own **User Exists** data-plane operation — `HEAD /api/hubs/{hub}/users/{userId}` (confirmed against current Microsoft docs, Aug 2026): **200** means at least one active connection is currently registered for this `userId` in the hub; **404** means none is. This is a **live check against Azure's own connection state**, called at the moment `smb-chat-service` needs the answer — not anything we track or infer locally ourselves, consistent with the earlier decision not to rebuild a `ConnectionRegistry` (single-device enforcement, above).
- `sendToUser(userId, event)` is always attempted regardless of this check's outcome — it's a safe no-op if nobody's connected (Web PubSub's send is fire-and-forget, no delivery confirmation returned). The User Exists check is what actually decides whether the notification fallback (push/email, scenario 12) also fires; the two calls are independent, not one gating the other's success.
- **This is a distinct mechanism from the `connected`/`disconnected` webhook system events** (scenario 9, §3 API row 11) — those are async signals Azure pushes to us over time, used only to reset the `offlineNotifiedAt` throttle; the User Exists check is a synchronous query made at message-delivery decision time, used per-message.
- **Known race, accepted:** a small window exists between the check and the moment delivery would land — e.g. the device disconnects a moment after the check reported "connected." The real-time push is then silently dropped *and* no notification fallback fires for that specific message. Not a correctness gap: the message is already persisted (write-before-push throughout §2) and surfaces on the customer's next catch-up (scenario 1/9) regardless — the only consequence is a delayed real-time notification, not a lost message.

**Notifications (offline agent response):**
- **Trigger:** the connectivity check above returns 404 (device disconnected) at the moment `smb-chat-service` would otherwise push an event to it. No new/separate "offline" detection is introduced.
- **Channels & frequency:** push fires on **every** inbound agent message while offline, unchanged from the original design. Email is **throttled to once per continuous-offline stretch** — the first offline agent message triggers both push and email; subsequent offline messages in the same stretch trigger push only. The throttle resets the moment the device reconnects (§4 item 6), so a later offline stretch in the same conversation gets a fresh email.
- **Content:** redacted on both channels, consistent with the existing push principle (§1.6) — no message body, no agent name; e.g. "You have a new message from your support agent." Applies equally to email.
- **Delivery mechanism:** both channels are brokered through the existing `notification-service` — `smb-chat-service` never calls FCM/APNs or an email provider directly, and never stores or looks up the customer's email address itself (assumed to be resolved by `notification-service` from the existing banking/identity profile).
- **Scope boundary:** this applies to inbound **agent messages** only (scenarios 6/7). The conversation-close notification (scenario 11) stays push-only, unthrottled — closure is a one-time terminal event, and email-on-close wasn't part of the stated requirement. Revisit if that's wanted later.

**Reconciliation:**
- Scoped to `HUMAN`-phase (D365) messages only — see "AI conversation" above for why the `AI` leg has no equivalent sweep.
- The job implements **ADR-0003 Option A — fixed-interval sweep only.** This is a *directed* choice, explicitly **not** ADR-0003's own preliminary recommendation (which was the hybrid Option C: a sparse fixed interval plus an immediate trigger on locally-observed failure signals). Proceeding with Option A simplifies the design: one cron bean, no flag table, no triggered/webhook-failure path.
- Confirmed against current D365 docs: `GET /conversation/{id}/messages` only ever returns `type: "message"` activities. This makes the sweep a **`MESSAGE`-only safety net** — it cannot backfill a lost `AGENT_ACCEPTED` or `CLOSED` webhook delivery (no equivalent history endpoint exists for those event types). A lost `CLOSED` is mitigated instead by the stale-open escalation (a conversation stuck open too long gets flagged for investigation); a lost `AGENT_ACCEPTED` is accepted as a residual, informational-only gap.

**API surface:**
- The full endpoint catalog is in §3 "API definition," now twelve rows. Two are **proposed**, not yet confirmed — see §4 item 3.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
C4Container
title Container diagram — Human Chat (Up By CBD ↔ D365 Omnichannel ↔ Conversational AI)

Person(customer, "SME Customer", "Uses the Up By CBD mobile app for banking and support chat")

Container_Boundary(bank, "Bank Chat Platform") {
    Container(web_pubsub, "Real-time Transport", "Azure Web PubSub", "Delivers inbound events to the connected device (ADR-0001). Mobile connects to this directly, not proxied through smb-chat-service.")
    Container(mobile_app, "Up By CBD", "iOS / Android", "Mobile banking app. Sends messages/attachments via REST; receives real-time push via a direct WebSocket connection.")
    Container(chat_service, "smb-chat-service", "Java, Spring Boot (BFF)", "Facade between the mobile app, D365 and chat-orchestrator-service. Routes each message to the right downstream channel based on Conversation.state (ADR-0005). Embeds the reconciliation sweep job (Spring @Scheduled, ADR-0003 Option A).")
    ContainerDb(chat_db, "smb-chat-service-db", "Azure SQL Database", "Compliance source of truth. Conversation, ConversationEvent, Attachment metadata (ADR-0002).")
    ContainerDb(blob_storage, "Attachment Storage", "Azure Blob Storage", "File bytes for message attachments — outbound (customer-uploaded), inbound (copied from D365), and AI-phase (fetched by the orchestrator via a facade-issued SAS URL).")
}

Container(chat_orchestrator, "chat-orchestrator-service", "Separate team/deployable, technology TBD", "Owns the AI conversation phase: intent routing, FAQ/RAG answers, flow state, model gateway to LLM providers (conversational_banking_service_context.md). Decides when to request escalation. Fetches attachment bytes directly via a facade-issued SAS URL carried in the request — same pattern as D365, not drawn as a separate edge.")
System_Ext(d365, "D365 Omnichannel", "Messaging API + webhook subscription. Human agent chat (ADR-0004: GA confirmed).")
System_Ext(notification_service, "Notification Service", "Existing shared platform service. Sends both push (FCM/APNs) and email on our behalf — redacted body only. We never talk to FCM/APNs or an email provider directly.")

Rel(customer, mobile_app, "Uses")
Rel(mobile_app, chat_service, "Sends messages, uploads attachments, escalates, reads, catches up", "HTTPS/REST")
Rel(mobile_app, web_pubsub, "Receives real-time push (direct connection)", "WSS")
Rel(chat_service, web_pubsub, "Issues connection tokens; pushes via sendToUser", "HTTPS (management API)")
Rel(chat_service, chat_db, "Reads/writes Conversation, ConversationEvent, Attachment metadata", "JDBC")
Rel(chat_service, blob_storage, "Uploads/downloads attachment file bytes", "HTTPS")
Rel(chat_service, d365, "Creates conversations, sends customer messages/attachments (HUMAN phase)", "HTTPS/REST")
Rel(d365, chat_service, "Delivers agent messages, AgentAccepted, AgentClosed/ConversationClosed", "HTTPS webhook")
Rel(chat_service, notification_service, "Requests push and/or email notification (device disconnected)", "HTTPS")
Rel(notification_service, mobile_app, "Delivers push notification", "APNs / FCM")
Rel(notification_service, customer, "Delivers email notification", "SMTP / email")
Rel(chat_service, chat_orchestrator, "Sends AI-phase message; receives streamed response (AI phase)", "HTTPS/REST (text/event-stream)")
:::

**Notes on this diagram:**
- `smb-chat-service` deliberately shows the reconciliation job as a *capability of the container*, not a separate box — it's a Spring `@Scheduled` bean in the same deployable, a Component-diagram-level (C3) distinction, not a Container-level (C2) one. Unlike the old FAQ SDK, `chat-orchestrator-service` is a genuinely separate deployable/team, which is why it *is* drawn as its own container.
- `smb-chat-service-db`, `Attachment Storage`, and `Real-time Transport` are modeled as containers *inside* the bank's own boundary, not as external systems — they're PaaS-hosted but are infrastructure this project provisions and owns the configuration of, unlike D365 and the notification service, which are genuinely third-party or separately-owned systems being integrated with.
- `chat-orchestrator-service` sits **outside** that boundary (different owning team, per `conversational_banking_service_context.md` §3) but is drawn as a `Container`, not a `System_Ext` — it's still a bank-owned system, just not one this project provisions.
- The `chat_service` ↔ `chat_orchestrator` call is a single synchronous HTTP relationship (ADR-0005) — `smb-chat-service` calls, `chat-orchestrator-service` streams its response back on the same connection (`text/event-stream`, to avoid blocking on a long-running generation). No broker sits between them.
- No edge is drawn between `chat_orchestrator` and `chat_service`/`blob_storage` for attachments — see the orchestrator's own description above; it mirrors D365's existing (also undrawn) direct SAS fetch rather than introducing a second access pattern.
- The mobile app's WebSocket connection to Web PubSub is direct, not proxied through `smb-chat-service` (ADR-0001 §6.1) — `smb-chat-service` only issues the connection token and calls `sendToUser`; the actual real-time data plane bypasses it.
- Push is delivered to `mobile_app` (device/OS-level), while email is delivered to `customer` (their inbox) — not modeled as flowing through `mobile_app`, since email arrives independently of the app being installed or running.

## Sequence diagram

12 scenarios covering the full conversation lifecycle, from app startup through escalation, messaging, delivery/read tracking, reconnect, reconciliation, close, and offline notification. All are drawn below.

### 1. App startup & connection lifecycle

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant wps as Azure Web PubSub

Note left of mobile: App launch (existing banking session/JWT already established)
mobile ->> chatsvc: GET /chat/current-session
chatsvc ->> db: SELECT latest non-closed Conversation for userId
db -->> chatsvc: Conversation (state, watermark) or none
chatsvc -->> mobile: { conversationId, state, watermark } or { none }

Note left of mobile: Connection opened only if a conversation exists (ADR-0001 §6.3) — never unconditionally on app foreground
alt conversation exists (state = AI or HUMAN)
    mobile ->> chatsvc: POST /chat/connection/negotiate
    chatsvc ->> wps: issue client access token (scoped to userId)
    wps -->> chatsvc: access token
    chatsvc -->> mobile: wss:// URL + token
    mobile ->> wps: open WebSocket (direct connection)
    Note left of mobile: Catch-up — fill any gap since last known watermark
    mobile ->> chatsvc: GET /chat/sessions/{id}/events?sinceSeq=watermark
    chatsvc ->> db: SELECT ConversationEvent WHERE sequenceNo > watermark
    db -->> chatsvc: events[]
    chatsvc -->> mobile: events[] (render transcript)
else no conversation exists
    Note left of mobile: No socket opened yet — waits until the user starts a chat (scenario 2: POST /chat/sessions)
end
:::

**Notes:**
- The connection-opened-only-if-a-conversation-exists gating is the conversation-state-scoped lifecycle from ADR-0001 §6.3, not the app-session-scoped model from the original architecture summary — deliberately superseded there.
- The catch-up call runs on *every* app start where a conversation exists, not just after a detected disconnect — it's cheap (indexed range query on `sequenceNo`) and removes any dependency on knowing whether a gap actually occurred.
- Auth (existing banking JWT) is assumed already established before this flow starts and isn't detailed here — out of scope for this diagram.

### 2. Start conversation + AI message exchange (REST, streamed response)

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant orch as chat-orchestrator-service
Participant wps as Azure Web PubSub
Participant notification as Notification Service

mobile ->> chatsvc: POST /chat/sessions
chatsvc ->> db: INSERT Conversation (state = AI)
db -->> chatsvc: conversationId
chatsvc -->> mobile: { conversationId, state: AI }

Note left of mobile: Conversation now exists — negotiate + open Web PubSub connection (same flow as scenario 1), not redrawn here

mobile ->> chatsvc: POST /chat/sessions/{id}/messages { contentType: TEXT, body: prompt }
chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=CUSTOMER, phase=AI, body=prompt)
db -->> chatsvc: sequenceNo assigned
Note left of chatsvc: Prompt persisted before calling the orchestrator — same "write first" principle as the D365 legs, so a crash before/during the call never loses the customer's original message
chatsvc -->> mobile: 202 Accepted { sequenceNo } — no answer yet
Note left of mobile: The reply is NOT in this response — mobile shows a "thinking" state and waits for the push below (§1.6, §4 item 10)

chatsvc ->> orch: POST /converse { conversationId, sequenceNo, text, attachments[] } — Accept: text/event-stream
Note over chatsvc,orch: Long-lived streamed response, not on mobile's request (already acked above). Orchestrator processing (routing, RAG, model calls — conversational_banking_service_context.md) is out of scope for this diagram

loop while the stream is open
    orch -->> chatsvc: event: delta { text }
    chatsvc ->> wps: sendToUser(userId, partial chunk) — best-effort live relay, never persisted, same ephemeral treatment as TYPING (§1.6)
    wps -->> mobile: deliver partial chunk over open WebSocket (if connected — safe no-op otherwise)
end

alt stream ends with event: done
    orch -->> chatsvc: event: done { messageId, contentType, text/richContent }
    chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AI_BOT, phase=AI, direction=INBOUND, externalMessageId=messageId, contentType, body/metadata)
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> wps: sendToUser(userId, final event) — always attempted, safe no-op if nobody's connected
    chatsvc ->> wps: HEAD /api/hubs/{hub}/users/{userId} (User Exists check, §1.6)
    wps -->> chatsvc: 200 (device connected) or 404 (device disconnected)
    alt 200 — device connected
        wps -->> mobile: deliver final event over open WebSocket
    else 404 — device disconnected
        chatsvc ->> notification: Trigger offline notification (push always, email throttled — scenario 12)
    end
else stream ends with event: escalate
    orch -->> chatsvc: event: escalate { reason }
    Note over chatsvc: Same internal escalate operation as scenario 3 — triggered here mid-stream rather than by a separate call. Not redrawn, see scenario 3.
else call fails or times out
    orch --x chatsvc: connection dropped / timeout
    Note right of chatsvc: No retry loop — logged and alerted (§4 item 10), not backfilled by anything, since there's no async channel to reconcile against (§1.6). The customer can simply resend.
end

Note left of mobile: After any exchange, the user may escalate to HUMAN (scenario 3) — or the orchestrator may request it itself
:::

**Notes:**
- The AI reply is **still not synchronous in the mobile-facing REST response** — that hasn't changed from the previous design. What changed is only the transport between `smb-chat-service` and `chat-orchestrator-service`: a direct REST call with a streamed response, not a message broker. `smb-chat-service` acks mobile (202) before making the downstream call, so a slow generation never ties up the original HTTP request.
- The live `delta` chunks give mobile a progressive "typing" effect, for free, as a consequence of streaming — but they're not the compliance record. Only the assembled final message (`event: done`) is persisted as a `ConversationEvent`, the same write-once-complete treatment the old synchronous FAQ answer got.
- The dedupe-then-push steps after `event: done` are deliberately drawn to mirror scenario 6 line for line — once the final answer exists, delivering it to the device is the exact same mechanism as a D365 agent message, just arriving from a different source.
- Both the customer's prompt and the orchestrator's answer are persisted as separate `ConversationEvent` rows (`origin=CUSTOMER` and `origin=AI_BOT` respectively) — consistent with the confirmed decision that `AI`-phase messages share the same compliance-store write path as `HUMAN`-phase ones.
- No retry loop is drawn on failure, matching the D365 outbound pattern (scenario 4) — the difference is there's no reconciliation sweep behind this one (§1.6), so a failure is a user-visible failure, not a silently-backfilled gap. §4 item 10 covers the retry/alerting policy still to be defined.
- Ordering is no longer a broker-level open question (§1.6) — the narrower remaining decision (whether a second send for the same conversation can run concurrently with this one) is noted there too.

### 3. Escalate AI → HUMAN

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant d365 as D365 Omnichannel
Participant wps as Azure Web PubSub

alt User-initiated (explicit "talk to a human" action)
    mobile ->> chatsvc: POST /chat/sessions/{id}/escalate
else Orchestrator-initiated (fraud keyword, out-of-policy request, or the user asked mid-conversation — conversational_banking_service_context.md)
    Note over chatsvc: Arrives as an event: escalate SSE event mid-stream, inside scenario 2's in-flight call for this conversation — not a separate inbound call, not redrawn here
end

Note over chatsvc: Same internal escalate operation either way (§1.6) — the trigger path doesn't change what happens next

chatsvc ->> db: SELECT ConversationEvent WHERE conversationId=id AND phase=AI ORDER BY sequenceNo
db -->> chatsvc: AI-phase events[]
chatsvc ->> chatsvc: Format AI-phase transcript into D365 context variables
chatsvc ->> d365: POST /conversation/create { contextVariables }
Note right of d365: Context variables are eventually consistent on D365's side (confirmed via docs) — the agent isn't guaranteed to see them the instant the conversation starts. Nothing to fix on our side — the D365 conversation is still created synchronously.
d365 -->> chatsvc: d365ConversationId
chatsvc ->> db: UPDATE Conversation SET state=HUMAN, d365ConversationId, escalatedAt=now
Note over chatsvc,db: One-way, terminal for the AI leg (§1.2/§1.6, ADR-0005) — no code path ever sets state back to AI, and chat-orchestrator-service has no further role in this conversation past this write
chatsvc ->> db: INSERT ConversationEvent (eventType=ESCALATED, phase=HUMAN, metadata={d365ConversationId, trigger})
db -->> chatsvc: sequenceNo assigned

alt User-initiated
    chatsvc -->> mobile: 200 OK { conversationId, state: HUMAN }
else Orchestrator-initiated
    Note over chatsvc: No pending HTTP request to answer — mobile learns of the transition the same way it learns of any async event
    chatsvc ->> wps: sendToUser(userId, ESCALATED event) — same push mechanism as scenarios 2/6
end

Note left of mobile: Web PubSub connection was already opened when the conversation was first created (scenario 1/2) — no reconnect needed here. Subsequent messages now route to D365 instead of chat-orchestrator-service (scenario 4).
:::

**Notes:**
- The AI-phase transcript read (`phase=AI`) is scoped by the same `(conversationId, phase, sequenceNo)` index called out in §1.6 — no full-table scan.
- The `ESCALATED` event is stamped `phase=HUMAN`, not `AI` — it marks the transition *into* the HUMAN phase, consistent with `phase` being "which phase this event belongs to" rather than "which phase produced it."
- `d365ConversationId` and `escalatedAt` on `Conversation` are denormalized mirrors — the `ESCALATED` `ConversationEvent` row is the audit source of truth; the `Conversation` row is the fast-lookup copy.
- D365's context-variables eventual-consistency behavior is shown as a note, not a retry/poll loop — there's nothing actionable on the facade side; it's a constraint on agent-side UX expectations, out of scope for this diagram.
- The two trigger paths converge deliberately — there is exactly one escalate code path in `smb-chat-service`, never two parallel implementations to keep in sync. The orchestrator-initiated path needs its own push to the mobile app (no HTTP request to answer, since it surfaces mid-stream inside scenario 2's call, not as a reply to a fresh request); the user-initiated path doesn't, since the REST response already carries the new state.

### 4. Outbound text/link message (HUMAN)

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant d365 as D365 Omnichannel

mobile ->> chatsvc: POST /chat/sessions/{id}/messages { contentType: TEXT, body }
Note over chatsvc: Validate body ≤ 6,000 chars — text or attachments must be present (§1.6 — enforced at facade, not left to D365 to reject)
chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=CUSTOMER, phase=HUMAN, direction=OUTBOUND, contentType=TEXT, body)
db -->> chatsvc: sequenceNo assigned
Note over chatsvc: Persisted before forwarding to D365 — same "write first" principle as scenario 2 — this row is also what reconciliation (scenario 10) diffs against later
chatsvc ->> d365: POST /conversation/{d365ConversationId}/messages { text: body }
alt D365 accepts
    d365 -->> chatsvc: 200 OK { activityId }
    chatsvc ->> db: UPDATE ConversationEvent SET externalMessageId=activityId WHERE id=...
    chatsvc -->> mobile: 200 OK { sequenceNo }
else D365 call fails/times out
    d365 -->> chatsvc: error / no response
    chatsvc -->> mobile: 5xx (row already persisted with externalMessageId=null)
    Note right of chatsvc: No retry loop here — the row's null externalMessageId makes it visible to the reconciliation sweep (scenario 10, ADR-0003), which is the actual safety net for this gap
end
:::

**Notes:**
- A link message is just `contentType: TEXT` with a URL in `body` — no distinct type or preview metadata (working assumption, §1.6; unresolved rich-preview requirement, §4 item 1).
- The row is written with `externalMessageId = null` at persist time and only backfilled once D365 confirms (`activityId`) — this is why `externalMessageId` has to be nullable for `HUMAN`-phase rows too, not just in-flight `AI`-phase ones (§1.6): outbound-in-flight is a second, narrower reason it's nullable for HUMAN rows specifically, closed by either this update or the reconciliation sweep.
- Behavior branch on `Conversation.state` (`AI` → call chat-orchestrator-service vs. `HUMAN` → D365 forward) lives entirely in `POST /chat/sessions/{id}/messages` per the endpoint table in §3 (API definition, row 4) — this diagram only shows the HUMAN branch; scenario 2 showed the AI branch.
- No explicit retry is drawn on D365 failure — deliberate, per ADR-0003's Option A design: the fixed-interval reconciliation sweep is the agreed mitigation for this gap, not a facade-level retry loop.

### 5. Outbound file message (HUMAN)

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant blob as Attachment Storage
Participant db as smb-chat-service-db
Participant d365 as D365 Omnichannel

mobile ->> chatsvc: POST /chat/sessions/{id}/attachments (file bytes)
Note over chatsvc: Validate mimeType ∈ {image/jpeg, image/png, application/pdf}, sizeBytes ≤ 10MB (§1.6) — facade-proxied upload, no direct-to-blob SAS from mobile
chatsvc ->> blob: Upload file bytes
blob -->> chatsvc: blobStorageRef
chatsvc ->> chatsvc: Generate contentUrl — SAS URL, fixed expiry (attachment.sas-token.expiry-minutes, §2)
chatsvc ->> db: INSERT Attachment (fileName, mimeType, sizeBytes, blobStorageRef, contentUrl) — eventId still null
db -->> chatsvc: attachmentId
chatsvc -->> mobile: 200 OK { attachmentId }

mobile ->> chatsvc: POST /chat/sessions/{id}/messages { contentType: FILE, attachmentId }
chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=CUSTOMER, phase=HUMAN, direction=OUTBOUND, contentType=FILE, attachmentId)
db -->> chatsvc: sequenceNo assigned
chatsvc ->> db: UPDATE Attachment SET eventId WHERE id=attachmentId
Note over chatsvc: contentUrl must be D365-fetchable before the message activity referencing it is sent — file lands in our Blob Storage first, D365 fetches it via contentUrl, never receives inline bytes (§1.6, confirmed against D365 docs)
chatsvc ->> d365: POST /conversation/{d365ConversationId}/messages { attachments: [{ contentType, contentUrl, name }] }
alt D365 accepts
    d365 -->> chatsvc: 200 OK { activityId }
    chatsvc ->> db: UPDATE ConversationEvent SET externalMessageId=activityId WHERE id=...
    chatsvc -->> mobile: 200 OK { sequenceNo }
else D365 call fails/times out
    d365 -->> chatsvc: error / no response
    chatsvc -->> mobile: 5xx (row already persisted with externalMessageId=null)
    Note right of chatsvc: Same reconciliation safety net as scenario 4 (ADR-0003) — no facade-level retry loop
end
:::

**Notes:**
- Two-call flow (`POST /attachments` then `POST /messages`) matches the endpoint catalog in §3 (rows 4–5) — upload and message-send are separate REST operations, not one combined call, so a file can be uploaded and confirmed before the customer commits to sending the message.
- `Attachment.eventId` is null between the two calls — an `Attachment` row can exist (file uploaded, stored, `contentUrl` minted) before the `ConversationEvent` that references it exists. The FK is backfilled once the message is persisted, not required at `Attachment` insert time.
- No virus scanning step is drawn — confirmed descoped (§1.6); the file is stored as uploaded.
- Same write-before-forward and reconciliation-as-safety-net pattern as scenario 4 — deliberately consistent rather than introducing file-specific retry/error handling.

### 6. Inbound agent text message

::: mermaid
sequenceDiagram
autonumber
Participant d365 as D365 Omnichannel
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant wps as Azure Web PubSub
Participant mobile as Up By CBD
Participant notification as Notification Service

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: message, activityId, text, from: agent }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Ack fast, then process off the webhook thread — avoids D365's retry-on-timeout behavior — the ack is not conditioned on the DB write below succeeding

chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, contentType=TEXT, body=text, externalMessageId=activityId)
alt externalMessageId not seen before
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> wps: sendToUser(userId, event) — always attempted, safe no-op if nobody's connected
    chatsvc ->> wps: HEAD /api/hubs/{hub}/users/{userId} (User Exists check, §1.6)
    wps -->> chatsvc: 200 (device connected) or 404 (device disconnected)
    alt 200 — device connected
        wps -->> mobile: deliver event over open WebSocket
    else 404 — device disconnected
        chatsvc ->> notification: Trigger offline notification (push always, email throttled — scenario 12)
    end
else externalMessageId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on externalMessageId — no-op
    Note over chatsvc: Idempotent dedupe (§1.6) — a retried webhook delivery is silently absorbed, never double-persisted or double-pushed
end
:::

**Notes:**
- Ack-then-process is the same pattern ADR-0003 establishes for webhook handling generally — this diagram is the first to actually show it, since scenarios 1–5 were all facade-initiated rather than webhook-initiated.
- `externalMessageId` (populated from D365's `activityId` for this phase) is the dedupe key (unique constraint, §1.6) — D365's own at-least-once webhook delivery (3-attempt retry) means duplicate deliveries are expected, not exceptional; this path is what makes them safe.
- "Device connected" is a live Web PubSub check (§1.6), not local state — see §1.6's "Connectivity check" group for the mechanism and its known race condition.
- Offline notification (push + throttled email) is only triggered when the check reports disconnected — not sent unconditionally alongside every Web PubSub push. Full mechanics in scenario 12, kept out of this diagram to stay focused on the inbound-message path.
- Read receipt / delivery-ack handling is a separate scenario (#8) — not shown here.

### 7. Inbound agent file message

::: mermaid
sequenceDiagram
autonumber
Participant d365 as D365 Omnichannel
Participant chatsvc as smb-chat-service
Participant blob as Attachment Storage
Participant db as smb-chat-service-db
Participant wps as Azure Web PubSub
Participant mobile as Up By CBD
Participant notification as Notification Service

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: message, activityId, attachments: [{ contentType, contentUrl, name }], from: agent }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Same ack-then-process pattern as scenario 6

alt externalMessageId not seen before
    chatsvc ->> d365: GET contentUrl (fetch file bytes from D365-hosted URL)
    d365 -->> chatsvc: file bytes
    chatsvc ->> blob: Upload file bytes — copy into our own Blob Storage
    blob -->> chatsvc: blobStorageRef
    chatsvc ->> chatsvc: Generate our own contentUrl (facade-brokered)
    chatsvc ->> db: INSERT Attachment (fileName, mimeType, sizeBytes, blobStorageRef, contentUrl)
    db -->> chatsvc: attachmentId
    chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, contentType=FILE, attachmentId, externalMessageId=activityId)
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> db: UPDATE Attachment SET eventId WHERE id=attachmentId
    chatsvc ->> wps: sendToUser(userId, event) — always attempted, safe no-op if nobody's connected
    chatsvc ->> wps: HEAD /api/hubs/{hub}/users/{userId} (User Exists check, §1.6)
    wps -->> chatsvc: 200 (device connected) or 404 (device disconnected)
    alt 200 — device connected
        wps -->> mobile: deliver event over open WebSocket
    else 404 — device disconnected
        chatsvc ->> notification: Trigger offline notification (push always, email throttled — scenario 12)
    end
    Note left of mobile: File bytes downloaded later via facade-brokered URL, not fetched directly from Blob Storage — see note below on the missing endpoint
else externalMessageId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on externalMessageId — no-op
end
:::

**Notes:**
- The copy-not-reference decision (§1.6) means our Blob Storage becomes the retention source of truth for inbound files too — the customer's later ability to view an agent-sent file never depends on D365 continuing to host it.
- The file download (D365 → facade) happens as part of the same async, post-ack processing as the metadata persist — a large/slow file download doesn't block the webhook ack.
- Idempotent dedupe on `externalMessageId` (same mechanism as scenario 6) covers this path too — a retried webhook won't re-download or re-copy the file.
- "Device connected" is the same live Web PubSub User Exists check as scenario 6 (§1.6) — not repeated in full here.
- **Gap this diagram surfaced:** there was no `GET` route for the mobile app to download an inbound attachment's bytes via the facade — only `POST /chat/sessions/{id}/attachments` (upload, outbound direction) existed. Addressed as `GET /chat/attachments/{attachmentId}` in the API catalog (§3, row 10) — still marked **proposed**, needs sign-off (§4 item 3).

### 8. Delivery ack + read receipt

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db

Note over mobile: Continues from scenario 6/7 — mobile just received a pushed event over its open WebSocket
mobile ->> chatsvc: POST /chat/sessions/{id}/delivered { sequenceNo }
chatsvc ->> db: UPDATE Conversation SET deliveryWatermark = sequenceNo WHERE deliveryWatermark < sequenceNo
chatsvc -->> mobile: 200 OK
Note over chatsvc: Monotonic guard — a late/out-of-order ack for an older sequenceNo never regresses the watermark. No ConversationEvent row is written here: delivery ack is ephemeral confirmation, same treatment as TYPING (§1.6) — no compliance/audit value, just the fast-lookup field advancing.

Note over mobile: Later, independently — user actually views the conversation screen (read watermark is a separate signal from delivery, ADR-0001 §6.3)
mobile ->> chatsvc: POST /chat/sessions/{id}/read { sequenceNo }
chatsvc ->> db: INSERT ConversationEvent (eventType=READ_RECEIPT, origin=CUSTOMER, phase=Conversation.state, relatedSequenceNo=sequenceNo)
db -->> chatsvc: sequenceNo assigned (the READ_RECEIPT row's own sequenceNo — distinct from relatedSequenceNo, the event being acknowledged)
chatsvc ->> db: UPDATE Conversation SET readWatermark = relatedSequenceNo WHERE readWatermark < relatedSequenceNo
chatsvc -->> mobile: 200 OK
:::

**Notes:**
- This is the two-watermark model described in full above, keyed by the single global `sequenceNo` spanning both AI and HUMAN phases rather than a per-phase counter.
- **Delivery ack and read receipt use the identical REST shape** (`POST /chat/sessions/{id}/{delivered|read} { sequenceNo }`) — a deliberate consistency decision, superseding an earlier draft that put delivery-ack on a Web PubSub upstream client message. REST was chosen over the socket: every other client→server interaction in this design already goes through the REST API, and it removes a dependency on Web PubSub's upstream-message/event-handler mechanism, which was never confirmed to behave as needed.
- **They still differ in what gets persisted**, mirroring the `TYPING`-vs-persisted-event distinction already established in §1.6: delivery ack only ever touches the denormalized `Conversation.deliveryWatermark` field (no audit row), while read receipt is fully persisted as a `ConversationEvent` *and* mirrored to `Conversation.readWatermark` — because read history was confirmed to have standalone audit/debugging value, delivery-ack history was not. That asymmetry is about compliance value, not transport.
- Both watermark updates use a `WHERE current < new` guard rather than an unconditional `SET` — necessary because delivery/read confirmations can arrive slightly out of order relative to `sequenceNo` over an unreliable network path, and the watermark must never regress.

### 9. Reconnect + catch-up

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant wps as Azure Web PubSub
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db

Note over mobile,wps: Prior state — mobile already has an open WebSocket connection (scenario 1)

alt Short transient network blip (seconds)
    mobile -x wps: Connection drops (brief blip)
    mobile ->> wps: Reconnect using connectionId + reconnectionToken (reliable subprotocol)
    wps -->> mobile: Resumed — same logical connection/session state
    Note over mobile,wps: Handled entirely by the Web PubSub client SDK (ADR-0001 §6.4) — facade not involved, no catch-up call needed since no gap actually occurred
else App backgrounded/suspended, then foregrounded later
    mobile -x wps: OS suspends app — socket stops being serviced (no pong replies)
    wps ->> chatsvc: System event: disconnected (CloudEvent — userId, connectionId, reason)
    chatsvc ->> chatsvc: Remove dead connection from userId's active set
    Note over mobile: Time passes while the app is backgrounded
    Note left of mobile: App foregrounded — this is a fresh connect, not a resume. Azure has no memory of "this device was here before" beyond the identity claim (ADR-0001 §6.4)
    mobile ->> chatsvc: POST /chat/connection/negotiate
    chatsvc ->> wps: Issue new client access token (scoped to userId)
    wps -->> chatsvc: access token
    chatsvc -->> mobile: wss:// URL + token
    mobile ->> wps: Open new WebSocket (direct connection)
    wps ->> chatsvc: System event: connected (CloudEvent — userId, connectionId)
    chatsvc ->> db: UPDATE Conversation SET offlineNotifiedAt = NULL WHERE offlineNotifiedAt IS NOT NULL
    Note over chatsvc: Resets the offline-notification email throttle (scenario 12, §1.6) — a later offline stretch in this conversation can trigger a fresh email
    mobile ->> chatsvc: GET /chat/sessions/{id}/events?sinceSeq=watermark
    chatsvc ->> db: SELECT ConversationEvent WHERE conversationId=id AND sequenceNo > watermark ORDER BY sequenceNo
    db -->> chatsvc: events[] (may span both AI- and HUMAN-phase rows)
    chatsvc -->> mobile: events[] (render transcript, replay anything missed while disconnected)
end
:::

**Notes:**
- The "app backgrounded" branch reuses scenario 1's negotiate → connect → catch-up shape exactly — the difference here is *why* it's happening (a genuine reconnect after a real disconnect, not first app launch) and that ADR-0001 §6.4's `disconnected` system event is shown explicitly as the trigger for removing the dead connection.
- Catch-up is phase-agnostic: a missed AI-phase message replays through the same `sinceSeq` query as a missed HUMAN-phase D365 message, since `ConversationEvent.sequenceNo` is one global sequence spanning both phases (§1.6). Nothing D365-specific about this path.
- The short-blip branch requires **no facade involvement at all** — it's purely the Web PubSub client SDK and Azure's own reliable-subprotocol recovery. It's included here (rather than omitted) specifically to make clear why the "app backgrounded" branch's more expensive fresh-connect-plus-catch-up path only fires for a real disconnect, not every network hiccup.
- **Gap this diagram surfaced, same category as #7:** ADR-0001 §6.4 says the `disconnected` CloudEvent fires to "our webhook/upstream endpoint," but no such endpoint existed — the original API list only had the D365 webhook receiver. Addressed as `POST /webhooks/webpubsub/events` in the API catalog (§3, row 11) — still marked **proposed**, needs sign-off (§4 item 3).
- The exact recovery-window duration for the short-blip branch is still an open ADR-0001 item (§8, "pull the current documented connection-recovery window") — this diagram assumes the branch exists but doesn't depend on the exact duration.
- The `connected` event and its throttle-reset write are new — added to support the offline-notification email throttle (scenario 12). Unlike `disconnected`, ADR-0001 doesn't currently document a `connected` system event explicitly; its existence and payload shape need confirming (§4 item 7). This is also why row 11's endpoint (§3) can no longer be described as purely informational/logging — it now performs one real write.

### 10. Reconciliation sweep

::: mermaid
sequenceDiagram
autonumber
Participant cron as reconciliationSweepJob<br/>(Spring @Scheduled, fixed interval)
Participant db as smb-chat-service-db
Participant d365 as D365 Omnichannel
Participant wps as Azure Web PubSub

Note over cron: Fires unconditionally on its cron schedule — Option A only (ADR-0003, as directed): no flag table, no triggered path, one bean
cron ->> db: SELECT Conversation WHERE state = HUMAN
db -->> cron: conversations[] — each with d365ConversationId + last-known MESSAGE watermark

loop for each HUMAN-state conversation
    cron ->> d365: GET /conversation/{d365ConversationId}/messages?StartTimeStamp=watermark
    d365 -->> cron: message activities since watermark (possibly empty)
    cron ->> cron: Diff returned activityIds against stored ConversationEvent.externalMessageId values WHERE phase=HUMAN AND eventType=MESSAGE
    alt gap found (externalMessageId not yet persisted)
        cron ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, externalMessageId=activityId, body/attachment) — same idempotent write path as the webhook (scenarios 6/7)
        db -->> cron: sequenceNo assigned
        cron ->> wps: sendToUser(userId, event) — deliver the backfilled message live, same as a normal inbound push
        Note over cron: The whole point of reconciliation is the customer eventually sees the message — a silent backfill with no push would leave the transcript correct but the customer none the wiser until their next catch-up
    end
    cron ->> db: Advance stored watermark for this conversation
end

Note over cron,db: Separately — stale-open escalation (ADR-0003 §4.2): a conversation stuck in state=HUMAN with no new activity across multiple sweep passes beyond a threshold is flagged for investigation, not reconciled forever
:::

**Notes:**
- Scope is `Conversation.state = HUMAN` only, sourced from our own state — never a D365 conversation-list call, per ADR-0003 §2's finding that D365's "don't poll" warning applies to the all-conversations endpoint, not the per-conversation messages endpoint this job actually uses.
- **This is a `MESSAGE`-only safety net.** Confirmed via docs (§1.6): `GET /conversation/{id}/messages` only ever returns `type: "message"` activities — so this sweep cannot backfill a lost `AGENT_ACCEPTED` or `CLOSED` webhook delivery; there's no equivalent history endpoint for those event types to diff against.
    - A lost `CLOSED` is mitigated by the **stale-open escalation** shown above — it detects a conversation that never received its close event, rather than trying to backfill the event itself.
    - A lost `AGENT_ACCEPTED` is accepted as a residual gap (informational/audit-only, doesn't block anything operationally) unless told otherwise.
- Diffing key is `externalMessageId` (D365's `activityId` for this phase) — the same idempotent-write/dedupe mechanism used by the webhook receiver (scenarios 6/7), so a message the webhook already delivered is never double-persisted or double-pushed just because it also shows up in the reconciliation fetch.
- This diagram reflects **Option A (fixed-interval only) as confirmed/directed** (§1.6) — not ADR-0003's own preliminary recommendation (Option C, hybrid with a triggered path). No flag table, no second cron bean, no webhook-receiver-writes-a-flag step — deliberately simpler than ADR-0003 §8's Option C diagram.
- Interval value itself (ADR-0003 §7 suggests 15 minutes as a starting point) is still pending confirmation against compliance's acceptable-staleness answer and Microsoft's actual rate limit (§4 item 4) — not fixed by this diagram.

### 11. Conversation close by agent

::: mermaid
sequenceDiagram
autonumber
Participant d365 as D365 Omnichannel
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant wps as Azure Web PubSub
Participant mobile as Up By CBD
Participant notification as Notification Service

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: event, activityId, name: ConversationClosed | AgentClosed }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Same ack-then-process pattern as scenarios 6/7/9

alt externalMessageId not seen before
    chatsvc ->> db: UPDATE Conversation SET state=CLOSED, closedAt=now, closedReason WHERE id=...
    chatsvc ->> db: INSERT ConversationEvent (eventType=CLOSED, origin=AGENT, phase=HUMAN, externalMessageId=activityId, metadata={reason})
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> wps: sendToUser(userId, event) — always attempted, safe no-op if nobody's connected
    chatsvc ->> wps: HEAD /api/hubs/{hub}/users/{userId} (User Exists check, §1.6)
    wps -->> chatsvc: 200 (device connected) or 404 (device disconnected)
    alt 200 — device connected
        wps -->> mobile: Deliver CLOSED event over open WebSocket
        Note left of mobile: After a short grace period (~30–60s, ADR-0001 §6.3) to let a trailing ack/read-receipt round-trip land
        mobile ->> wps: Close WebSocket — conversation-state-scoped lifecycle ends here, not app-session-scoped
    else 404 — device disconnected
        chatsvc ->> notification: Send push notification (redacted body only)
        notification -->> mobile: Deliver push (APNs/FCM)
        Note left of mobile: Next app foreground follows scenario 9's reconnect flow — GET /chat/current-session finds state=CLOSED and does not reopen a socket (ADR-0001 §6.3)
    end
else externalMessageId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on externalMessageId — no-op
end
:::

**Notes:**
- Reuses the ack-then-process and idempotent-dedupe pattern from scenarios 6, 7, and 9 — `CLOSED` is just another D365 activity type flowing through the same webhook receiver, not a special-cased endpoint.
- "Device connected" is the same live Web PubSub User Exists check as scenarios 6/7 (§1.6) — not repeated in full here.
- **Push only here, no email, no throttle** — unlike the offline-notification flow in scenario 12, conversation close is a one-time terminal event, and email-on-close wasn't part of the stated notification requirement (§1.6). Revisit if that's wanted later.
- `state=CLOSED` is terminal (§1.6) — this is the one place the conversation-state-scoped connection lifecycle (ADR-0001 §6.3) actually tears the socket down, as opposed to scenario 1/2/3 where it's opened or kept open.
- **Who initiates the socket close is modeled as client-side**, after the grace period, matching ADR-0001 §6.3's rationale of bounding connection cost/exposure to "while there's something to protect" — once `CLOSED`, there isn't. This is a design choice to confirm in Phase 3, not a documented Azure requirement: Web PubSub also exposes a server-side `closeClientConnection` management-API primitive if a forced/immediate close is later preferred over waiting on the client.
- A lost `CLOSED` webhook delivery is **not** backfilled by this diagram — per scenario 10 / ADR-0003 §4.2, the mitigation is the reconciliation sweep's stale-open escalation (detecting a conversation stuck open too long), not a retry of this specific event.
- **This is the only path to `CLOSED` in the current design** — it requires an agent to close a D365 conversation, which requires having escalated first. A conversation that stays in `AI` phase and never escalates has no defined terminal state at all (the customer can simply stop using it). Not a regression introduced by this change — the old `FAQ`-phase design had the identical gap — but worth flagging now that `AI`-phase conversations are expected to be longer and more frequent (§4 item 12).

### 12. Offline notification: push + email (throttled)

::: mermaid
sequenceDiagram
autonumber
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant notification as Notification Service
Participant mobile as Up By CBD
Participant customer as SME Customer (inbox)

Note over chatsvc: Entered from scenario 6 or 7's "device disconnected" branch — one inbound agent message while the customer is offline

chatsvc ->> notification: Send push notification (redacted body only)
notification -->> mobile: Deliver push (APNs/FCM)

chatsvc ->> db: SELECT Conversation.offlineNotifiedAt WHERE id=conversationId
db -->> chatsvc: offlineNotifiedAt (null or timestamp)

alt offlineNotifiedAt is null (first offline agent message since the last reconnect)
    chatsvc ->> notification: Send email notification (redacted body only)
    notification -->> customer: Deliver email (SMTP)
    chatsvc ->> db: UPDATE Conversation SET offlineNotifiedAt = now WHERE id=conversationId
else offlineNotifiedAt already set (already notified this offline stretch)
    Note over chatsvc: Email skipped — already sent one for this continuous-offline stretch (§1.6). Push above is unaffected: it fires on every offline agent message regardless.
end
:::

**Notes:**
- Push is unconditional on every call into this flow; email is the only throttled leg — push stays unthrottled because OS-level notifications don't carry the same inbox-spam risk as repeated emails (§1.6).
- `offlineNotifiedAt` is reset only by the `connected` system-event handler added to scenario 9 — that's the sole way this flow can send a fresh email for the same conversation, once the customer reconnects and later goes offline again.
- Both channels are brokered entirely through `notification-service` (§1.6) — `smb-chat-service` holds no email address and never talks to FCM/APNs or an email provider directly.
- Content is redacted on both channels — no message body, no agent name, matching the existing push-redaction principle (§1.6).
- The throttle's read-then-write (`SELECT` then conditional `UPDATE`) is not compare-and-swap guarded the way the watermarks are (§3, Watermark model) — a rare race between two near-simultaneous offline messages could in theory send two emails instead of one. Accepted as a low-stakes gap for a notification-convenience feature, not a compliance-critical path; revisit if a stricter guarantee is later required.
- This closes out the 12-scenario set — all sequence diagrams are now drawn.

# 3. Design & Implementation

## Design & UX (Figma)

_(link to figma)_

**Status: pending** — needs a Figma link (external input, not yet provided).

## Backlog

User journey catalog derived from the confirmed scope (§1.2) and the 12 sequence diagrams (§2) — each user journey is traceable to the diagram(s) that specify its behavior and the API endpoint(s) it exercises (§3, API definition). This is the input for turning the design into a phased delivery plan (roadmap Phase 7), not a sprint-level breakdown.

### User Journeys

| # | User journey | Description | Sequence diagram(s) | Endpoint(s) |
|---|---|---|---|---|
| 1 | Conversation startup & connection lifecycle | Discover an existing conversation on app launch, open the real-time connection only if one exists, and catch up on missed events | #1 | `GET /chat/current-session`, `POST /chat/connection/negotiate`, `GET /chat/sessions/{id}/events` |
| 2 | AI conversation | Start a conversation and exchange messages with `chat-orchestrator-service` over REST, response streamed via text/event-stream | #2 | `POST /chat/sessions`, `POST /chat/sessions/{id}/messages` |
| 3 | Escalation to a human agent | Hand off from AI to D365, including the AI-phase transcript as context — user-initiated or orchestrator-initiated | #3 | `POST /chat/sessions/{id}/escalate` |
| 4 | Outbound text/link messaging (HUMAN) | Send a text or link message to the live agent, with facade-side validation and reconciliation as the failure safety net | #4 | `POST /chat/sessions/{id}/messages` |
| 5 | Outbound file messaging (HUMAN) | Upload an image/PDF via the facade, then send it to the agent as an attachment | #5 | `POST /chat/sessions/{id}/attachments`, `POST /chat/sessions/{id}/messages` |
| 6 | Inbound agent text messaging | Receive an agent's text message via webhook and push it to the device (or fall back to a redacted notification) | #6 | `POST /webhooks/d365/conversations/{id}/activities` |
| 7 | Inbound agent file messaging | Receive an agent's file via webhook, copy it into our own Blob Storage, and make it downloadable via the facade | #7 | `POST /webhooks/d365/conversations/{id}/activities`, `GET /chat/attachments/{attachmentId}` **(proposed, §4 item 3)** |
| 8 | Delivery ack & read receipts | Track the two-watermark model — transport-level delivery vs. product-level read (§3, Watermark model) | #8 | `POST /chat/sessions/{id}/delivered`, `POST /chat/sessions/{id}/read` |
| 9 | Reconnect & catch-up | Recover from a short network blip or a real backgrounding episode without losing events | #9 | `POST /chat/connection/negotiate`, `GET /chat/sessions/{id}/events`, `POST /webhooks/webpubsub/events` **(proposed, §4 item 3)** |
| 10 | Reconciliation sweep | Backfill any `MESSAGE` event a lost/failed D365 webhook delivery would otherwise have dropped | #10 | Internal cron job — no client-facing endpoint |
| 11 | Conversation close by agent | Terminate the conversation when the agent closes it, notify the device, and tear down the real-time connection | #11 | `POST /webhooks/d365/conversations/{id}/activities` |
| 12 | Offline notification (push + email) | Notify a disconnected customer that an agent responded — push every time, email throttled to once per continuous-offline stretch | #6, #7, #9, #12 | None directly — brokered via `notification-service`; throttle reset piggybacks on `POST /webhooks/webpubsub/events` **(proposed, §4 item 3)** |

**Cross-cutting, not separately numbered above** (each spans multiple user journeys rather than being one): attachment type/size validation (§1.6), idempotent webhook/bus dedupe via `externalMessageId` (§1.6), and the watermark model itself (§3) all apply across user journeys 2–11 rather than to a single one.

**Explicitly not in this backlog** — see §1.2 for the full out-of-scope list (multi-device fan-out, attachment virus scanning, unrelated event types over the same channel, direct-to-blob upload) and §4 for items pending confirmation before they can be scoped as user journeys (rich link previews, the two proposed endpoints).

**Suggested build order:** roughly user journeys 1–2 (facade skeleton + compliance-store write path) → 3–5 (D365 integration, outbound direction) → 6–7 (inbound webhook + real-time delivery) → 8–9 (watermark/reconnect polish) → 10–11 (reconciliation + close) → 12 (offline notification — depends on 6/7's disconnected branch and 9's `connected`-event throttle reset already existing), consistent with the phased approach the roadmap's Phase 7 already outlines — not a commitment to sprint boundaries, just a dependency-respecting order.

### Decisions

- [ADR-0001: Real-time transport](./adr-0001-realtime-transport.md) — Azure Web PubSub
- [ADR-0002: Channel System technology](./adr-0002-channel-system-technology.md) — Azure SQL Database only
- [ADR-0003: Reconciliation job](./adr-0003-reconciliation-job.md) — fixed-interval cron (Option A)
- [ADR-0004: Webhook GA status](./adr-0004-webhook-ga-status.md) — resolved, GA confirmed
- [ADR-0005: AI orchestrator integration](./adr-0005-ai-orchestrator-integration.md) — chat-orchestrator-service extraction, synchronous REST with a text/event-stream response for long-running generation

### Tests

_(provide Tests link)_

**Status: pending.**

## Watermark model (delivery vs. read)

The design uses two independent watermarks on `Conversation` — `deliveryWatermark` and `readWatermark` — plus a third, unrelated cursor used only inside the reconciliation job. All three get called "watermark" informally, so this section is explicit about what each one actually *is*.

**What a watermark is, concretely:** for `deliveryWatermark` and `readWatermark`, a watermark is **`ConversationEvent.sequenceNo`** — a plain integer, database-generated and globally monotonic, assigned atomically at insert time. It is **not** a UUID and **not** a timestamp. Two other per-event fields could be mistaken for the watermark and deliberately aren't used as one:
- `externalMessageId` — the idempotent-dedupe key (unique constraint) for any row sourced from an external async system: D365's own Bot Framework `activityId` (string, UUID-shaped) for `HUMAN`-phase rows, `chat-orchestrator-service`'s own assigned message ID for `AI`-phase rows. It has no ordering meaning and isn't comparable across event types.
- `occurredAt` — the source system's own timestamp for when the event happened (client-send time, D365 activity time, orchestrator response time, read-action time). Display-ordering only — it isn't reliably monotonic across delivery paths (a webhook/bus retry or reconciliation backfill can insert a row whose `occurredAt` is earlier than rows already stored), so it can never safely answer "has the client seen everything up to X."

So a watermark value like `deliveryWatermark = 482` means exactly one thing: "the device has acknowledged every `ConversationEvent` up to and including `sequenceNo = 482`," full stop — nothing about *when* those events occurred.

**Why there are two:**
- **`deliveryWatermark`** — a transport-level "the device received this" signal. Advanced by the delivery-ack flow (scenario 8): the client calls `POST /chat/sessions/{id}/delivered { sequenceNo }`, and only `Conversation.deliveryWatermark` changes — no `ConversationEvent` row is written, same ephemeral treatment as `TYPING` (§1.6).
- **`readWatermark`** — a product-level "the customer actually looked at this" signal. Advanced by `POST /chat/sessions/{id}/read { sequenceNo }` (scenario 8), which **does** persist a `ConversationEvent` (`eventType=READ_RECEIPT`, `relatedSequenceNo=sequenceNo`) for audit/debugging value, in addition to updating the denormalized field.
- The two are independent and routinely diverge: a backgrounded app can receive-and-ack a push (advancing `deliveryWatermark`) well before the customer opens the conversation screen and actually reads it (advancing `readWatermark`). Neither watermark is derived from the other.

**How a watermark advances:** always via a guarded conditional update, never an unconditional `SET` —
```
UPDATE Conversation SET deliveryWatermark = :new WHERE deliveryWatermark < :new
UPDATE Conversation SET readWatermark    = :new WHERE readWatermark    < :new
```
The guard exists because acknowledgments can arrive out of order relative to `sequenceNo` over an unreliable network path — a duplicate or late ack for an older `sequenceNo` must never regress the watermark backward. Regressing it would cause the catch-up query (`GET /chat/sessions/{id}/events?sinceSeq=watermark`, scenario 1/9) to needlessly replay events the client already has.

**The unrelated third "watermark":** the reconciliation sweep (scenario 10) also tracks a per-conversation "last-known watermark" it passes to D365 as `StartTimeStamp` on `GET /conversation/{id}/messages?StartTimeStamp=...` for incremental fetch. This is a **timestamp**, not a `sequenceNo` — it's D365's own cursor semantics on its history endpoint, not our internal event-ordering field, and the two must not be confused or reused interchangeably. It exists purely to avoid re-fetching a conversation's entire D365 message history on every sweep tick; it has no relationship to `deliveryWatermark`/`readWatermark` and is never exposed to the mobile client.

## API definition

Full endpoint catalog, consolidated from the sequence diagrams in §2 above — nine endpoints confirmed during design, two (rows 10–11) that diagramming itself surfaced as gaps and are marked **proposed**, not yet confirmed (§4 item 3), and one (row 12) added to replace an earlier non-REST design for delivery-ack (§4 item 2, resolved). Formal swagger/OpenAPI spec is the remaining follow-on work; this table is the surface to formalize.

| # | Method & URL | Purpose | Phase | Data model read | Data model written |
|---|---|---|---|---|---|
| 1 | `POST /chat/connection/negotiate` | Issue a Web PubSub client access token scoped to `userId`. Called conditionally per ADR-0001 §6.3 — only when a conversation is `AI`/`HUMAN`, not unconditionally on app foreground. | AI, HUMAN | `Conversation` (state check, caller-side) | — (stateless; token minted by Web PubSub, not persisted) |
| 2 | `GET /chat/current-session` | App-startup discovery — does an active/recent conversation exist for this user, and what's its watermark? | any | `Conversation` (latest non-closed row for `userId`) | — |
| 3 | `POST /chat/sessions` | Start a new conversation. | — → AI | — | `Conversation` (insert, `state=AI`) |
| 4 | `POST /chat/sessions/{id}/messages` | Send a message (text/link → `body`, or `attachmentId` reference for a file). Behavior branches on `Conversation.state`: `AI` calls chat-orchestrator-service over REST (`POST /converse`, streamed response); `HUMAN` forwards to D365. | AI, HUMAN | `Conversation.state` (branch); `Attachment` (if `contentType=FILE`) | `ConversationEvent` (insert `eventType=MESSAGE`, `origin=CUSTOMER`); in the AI branch, the orchestrator's streamed reply later lands as a second insert (`origin=AI_BOT`) once the stream completes (scenario 2), not this call; `externalMessageId` backfilled on the same row after the D365 call in the HUMAN branch |
| 5 | `POST /chat/sessions/{id}/attachments` | Upload a file (facade-proxied — mobile sends bytes here, facade writes to Blob Storage and returns an `attachmentId`). Enforces 10 MB / image-or-PDF constraint. | AI, HUMAN (facade-proxied upload applies to both phases; the orchestrator fetches AI-phase attachment bytes via a facade-issued SAS URL, §1.6) | — | `Attachment` (insert; `eventId` left null until the follow-up `POST .../messages` call links it) |
| 6 | `POST /chat/sessions/{id}/escalate` | Transition `AI` → `HUMAN`: creates the D365 conversation, passes the AI-phase transcript as context variables, persists an `ESCALATED` event. User-initiated REST call; the orchestrator-initiated path (scenario 3) reaches the same internal operation via an `event: escalate` SSE event mid-stream instead, not this endpoint. One-way/terminal — no reverse endpoint exists (§1.6). | AI → HUMAN | `ConversationEvent` (all `phase=AI` rows, to build the transcript) | `Conversation` (update `state=HUMAN`, `d365ConversationId`, `escalatedAt`); `ConversationEvent` (insert `eventType=ESCALATED`, `phase=HUMAN`) |
| 7 | `GET /chat/sessions/{id}/events?sinceSeq=` | Catch-up — "everything since my last watermark," keyed on `sequenceNo`. Used at app startup, on reconnect, and during first connect. | any | `ConversationEvent` (`WHERE sequenceNo > sinceSeq`, spans both phases) | — |
| 8 | `POST /chat/sessions/{id}/read` | Read receipt — persists a `READ_RECEIPT` event and advances `Conversation.readWatermark`. | AI, HUMAN | — | `ConversationEvent` (insert `eventType=READ_RECEIPT`, `relatedSequenceNo`); `Conversation` (update `readWatermark`, guarded `WHERE readWatermark < new`) |
| 9 | `POST /webhooks/d365/conversations/{conversationId}/activities` | Inbound webhook receiver for D365 activities (agent messages, `AgentAccepted`, `AgentClosed`/`ConversationClosed`). Ack fast (<10s per ADR-0003), process async. | HUMAN | `ConversationEvent.externalMessageId` (dedupe check, populated from D365's `activityId`) | `ConversationEvent` (insert — `eventType` varies: `MESSAGE`/`AGENT_ACCEPTED`/`CLOSED`); `Attachment` (insert, for inbound file messages); `Conversation` (update `state=CLOSED`/`closedAt`/`closedReason` on close; `deliveryWatermark` untouched here — that's scenario 8) |
| 10 | `GET /chat/attachments/{attachmentId}` **(proposed)** | Facade-brokered download of an inbound (agent-sent) attachment's bytes. Surfaced as a gap while drawing sequence diagram #7 — no download route existed for the inbound direction, only upload (#5, outbound). | HUMAN | `Attachment` (`blobStorageRef`/`contentUrl`); `ConversationEvent`/`Conversation` (ownership check — `attachmentId` → `eventId` → `conversationId` → `userId`, so a user can't fetch another user's file by guessing an id) | — |
| 11 | `POST /webhooks/webpubsub/events` **(proposed)** | Receiver for Web PubSub system events (`azure.webpubsub.sys.disconnected`, `azure.webpubsub.sys.connected`, and similar CloudEvents). Surfaced as a gap while drawing sequence diagram #9 — ADR-0001 §6.4 describes the `disconnected` event firing to "our webhook/upstream endpoint" but no such endpoint was in the original list; the `connected` event isn't documented in ADR-0001 at all yet (§4 item 7). | n/a (connection-level, not conversation-level) | `Conversation.offlineNotifiedAt` (on `connected`, to check whether a reset is needed) | On `disconnected`: none persisted (Web PubSub's own active-connection set is what changes). On `connected`: `Conversation` (`offlineNotifiedAt` reset to null, if set) — the one real write this endpoint performs, added to support the offline-notification email throttle (scenario 12) |
| 12 | `POST /chat/sessions/{id}/delivered` | Delivery ack — advances `Conversation.deliveryWatermark`. Same request shape as row 8's read receipt (`{ sequenceNo }`); replaces an earlier design that put this on a Web PubSub upstream client message (§4 item 2, resolved). | AI, HUMAN | — | `Conversation` (update `deliveryWatermark`, guarded `WHERE deliveryWatermark < new`) — no `ConversationEvent` row, same ephemeral treatment as `TYPING` (§1.6) |

**Notes:**
- Rows 10 and 11 are **proposed, not confirmed** — they close gaps found by diagramming rather than being part of the originally scoped 9-endpoint list. Flag for explicit sign-off before they go into the formal swagger spec, same as any other API surface decision.
- Row 12 is confirmed, not proposed — unlike rows 10/11, it isn't a gap awaiting sign-off, it's a deliberate replacement of a prior design choice.
- "Data model written" for row 9 is the widest of any endpoint — it's the single inbound funnel for three different `eventType` outcomes depending on the D365 activity's own `type`/`name`. Worth keeping that branching explicit in the eventual OpenAPI spec (e.g. via a discriminated request/response schema) rather than flattening it into one generic "webhook payload" shape.
- Row 12 (delivery-ack) and row 8 (read receipt) are intentionally near-identical in shape — same `{ sequenceNo }` body, same conversation-scoped path — differing only in what each persists (see scenario 8's notes).

# 4. Open Items & Follow-ups

Items surfaced during this design pass that don't yet have a final answer. Each is referenced from wherever it's load-bearing in §1/§2/§3 rather than repeated in full there.

1. **Rich link-preview metadata** — unconfirmed requirement. The current design treats a link as a plain `TEXT` message; if link-preview metadata (title/thumbnail/etc.) turns out to be required, `ConversationEvent` needs a distinct content type or extra fields.
2. ~~**Delivery-ack transport**~~ — **resolved.** Originally modeled as a Web PubSub upstream client message; switched to a REST endpoint (`POST /chat/sessions/{id}/delivered`, §3 row 12) mirroring read-receipt's shape, for consistency with every other client→server interaction and to drop the dependency on Web PubSub's unconfirmed upstream-message/event-handler mechanism.
3. **Two proposed API rows** — `GET /chat/attachments/{attachmentId}` (§3, row 10) and `POST /webhooks/webpubsub/events` (§3, row 11) — need explicit sign-off before they go into a formal swagger/OpenAPI spec.
4. **Reconciliation interval value** — a 15-minute starting point (ADR-0003 §7) is still pending confirmation against compliance's acceptable-staleness answer and Microsoft's actual (currently undocumented) rate limit for the per-conversation messages endpoint.
5. **AI-transcript context-variable payload size limit** — the size limit D365 enforces on the context variables sent at escalation (scenario 3) is undocumented. Worth confirming empirically in a sandbox tenant before assuming an arbitrarily long AI-phase transcript can be sent as one value; may need a truncation strategy (e.g. last N exchanges) if a limit is hit in practice.
6. **`notification-service` API contract** — payload shape, delivery guarantees (at-least-once? fire-and-forget?), retry/failure behavior, and rate limits for `notification-service` are all assumed rather than confirmed. Needed before scenario 12 (offline notification) can be built.
7. **Web PubSub `connected` system event** — scenario 9 and scenario 12 assume Azure fires a `connected` CloudEvent (symmetric to the `disconnected` one ADR-0001 §6.4 already documents) to the same webhook receiver, and that this is what resets the email throttle. ADR-0001 currently only discusses `disconnected` — the `connected` event's existence and payload shape should be confirmed against current Web PubSub docs before this is built.
8. **Email audit/compliance record** — the offline-notification email is currently modeled as ephemeral (not persisted as a `ConversationEvent`), consistent with how push is treated. Email may carry different compliance expectations (e.g. proof that an outbound customer communication was sent) — worth confirming with compliance whether this needs its own audit trail, separate from the throttle-only `offlineNotifiedAt` field.
9. **User Exists SDK wrapper** — the underlying REST operation (`HEAD /api/hubs/{hub}/users/{userId}`) is confirmed against current Microsoft docs, but the native Java SDK's exact method name/signature for it wasn't independently verified (only inferred from the SDK already providing first-class wrappers for `sendToUser`/token issuance, ADR-0001 §2). Confirm before scenarios 6/7/11 are built.
10. **chat-orchestrator-service call: timeout, retry, and streaming-connection infra** (ADR-0005) — no broker sits behind this call, so `smb-chat-service` needs its own failure policy (retry? how many times, with what backoff, before surfacing a failure to the user?) and the infrastructure between the two services (load balancers, API gateways, proxies) needs confirmed support for long-lived streamed HTTP connections, including idle-timeout settings and a heartbeat/keep-alive convention so intermediaries don't kill an in-progress generation as idle.
11. **Mobile "thinking" state for AI replies** — the AI answer still isn't returned synchronously in the original REST response (scenario 2); the mobile app needs a loading/thinking indicator for `AI`-phase sends that it previously only needed for `HUMAN`-phase ones, and ideally renders the live `delta` chunks progressively rather than just a spinner. A UX/design item, not purely backend.
12. **`POST /converse` request/response (SSE event) schema** (ADR-0005) — the shapes sketched in ADR-0005 (`delta`, `done`, `escalate` event types) are illustrative, not confirmed; need sign-off jointly with whoever owns `chat-orchestrator-service`, same as any other API-surface decision in this document.
13. **Per-conversation send serialization** (ADR-0005) — whether `smb-chat-service` rejects or queues a second `AI`-phase send for a conversation that already has an in-flight stream open, and what timeout governs "in-flight" (a stuck/abandoned stream shouldn't permanently block a conversation). Narrower than the old broker-ordering question, but still needs an answer before build.
14. **Terminal state for a never-escalated `AI`-phase conversation** (scenario 11) — `CLOSED` today is only reachable via D365 agent closure, which requires escalation first. Whether an `AI`-only conversation needs its own defined end state (an idle timeout, an explicit "end chat," or neither) is unresolved — carried over from the old `FAQ`-phase design, not introduced by this change, but worth revisiting now that `AI`-phase conversations are expected to be the common case rather than a brief first exchange.
