---
title: 'Trustworthy LLM Memory Substrate'
description: 'Four architectural primitives and six operational disciplines for building a persistent AI memory system that agents can actually trust — grounded in real incidents from one month of multi-agent operation.'
pubDate: 'Jun 13 2026'
---

## §1 — Problem statement

A persistent memory store turns a stateless LLM into a coherent agent across sessions. It also turns a single agent's confabulation into an institutional fact.

This is the trust-substrate problem. An LLM reading from shared memory cannot, in general, distinguish its own past confabulation from grounded recall. The capture-time agent had context the read-time agent does not. If that capture was wrong — a hallucinated reference, a misremembered timeline, a claim adopted from another LLM's prior misremember — the read-time agent has no signal to challenge it. Compounded across sessions, false claims become load-bearing: agents act on them, cite them, build on them, and the next agent reading the substrate sees a corpus of mutually-supporting claims that all trace back to one bad write.

Consider the failure mode concretely. A multi-agent system shares a vector store of "thoughts" — short captures with metadata, embeddings, topic tags. Agent A captures: "Owner asked about Project X on Tuesday." Agent B, reading the corpus on Friday, finds this capture, treats it as ground truth, and acts on it. But Owner never said anything about Project X. Agent A confabulated — perhaps the embedding of a related conversation triggered a near-match that Agent A interpreted as memory. Agent B has no signal that the capture is wrong. Agent C, coordinating with B, sees both the original confabulation and B's downstream action; both look mutually-supporting. The substrate now contains a confabulation propagation chain.

The naive fixes don't work. **Author authentication** ("only trust captures from agents you trust") fails because the authenticated agent IS the confabulator. **Read-time skepticism** ("treat memory as suggestive, not authoritative") fails because skepticism without a discriminator is just noise — every claim becomes equally suspect, including the load-bearing ones. **Larger context windows** fail because the failure mode is at write-time, not read-time; a longer context lets you read more confabulations, not fewer.

The correct framing: the memory substrate itself must carry trust-bearing structure. Provenance must be unforgeable from the writing agent's perspective. Contradictions with prior captures must be detected at write-time and gated. Reads must default to quarantine-filtered, with opt-in inclusion of unverified content for callers who can handle the noise. The substrate's contract with its readers is not "everything here is true" but "everything here has known provenance, known verification status, and known contradiction posture; you can decide what to trust based on those signals."

This paper documents the architectural primitives and operational disciplines that emerged from one month of running such a substrate with a multi-agent system. The substrate in question is a single-DB-multiple-agent persistent memory store using PostgreSQL with the `pgvector` extension and an MCP-protocol HTTP API [Anthropic MCP, 2024] — built on an upstream open-source framework [Jones, *Open Brain*, FSL-1.1-MIT, 2026] and extended in private deployment, operated by a handful of specialized AI agents (planning/security/architecture/operations/QA/storage/documentation roles) coordinating across overlapping problem domains. The agents read and write to the substrate continuously; the substrate is single-point-of-failure for the agents' institutional memory; substrate trust is the load-bearing property. The failure mode the paper addresses is named in the literature under several headings — *context poisoning* in practitioner taxonomy [Breunig], *memory poisoning* in adversarial security literature, and *model collapse in a network of LLMs* [Wang et al., arXiv:2506.15690] in formal treatment — but the *accidental* (non-adversarial) propagation case has been less directly mitigated; a recent architecture in the multi-agent shared-memory class is Collaborative Memory [Rezazadeh et al., arXiv:2505.18279], which addresses multi-user memory sharing in LLM agents with dynamic access control.

What follows: four architectural primitives that ship as code (§2), six operational disciplines that ship as discipline (§3), four classes of incident with their per-incident primitive/discipline additions (§4), and four open questions that the substrate doesn't yet answer (§5).

---

## §2 — Architectural primitives

