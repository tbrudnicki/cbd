# Solution Design — planning pass

Status: planning input for authoring `sd-template.md` — not the SD itself
Relates to: [sd-roadmap.md](./sd-roadmap.md), [adr-0001](./adr-0001-realtime-transport.md), [adr-0002](./adr-0002-channel-system-technology.md), [adr-0003](./adr-0003-reconciliation-job.md), [adr-0004](./adr-0004-webhook-ga-status.md)

This scopes what needs to go into the actual Solution Design document (per `sd-template.md`'s structure) before drafting it: data models, C2 diagram elements, the sequence-diagram scenario list, and the API surface. Written against the locked assumptions below.

## Progress tracker

Update the status column as work lands — this is the at-a-glance view; the numbered sections below are the detail.

**Planning (this document)**

| Item | Status |
|---|---|
| Locked assumptions & ADR decisions | ✅ Done (§1) |
| Data models — `Conversation`, `ConversationEvent`, `Attachment` | ✅ Done (§2) |
| C2 diagram — elements identified | ✅ Done (§3) |
| C2 diagram — actually drawn | ✅ Done — see [solution-design.md](./solution-design.md) |
| Sequence diagram scenarios — identified | ✅ Done, 11 scenarios (§4) |
| Open questions from planning pass | ✅ All resolved (§5) |
| API endpoint list | ✅ Done (§7) |

**Sequence diagrams** — each maps to a scenario in §4; draw one at a time, roughly in this order since later ones depend on earlier ones (connection lifecycle and FAQ basics before escalation; escalation before the D365-dependent flows)

| # | Scenario | Status |
|---|---|---|
| 1 | App startup & connection lifecycle | ✅ Done — [solution-design.md](./solution-design.md) |
| 2 | Start conversation + first FAQ prompt | ✅ Done — [solution-design.md](./solution-design.md) |
| 3 | Escalate FAQ → HUMAN | ✅ Done — [solution-design.md](./solution-design.md) |
| 4 | Outbound text/link message (HUMAN) | ✅ Done — [solution-design.md](./solution-design.md) |
| 5 | Outbound file message (HUMAN) | ✅ Done — [solution-design.md](./solution-design.md) |
| 6 | Inbound agent text message | ✅ Done — [solution-design.md](./solution-design.md) |
| 7 | Inbound agent file message | ✅ Done — [solution-design.md](./solution-design.md) |
| 8 | Delivery ack + read receipt | ✅ Done — [solution-design.md](./solution-design.md) |
| 9 | Reconnect + catch-up | ✅ Done — [solution-design.md](./solution-design.md) |
| 10 | Reconciliation sweep | ✅ Done — [solution-design.md](./solution-design.md) |
| 11 | Conversation close by agent | ✅ Done — [solution-design.md](./solution-design.md) |

**Final SD document** (`sd-template.md` sections, to be authored into an actual `solution-design.md` deliverable once the above is drawn)

| Template section | Status |
|---|---|
| 1. Introduction | 🟡 In progress — purpose/scope/related-docs/naming drafted, see [solution-design.md](./solution-design.md); document owner & approval chain still TBD (roadmap §4.5) |
| 2. Architecture — C2 diagram | ✅ Done — [solution-design.md](./solution-design.md) |
| 2. Architecture — Sequence diagrams | ✅ Done — all 11 of 11 drawn, see [solution-design.md](./solution-design.md) |
| 3. Design & Implementation — User Journey | ⬜ Not started — needs a Figma link, external input |
| 3. Design & Implementation — Backlog/Features | ⬜ Not started |
| 3. Design & Implementation — Decisions | ✅ Done — [solution-design.md](./solution-design.md) links ADR-0001 through ADR-0004 |
| 3. Design & Implementation — Tests | ⬜ Not started |
| 3. Design & Implementation — API definition | 🟡 In progress — full endpoint catalog with data-model mapping done, see [solution-design.md](./solution-design.md); formal swagger/OpenAPI spec still outstanding |

## 1. Locked assumptions

**From ADRs (as directed):**
- **ADR-0001**: Azure Web PubSub (not SignalR, not self-managed).
- **ADR-0002**: Option A — Azure SQL Database only, no separate event log/Kafka/Event Hubs.
- **ADR-0003**: Option A — **fixed-interval reconciliation only**, no triggered/flag-based path. (Note: this is *not* the ADR's preliminary recommendation, which was the hybrid Option C — proceeding with A as explicitly directed. This simplifies the design: one cron bean, no flag table, no webhook-failure-triggered path.)
- **ADR-0004**: Messaging API + webhook subscription available (GA confirmed).

**Naming:**
- Mobile app: **Up By CBD**
- Facade/BFF: **smb-chat-service** (Java/Spring Boot)

**New business requirements introduced this pass:**
- Support sending **text, links, and files**. (Working assumption: a link is just a `TEXT` message containing a URL — no distinct type/preview metadata unless told otherwise, since no rich-preview requirement was stated. Flagged in §5.)
- **Three conversation states.** Reading of the described flow: **FAQ → HUMAN → CLOSED**. Working assumption on exact names — flagged in §5, cheap to rename later, not blocking.
  - **FAQ**: default state on conversation start. Every user message is routed to an **FAQ SDK embedded in-process inside smb-chat-service** (not a separate deployed service, per how it was described) — no D365 involvement at all in this state.
  - **HUMAN**: entered when the user explicitly escalates after at least one FAQ prompt. Triggers D365 conversation creation (`POST /conversation/create`). Spans D365's own internal `open`/`wrap-up` sub-lifecycle as a single state at the smb-chat-service level.
  - **CLOSED**: terminal, reached when the D365 agent closes the conversation (`ConversationClosed`/`AgentClosed` webhook event).

**Confirmed this session:**
- FAQ-phase messages **are** persisted in the same smb-chat-service-db (Azure SQL) as HUMAN-phase messages — one shared event table (`ConversationEvent`, §2 — generalized from a plain `Message` table during data-model discussion), one write path, for the full customer interaction history. (This eliminates the need for a separate FAQ-interaction table — noted in §2. Since both phases share a table, every row carries an explicit `phase` discriminator so the two remain cleanly distinguishable — §2.)
- File attachments are **uploaded via the facade** (smb-chat-service proxies the upload to Blob Storage and generates the `contentUrl`) — not direct-to-blob via SAS token from the mobile app. Consistent with the existing "one ingress path through the facade" principle from ADR-0001's outbound-message discussion.

**Fact checked live against current D365 docs (Aug 2026), load-bearing for the file-attachment design:**
- The Messaging API's message payload takes attachments as `{ contentType, contentUrl, name }` — a **fetchable URL**, not inline bytes. This confirms the file must land somewhere D365's infrastructure can reach *before* the message activity referencing it is sent — shapes sequence diagram #5 in §4.
- Message `text` is capped at **6,000 characters**. Either `text` or `attachments` must be present — a request with neither is rejected. Worth enforcing this at the facade validation layer for both FAQ and HUMAN phases, not just outbound-to-D365 traffic.

## 2. Data models

One write path (per the confirmed FAQ-persistence decision) means both conversation phases share the same event table — no separate FAQ log.

**Generalized from `Message` to `ConversationEvent`** (design discussion): a message is one of several things that can happen in a conversation's lifecycle — D365's own Bot Framework Activity Schema already models inbound traffic this way (`message`, `typing`, `event`, `endOfConversation`, with named sub-events like `AgentAccepted`/`AgentClosed`/`ConversationClosed`), so mirroring that shape avoids forcing every non-message occurrence into either an ill-fitting `Message` row or a mutable `Conversation` field that can only ever hold the latest value and loses history if something happens more than once. `Conversation`'s own `escalatedAt`/`closedAt`/`readWatermark`-style fields are kept as fast denormalized "current state" mirrors, but the append-only `ConversationEvent` log underneath them is now the actual source of truth and audit trail.

**Conversation**
| Field | Notes |
|---|---|
| `id` (PK) | |
| `userId` | FK to banking identity |
| `state` | enum `{FAQ, HUMAN, CLOSED}` — naming assumption, §5 |
| `d365ConversationId` | nullable until escalation |
| `d365SubState` | nullable; mirrors D365's `open`/`wrap-up`/`closed` while `state=HUMAN`; this is the field ADR-0003's reconciliation job scopes against |
| `createdAt`, `escalatedAt` (nullable), `closedAt` (nullable) | denormalized mirrors of the `ESCALATED`/`CLOSED` events below — fast lookup only, not the audit record itself |
| `closedReason` | denormalized mirror of the `CLOSED` event's `metadata`; e.g. `AGENT_CLOSED` |
| `deliveryWatermark` | last acked event sequence (ADR-0001's delivery watermark) |
| `readWatermark` | denormalized mirror of the latest `READ_RECEIPT` event's `relatedSequenceNo` — fast lookup for UI read-checkmarks/badges without querying the event log each time |

**ConversationEvent** (generalized from `Message`)
| Field | Notes |
|---|---|
| `id` (PK) | |
| `conversationId` (FK) | |
| `eventType` | enum `{MESSAGE, AGENT_ACCEPTED, ESCALATED, CLOSED, READ_RECEIPT}` — **discriminator.** `TYPING` is deliberately *not* in this list: relayed live over Web PubSub only, never persisted — pure ephemeral noise with no compliance value, would bloat the log for nothing. Catalog confirmed as sufficient for now; extensible if a new D365 activity type needs capturing later. |
| `sequenceNo` | **DB-generated, globally monotonic identity — not a hand-maintained per-conversation counter, not a linked-list pointer.** Reflects persistence order ("what did we learn about, and when"), assigned atomically by the database at insert time regardless of writer (webhook handler, outbound REST handler, reconciliation backfill can all write concurrently with no race). **This is the sync-critical field** — watermarks, catch-up ("everything since X"), and reconciliation diffing all key off it, and it is never reordered or mutated once assigned. A late-arriving/backfilled event simply gets whatever value is next available at actual insert time, even though it logically "happened" earlier — see rationale below. Now spans *all* event types, so the transcript UI renders one ordered feed (messages interleaved with "agent joined," "conversation closed," etc.) rather than needing a client-side merge of separate tables. |
| `occurredAt` | The source system's own timestamp for when the event actually happened — client-send time (customer message), D365 activity timestamp (agent message/`AgentAccepted`/`CLOSED`), FAQ SDK response time (bot message), read-action time (`READ_RECEIPT`). **Display-ordering only**, never used for watermarks/sync, since it isn't reliably monotonic across delivery paths the way `sequenceNo` is. |
| `phase` | enum `{FAQ, HUMAN}` — **explicit discriminator, stamped at write time.** Not derivable from `origin` alone (a `CUSTOMER`- or `SYSTEM`-authored row looks identical in either phase) and deliberately not inferred from `activityId` nullability, which would be an implicit/fragile discriminator. This is the field every phase-scoped query filters on — including ADR-0003's reconciliation job, which must cleanly select only `HUMAN`-phase rows. |
| `origin` | enum `{CUSTOMER, FAQ_BOT, AGENT, SYSTEM}` — orthogonal to `phase`: says *who* produced the event, not *which* phase it belongs to |
| `direction` | enum `{INBOUND, OUTBOUND}` — applies to `MESSAGE`; other event types are mostly one-directional by nature (e.g. `CLOSED` is always agent/system-originated) |
| `contentType` | enum `{TEXT, FILE}` — **populated only when `eventType = MESSAGE`**; link = `TEXT`, §5 |
| `body` | nullable; populated only for `MESSAGE`; enforce ≤6,000 chars (§1) |
| `attachmentId` (FK, nullable) | populated only for `MESSAGE` |
| `relatedSequenceNo` (nullable) | populated only for `READ_RECEIPT` — the `sequenceNo` of the event being acknowledged as read. Kept as a first-class column rather than buried in `metadata` since it's the one cross-type-reusable "points at another event" field (delivery-ack style events, if ever added, would use the same column). |
| `metadata` | nullable JSON — for the low-frequency/one-off event types (`AGENT_ACCEPTED`'s agent id/name, `CLOSED`'s reason/who-closed-it) that don't justify dedicated columns each. Deliberately not used for `MESSAGE`'s core fields, which stay as real typed columns for query/index-ability. |
| `activityId` | D365 Bot Framework activity ID; **nullable for `FAQ`-phase rows** (never reach D365) as a *consequence* of `phase`, not as the thing that defines it; unique constraint drives idempotent dedupe for `HUMAN`-phase rows, same as ADR-0002/0003. Applies to any D365-sourced event type, not just `MESSAGE`. |
| `persistedAt` | our system's write timestamp — audit/debugging use, distinct from `occurredAt` |

**Why `READ_RECEIPT` is persisted, not just a `Conversation.readWatermark` field:** confirmed as a deliberate early-stage choice — the event gives a full audit trail of every read-position change (useful for debugging delivery/read behavior while the system is new), at the cost of one more row per read action. Acceptable to drop down to watermark-only later if the volume/value trade-off doesn't hold up; the denormalized `Conversation.readWatermark` field already coexists as the fast-lookup path regardless, so removing the event rows later wouldn't be a breaking change to anything reading current state — only to anything relying on read-history.

**Ordering & corruption-robustness rationale:** a linked-list ordering (`previousEventId`/`nextEventId`) was considered and rejected. Late delivery — via webhook retries, reconciliation backfill, or async agent delivery — is an expected, routine occurrence in this architecture, not an edge case, and a linked list can only accommodate an out-of-order insert by mutating an existing row's pointer to splice the new node in. That both creates write contention (must locate and lock the correct insertion point) and directly conflicts with the append-only/no-mutation principle already established for these rows in ADR-0002 (audit rows shouldn't be rewritten after the fact). Splitting sync-order (`sequenceNo`, DB-atomic, monotonic, immutable) from display-order (`occurredAt`, source-timestamp, informational) means a late-arriving event is a plain insert that touches nothing else — no chain to walk, no pointer to fix up, no existing row mutated. Referential integrity (FK constraints) and idempotent dedupe (`activityId` unique constraint) cover the remaining corruption concerns without needing any chain structure. The one place a chain-like structure would genuinely earn its keep is **tamper-evidence** (a hash chain over content, for detecting after-the-fact alteration) — a different problem from ordering, still an open question from ADR-0002 §6.4/§7, and buildable later keyed off `sequenceNo` order without disturbing this design.

Recommend an index on `(conversationId, phase, sequenceNo)` — the shape both the mobile catch-up query and the reconciliation job's scoping query need — plus a secondary index on `(conversationId, eventType)` for targeted queries (e.g. "just the read-receipt history"). Exact index/type detail is still Phase 3 work; noting the shape here since it falls directly out of the `phase`/`eventType` fields.

**Attachment**
| Field | Notes |
|---|---|
| `id` (PK) | |
| `eventId` (FK to `ConversationEvent`) | renamed from `messageId` — an attachment belongs to a `ConversationEvent` row where `eventType = MESSAGE` |
| `fileName` | |
| `mimeType` | **constrained, confirmed**: allowlist `{image/jpeg, image/png, application/pdf}` — images and PDFs only. Enforced at facade validation on upload, not left to the client. |
| `sizeBytes` | **constrained, confirmed**: max **10 MB**. Enforced at facade validation on upload. |
| `blobStorageRef` | our Blob Storage container/path — plain storage, no scanning step (confirmed: virus scanning not required at this time; simply stored as-is) |
| `contentUrl` | the D365-fetchable URL handed to the Messaging API — a SAS URL with a **fixed expiry duration, configurable at service deployment** (an application property, e.g. `attachment.sas-token.expiry-minutes`) rather than computed per-request or negotiated dynamically with D365 |
| `createdAt` | |

~~`virusScanStatus`~~ — **descoped.** Confirmed: no scanning requirement at this time; files are stored as uploaded. Noted here rather than silently dropped, in case this needs revisiting later (e.g. if a compliance/security review later requires it).

~~**ConnectionRegistry**~~ — **removed.** Its only stated purpose was supporting single-device enforcement (refuse/displace) at the smb-chat-service layer. Confirmed that enforcement is handled upstream by the Identity Provider (one session per IDP profile) — not something this system needs to build. Azure Web PubSub's own `sendToUser` primitive already routes by `userId` without a connectionId lookup table (ADR-0001 §6.2), so there was no second purpose keeping this entity alive. See ADR-0001 §8.

Exact column types/indexes are Phase 3 detailed-design work, not this pass — this is entity/field scoping only.

## 3. C2 container diagram — elements to include

- **Person**: SME customer
- **Container**: Up By CBD (mobile app)
- **Container**: smb-chat-service (Java/Spring Boot BFF) — show as containing two internal components, not separate containers: the **embedded FAQ SDK** (confirmed: pure in-process function call, no external dependency — nothing to show leaving the container for the FAQ path) and the **reconciliation sweep job** (Spring `@Scheduled` bean, per ADR-0003 Option A)
- **Container**: smb-chat-service-db (Azure SQL Database) — `Conversation`, `ConversationEvent`, `Attachment`
- **Container**: Azure Blob Storage (file bytes)
- **Container**: Azure Web PubSub
- **External system**: Push notification gateway (FCM/APNs) — assumed pre-existing shared infra, not built by this project
- **External system**: D365 Omnichannel / Messaging API
- **External system**: Microsoft Entra ID (OAuth2 client-credentials)

This C2 element list is now complete — no unconfirmed external dependencies remain.

## 4. Sequence diagram scenarios

Proposed set for the SD's "Sequence diagram" section — each maps to a `sequenceDiagram` block per the template:

1. **App startup & connection lifecycle** — auth, `GET /chat/current-session`, conditional Web PubSub negotiate+connect (conversation-state-scoped per ADR-0001 §6.3)
2. **Start conversation + first FAQ prompt** — `POST /chat/sessions` (state=FAQ), in-process FAQ SDK call, synchronous response
3. **Escalate FAQ → HUMAN** — `POST /chat/sessions/{id}/escalate`: read the conversation's `FAQ`-phase `ConversationEvent` rows, format into one or more context variables, include them in D365's `POST /conversation/create` request body, transition `Conversation.state = HUMAN`, persist an `ESCALATED` event. **Confirmed: FAQ transcript is included as context passed to D365.** Design caveat worth showing in the diagram itself: D365's context-variables API is documented as **eventually consistent** — the agent isn't guaranteed to see it the instant the conversation starts, even though we send it at creation time. Nothing to fix on our side; just don't design any agent-side behavior that assumes synchronous visibility.
4. **Outbound text/link message (HUMAN)** — mobile → facade → smb-chat-service-db → D365 (existing pattern, restated with current naming)
5. **Outbound file message (HUMAN)** — upload via facade-proxy (§1) → Blob Storage → `contentUrl` generated → `Attachment` + `ConversationEvent` (`eventType=MESSAGE`) persisted → forwarded to D365 with `attachments[]`
6. **Inbound agent text message** — webhook → facade → smb-chat-service-db → Web PubSub push (reuse ADR-0001 pattern)
7. **Inbound agent file message** — webhook with `attachments[].contentUrl` → **confirmed: download and copy into our own Blob Storage** (matches the "our store is source of truth" principle — retention isn't dependent on D365 continuing to host the file) → persist `Attachment` + `ConversationEvent` → push notification → mobile downloads via facade-brokered URL
8. **Delivery ack + read receipt** — reuse ADR-0001 pattern, now keyed by `sequenceNo` spanning both phases
9. **Reconnect + catch-up** — reuse ADR-0001 §6.4; catch-up now may replay FAQ-phase messages too, not just D365-sourced ones
10. **Reconciliation sweep** — ADR-0003 Option A: single fixed-interval cron bean, scoped to `Conversation.state = HUMAN` and filtering `ConversationEvent.phase = HUMAN, eventType = MESSAGE` when diffing against D365. **Confirmed via docs (Aug 2026): `GET /conversation/{id}/messages`'s response only ever contains `type: "message"` activities** — so this reconciliation path is a `MESSAGE`-only safety net. It does **not** backfill a lost `AGENT_ACCEPTED`/`CLOSED` webhook delivery; those event types have no equivalent history endpoint to reconcile against and depend entirely on the webhook's 3-attempt delivery. For `CLOSED` specifically, ADR-0003 §4.2's stale-open escalation is the actual mitigation (detects a conversation that never received its close event, rather than backfilling the event itself). A lost `AGENT_ACCEPTED` is lower-stakes (informational/audit only, doesn't block anything operationally) and is accepted as a residual gap unless told otherwise.
11. **Conversation close by agent** — webhook `ConversationClosed`/`AgentClosed` → `state=CLOSED` → notify mobile → close Web PubSub connection (conversation-state-scoped lifecycle, ADR-0001 §6.3)
~~12. **Single-device displacement**~~ — **removed from scope.** Not a scenario this system needs to diagram: the single-device constraint is enforced upstream by the Identity Provider, not by smb-chat-service or Web PubSub, so there's no refuse/displace behavior of ours to show. See ADR-0001 §6.2/§8.

## 5. Open questions surfaced by this pass

- [x] ~~Confirm conversation state names~~ — **confirmed**: `FAQ` / `HUMAN` / `CLOSED`.
- [x] ~~Does the FAQ SDK depend on any external service itself?~~ — **confirmed: no.** The FAQ SDK is an in-process backend function call inside smb-chat-service — no external call (no LLM API, no search index, nothing leaving the process) happens during a FAQ-phase interaction. Simplifies the C2 diagram (§3): no external system needed for the FAQ path, and no third-party data-sharing/vendor-review question for the security review either, since nothing about the customer's message ever leaves smb-chat-service during that phase.
- [x] ~~Attachment constraints~~ — **confirmed**: 10 MB max, images + PDFs only (`image/jpeg`, `image/png`, `application/pdf`), no virus scanning required for now (plain storage).
- [x] ~~SAS token expiry vs. D365's fetch timing~~ — **confirmed**: fixed expiry duration, configurable at service deployment (not dynamic/per-request).
- [x] ~~Inbound attachments: copy vs. reference~~ — **confirmed: copy into our own Blob Storage.**
- [x] ~~Escalation context handoff~~ — **confirmed: yes**, FAQ transcript is passed to D365 as context variables at escalation. New follow-on item, not yet resolved: **context-variable payload size limit is undocumented** — worth confirming empirically in sandbox before assuming an arbitrarily long FAQ transcript can be sent as one value; may need a truncation strategy (e.g. last N exchanges) as a safety measure if a limit is hit in practice.
- [x] ~~Single-device enforcement mechanism~~ — **resolved: out of scope.** Enforced upstream by the Identity Provider (one session per IDP profile); smb-chat-service builds no refuse/displace logic. `ConnectionRegistry` removed from the data model (§2) and diagram #12 removed from scope (§4) as a consequence.

**All open questions from this pass are now resolved.**

**New follow-on surfaced while drawing sequence diagrams:**
- [ ] **Delivery-ack transport mechanism** — sequence diagram #8 ([solution-design.md](./solution-design.md)) assumes delivery acks travel client → Web PubSub upstream message → facade event-handler callback (per ADR-0001 §2/§7's event-handler model), since no REST endpoint for it exists in the §7 endpoint list. This is a design inference, not a confirmed decision — worth explicitly confirming once ADR-0001 is finalized, since the ack path is otherwise unspecified.
- [ ] **Inbound attachment download endpoint missing** — sequence diagram #7 surfaced that the §7 endpoint list has no `GET` route for the mobile app to download an inbound (agent-sent) attachment via the facade. Added as `GET /chat/attachments/{attachmentId}` to §7 — needs to be carried into the eventual swagger spec.
- [ ] **Web PubSub system-events endpoint missing** — sequence diagram #9 surfaced that ADR-0001 §6.4's `disconnected` CloudEvent needs somewhere to land ("our webhook/upstream endpoint"), but §7's endpoint list only has the D365 webhook receiver. Needs a dedicated route, e.g. `POST /webhooks/webpubsub/events` — add when the endpoint list is next revisited.

## 6. Suggested next step

See the **Progress tracker** at the top of this document for the live status — data models and the C2 element list are locked, so the next unblocked work is drawing the actual C2 diagram and working through the 11 sequence diagrams in §4, roughly in listed order (they build on each other: connection lifecycle and FAQ-phase basics before escalation, escalation before the D365-dependent flows).

## 7. API endpoint list

Feeds the SD template's "API definition" section — formal swagger/OpenAPI spec is separate follow-on work, this is the surface to formalize.

| Endpoint | Purpose |
|---|---|
| `POST /chat/connection/negotiate` | Issue a Web PubSub access token, scoped to `userId`. Called conditionally per ADR-0001 §6.3 — only when a conversation is `FAQ`/`HUMAN`, not unconditionally on app foreground. |
| `GET /chat/current-session` | App-startup discovery — does an active/recent conversation exist for this user, and what's its watermark? |
| `POST /chat/sessions` | Start a new conversation (`state = FAQ`). |
| `POST /chat/sessions/{id}/messages` | Send a message (text/link → `body`, or `attachmentId` reference for a file). Behavior branches on `Conversation.state`: `FAQ` routes to the in-process FAQ SDK call; `HUMAN` forwards to D365. |
| `POST /chat/sessions/{id}/attachments` | Upload a file (facade-proxied per confirmed decision — mobile sends bytes here, facade writes to Blob Storage and returns an `attachmentId`). Enforces the 10 MB / image-or-PDF constraint (§2). |
| `POST /chat/sessions/{id}/escalate` | Transition `FAQ` → `HUMAN`: creates the D365 conversation, passes the FAQ transcript as context variables, persists an `ESCALATED` event. |
| `GET /chat/sessions/{id}/events?sinceSeq=` | Catch-up — "everything since my last watermark," keyed on `ConversationEvent.sequenceNo`. Renamed from the earlier `/messages` framing now that this returns all event types, not just messages. |
| `POST /chat/sessions/{id}/read` | Read receipt — persists a `READ_RECEIPT` event and advances `Conversation.readWatermark`. |
| `POST /webhooks/d365/conversations/{conversationId}/activities` | Inbound webhook receiver for D365 activities (agent messages, `AgentAccepted`, `AgentClosed`/`ConversationClosed`). Ack fast (<10s per ADR-0003), process async. |
| `GET /chat/attachments/{attachmentId}` | **Gap surfaced while drawing sequence diagram #7** ([solution-design.md](./solution-design.md)) — no facade-brokered download route currently exists for the mobile app to fetch an inbound (agent-sent) attachment's bytes. `POST /chat/sessions/{id}/attachments` only covers the outbound (upload) direction. Needs to be added before the swagger spec is drafted. |
