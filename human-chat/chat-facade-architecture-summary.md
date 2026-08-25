# Human chat facade — Dynamics 365 Omnichannel integration

## Context

- Banking mobile app for SMEs. Building a Java backend facade between the mobile app and Microsoft Dynamics 365 (Contact Center / Omnichannel for Customer Service) human agent chat.
- Hard business requirement: **all human chat interactions must flow through, and be stored in, our own channel system** (compliance/audit store). The mobile app must never talk to D365 directly.

## API choice: Messaging API (not Chat SDK, not Direct Line)

Dynamics 365 offers three integration paths. We're using the **Messaging API**.

- **Omnichannel Chat SDK (React Native)** — client-side SDK meant to run inside the mobile app. Rejected: bypasses our backend, conflicts with the compliance requirement.
- **Direct Line API 3.0** — lower-level "build your own connector" option. Viable but more plumbing, no native webhook push.
- **Messaging API** ✅ — explicitly designed for service-to-service/back-end integration, not direct client invocation. Sends messages on our behalf, and delivers agent responses to us via **webhook subscription** rather than requiring us to poll.

Key facts:
- Base URL format: `https://m-{org_id}.{geo}.omnichannelengagementhub.com`
- Auth: Microsoft Entra ID (Azure AD), OAuth2 client-credentials grant. Scope: `https://{org_id}-c.{zone}.dynamics.com/.default`. Requests carry `Authorization: Bearer {token}`, `channel-id`, `organization-id` headers.
- Outbound: REST call to post a customer message/activity into the D365 conversation.
- Inbound: webhook endpoint we host at `{webhook_url}/v3/conversations/{conversationId}/activities`, receiving agent messages, typing indicators, and session-closed events as Bot Framework Activity Schema payloads.
- Reconciliation/backfill: `/conversation/{conversationId}/messages` endpoint retrieves the full message history for an active conversation — Microsoft explicitly says not to use conversation-list endpoints as a *polling mechanism*, so this is a safety net, not the primary path.

**Important operational caveat:** the webhook capability is currently a **preview feature** per Microsoft's docs — confirm GA status/support commitment with our Microsoft contact before finalizing.

## Core architecture

```
Mobile app  <---REST/WebSocket--->  Chat facade (Java/Spring Boot)  <---REST + webhook--->  D365 Messaging API
                                            |
                                            v
                                   Channel system (compliance store, source of truth)
```

- **Outbound (customer → agent):** Mobile → Facade → **persist to Channel System first** → POST to D365 Messaging API → ack to mobile.
- **Inbound (agent → customer):** D365 webhook POST → Facade validates + dedupes by activity ID → persist to Channel System → deliver to mobile (WebSocket if connected, else push notification).
- Channel System is the source of truth — write to it before forwarding downstream on either leg, so a downstream failure never loses the record.

## Webhook reliability — retry limits and the reconciliation pattern

- D365's webhook retries **up to 3 times, 10-second timeout each, then gives up permanently** — no dead-letter queue, no extended retry window on Microsoft's side.
- Webhook receiver must **ack fast (well under 10s) and process asynchronously** — validate/dedupe, queue, return 200 immediately; do the heavy lifting off the request thread.
- Because the retry budget is thin, the webhook cannot be the only inbound path. Pair it with a **sparse reconciliation job**:
  - Track a last-known-good watermark/last-activity-ID per open conversation.
  - Periodically (not continuously) call `/conversation/{id}/messages` for open conversations, and explicitly after any detected outage of our own webhook endpoint.
  - Diff against what's already stored, backfill gaps using the same idempotent write path (dedup by activity ID) as the webhook.
- This keeps us compliant with Microsoft's "don't poll continuously" guidance while closing the real gap: an outage on our side during the 3-retry window.

## Real-time delivery to the mobile app