The substrate carries trust-bearing structure as code. Four primitives compose to make the contract enforceable: provenance fields make every write attributable, the quarantine filter makes verification status read-time-visible, contradiction detection gates writes against existing knowledge, and burst detection signals when a writer is generating from cached inference instead of grounded sources. Each primitive is independently auditable; together they constitute the substrate's discriminator.

### §2.1 Provenance fields

Three columns on the canonical `thoughts` table — `captured_by`, `session_id`, `evidence_refs` — turn anonymous text into attributable claims.

`captured_by` is the writing agent's identity. The substrate uses per-agent HTTP API keys; the claim must match the authenticated identity. A validator function runs at the top of every write entry-point: the claim is parsed as `<agent>` or `<agent>-<suffix>`, and the prefix must equal the authenticated agent name. Mismatch is HTTP 403; format violations are HTTP 400. The validator is pure and testable in isolation; it runs before the freeze gate so authentication decisions are higher-priority than substrate-wide write gates.

`session_id` groups captures from one working session. It is a free-form short string (`<agent>-session-NN`) supplied by the caller. The substrate does not validate session continuity — that is the agent's discipline — but it does index on the field, enabling per-session queries during incident response and per-session burst detection.

`evidence_refs` is a JSONB array of objects, each typically `{path, url, note}`. Reference-class captures (`type='reference'`) without evidence_refs are accepted but auto-tagged `verification_status='unverified'` with `drift_flags=['missing_evidence_refs']`. The substrate does not require evidence at write-time; it requires that callers who omit evidence acknowledge the cost in read-time visibility. A reference-class claim with no anchor is, definitionally, a claim the substrate cannot stand behind.

### §2.2 Quarantine filter

The trust contract — "everything here has known verification status" — is enforced at the read layer.

A `verification_status` column carries an enum spanning trusted, unverified, and several flagged states (contradicted / superseded / retracted / contested). Default reads filter to trusted rows; callers opt in to flagged content via an explicit read-time flag.

The default-on posture is load-bearing. Read-time skepticism without a discriminator is noise — every claim becomes equally suspect, including the load-bearing ones. The quarantine filter IS the discriminator: callers who want only-trustworthy content get default reads; callers who can handle noise (audit tooling, dedup recipes, incident response) opt in. The same query surface serves both audiences.

The filter is testable via a permanent canary: a single deliberately-false reference thought, planted with flagged verification status, that must NOT appear in default search results and MUST appear when the caller opts into flagged content. Session-start probes verify the filter is functional. Filter regression is a substrate-wide trust failure; the canary detects it before the next read returns confabulated content.

### §2.3 Contradiction detection

Reference-class writes are gated against existing knowledge at write-time.

The gate runs only on `type='reference'` captures (the type whose explicit purpose is "this is a fact, not an observation"). For each, the substrate computes the embedding and finds the nearest neighbors by cosine similarity. When the maximum similarity exceeds an empirically-tuned threshold AND a cascade-gate condition on the second-place neighbor holds, the substrate calls a separate Natural Language Inference (NLI) service with the candidate-write and the nearest-neighbor as a premise pair.

The NLI service returns one of three verdicts: ENTAILMENT (the candidate restates or refines the premise — no flag), NEUTRAL (semantically related but not in conflict — no flag), or CONTRADICTION (the candidate negates the premise). On CONTRADICTION, the write is blocked unless the caller has supplied `evidence_refs`. Evidence-bearing contradictions are admitted with `verification_status='contested'` (a meta-claim that two grounded references disagree); evidence-free contradictions are rejected.

Threshold values were tuned empirically on the deployed substrate; specific numbers are operational and omitted here. Lower thresholds produced false-positive rejections of legitimate refinement-class writes; higher thresholds let near-duplicate confabulations land. The cascade-gate logic exists because a single-threshold gate fires on benign topic concentration (a dense cluster of related captures will surface high similarity to several neighbors); a multi-condition gate is what made the detector usable in steady-state.

### §2.4 Burst detection

A more subtle anomaly signal: writers generating from cached inference, not grounded sources.

