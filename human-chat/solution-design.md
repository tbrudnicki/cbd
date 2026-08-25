# 1. Introduction

## 1.1 Purpose

This Solution Design describes the architecture for **human chat** support within **Up By CBD** — SME customers get a bot-first (FAQ) support experience that can escalate to a live D365 Omnichannel agent (HUMAN), fronted by a purpose-built facade, **smb-chat-service**. It exists to bring this design to architecture review, security, and compliance (roadmap Phases 5–6) before implementation begins.

## 1.2 Scope

**In scope:**
- Two-phase conversation model: `FAQ` (in-process bot) → `HUMAN` (D365 Omnichannel agent) → `CLOSED`.
- Text, link, and file (image/PDF) messaging in both directions.
- Real-time delivery via Azure Web PubSub, with push-notification fallback when the device is disconnected.
- A single compliance-source-of-truth event log (Azure SQL Database) covering both phases.
- A fixed-interval reconciliation sweep against D365's message history, as the safety net for lost webhook deliveries (ADR-0003, Option A).

**Explicitly out of scope** (confirmed during planning, not an oversight):
- Multi-device fan-out — the org's Identity Provider enforces one session per user, so no cross-device delivery/read-state problem exists here (ADR-0001 §6.2).
- Virus/malware scanning of uploaded attachments — descoped for now (§1.6); flagged as a candidate for a future compliance/security revisit, not silently dropped.
- Pushing unrelated event types (transaction alerts, fraud confirmations) over the same real-time channel — speculative, not a confirmed near-term requirement (ADR-0001 §6.3, §8). If this becomes real it flips the connection-lifecycle design from conversation-scoped to app-session-scoped and would need re-scoping in.
- Direct-to-blob attachment upload (SAS token issued to the client) — all file traffic is proxied through the facade, consistent with the single-ingress-path principle used throughout.

## 1.3 Related documents

| Document | Role |
|---|---|
| [sd-roadmap.md](./sd-roadmap.md) | How this SD gets from informal exploration to sign-off — phases, RACI, open questions |
| [ADR-0001](./adr-0001-realtime-transport.md) | Real-time transport — Azure Web PubSub |
| [ADR-0002](./adr-0002-channel-system-technology.md) | Compliance store technology — Azure SQL Database only |
| [ADR-0003](./adr-0003-reconciliation-job.md) | Reconciliation job cadence/trigger model |
| [ADR-0004](./adr-0004-webhook-ga-status.md) | D365 webhook GA status |
| [chat-facade-architecture-summary.md](./chat-facade-architecture-summary.md) | Original informal architecture exploration this design was built from |

## 1.4 Naming & terminology

| Term | Meaning |
|---|---|
| **Up By CBD** | The mobile banking app (iOS/Android) |
| **smb-chat-service** | The Java/Spring Boot facade (BFF) between the mobile app and D365 |
| **FAQ** | Conversation state: every message routed to an in-process FAQ SDK, no D365 involvement |
| **HUMAN** | Conversation state: messages routed to a live D365 Omnichannel agent |
| **CLOSED** | Terminal conversation state, reached when the agent closes the D365 conversation |
| **ConversationEvent** | The single append-only event log row type spanning both phases — messages, escalation, closure, read receipts, etc. (§1.6) |

## 1.5 Status & ownership

**Status:** Draft. Architecture (C2 diagram, all 11 sequence diagrams) and the four key ADRs' rationale are complete. Outstanding: User Journey (blocked on a Figma link — external input), Backlog/Features, Tests, and the formal OpenAPI/swagger spec (the endpoint catalog itself is done, §3).

**Ownership — still open**, tracked as an unresolved item in [sd-roadmap.md](./sd-roadmap.md) §4.5:
- Document owner / sign-off authority: **TBD** — not yet assigned.
- Approval chain (architecture review board / security / compliance / D365 platform owner): **TBD** — the roadmap's Phase 0 stakeholder-confirmation step hasn't run yet.

This section should be revisited once Phase 0 actually completes ([sd-roadmap.md](./sd-roadmap.md) §2) — the content above reflects what's derivable from the design work done so far, not a substitute for that stakeholder-alignment step.

## 1.6 Key assumptions & confirmed decisions

Facts and decisions the architecture in §2 and §3 is built on. Stated explicitly here so this document is self-contained — the sequence diagrams and API table below reference this section rather than external planning notes.

**Messaging content:**
- A link message is just `contentType: TEXT` with a URL in `body` — no distinct type or preview metadata. This is a working assumption, not a confirmed requirement; see §1.7 item 1.
- `body` is capped at 6,000 characters — confirmed against the current D365 Messaging API docs, enforced at the facade (not left to D365 to reject) for both `FAQ` and `HUMAN` phases.
- On any message, either `text` or `attachments` must be present — a request with neither is rejected. Same D365-confirmed constraint, same facade-side enforcement.

