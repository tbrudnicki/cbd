# ADR-0004: D365 webhook subscription — GA status and residual risk

Status: **resolved — GA confirmed; residual-risk posture proposed, pending sign-off** (see §5)
Relates to: [chat-facade-architecture-summary.md](./chat-facade-architecture-summary.md), [sd-roadmap.md](./sd-roadmap.md) Phase 2, [adr-0003-reconciliation-job.md](./adr-0003-reconciliation-job.md)
Decision owner: TBD (backend eng lead + Microsoft account contact — confirm in Phase 0)

## 1. Context

The original architecture summary flagged an "important operational caveat": the D365 webhook subscription capability — the mechanism this entire inbound design depends on — was a **preview feature** at the time, with an explicit open question to *"confirm GA status/support commitment with our Microsoft contact before finalizing."* This ADR was scoped to answer that question and define a fallback posture if GA turned out to be delayed, denied, or the feature got discontinued.

## 2. Finding: the feature is already GA

Checked directly against Microsoft's official Dynamics 365 release-plan documentation (fetched live, Aug 2026):

> **"Use messaging APIs to manage customer chat conversations"** — Dynamics 365 Contact Center, 2025 release wave 2
> | Enabled for | Public preview | General availability |
> |---|---|---|
> | Users by admins, makers, or analysts | ✅ Sep 15, 2025 | ✅ **Feb 24, 2026** |

The feature description on that same page explicitly names the webhook capability as part of what shipped: *"A webhook subscription lets your services receive real-time updates about conversation events... when an agent sends a message, when a system message is generated, or when an agent closes the conversation."* This isn't a separate, still-in-preview sub-feature riding along on a GA'd parent — the webhook subscription is the thing that went GA.

As of this ADR (Aug 2026), the feature has been GA for **roughly six months**. The current product reference page (`intro-messaging-apis`) carries no preview/experimental banner and explicitly recommends this exact pattern — *"Messaging APIs are intended for back-end integrations. They're not recommended for use in mobile or web client applications..."* — which matches this architecture's core compliance requirement (mobile never talks to D365 directly) precisely, not coincidentally.

**This resolves the original summary's single biggest flagged risk.** The scenario this ADR was originally meant to plan around — building a compliance-critical system around a feature Microsoft might never finish, delay indefinitely, or pull — did not happen.

## 3. What this does and doesn't resolve

**Resolved:**
- Whether to build the design around a feature with no committed ship date — no longer a real risk; it shipped and has been stable in GA for ~6 months as of this writing.
- The original "risk acceptance" framing this ADR was scoped for — there's no longer a preview-risk decision to make.

**Not resolved by GA status alone — genuinely separate questions, still open:**
1. **Licensing/tenant availability** — GA on the product doesn't confirm it's included in *our specific* D365 licensing tier. Still an open item (roadmap, ADR-0001 §8 area) — GA removes "will Microsoft ever ship this" risk, not "is it in our contract" risk.
2. **Undocumented rate limits** (already flagged in ADR-0003 §2/§6) — a feature can be GA and still have thin public documentation on operational limits. GA reduces the risk of the *feature disappearing*; it doesn't by itself produce the missing rate-limit number. Still needs direct confirmation or sandbox testing.
3. **"Recently GA" is not the same risk profile as "GA for years."** Six months of GA history is real, but shorter than a mature, heavily-battle-tested API. Edge cases, doc gaps, or minor breaking changes in a point release are more plausible for a ~6-month-old GA surface than for a decade-old one. This is a reason to keep — not to remove — the belt-and-suspenders reconciliation design from ADR-0003, even though it's no longer covering for a "might get cancelled" risk.
4. **A public docs page is not a contractual support commitment.** The original open question asked for confirmation *"with our Microsoft contact"* specifically — a docs page proves the feature shipped, not that Microsoft has committed to a support SLA for it at our tier. Worth still getting this in writing (§5), just as a lighter-weight confirmation than originally scoped, not a negotiation from a position of "is this even real yet."

## 4. Why the fallback design still earns its keep

Even though the original "what if GA never happens" scenario is moot, the reconciliation job design (ADR-0003) isn't wasted effort — it was never *only* insurance against a stalled preview. It also covers the permanent, GA-independent fact that the webhook's retry budget is thin (3 attempts, 10s timeout, then permanent give-up, per ADR-0003 §2) and Microsoft's service will never notify us when it gives up. That risk is orthogonal to preview-vs-GA status and doesn't go away just because the feature shipped. No design change is needed here — this section exists to make explicit that ADR-0003's job isn't rendered redundant by this finding, just no longer double-duty as a "the whole feature might vanish" hedge.

## 5. Open items

- [ ] Confirm the Messaging API + webhook subscription is included in our specific D365 licensing tier (separate from GA status — see §3.1).
- [ ] Get written confirmation from the Microsoft account contact of support commitment/SLA at our tier, even though public docs confirm GA — a lighter check than the original "is this even going to ship" question, but still worth closing formally for a compliance-relevant dependency.
- [ ] Confirm the undocumented rate limit for the per-conversation messages endpoint (carried over from ADR-0003 §6 — unrelated to GA status, still open).
- [ ] Re-check this page (or equivalent current documentation) close to build/launch time — GA was ~6 months ago as of this ADR; re-verify nothing has changed (deprecation notices, breaking changes) rather than treating this finding as permanently frozen.

## 6. Decision

**Proceed with the design as-is.** No fallback/delay posture is needed on account of preview risk — that risk is resolved. Retain ADR-0003's reconciliation job exactly as designed, understood now as covering the permanent webhook-retry-budget risk rather than a preview-feature risk. Remaining open items (§5) are licensing/SLA confirmations, not architecture decisions, and don't block continued design work — they block production launch readiness (Phase 5/6 in the roadmap).