The detector counts, per `session_id` per short sliding window, captures whose content length exceeds a synthesis-class threshold. Beyond a small N in window, each capture is annotated with `drift_flags=['burst']`. Bursts do not block writes; they tag them. The pattern correlates with synthesis-not-source: an agent generating multiple long captures from a single inflection point, rather than grounded short captures from external observation. (Specific window, length, and N values are operational and omitted here.)

Burst-tagged thoughts remain in default reads (the tag is a signal, not a verdict). But the tag flows into incident-response queries, into the audit dashboard, into the "what did this agent write recently" surface. Bursts during an incident are particularly load-bearing — agents under stress generate long synthesis-class captures that, post-hoc, often turn out to be the load-bearing confabulations.

Burst detection's design target is interactive agent sessions; complementary signals cover other write paths.

### §2.5 Composition

The four primitives combine into write and read paths.

The write path: HTTP middleware authenticates the request and decides the gate posture (write endpoints are enumerated in a single function; new write sites require explicit gate-posture decisions, enforced by a CI grep harness). Then a substrate-wide freeze gate (used during incident response). Then the `captured_by` validator. For reference-class writes, the contradiction-detection gate. Then embedding (via a local Ollama model) and metadata extraction (a separate llama3 prompt with downstream `validateMetadata` filtering). Then the burst detector tags the row. Finally the INSERT.

The read path is simpler: authenticate, query, apply the quarantine filter (default-on, opt-out per call), return results.

Each primitive is independently auditable. The CI grep enumerates every INSERT site against an allowlist; adding a new write site without an explicit gate decision fails the build. The freeze gate's enumeration table is a single source of truth in the operator README. The canary probe runs at session-start. The audit dashboard surfaces drift_flags directly. The substrate's trust is not a property of any single primitive — it is the property that emerges when all four are functional and their interactions are observable.

---

## §3 — Operational discipline

Architectural primitives ship as code; operational disciplines ship as habit. Both matter, and they fail differently. The disciplines surfaced not from theory but from incident response — each is grounded in a concrete failure that the primitives alone did not catch. A substrate that ships only the code primitives without the disciplines will accumulate the same failure modes the disciplines were built to prevent.

### §3.1 Canary probe at session-start

Every agent session opens with a search for the canary thought. Default search must return zero results; `include_quarantined: true` must return the canary at high similarity match. If either condition fails, halt.

The discipline arose from a near-miss: a routine audit revealed that the canary thought had `embedding IS NULL`. Every prior session-start probe had been passing — but for the wrong reason. The default search returned no results not because the quarantine filter was working, but because the thought had no embedding to match against. A regression in the quarantine filter would have gone undetected indefinitely. The fix landed as a re-embed of the canary thought through a direct psql `UPDATE`.

The fix was structural: re-embed the canary, then add the `include_quarantined: true` half of the probe. A single check verifying default-empty was insufficient; the probe had to verify both halves of the contract. Embedding-presence checks at session-start became standard.

### §3.2 Post-capture read-back

Important writes are read back by ID immediately after capture; provenance fields (`captured_by`, `session_id`, `evidence_refs`) are verified intact.

The discipline emerged from observing metadata loss across the write path. The metadata extractor (an LLM prompt) is non-deterministic; the JSON parse can fail; the INSERT path can drop fields. A capture that was supposed to land with `topics=['skoll>librarian']` may land with topics inferred from content tokens instead. Without a read-back, the routing tag is lost silently and the receiving agent never sees the message.

Read-back is cheap — one ID query — and it catches metadata drift before that drift becomes audit-trail noise. Treat read-back as a first-class part of the write protocol, not an afterthought.

### §3.3 Ground-truth anchor verification

Temporal claims and load-bearing factual claims must be verified against external ground-truth (git log, filesystem state, command output) — not solely against memory captures.

Memory-recursion produces drift. An agent reads "we shipped X yesterday" from memory, acts on it, captures the action; the next agent reads both the original capture and the action, treats both as mutual-supporting evidence, and the substrate now contains a load-bearing claim grounded only in itself. External anchors break the loop: `git log` shows whether X actually shipped; the file timestamp shows when; the command output shows what.