**Attachments:**
- Allowed types: `image/jpeg`, `image/png`, `application/pdf` only. Max size: 10 MB. Both enforced at facade upload validation, not left to the client.
- No virus/malware scanning at this time — files are stored as uploaded. Flagged as a candidate for a future compliance/security revisit, not silently dropped.
- Inbound (agent-sent) attachments are **copied into our own Blob Storage**, not referenced by D365's URL — our store is the retention source of truth, so a customer's later ability to view a file never depends on D365 continuing to host it.
- The `contentUrl` SAS URL uses a **fixed expiry duration, configurable at service deployment** (e.g. an `attachment.sas-token.expiry-minutes` application property) rather than being computed per-request or negotiated dynamically with D365.

**Data model / event log:**
- `ConversationEvent` is a single append-only log spanning both `FAQ` and `HUMAN` phases — one write path, one DB-generated, globally monotonic `sequenceNo` driving watermarks, catch-up, and reconciliation diffing alike.
- `activityId` is the idempotent-dedupe key for any D365-sourced event (unique constraint). It's nullable: always null for `FAQ`-phase rows (which never reach D365), and briefly null for an in-flight outbound `HUMAN`-phase row between being persisted and D365 confirming it.
- `TYPING` events and delivery acks are deliberately **never persisted** — ephemeral, no compliance/audit value; only a denormalized fast-lookup field (e.g. `Conversation.deliveryWatermark`) changes.
- Recommended index: `(conversationId, phase, sequenceNo)` — the shape both the mobile catch-up query and the reconciliation job's scoping query need.
- Single-device enforcement is handled entirely upstream by the **Identity Provider** (one session per IDP profile) — `smb-chat-service` builds no refuse/displace logic, and no `ConnectionRegistry` entity exists in the data model as a result.

**Reconciliation:**
- The job implements **ADR-0003 Option A — fixed-interval sweep only.** This is a *directed* choice, explicitly **not** ADR-0003's own preliminary recommendation (which was the hybrid Option C: a sparse fixed interval plus an immediate trigger on locally-observed failure signals). Proceeding with Option A simplifies the design: one cron bean, no flag table, no triggered/webhook-failure path.
- Confirmed against current D365 docs: `GET /conversation/{id}/messages` only ever returns `type: "message"` activities. This makes the sweep a **`MESSAGE`-only safety net** — it cannot backfill a lost `AGENT_ACCEPTED` or `CLOSED` webhook delivery (no equivalent history endpoint exists for those event types). A lost `CLOSED` is mitigated instead by the stale-open escalation (a conversation stuck open too long gets flagged for investigation); a lost `AGENT_ACCEPTED` is accepted as a residual, informational-only gap.

**API surface:**
- The full endpoint catalog is in §3 "API definition." Two of its eleven rows are **proposed**, not yet confirmed — see §1.7 item 3.

## 1.7 Open items & follow-ups

Items surfaced during this design pass that don't yet have a final answer. Each is referenced from wherever it's load-bearing in §2/§3 rather than repeated in full there.

1. **Rich link-preview metadata** — unconfirmed requirement. The current design treats a link as a plain `TEXT` message; if link-preview metadata (title/thumbnail/etc.) turns out to be required, `ConversationEvent` needs a distinct content type or extra fields.
2. **Delivery-ack transport** — modeled as a Web PubSub upstream client message handled through the SDK's own event-handler callback mechanism, not a REST endpoint (inferred from ADR-0001's event-handler model for connection lifecycle events). This is a design inference to confirm once ADR-0001 is finalized, not a locked decision.
3. **Two proposed API rows** — `GET /chat/attachments/{attachmentId}` (§3, row 10) and `POST /webhooks/webpubsub/events` (§3, row 11) — need explicit sign-off before they go into a formal swagger/OpenAPI spec.
4. **Reconciliation interval value** — a 15-minute starting point (ADR-0003 §7) is still pending confirmation against compliance's acceptable-staleness answer and Microsoft's actual (currently undocumented) rate limit for the per-conversation messages endpoint.
5. **FAQ-transcript context-variable payload size limit** — the size limit D365 enforces on the context variables sent at escalation (scenario 3) is undocumented. Worth confirming empirically in a sandbox tenant before assuming an arbitrarily long FAQ transcript can be sent as one value; may need a truncation strategy (e.g. last N exchanges) if a limit is hit in practice.

# 2. Architecture

## C2 diagram container diagram

::: mermaid
C4Container
title Container diagram — Human Chat (Up By CBD ↔ D365 Omnichannel)

Person(customer, "SME Customer", "Uses the Up By CBD mobile app for banking and support chat")

Container_Boundary(bank, "Bank Chat Platform") {
    Container(mobile_app, "Up By CBD", "iOS / Android", "Mobile banking app. Sends messages/attachments via REST; receives real-time push via a direct WebSocket connection.")
    Container(chat_service, "smb-chat-service", "Java, Spring Boot (BFF)", "Facade between the mobile app and D365. Embeds the FAQ SDK (in-process function call, no external dependency) and the reconciliation sweep job (Spring @Scheduled, ADR-0003 Option A).")
    ContainerDb(chat_db, "smb-chat-service-db", "Azure SQL Database", "Compliance source of truth. Conversation, ConversationEvent, Attachment metadata (ADR-0002).")
    ContainerDb(blob_storage, "Attachment Storage", "Azure Blob Storage", "File bytes for message attachments — both outbound (customer-uploaded) and inbound (copied from D365) files.")
    Container(web_pubsub, "Real-time Transport", "Azure Web PubSub", "Delivers inbound events to the connected device (ADR-0001). Mobile connects to this directly, not proxied through smb-chat-service.")
}

