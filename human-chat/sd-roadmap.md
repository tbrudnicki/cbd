# Solution Design Roadmap — Human Chat Facade (D365 Omnichannel Integration)

Status: draft / working document
Source input: [chat-facade-architecture-summary.md](./chat-facade-architecture-summary.md) (informal architecture exploration with an AI agent)
Purpose: this document plans **how we get from the informal summary to a signed-off, official Solution Design (SD) document**. It is not the SD itself — it's the roadmap for producing it.

---

## 1. What we're starting from

The summary already gives us a working architecture hypothesis:

- Java/Spring Boot facade between mobile app and D365 Omnichannel, using the **Messaging API**
- Channel System (compliance store) as source of truth, written before any downstream hop
- Webhook-primary + sparse-reconciliation inbound pattern
- Azure SignalR/Web PubSub for real-time delivery, push as backgrounded fallback
- User/device-scoped connection lifecycle, decoupled from conversation lifecycle
- Two-watermark model (delivery-ack vs read-receipt)

This is strong enough to design around, but it's a single AI-assisted exploration, not a reviewed architecture. Several decisions are still open (§4), and none of it has been validated against our actual D365 tenant, compliance function, or platform standards. The SD phases below exist to close those gaps and produce something an architecture review board / compliance / security can actually sign off on.

---

## 2. Phases

### Phase 0 — Framing & stakeholder alignment
**Goal:** agree who needs to be in the room and what "done" looks like before writing anything formal.