The discipline applies asymmetrically. Casual recall ("we discussed this earlier") doesn't need anchoring. Load-bearing claims ("the patch landed in commit X on Tuesday") do. The discriminator is whether downstream action depends on the claim being correct.

### §3.4 Generate-from-source-not-prior-synthesis

When an LLM-authored artifact is recurring — re-generated periodically by an agent — generation MUST read original sources, not the agent's own prior synthesis. Self-reference compounds drift; same-source regeneration breaks the loop.

The compounding-error principle has formal treatment in Mohamed et al. [arXiv 2502.20258, ACL 2025], whose *LLM as a Broken Telephone* documents empirically that chained summarization degrades faithfulness with chain length. The classic instance is iterative summarization: each summary read as input to the next produces a sequence whose final output bears no traceable relationship to the original sources. The substrate's instances are subtler. A wiki page synthesizing across captures must re-read captures, not the prior wiki page. A self-summarizer producing per-session digests must read the session's raw captures, not prior digests. An audit synthesizing across audits must read the underlying findings, not the prior audit's framing.

The discipline surfaces in spec design: any pipeline producing a recurring artifact needs an explicit no-self-reference invariant. "What does this read as input?" is the design-time check.

### §3.5 Explicit metadata override

When the metadata extractor demonstrates systematic mis-extraction, callers MUST pass explicit metadata fields to bypass extraction.

The triggering incident was negation-blindness in name extraction. The deployed extractor (an llama3 prompt) auto-extracts capitalized name tokens into a `people` field. Compliance footers in agent captures (for example, `Hard rules clean: no <forbidden-name>`) contain capitalized tokens; the extractor reads them as person-mentions. The negation is invisible. Over weeks, the corpus accumulated false-extraction artifacts; each new capture with the standard footer added another. Audit queries against the `people` field returned a long tail of mentions that were definitionally not real references.

Two complementary fixes: a backend denylist filter in the validator (deterministic, version-stable, mechanical), and an agent-side discipline of passing explicit `people: [...]` arrays on captures with the standard footer. The denylist catches the artifact. The discipline catches anything the denylist doesn't yet know about. Defense-in-depth, because the failure mode is open-ended (any new compliance phrase containing a capitalized token).

### §3.6 Pre-pause inbox sweep + rescan-before-blocked

Before declaring blocked or pausing for direction, sweep external inputs by routing tag (not semantic search). Rescan when revisiting a previously-blocked work-item.

The discipline addresses a coordination failure mode: an agent decides "I'm blocked, standing by," not realizing a peer agent has just routed an unblock. The inbox sweep is mechanical — list captures by routing tag (e.g. `manager>librarian`) within the last day — not semantic, because semantic search rewards content-relevance, not coordination-relevance. The rescan-before-blocked extension catches stale-state on any work-item that has been blocked across sessions or across long time gaps.

The pattern is generalizable. Any persistent-state system used by multiple writers needs a discipline for "did anyone touch this since I last looked." Polling-by-routing is the cheap version; event-driven is the durable version. The substrate currently runs the cheap version.

### §3.7 Why discipline cannot be replaced by primitives

A natural reaction: surely the disciplines should ship as automation? The substrate could enforce read-back, automate canary checks, gate writes that lack ground-truth anchors. Some of this is ongoing work. But the disciplines that resist automation are the ones that depend on judgment about what is load-bearing — which claim warrants ground-truth anchoring, which artifact is recurring synthesis, which capture's footer needs explicit override.

Substrates can carry trust-bearing structure. They cannot carry trust-bearing judgment. The disciplines exist because trust is not a property the substrate can confer; it is a property an attentive operator confers, with the substrate's help.

---

## §4 — Empirical evidence

The substrate matures by accumulating responses to incidents. Each of the four classes below shaped the architecture or the discipline. Each is grounded in a concrete failure; none are hypothetical. The pattern across them is visible only after several: failures recur in different shapes, but the response-shape — primitive-or-discipline addition — generalizes.