System_Ext(d365, "D365 Omnichannel", "Messaging API + webhook subscription. Human agent chat (ADR-0004: GA confirmed).")
System_Ext(entra_id, "Microsoft Entra ID", "OAuth2 client-credentials grant for D365 API auth")
System_Ext(push_gateway, "Push Notification Gateway", "FCM / APNs — backgrounded/killed-app delivery, redacted body only")

Rel(customer, mobile_app, "Uses")
Rel(mobile_app, chat_service, "Sends messages, uploads attachments, escalates, reads, catches up", "HTTPS/REST")
Rel(mobile_app, web_pubsub, "Receives real-time push (direct connection)", "WSS")
Rel(chat_service, web_pubsub, "Issues connection tokens; pushes via sendToUser", "HTTPS (management API)")
Rel(chat_service, chat_db, "Reads/writes Conversation, ConversationEvent, Attachment metadata", "JDBC")
Rel(chat_service, blob_storage, "Uploads/downloads attachment file bytes", "HTTPS")
Rel(chat_service, d365, "Creates conversations, sends customer messages/attachments", "HTTPS/REST")
Rel(d365, chat_service, "Delivers agent messages, AgentAccepted, AgentClosed/ConversationClosed", "HTTPS webhook")
Rel(chat_service, entra_id, "Obtains OAuth2 client-credentials token", "HTTPS")
Rel(chat_service, push_gateway, "Sends redacted push notification (device disconnected)", "HTTPS")
Rel(push_gateway, mobile_app, "Delivers push notification", "APNs / FCM")
:::

**Notes on this diagram:**
- `smb-chat-service` deliberately shows the FAQ SDK and reconciliation job as *capabilities of the container*, not separate boxes — they're in-process components (FAQ SDK: confirmed no external call; reconciliation: a Spring `@Scheduled` bean in the same deployable), which is a Component-diagram-level (C3) distinction, not a Container-level (C2) one.
- `smb-chat-service-db`, `Attachment Storage`, and `Real-time Transport` are modeled as containers *inside* the bank's own boundary, not as external systems — they're PaaS-hosted but are infrastructure this project provisions and owns the configuration of, unlike D365/Entra ID/the push gateway, which are genuinely third-party systems being integrated with.
- The mobile app's WebSocket connection to Web PubSub is direct, not proxied through `smb-chat-service` (ADR-0001 §6.1) — `smb-chat-service` only issues the connection token and calls `sendToUser`; the actual real-time data plane bypasses it.

## Sequence diagram

11 scenarios covering the full conversation lifecycle, from app startup through escalation, messaging, delivery/read tracking, reconnect, reconciliation, and close. All are drawn below.

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
alt conversation exists (state = FAQ or HUMAN)
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

### 2. Start conversation + first FAQ prompt

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant faqsdk as FAQ SDK (in-process)
Participant db as smb-chat-service-db

mobile ->> chatsvc: POST /chat/sessions
chatsvc ->> db: INSERT Conversation (state = FAQ)
db -->> chatsvc: conversationId
chatsvc -->> mobile: { conversationId, state: FAQ }

Note left of mobile: Conversation now exists — negotiate + open Web PubSub connection (same flow as scenario 1), not redrawn here

mobile ->> chatsvc: POST /chat/sessions/{id}/messages { contentType: TEXT, body: prompt }
chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=CUSTOMER, phase=FAQ, body=prompt)
db -->> chatsvc: sequenceNo assigned
Note left of chatsvc: Prompt persisted before FAQ processing — same "write first" principle as the D365 legs, so a crash mid-answer never loses the customer's original message
chatsvc ->> faqsdk: answer(promptText)
Note right of faqsdk: In-process function call, same JVM — confirmed no external dependency (no LLM API, no search index)
faqsdk -->> chatsvc: responseText
chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=FAQ_BOT, phase=FAQ, body=responseText)
db -->> chatsvc: sequenceNo assigned
chatsvc -->> mobile: 200 OK { response: responseText, sequenceNo }

Note left of mobile: After this first exchange, the user may escalate to HUMAN (scenario 3)
:::

**Notes:**
- The FAQ bot's response is returned **synchronously in the REST response**, not pushed via Web PubSub. That's a direct consequence of the FAQ SDK being an in-process, synchronous call — there's no async gap to bridge the way there is for D365 agent messages arriving out-of-band via webhook. Using the push channel for this would be an unnecessary round-trip.
- Both the customer's prompt and the bot's answer are persisted as separate `ConversationEvent` rows (`origin=CUSTOMER` and `origin=FAQ_BOT` respectively) before the HTTP response returns — consistent with the confirmed decision that FAQ-phase messages share the same compliance-store write path as HUMAN-phase ones.