- **Not polling.** Use a persistent connection while the app is foregrounded — recommended: **Azure SignalR Service or Azure Web PubSub** rather than a self-managed WebSocket gateway + Redis pub/sub, since it's Azure-native and handles multi-instance fan-out ("send to user") for us.
- **Backgrounded/killed app:** native push (FCM/APNs). Use the **same visible push mechanism already used for transfers**, triggered when a message is delivered via webhook/reconciliation — but keep the notification body **generic/redacted** ("New message from your support agent"), not the actual chat content, to avoid sensitive data transiting Apple/Google push infrastructure.
- Silent/background push (content-available/data message) is a secondary tool for quietly refreshing local state (e.g. unread badge) — not reliable enough (can be throttled/dropped by the OS) to be the primary "tell the user" mechanism.
- **Reconnect + catch-up is mandatory regardless of push**, since sockets don't survive backgrounding for long:
  - On app resume/foreground and on network change, reconnect (lean on SignalR/Web PubSub client SDK's built-in reconnect-with-backoff).
  - After reconnecting, call the facade with "everything since my last watermark" to fill any gap.
- Push and reconnect solve different halves of the problem: push gets the user's attention when the app isn't running; reconnect+catch-up guarantees correctness once the app *is* open, regardless of how it got there.

## Connection lifecycle vs. conversation lifecycle

Two separate lifecycles — don't conflate them:

1. **Connection lifecycle — scoped to user/device, not conversation.** App authenticates (existing banking session/JWT) and opens one connection on app foreground, independent of whether a chat is active. Can sit open with no conversation happening.
2. **Conversation lifecycle — its own state machine** (created → open → wrap-up → closed), mirroring D365's own conversation states. Created only when the customer actually starts a chat.

Why scope the connection to the user rather than the conversation:
- No conversation ID exists at app launch; forcing the connection to require one would mean creating placeholder conversations just to connect.
- One connection can serve multiple conversations over its life.
- Leaves room to push other things (transaction alerts, fraud confirmations) over the same channel later.

**App startup flow:**
1. App authenticates, opens its user/device-scoped connection (unconditional).
2. App calls `GET /chat/current-session` on the facade to discover whether an active/recent conversation exists — returns conversation ID + watermark if so. This is a REST call, not something asked over the socket.
3. If none exists and the user starts a chat, `POST /chat/sessions` creates one and returns the new conversation ID.

**Server-side routing:** facade stores a `conversationId → userId` mapping the moment a conversation is created. When a webhook activity arrives for a conversation, the facade looks up the owning user and pushes to whatever connection that user currently has open — routing depends on the DB mapping, not on when/how the connection was established. SignalR/Web PubSub's "send to user" primitive handles multi-device fan-out automatically.

## Watermarks — two distinct kinds

- **Delivery/sync watermark ("what has this device actually received")** — requires an explicit client ack, because a successful socket write isn't proof the app received and persisted the message (app could crash mid-processing, connection could drop mid-frame). Flow: server pushes → app persists → app sends `{"ackedUpTo": "<messageId>"}` → server advances the stored watermark only then. If no ack arrives, the watermark stays put, so the next catch-up call re-delivers rather than silently skips (same at-least-once + idempotent-consumer pattern as the webhook).
- **Read watermark ("what has the user actually seen")** — only the client can supply this; the server can't infer it from delivery. Separate, deliberate call: `POST /chat/sessions/{id}/read { lastReadMessageId }`, fired when the message actually scrolls into view or the chat screen opens. Don't conflate with delivery — a message can sit delivered and unread for a while.

## Open questions

- [ ] Confirm D365 licensing tier and that Messaging API (and its webhook subscription) is available/supported for it; confirm GA timeline for the webhook feature (currently preview).
- [ ] Confirm workstream/unified-routing configuration on the D365 admin side (queues, skill groups for SME banking support) — a prerequisite, not something the facade controls.
- [ ] Decide: Azure SignalR Service vs Azure Web PubSub vs self-managed WebSocket + Redis — pick based on cost, existing Azure footprint, and team familiarity.
- [ ] Multi-device behavior: if a user is logged in on phone + tablet simultaneously, do both receive pushes/socket messages? How is read state reconciled across devices?
- [ ] Reconciliation job cadence and trigger conditions — fixed interval only, or also triggered by health-check failures on the webhook receiver? What's an acceptable staleness window before compliance cares?
- [ ] Channel System technology choice — confirm Kafka (or equivalent event log) + Postgres, or whatever fits existing platform standards, and confirm retention/encryption requirements from compliance/legal.
- [ ] PII/data residency: does D365's geo-region and our Channel System's storage region need to match for regulatory reasons?
- [ ] Push notification content policy — confirm with compliance/security that redacted push bodies ("new message from your agent") are sufficient, or whether even that metadata needs extra handling.
- [ ] Exact webhook payload/header structure for our specific D365 tenant tier — grab the current Microsoft Learn reference before implementation, since this was a preview feature at time of writing and details may shift.
- [ ] Failure/alerting strategy: what should trigger a page — webhook receiver down, reconciliation job finding persistent gaps, D365 auth token failures, etc.