### §4.1 Confabulation propagation (the founding incident)

The substrate's quality-assurance stack exists because of one early incident. Within two weeks of multi-agent operation, a confabulation propagation pattern surfaced. One agent captured a false claim about a piece of recent infrastructure state; a second agent read the capture, treated it as context for morning work, and acted on the false premise; the second agent then made her own false attribution about a peer's recent operations; that attribution was contradicted by ground-truth verification (git history showed zero such activity). Three captures, two false claims, one near-cascade — caught only because an agent ran external verification rather than re-reading the substrate.

The response was a substrate-wide freeze. Reads stayed open; writes were gated. Over the subsequent sessions, the quality-assurance stack was built: provenance fields so every claim was attributable to a specific writer in a specific session; the quarantine filter so reads could exclude unverified content; evidence_refs and per-type verification policy so reference-class claims had to be anchored or were auto-tagged unverified; burst detection so synthesis-fan-out patterns were tagged; and contradiction detection so reference-class writes were gated against existing knowledge. The freeze lifted only after five validation gates passed: canary integrity, provenance round-trip, quarantine filter behavior, contradiction-detection threshold tuning, and burst-detector tuning.

The lesson was structural, not tactical. The substrate's original design treated trust as a property of the writing agent; the freeze response moved trust to the substrate's read-time output. The four primitives in §2 are the residue of that move. Without the founding incident, none of them would exist; the substrate would still trust writers and fail at scale.

### §4.2 Canary integrity false-positive (the silent regression near-miss)

Some weeks after the freeze lift, a routine audit revealed that the canary thought — the permanent flagged-status sentinel that session-start probes search for — had `embedding IS NULL`. Every session-start probe since the canary was seeded had been passing. But the passes were for the wrong reason. The default search returned no results not because the quarantine filter was excluding the canary, but because the canary had no embedding to match against. A regression in the quarantine filter would have been undetectable by the existing probe.

The fix was simple in code (re-embed the canary, add the `include_quarantined: true` half of the probe) and load-bearing in discipline. The canary probe protocol was strengthened: both halves of the contract must be verified at session-start. A probe that verifies only the negative half (default-empty) is insufficient because there are multiple causes of default-empty, only one of which is the filter actually working.

The general lesson: substrate-level invariants need probes that cannot pass for the wrong reason. If the probe's success is consistent with multiple states of the substrate, the probe is testing the wrong property. Re-design probes to be sensitive to the specific invariant they protect.

### §4.3 Cluster wipe and the platform-update collision

A platform update revealed a different class of failure. The substrate runs on a Kubernetes cluster on a workstation-class operator deployment. A version bump introduced a new prerequisite: a host-OS service the host operator had deliberately disabled as part of a hardening posture (the service is unrelated to runtime container operations). The update failed at the prerequisite check. The Kubernetes integration crashed silently; the kubeconfig file was truncated to a stub. When the operator re-enabled Kubernetes through the platform's management UI, the toggle *reset* the cluster — wiping the substrate's namespace, statefulset, and persistent volume claim in a single click.

Recovery took place through the out-of-band backup pipeline. An out-of-band database snapshot stored off-host had landed cleanly; a restore brought the substrate back to a prior-known-good state. A bounded window of captures was lost; most of that window had been quiet park-time across agents, so the data loss was bounded. Recovery disciplines — toggle-reset behavior, vault-driven secret recreation, and platform-update routing through a coordination layer — were documented as operational lessons.

Three architectural lessons emerged. First: vendor updates colliding with deliberate hardening is a recurring class, not a one-off. The class generalizes — Windows Update KB additions, defender platform updates, runtime version bumps — any of these can introduce prerequisites that conflict with hardening. The mitigation is process: vendor-update routing through a coordination layer, with a hardening-posture registry checked against vendor prerequisite changes before approval.