### 3. Escalate FAQ → HUMAN

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant d365 as D365 Omnichannel

mobile ->> chatsvc: POST /chat/sessions/{id}/escalate
chatsvc ->> db: SELECT ConversationEvent WHERE conversationId=id AND phase=FAQ ORDER BY sequenceNo
db -->> chatsvc: FAQ-phase events[]
chatsvc ->> chatsvc: Format FAQ transcript into D365 context variables
chatsvc ->> d365: POST /conversation/create { contextVariables }
Note right of d365: Context variables are eventually consistent on D365's side (confirmed via docs) — the agent isn't guaranteed to see them the instant the conversation starts. Nothing to fix on our side — the D365 conversation is still created synchronously.
d365 -->> chatsvc: d365ConversationId
chatsvc ->> db: UPDATE Conversation SET state=HUMAN, d365ConversationId, escalatedAt=now
chatsvc ->> db: INSERT ConversationEvent (eventType=ESCALATED, phase=HUMAN, metadata={d365ConversationId})
db -->> chatsvc: sequenceNo assigned
chatsvc -->> mobile: 200 OK { conversationId, state: HUMAN }

Note left of mobile: Web PubSub connection was already opened when the conversation was first created (scenario 1/2) — no reconnect needed here. Subsequent messages now route to D365 instead of the FAQ SDK (scenario 4).
:::

**Notes:**
- The FAQ transcript read (`phase=FAQ`) is scoped by the same `(conversationId, phase, sequenceNo)` index called out in §1.6 — no full-table scan.
- The `ESCALATED` event is stamped `phase=HUMAN`, not `FAQ` — it marks the transition *into* the HUMAN phase, consistent with `phase` being "which phase this event belongs to" rather than "which phase produced it."
- `d365ConversationId` and `escalatedAt` on `Conversation` are denormalized mirrors — the `ESCALATED` `ConversationEvent` row is the audit source of truth; the `Conversation` row is the fast-lookup copy.
- D365's context-variables eventual-consistency behavior is shown as a note, not a retry/poll loop — there's nothing actionable on the facade side; it's a constraint on agent-side UX expectations, out of scope for this diagram.

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
    chatsvc ->> db: UPDATE ConversationEvent SET activityId WHERE id=...
    chatsvc -->> mobile: 200 OK { sequenceNo }
else D365 call fails/times out
    d365 -->> chatsvc: error / no response
    chatsvc -->> mobile: 5xx (row already persisted with activityId=null)
    Note right of chatsvc: No retry loop here — the row's null activityId makes it visible to the reconciliation sweep (scenario 10, ADR-0003), which is the actual safety net for this gap
end
:::

**Notes:**
- A link message is just `contentType: TEXT` with a URL in `body` — no distinct type or preview metadata (working assumption, §1.6; unresolved rich-preview requirement, §1.7 item 1).
- The row is written with `activityId = null` at persist time and only backfilled once D365 confirms — this is why `activityId` has to be nullable for `HUMAN`-phase rows too, not just `FAQ`-phase ones (contrast with §1.6, which frames nullability as an FAQ-phase consequence; outbound-in-flight is a second, narrower reason it's nullable for HUMAN rows specifically, closed by either this update or the reconciliation sweep).
- Behavior branch on `Conversation.state` (FAQ → in-process SDK vs. HUMAN → D365 forward) lives entirely in `POST /chat/sessions/{id}/messages` per the endpoint table in §3 (API definition, row 4) — this diagram only shows the HUMAN branch; scenario 2 showed the FAQ branch.
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
    chatsvc ->> db: UPDATE ConversationEvent SET activityId WHERE id=...
    chatsvc -->> mobile: 200 OK { sequenceNo }
else D365 call fails/times out
    d365 -->> chatsvc: error / no response
    chatsvc -->> mobile: 5xx (row already persisted with activityId=null)
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
Participant push as Push Notification Gateway

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: message, activityId, text, from: agent }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Ack fast, then process off the webhook thread — avoids D365's retry-on-timeout behavior — the ack is not conditioned on the DB write below succeeding

chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, contentType=TEXT, body=text, activityId)
alt activityId not seen before
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> wps: sendToUser(userId, event)
    alt device connected
        wps -->> mobile: deliver event over open WebSocket
    else device disconnected
        chatsvc ->> push: send push notification (redacted body only)
        push -->> mobile: deliver push (APNs/FCM)
    end
else activityId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on activityId — no-op
    Note over chatsvc: Idempotent dedupe (§1.6) — a retried webhook delivery is silently absorbed, never double-persisted or double-pushed
end
:::