- Confirm the SD's audience and approval chain (architecture review board? security? compliance/legal? D365 platform owner?)
- Identify and engage stakeholders (see §5 RACI)
- Confirm document template/standard if the org has one (many shops have a mandated SD/ADR template — don't build a bespoke structure if one exists)
- Set target review date(s) / any external deadline (e.g. contact center program milestone)

**Exit criteria:** stakeholder list confirmed, template chosen, review checkpoint dates set.

### Phase 1 — Requirements & constraints gathering
**Goal:** turn assumptions in the summary into confirmed requirements.

- Functional requirements: what "human chat" must support (text only? attachments? typing indicators? transfer/escalation UX? session timeout/re-open behavior?)
- Non-functional requirements: expected concurrent chat volume, peak load, latency targets, availability target/SLA
- Compliance/regulatory requirements: what must be logged, retention period, encryption at rest/in transit, data residency
- Licensing/commercial constraints: confirmed D365 tier, Messaging API + webhook entitlement
- Existing platform standards: mandated tech for event log/DB, push infra, auth, observability stack — so the design doesn't invent things that duplicate existing platform capability

**Exit criteria:** a requirements list stakeholders have reviewed (even informally), not just the AI-assisted assumptions.

### Phase 2 — Architecture decision resolution (ADRs)
**Goal:** close the open decisions in §4 with an actual decision + rationale, not just a flagged question.

For each open item, produce a short ADR: decision, options considered, why, consequences. Candidates likely to need one each:
- Real-time transport: Azure SignalR vs Web PubSub vs self-managed — comparison drafted in [adr-0001-realtime-transport.md](./adr-0001-realtime-transport.md), decision still pending stakeholder workshop
- Channel System technology (Azure SQL Database vs. Kafka/Event Hubs + Azure SQL; **Postgres is not an allowed org DB standard**) — comparison drafted in [adr-0002-channel-system-technology.md](./adr-0002-channel-system-technology.md), decision still pending stakeholder input
- Reconciliation job cadence/trigger model — comparison drafted in [adr-0003-reconciliation-job.md](./adr-0003-reconciliation-job.md), decision still pending stakeholder input
- ~~Multi-device fan-out and read-state reconciliation approach~~ — **discarded, no ADR needed.** Confirmed constraint: only one device may be associated with a given user at a time, so no fan-out scenario exists. See ADR-0001 §6.2 (updated) and the resolved open question below.
- ~~Webhook preview-feature risk acceptance (or fallback plan if GA is delayed/denied)~~ — **resolved.** See [adr-0004-webhook-ga-status.md](./adr-0004-webhook-ga-status.md): the webhook subscription capability reached GA on Feb 24, 2026 (confirmed against Microsoft's official release-plan docs) — the "might never ship" risk this was scoped for no longer exists. Remaining items are licensing/SLA confirmations, not an architecture decision.

**Exit criteria:** every item in §4 has either an ADR or an explicit "deferred, revisit at X" note with an owner.

### Phase 3 — Detailed design
**Goal:** the technical meat of the SD — enough for engineers to build from and reviewers to critique.

- Component diagram (mobile app, facade, Channel System, D365, SignalR/Web PubSub, push infra)
- Sequence diagrams: outbound message flow, inbound webhook flow, reconciliation flow, app resume/reconnect+catch-up flow, connection auth flow
- Data model: Channel System schema (conversation, message, watermark, delivery/read state), conversationId→userId mapping
- API contracts: facade↔mobile (REST + socket message shapes), facade↔D365 (request/response, headers, auth)
- State machines: conversation lifecycle (created→open→wrap-up→closed), connection lifecycle
- Idempotency/dedup strategy detail (activity ID dedup, at-least-once delivery semantics)
- Error handling & retry design: D365 API failures, webhook validation failures, SignalR/Web PubSub outages, push provider failures

**Exit criteria:** design is detailed enough that a second engineer could estimate/build from it without re-deriving decisions.

### Phase 4 — Non-functional design
**Goal:** the parts that get skipped if you only design the happy path.

- Capacity/scalability plan (expected concurrent conversations, connection count, facade horizontal scaling)
- Observability: metrics, logs, traces, dashboards — what signals prove the system is healthy
- Alerting/paging strategy (ties to open question on failure triggers)
- Disaster recovery / failover (facade instance loss, Channel System unavailability, D365 outage, Azure region issue)
- Security design: token handling, secret storage, webhook endpoint hardening (signature/origin validation), PII handling in logs, push payload content policy

**Exit criteria:** NFRs have concrete targets (numbers), not just qualitative statements.

### Phase 5 — Compliance & security review
**Goal:** get the design in front of the people who can veto it, before it's "final."

- Formal security review (threat model walkthrough: webhook spoofing, replay, token theft, push content leakage)
- Compliance/legal review: audit store completeness, retention, data residency, redacted-push adequacy
- D365/Microsoft-side confirmation: licensing tier, webhook GA status/support commitment, workstream/routing config ownership

**Exit criteria:** documented sign-off or documented list of required changes, from security and compliance specifically (not just architecture peers).

### Phase 6 — Review & sign-off
**Goal:** formal approval.

- Circulate SD draft to architecture review board / relevant leads
- Incorporate feedback, track as revisions (keep a changelog in the doc)
- Final sign-off from accountable owner(s)

**Exit criteria:** SD marked approved, version-locked.

### Phase 7 — Implementation readiness handoff
**Goal:** bridge from "designed" to "buildable."

- Break design into epics/stories or a phased delivery plan (e.g. facade skeleton + Channel System write path first, then D365 integration, then real-time delivery, then reconciliation)
- Identify what needs a spike/POC before full build (D365 sandbox access and webhook behavior end-to-end — now GA per ADR-0004, but still worth a spike to validate against our actual tenant before committing; the chosen real-time transport's `sendToUser` delivery and reconnect/recovery behavior per ADR-0001)
- Confirm test strategy (contract tests against D365 sandbox, chaos/failure injection for the reconciliation gap scenario)

**Exit criteria:** engineering has a backlog it can start pulling from, with explicit spikes called out for anything the SD had to assume rather than verify.

---

## 3. Suggested document outline for the final SD

(Adjust to match org template if one exists — see Phase 0.)

1. Executive summary
2. Business context & requirements
3. Scope & non-goals
4. Architecture overview (diagram + narrative)
5. Key architecture decisions (ADRs, linked or embedded)
6. Detailed design (components, sequence diagrams, data model, API contracts, state machines)
7. Non-functional requirements & design (capacity, availability, observability, DR)
8. Security & compliance design
9. Risks & open items with owners
10. Rollout / implementation plan
11. Appendix: glossary, D365 reference links, alternatives considered and rejected (Chat SDK, Direct Line)

---

## 4. Open questions — consolidated and expanded

Carried over from the architecture summary, grouped, plus follow-ons surfaced while planning this roadmap. These are blocking inputs to the phases above, not just background reading — each should have an owner before the phase that needs it starts.

### 4.1 D365 / Microsoft-side (blocks Phase 1, 2, 5)
- [ ] Confirm D365 licensing tier includes Messaging API + webhook subscription.
- [x] ~~Confirm GA timeline for webhook capability~~ — **resolved.** See [adr-0004-webhook-ga-status.md](./adr-0004-webhook-ga-status.md): GA confirmed Feb 24, 2026 against Microsoft's official release-plan docs. Remaining: get written support-commitment/SLA confirmation from the Microsoft contact (lighter-weight than the original "will this ever ship" question).
- [ ] Confirm workstream/unified-routing configuration (queues, skill groups for SME banking support) and who owns that config.
- [ ] Get the current Microsoft Learn reference for webhook payload/header structure for our specific tenant tier (worth re-checking periodically even post-GA — feature is only ~6 months out of preview as of this writing).
- [ ] Confirm rate limits / throughput limits on the Messaging API for our tenant.

### 4.2 Business / compliance / legal (blocks Phase 1, 5)
- [ ] What exactly must the Channel System capture for audit (message content, metadata, timestamps, agent identity, disposition codes)?
- [ ] Retention period and deletion/right-to-erasure requirements for chat records.
- [ ] Encryption requirements at rest and in transit — any specific standard mandated (e.g. FIPS, org-specific)?
- [ ] Data residency: must D365's geo-region match Channel System's storage region?
- [ ] Push notification content policy: is a redacted body ("new message from your agent") sufficient, or does even that metadata need extra handling/legal sign-off?
- [ ] Acceptable staleness window for the reconciliation job before it's a compliance concern (i.e. how long can a gap between D365 and our store persist before it's a reportable issue)?

### 4.3 Technical / platform (blocks Phase 2, 3, 4)
- [ ] SignalR vs Web PubSub vs self-managed WebSocket+Redis — decide based on cost, existing Azure footprint, team familiarity (needs an ADR, see Phase 2).
- [ ] Channel System technology — see [adr-0002-channel-system-technology.md](./adr-0002-channel-system-technology.md): **org DB standard is Microsoft SQL Server/Azure SQL — Postgres is not allowed.** Preliminary recommendation is Azure SQL Database only (no separate event log), pending compliance's answer on tamper-evidence/WORM requirements and platform engineering's answer on whether Kafka/Event Hubs is already a mandated org-wide pattern we should reuse instead.
- [ ] Reconciliation job cadence and trigger conditions — see [adr-0003-reconciliation-job.md](./adr-0003-reconciliation-job.md): preliminary recommendation is a hybrid model (sparse fixed-interval baseline + immediate trigger on locally-observed failure signals), with a 15-minute starting interval pending compliance confirmation of the acceptable staleness window and confirmation of the D365 Messaging API's actual rate limit.
- [x] ~~Multi-device behavior~~ — **resolved: not applicable.** Confirmed platform constraint: only one device may be associated with a given user at a time, so no fan-out or cross-device read-state reconciliation scenario exists. Replaced by a new open item: how is the single-device constraint technically enforced (does a new device's connection attempt get refused, or does it silently displace/disconnect the previous one)? See ADR-0001 §6.2 and §8.
- [ ] Failure/alerting strategy: what specifically pages someone — webhook receiver down, reconciliation finding persistent gaps, D365 auth token failures, SignalR/Web PubSub connection failures?
- [ ] Auth: does the facade's D365 client-credentials token get cached/refreshed centrally, and what's the blast radius/runbook if that token starts failing mid-day?
- [ ] Expected scale: concurrent conversations, peak concurrent connections, message volume — needed for capacity design (Phase 4) and to sanity-check SignalR/Web PubSub tier choice.

### 4.4 Mobile client (blocks Phase 3)
- [ ] Exact reconnect/backoff behavior expected from the mobile team — do they already have a pattern for this from other features, or is this net-new?
- [ ] Is pushing other event types over this same channel (transaction alerts, fraud confirmations) a real, near-term requirement or a speculative option kept open by the earlier AI-assisted exploration? This is now a decision-blocking input, not just a nice-to-know — see [adr-0001-realtime-transport.md](./adr-0001-realtime-transport.md) §6.3: if real, the connection should be app-session-scoped; if not, it should be conversation-state-scoped (cheaper, smaller exposure window). If real, it also shapes the message envelope now rather than later.
- [ ] Offline/queued-message UX on the client: if a customer sends a message with no connectivity, what's the expected behavior (queue-and-retry, block send, etc.)?

### 4.5 Process / ownership
- [ ] Who is the accountable owner for this SD (sign-off authority)?
- [ ] Does an architecture review board exist and what's its cadence/submission process?
- [ ] Who owns the D365 admin-side configuration long-term (routing, queues) once live — is that us or a separate contact-center ops team?

---

## 5. Stakeholders (draft RACI — confirm in Phase 0)

| Area | Likely stakeholder | Role |
|---|---|---|
| Facade architecture & build | Backend/platform engineering (us) | Responsible |
| D365 config, licensing, webhook GA status | Microsoft account contact / D365 admin team | Consulted, source of truth for §4.1 |
| Compliance store requirements | Compliance/legal | Approve, source of truth for §4.2 |
| Security review | Security team | Approve |
| Mobile integration | Mobile app team | Consulted, source of truth for §4.4 |
| Azure infra (SignalR/Web PubSub, push) | Platform/infra team | Consulted |
| Final sign-off | Architecture review board / eng leadership | Accountable |

---

## 6. Immediate next steps

1. Run Phase 0: confirm stakeholder list and SD template, get this roadmap itself reviewed.
2. Start chasing §4.1 (D365/Microsoft) and §4.2 (compliance) items in parallel — they're the longest-lead-time, most likely to block later phases, and neither depends on us writing anything first.
3. Begin Phase 1 requirements gathering using the summary's architecture as a strawman to react to, rather than starting from a blank page.