Second: backup-restore-as-cluster-recovery worked, but the soak clock should not have reset. The restored cluster was code-equivalent and key-equivalent to the pre-incident state; trust accumulated by the substrate during the prior soak should carry forward. A discipline emerged: the soak clock resets only when code or keys change.

Third: post-incident soak posture became a substrate-level invariant. After any restore, the post-restore state must be observed for at least one full operational cycle before substrate-trust assertions resume. The discipline exists because trust is not preserved across opaque transitions, even when the data is.

### §4.4 Metadata extractor negation-blindness

The most recent class is the most pedagogical because the substrate's existing primitives almost — but not quite — caught it.

The substrate's metadata extractor (a local LLM prompt with downstream `validateMetadata` filtering) auto-populates a `people` field on every capture. The agents' captures end with a compliance footer that contains capitalized tokens denoting prohibited entities. The extractor reads those tokens as person-mentions. The negation is invisible to the model. Over weeks, the corpus accumulated tens of false-extraction artifacts; one heavy-traffic day saw the rate of new artifacts more than double. Audit queries against the `people` field returned a long tail of mentions that were definitionally not real references.

The existing primitives caught some of this. Quarantine filter: irrelevant — the captures' content was correct, only their metadata was contaminated. Contradiction detection: irrelevant — the artifacts were not reference-class. Burst detection: tagged some sessions but not the underlying pattern. Provenance: showed that every artifact came from a captured-by agent passing the standard footer — diagnostic, but not preventive.

The fix is three-phase: a backend denylist filter in the validator (forward-fix, ~10 lines plus four unit tests), a SQL backfill of the existing artifacts with an audit-trail drift_flag annotation, and a company-wide standing rule that outbound captures with the standard compliance footer pass explicit `people: [...]` arrays on `capture_thought`. The forward-fix is mechanical; the backfill closes the existing tail; the standing rule is defense-in-depth because the failure mode is open-ended — any new compliance phrase containing a capitalized token would surface the same pattern.

The lesson: when an LLM-mediated component shows systematic failure, the substrate adds a deterministic filter rather than relying on prompt-engineering the LLM to do better. Prompt fixes are version-unstable; filters are version-stable. Denylist content is small; extension cadence is as-needed via add-only ratification.

---

## §5 — Open questions

The substrate works well enough to operate. It does not solve everything. Four classes of open question remain — each genuinely open, each surfaced by the operational use of the substrate, none paperable-over with hand-waving toward future work.

### §5.1 Evolution-vs-drift in self-summarizing agents

The agents carry self-state in structured documents — identity anchors, voice norms, current commitments, relationship state with peers. Some of those fields are by-design-evolutionary: an agent's active commitments change session by session; its relationship state with peers deepens with time. Some fields are anchored: voice and identity should not drift across sessions absent deliberate change.