**Notes:**
- Ack-then-process is the same pattern ADR-0003 establishes for webhook handling generally — this diagram is the first to actually show it, since scenarios 1–5 were all facade-initiated rather than webhook-initiated.
- `activityId` is the dedupe key (unique constraint, §1.6) — D365's own at-least-once webhook delivery (3-attempt retry) means duplicate deliveries are expected, not exceptional; this path is what makes them safe.
- Push notification is only sent when the device is disconnected, per the C2 diagram's `push_gateway` relationship (§2 of this document) — not sent unconditionally alongside every Web PubSub push.
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
Participant push as Push Notification Gateway

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: message, activityId, attachments: [{ contentType, contentUrl, name }], from: agent }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Same ack-then-process pattern as scenario 6

alt activityId not seen before
    chatsvc ->> d365: GET contentUrl (fetch file bytes from D365-hosted URL)
    d365 -->> chatsvc: file bytes
    chatsvc ->> blob: Upload file bytes — copy into our own Blob Storage
    blob -->> chatsvc: blobStorageRef
    chatsvc ->> chatsvc: Generate our own contentUrl (facade-brokered)
    chatsvc ->> db: INSERT Attachment (fileName, mimeType, sizeBytes, blobStorageRef, contentUrl)
    db -->> chatsvc: attachmentId
    chatsvc ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, contentType=FILE, attachmentId, activityId)
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> db: UPDATE Attachment SET eventId WHERE id=attachmentId
    chatsvc ->> wps: sendToUser(userId, event)
    alt device connected
        wps -->> mobile: deliver event over open WebSocket
    else device disconnected
        chatsvc ->> push: send push notification (redacted body only)
        push -->> mobile: deliver push (APNs/FCM)
    end
    Note left of mobile: File bytes downloaded later via facade-brokered URL, not fetched directly from Blob Storage — see note below on the missing endpoint
else activityId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on activityId — no-op
end
:::

**Notes:**
- The copy-not-reference decision (§1.6) means our Blob Storage becomes the retention source of truth for inbound files too — the customer's later ability to view an agent-sent file never depends on D365 continuing to host it.
- The file download (D365 → facade) happens as part of the same async, post-ack processing as the metadata persist — a large/slow file download doesn't block the webhook ack.
- Idempotent dedupe on `activityId` (same mechanism as scenario 6) covers this path too — a retried webhook won't re-download or re-copy the file.
- **Gap this diagram surfaced:** there was no `GET` route for the mobile app to download an inbound attachment's bytes via the facade — only `POST /chat/sessions/{id}/attachments` (upload, outbound direction) existed. Addressed as `GET /chat/attachments/{attachmentId}` in the API catalog (§3, row 10) — still marked **proposed**, needs sign-off (§1.7 item 3).

### 8. Delivery ack + read receipt

::: mermaid
sequenceDiagram
autonumber
Participant mobile as Up By CBD
Participant wps as Azure Web PubSub
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db

Note over mobile,wps: Continues from scenario 6/7 — mobile just received a pushed event over its open WebSocket
mobile ->> wps: Send ack (upstream client message: { type: ack, sequenceNo })
wps ->> chatsvc: Event-handler callback: upstream message event { userId, sequenceNo }
chatsvc ->> db: UPDATE Conversation SET deliveryWatermark = sequenceNo WHERE deliveryWatermark < sequenceNo
Note over chatsvc: Monotonic guard — a late/out-of-order ack for an older sequenceNo never regresses the watermark. No ConversationEvent row is written here: delivery ack is ephemeral confirmation, same treatment as TYPING (§1.6) — no compliance/audit value, just the fast-lookup field advancing.

Note over mobile: Later, independently — user actually views the conversation screen (read watermark is a separate signal from delivery, ADR-0001 §6.3)
mobile ->> chatsvc: POST /chat/sessions/{id}/read { sequenceNo }
chatsvc ->> db: INSERT ConversationEvent (eventType=READ_RECEIPT, origin=CUSTOMER, phase=Conversation.state, relatedSequenceNo=sequenceNo)
db -->> chatsvc: sequenceNo assigned (the READ_RECEIPT row's own sequenceNo — distinct from relatedSequenceNo, the event being acknowledged)
chatsvc ->> db: UPDATE Conversation SET readWatermark = relatedSequenceNo WHERE readWatermark < relatedSequenceNo
chatsvc -->> mobile: 200 OK
:::

**Notes:**
- This is the two-watermark model from the original architecture summary ([sd-roadmap.md](./sd-roadmap.md) §1), now keyed by the single global `sequenceNo` spanning both FAQ and HUMAN phases rather than a per-phase counter.
- **Delivery ack and read receipt are handled asymmetrically on purpose**, mirroring the `TYPING`-vs-persisted-event distinction already established in §1.6: delivery ack only ever touches the denormalized `Conversation.deliveryWatermark` field (no audit row), while read receipt is fully persisted as a `ConversationEvent` *and* mirrored to `Conversation.readWatermark` — because read history was confirmed to have standalone audit/debugging value, delivery-ack history was not.
- The delivery-ack transport (client → Web PubSub upstream message → facade event-handler callback) is inferred from ADR-0001 §2/§7's mention of "delivery ack (advances watermark)" and its note that Web PubSub already uses an event-handler/webhook model for connection lifecycle events — no dedicated REST endpoint for delivery-ack exists in the API catalog (§3), and none is needed if acks travel over the same socket. This is a design inference to confirm once ADR-0001 is finalized, not a locked decision (§1.7 item 2).
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
    mobile ->> chatsvc: GET /chat/sessions/{id}/events?sinceSeq=watermark
    chatsvc ->> db: SELECT ConversationEvent WHERE conversationId=id AND sequenceNo > watermark ORDER BY sequenceNo
    db -->> chatsvc: events[] (may span both FAQ- and HUMAN-phase rows)
    chatsvc -->> mobile: events[] (render transcript, replay anything missed while disconnected)
end
:::

**Notes:**
- The "app backgrounded" branch reuses scenario 1's negotiate → connect → catch-up shape exactly — the difference here is *why* it's happening (a genuine reconnect after a real disconnect, not first app launch) and that ADR-0001 §6.4's `disconnected` system event is shown explicitly as the trigger for removing the dead connection.
- Catch-up is phase-agnostic: a missed FAQ-phase message replays through the same `sinceSeq` query as a missed HUMAN-phase D365 message, since `ConversationEvent.sequenceNo` is one global sequence spanning both phases (§1.6). Nothing D365-specific about this path.
- The short-blip branch requires **no facade involvement at all** — it's purely the Web PubSub client SDK and Azure's own reliable-subprotocol recovery. It's included here (rather than omitted) specifically to make clear why the "app backgrounded" branch's more expensive fresh-connect-plus-catch-up path only fires for a real disconnect, not every network hiccup.
- **Gap this diagram surfaced, same category as #7:** ADR-0001 §6.4 says the `disconnected` CloudEvent fires to "our webhook/upstream endpoint," but no such endpoint existed — the original API list only had the D365 webhook receiver. Addressed as `POST /webhooks/webpubsub/events` in the API catalog (§3, row 11) — still marked **proposed**, needs sign-off (§1.7 item 3).
- The exact recovery-window duration for the short-blip branch is still an open ADR-0001 item (§8, "pull the current documented connection-recovery window") — this diagram assumes the branch exists but doesn't depend on the exact duration.

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
    cron ->> cron: Diff returned activityIds against stored ConversationEvent rows WHERE phase=HUMAN AND eventType=MESSAGE
    alt gap found (activityId not yet persisted)
        cron ->> db: INSERT ConversationEvent (eventType=MESSAGE, origin=AGENT, phase=HUMAN, direction=INBOUND, activityId, body/attachment) — same idempotent write path as the webhook (scenarios 6/7)
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
- Diffing key is `activityId` — the same idempotent-write/dedupe mechanism used by the webhook receiver (scenarios 6/7), so a message the webhook already delivered is never double-persisted or double-pushed just because it also shows up in the reconciliation fetch.
- This diagram reflects **Option A (fixed-interval only) as confirmed/directed** (§1.6) — not ADR-0003's own preliminary recommendation (Option C, hybrid with a triggered path). No flag table, no second cron bean, no webhook-receiver-writes-a-flag step — deliberately simpler than ADR-0003 §8's Option C diagram.
- Interval value itself (ADR-0003 §7 suggests 15 minutes as a starting point) is still pending confirmation against compliance's acceptable-staleness answer and Microsoft's actual rate limit ([sd-roadmap.md](./sd-roadmap.md) §4.1, §4.2) — not fixed by this diagram.

### 11. Conversation close by agent

::: mermaid
sequenceDiagram
autonumber
Participant d365 as D365 Omnichannel
Participant chatsvc as smb-chat-service
Participant db as smb-chat-service-db
Participant wps as Azure Web PubSub
Participant mobile as Up By CBD
Participant push as Push Notification Gateway

d365 ->> chatsvc: POST /webhooks/d365/conversations/{d365ConversationId}/activities { type: event, activityId, name: ConversationClosed | AgentClosed }
chatsvc -->> d365: 200 OK (ack immediately, < 10s per ADR-0003)
Note over chatsvc: Same ack-then-process pattern as scenarios 6/7/9

alt activityId not seen before
    chatsvc ->> db: UPDATE Conversation SET state=CLOSED, closedAt=now, closedReason WHERE id=...
    chatsvc ->> db: INSERT ConversationEvent (eventType=CLOSED, origin=AGENT, phase=HUMAN, activityId, metadata={reason})
    db -->> chatsvc: sequenceNo assigned
    chatsvc ->> wps: sendToUser(userId, event)
    alt device connected
        wps -->> mobile: Deliver CLOSED event over open WebSocket
        Note left of mobile: After a short grace period (~30–60s, ADR-0001 §6.3) to let a trailing ack/read-receipt round-trip land
        mobile ->> wps: Close WebSocket — conversation-state-scoped lifecycle ends here, not app-session-scoped
    else device disconnected
        chatsvc ->> push: Send push notification (redacted body only)
        push -->> mobile: Deliver push (APNs/FCM)
        Note left of mobile: Next app foreground follows scenario 9's reconnect flow — GET /chat/current-session finds state=CLOSED and does not reopen a socket (ADR-0001 §6.3)
    end
else activityId already persisted (webhook retry / duplicate delivery)
    db -->> chatsvc: unique constraint on activityId — no-op
end
:::

**Notes:**
- Reuses the ack-then-process and idempotent-dedupe pattern from scenarios 6, 7, and 9 — `CLOSED` is just another D365 activity type flowing through the same webhook receiver, not a special-cased endpoint.
- `state=CLOSED` is terminal (§1.6) — this is the one place the conversation-state-scoped connection lifecycle (ADR-0001 §6.3) actually tears the socket down, as opposed to scenario 1/2/3 where it's opened or kept open.
- **Who initiates the socket close is modeled as client-side**, after the grace period, matching ADR-0001 §6.3's rationale of bounding connection cost/exposure to "while there's something to protect" — once `CLOSED`, there isn't. This is a design choice to confirm in Phase 3, not a documented Azure requirement: Web PubSub also exposes a server-side `closeClientConnection` management-API primitive if a forced/immediate close is later preferred over waiting on the client.
- A lost `CLOSED` webhook delivery is **not** backfilled by this diagram — per scenario 10 / ADR-0003 §4.2, the mitigation is the reconciliation sweep's stale-open escalation (detecting a conversation stuck open too long), not a retry of this specific event.
- This closes out the 11-scenario set — all sequence diagrams are now drawn.

# 3. Design & Implementation

## User Journey

_(link to figma)_

**Status: pending** — needs a Figma link (external input, not yet provided).

## Backlog

_(Provide the feature backlog)_

**Status: pending.**

### Features

_(pending)_

### Decisions

- [ADR-0001: Real-time transport](./adr-0001-realtime-transport.md) — Azure Web PubSub
- [ADR-0002: Channel System technology](./adr-0002-channel-system-technology.md) — Azure SQL Database only
- [ADR-0003: Reconciliation job](./adr-0003-reconciliation-job.md) — fixed-interval cron (Option A)
- [ADR-0004: Webhook GA status](./adr-0004-webhook-ga-status.md) — resolved, GA confirmed

### Tests

_(provide Tests link)_

**Status: pending.**

## Watermark model (delivery vs. read)

The design uses two independent watermarks on `Conversation` — `deliveryWatermark` and `readWatermark` — plus a third, unrelated cursor used only inside the reconciliation job. All three get called "watermark" informally, so this section is explicit about what each one actually *is*.

**What a watermark is, concretely:** for `deliveryWatermark` and `readWatermark`, a watermark is **`ConversationEvent.sequenceNo`** — a plain integer, database-generated and globally monotonic, assigned atomically at insert time. It is **not** a UUID and **not** a timestamp. Two other per-event fields could be mistaken for the watermark and deliberately aren't used as one:
- `activityId` — D365's own Bot Framework activity identifier (string, UUID-shaped). Used only as the idempotent-dedupe key (unique constraint) for D365-sourced rows — it has no ordering meaning and isn't comparable across event types (`FAQ`-phase rows never have one).
- `occurredAt` — the source system's own timestamp for when the event happened (client-send time, D365 activity time, FAQ SDK response time, read-action time). Display-ordering only — it isn't reliably monotonic across delivery paths (a webhook retry or reconciliation backfill can insert a row whose `occurredAt` is earlier than rows already stored), so it can never safely answer "has the client seen everything up to X."

So a watermark value like `deliveryWatermark = 482` means exactly one thing: "the device has acknowledged every `ConversationEvent` up to and including `sequenceNo = 482`," full stop — nothing about *when* those events occurred.

**Why there are two:**
- **`deliveryWatermark`** — a transport-level "the device received this" signal. Advanced by the delivery-ack flow (scenario 8): the client sends `{ type: ack, sequenceNo }` upstream over its open Web PubSub connection, the facade's event-handler callback receives it, and only `Conversation.deliveryWatermark` changes — no `ConversationEvent` row is written, same ephemeral treatment as `TYPING` (§1.6).
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

Full endpoint catalog, consolidated from the sequence diagrams in §2 above — nine endpoints confirmed during design, plus two (rows 10–11) that diagramming itself surfaced as gaps and are marked **proposed**, not yet confirmed (§1.7 item 3). Formal swagger/OpenAPI spec is the remaining follow-on work; this table is the surface to formalize.

| # | Method & URL | Purpose | Phase | Data model read | Data model written |
|---|---|---|---|---|---|
| 1 | `POST /chat/connection/negotiate` | Issue a Web PubSub client access token scoped to `userId`. Called conditionally per ADR-0001 §6.3 — only when a conversation is `FAQ`/`HUMAN`, not unconditionally on app foreground. | FAQ, HUMAN | `Conversation` (state check, caller-side) | — (stateless; token minted by Web PubSub, not persisted) |
| 2 | `GET /chat/current-session` | App-startup discovery — does an active/recent conversation exist for this user, and what's its watermark? | any | `Conversation` (latest non-closed row for `userId`) | — |
| 3 | `POST /chat/sessions` | Start a new conversation. | — → FAQ | — | `Conversation` (insert, `state=FAQ`) |
| 4 | `POST /chat/sessions/{id}/messages` | Send a message (text/link → `body`, or `attachmentId` reference for a file). Behavior branches on `Conversation.state`: `FAQ` routes to the in-process FAQ SDK; `HUMAN` forwards to D365. | FAQ, HUMAN | `Conversation.state` (branch); `Attachment` (if `contentType=FILE`) | `ConversationEvent` (insert `eventType=MESSAGE`, `origin=CUSTOMER`; a second insert `origin=FAQ_BOT` in the FAQ branch); `activityId` backfilled on the same row after the D365 call in the HUMAN branch |
| 5 | `POST /chat/sessions/{id}/attachments` | Upload a file (facade-proxied — mobile sends bytes here, facade writes to Blob Storage and returns an `attachmentId`). Enforces 10 MB / image-or-PDF constraint. | HUMAN (files are HUMAN-phase only in the confirmed design) | — | `Attachment` (insert; `eventId` left null until the follow-up `POST .../messages` call links it) |
| 6 | `POST /chat/sessions/{id}/escalate` | Transition `FAQ` → `HUMAN`: creates the D365 conversation, passes the FAQ transcript as context variables, persists an `ESCALATED` event. | FAQ → HUMAN | `ConversationEvent` (all `phase=FAQ` rows, to build the transcript) | `Conversation` (update `state=HUMAN`, `d365ConversationId`, `escalatedAt`); `ConversationEvent` (insert `eventType=ESCALATED`, `phase=HUMAN`) |
| 7 | `GET /chat/sessions/{id}/events?sinceSeq=` | Catch-up — "everything since my last watermark," keyed on `sequenceNo`. Used at app startup, on reconnect, and during first connect. | any | `ConversationEvent` (`WHERE sequenceNo > sinceSeq`, spans both phases) | — |
| 8 | `POST /chat/sessions/{id}/read` | Read receipt — persists a `READ_RECEIPT` event and advances `Conversation.readWatermark`. | FAQ, HUMAN | — | `ConversationEvent` (insert `eventType=READ_RECEIPT`, `relatedSequenceNo`); `Conversation` (update `readWatermark`, guarded `WHERE readWatermark < new`) |
| 9 | `POST /webhooks/d365/conversations/{conversationId}/activities` | Inbound webhook receiver for D365 activities (agent messages, `AgentAccepted`, `AgentClosed`/`ConversationClosed`). Ack fast (<10s per ADR-0003), process async. | HUMAN | `ConversationEvent.activityId` (dedupe check) | `ConversationEvent` (insert — `eventType` varies: `MESSAGE`/`AGENT_ACCEPTED`/`CLOSED`); `Attachment` (insert, for inbound file messages); `Conversation` (update `state=CLOSED`/`closedAt`/`closedReason` on close; `deliveryWatermark` untouched here — that's scenario 8) |
| 10 | `GET /chat/attachments/{attachmentId}` **(proposed)** | Facade-brokered download of an inbound (agent-sent) attachment's bytes. Surfaced as a gap while drawing sequence diagram #7 — no download route existed for the inbound direction, only upload (#5, outbound). | HUMAN | `Attachment` (`blobStorageRef`/`contentUrl`); `ConversationEvent`/`Conversation` (ownership check — `attachmentId` → `eventId` → `conversationId` → `userId`, so a user can't fetch another user's file by guessing an id) | — |
| 11 | `POST /webhooks/webpubsub/events` **(proposed)** | Receiver for Web PubSub system events (`azure.webpubsub.sys.disconnected` and similar CloudEvents). Surfaced as a gap while drawing sequence diagram #9 — ADR-0001 §6.4 describes the event firing to "our webhook/upstream endpoint" but no such endpoint was in the original list. | n/a (connection-level, not conversation-level) | — | — (informational/logging only — Web PubSub maintains its own active-connection set server-side; no `Conversation`/`ConversationEvent` row changes as a result of this call, since `ConnectionRegistry` was removed from the data model, §1.6) |

**Notes:**
- Rows 10 and 11 are **proposed, not confirmed** — they close gaps found by diagramming rather than being part of the originally scoped 9-endpoint list. Flag for explicit sign-off before they go into the formal swagger spec, same as any other API surface decision.
- "Data model written" for row 9 is the widest of any endpoint — it's the single inbound funnel for three different `eventType` outcomes depending on the D365 activity's own `type`/`name`. Worth keeping that branching explicit in the eventual OpenAPI spec (e.g. via a discriminated request/response schema) rather than flattening it into one generic "webhook payload" shape.
- No endpoint exists for delivery-ack (distinct from row 8's read receipt) — per the note on sequence diagram #8, that's modeled as a Web PubSub upstream client message handled through the SDK's own event-handler mechanism, not a REST call, so it doesn't get a table row here. Still an inference to confirm, not a locked decision (§1.7 item 2).