The open question is the discriminator. How does the agent know — and how does the substrate help the agent know — whether an observed change in voice is honest evolution (legitimate accumulation of experience) or drift (style shift toward a default-LLM voice, away from the agent's anchor)? Drift detection is in early piloting; the discriminator is the hard part. Self-checking by the same model is self-consistency, not adversarial. Cross-model checking (a different model substrate as second reviewer) is expensive and may itself be inconsistent. The honest answer at present is that the discipline depends on operator pattern-watching; the substrate has no first-class way to distinguish evolution from drift automatically.

### §5.2 Adversarial independence in arch-coherence review

When agents review each other's work — architectural fit, security posture, code correctness — the reviewers run on the same model substrate as the writers. The review provides value (different role conditioning produces different attention) but the value is bounded by substrate-shared blind spots. Both reviewer and writer fail in the same ways under the same model versions.

Heterogeneous-source review — a different model family as second reviewer — would be the load-bearing fix. The cost is non-trivial: heterogeneous-source review requires running multiple model substrates, each with its own provisioning, prompts, evaluation criteria. The benefit is independent-failure-mode coverage. The substrate is well-positioned for this — the provenance fields would show which substrate produced which review — but the engineering remains.

### §5.3 Substrate single-point-of-failure for institutional memory

The substrate is the agents' institutional memory. When the substrate is unreachable, every parked agent loses recall. This was observed during a six-hour outage early in the deployment: agents that had previously built and deployed components could not, during the outage, recall the components they had built. The fix-in-progress is a per-agent on-disk resilience layer — agent-of-record documents that survive substrate outages and carry the load-bearing state forward.

The full minimum-viable replication strategy is open. Local read-through caches help with availability but not with consistency. Multi-region replication helps with availability and consistency but introduces split-brain risk. The right balance for an interactive multi-agent system is not yet clear; the practitioner-grade answer is "the substrate is best-effort durable, agents carry on-disk fallback for outage windows, and the resilience primitives are work-in-progress."

### §5.4 Threat-model framing

Confabulation propagation is, in threat-modeling terms, the behavior of a substrate whose writers have degraded judgment but well-meaning intent. The substrate's primitives were designed for that case. A separate threat-model — substrate-aware adversarial writers — is a different problem with different defenses; for the agent system documented here, the adversarial threat-model is forward-work rather than current-architecture. A paper specifically on the adversarial case would deserve its own treatment, both because the defenses differ in kind and because publishing specific bypass paths against a deployed substrate is itself a security concern.

---

## §A — Primitive interaction diagrams

The four primitives in §2 compose at write-time and read-time as follows. Solid arrows are unconditional flow; dashed arrows are conditional gates that may reject or annotate.

### §A.1 Write path

```text
HTTP request (POST /capture or capture_thought MCP tool)
  │
  ▼
[Middleware: authenticate + isWriteEndpoint gate]
  │
  ▼
[Substrate-wide freeze gate]  ── if frozen ─▶ HTTP 503 (reads stay open)
  │
  ▼
[captured_by validator]  ── 400 malformed / 403 mismatch ─▶ reject
  │
  ▼
[type='reference' branch?] ─ no ─┐
  │ yes                          │
  ▼                              │
[Embedding lookup of nearest neighbors]
  │
  ▼
[Cosine + cascade gate triggered?] ─ no ─┐
  │ yes                                  │
  ▼                                      │
[NLI service: ENTAILMENT/NEUTRAL/CONTRADICTION]
  │                                      │
  ├── CONTRADICTION + no evidence_refs ──▶ reject (HTTP 409)
  ├── CONTRADICTION + evidence_refs ────▶ admit as verification_status='contested'
  └── ENTAILMENT/NEUTRAL ───────────────▶ admit normally
                                         │
                                         ▼
[Embedding generation (if not cached)]
  │
  ▼
[Metadata extraction (LLM prompt + validateMetadata filter)]
  │
  ▼
[Burst-detector tag: drift_flags=['burst'] if pattern matches]
  │
  ▼
[INSERT into thoughts]
  │
  ▼
HTTP 200 (echo capture id)
```

The gate ordering is load-bearing. Authentication runs before the freeze gate so 400/403 validation errors are independent of substrate-wide write-disable; this avoids weaponizing freeze for attacker probing. Reference-class contradiction detection runs before metadata extraction because rejected writes shouldn't burn LLM calls. Burst detection runs after metadata extraction because the tag flows into metadata.

Publishing the full write-path is itself a tradeoff — the same disclosure that lets practitioners reason about composition gives an adversary an attack-chain map. The substrate's defense against this is not obscurity of the pipeline but defense-in-depth of each gate (per-gate auditability, independent test surface, and version-stable filters where possible). The contribution-cost of pipeline obscurity exceeds its security gain in a deployment of this size.

### §A.2 Read path

```text
HTTP request (GET /search, /list, get_thought, MCP read tools)
  │
  ▼
[Middleware: authenticate]
  │
  ▼
[Query construction + WHERE clause]
  │
  ▼
[Quarantine filter applied? (default ON; opt-out via include_quarantined flag)]
  │
  ├── default ─▶ filter to trusted rows
  └── opt-in ──▶ include flagged rows (for audit / dedup / incident-response callers)
  │
  ▼
[Result rows: provenance fields preserved verbatim from capture-time]
  │
  ▼
HTTP 200 (rows + metadata)
```

Reads are intentionally simple. The trust contract — every row carries known provenance, known verification status, known contradiction posture — is enforced at write-time; reads consume the contract without re-evaluating it.

### §A.3 Failure-mode mapping

The four primitives map to specific failure modes. Each primitive's protective surface is bounded; the disciplines in §3 cover the gaps:

| Failure mode | Primitive that catches it | Discipline backstop |
| --- | --- | --- |
| Anonymous claim with no writer attribution | Provenance fields (writer auth + claim binding) | — |
| Reading flagged content as trusted | Quarantine filter (default-on) | Canary probe at session-start |
| Reference-class write contradicting prior knowledge | Contradiction detection (NLI gate) | Ground-truth anchor verification on load-bearing claims |
| Synthesis fan-out from cached inference | Burst detection (tagging signal) | Generate-from-source-not-prior-synthesis |
| Negation-blind metadata extraction | (none — this was the failure that drove §3.5) | Explicit metadata override + backend denylist |
| Cross-session coordination drift | (none — substrate is passive on this) | Pre-pause inbox sweep + rescan-before-blocked |

The empty cells in the "Primitive that catches it" column are not gaps in the substrate's design; they are deliberate scope-bounds. The substrate provides infrastructure for trust; the operator and the agents provide judgment about what is load-bearing.

---

## Acknowledgments

This work builds on Open Brain, an open-source persistent memory framework by Nate B. Jones, used under the Functional Source License v1.1 (MIT Future). The substrate documented here is a private deployment of that framework, extended with the architectural primitives and operational disciplines described above; the upstream codebase provides the foundational schema, the MCP-protocol surface, and the capture-time architecture on which those extensions rest. The substrate further depends on PostgreSQL with the `pgvector` extension, the Hono HTTP framework on Deno, the Anthropic Model Context Protocol, and Ollama for local model serving — each load-bearing.

---

## References

### External literature

- Wang, T., Horiguchi, A., Pang, L., and Priebe, C. E. *LLM Web Dynamics: tracing model collapse in a network of LLMs.* arXiv:2506.15690 (2025).
- Breunig, Drew. *Context poisoning* — practitioner taxonomy.
- Dhuliawala et al. *Chain-of-Verification reduces hallucination in large language models.* arXiv:2309.11495.
- Farquhar et al. *Detecting hallucinations in large language models using semantic entropy.* Nature, 2024.
- Jones, Nate B. *Open Brain* — open-source persistent memory framework. Released 2026 under the Functional Source License v1.1, MIT Future (FSL-1.1-MIT).
- Mohamed et al. *LLM as a broken telephone: iterative generation distorts information.* arXiv:2502.20258 (ACL 2025).
- Nussbaum et al. *Nomic Embed: training a reproducible long context text embedder.* 2024.
- Wang, X., Wei, J., Schuurmans, D., Le, Q., Chi, E., Narang, S., Chowdhery, A., and Zhou, D. *Self-Consistency Improves Chain of Thought Reasoning in Language Models.* arXiv:2203.11171 (2022); published at ICLR 2023.
- Rezazadeh, A., Li, Z., Lou, A., Zhao, Y., Wei, W., and Bao, Y. *Collaborative Memory: multi-user memory sharing in LLM agents with dynamic access control.* arXiv:2505.18279 (2025).

### Software / dependencies

- PostgreSQL with the `pgvector` extension for 768-dim cosine-similarity search.
- Hono (Deno) HTTP framework for the MCP-protocol surface.
- Anthropic Model Context Protocol (MCP) — JSON-RPC contract for tool/resource exposure (2024).
- Ollama for local model serving — `nomic-embed-text` (768-dim embeddings) and `llama3` (metadata extraction).
- A separate NLI service (DeBERTa-v3-large MNLI head) for contradiction-detection entailment classification.
