---
title: 'A Trust Substrate for LLM Memory'
description: 'When you give multiple LLM agents a shared memory store, you also give one agent the power to make its hallucinations everyone else''s ground truth. Here is what that looks like, and what to do about it.'
pubDate: 'Apr 28 2026'
---

A few weeks into running a small fleet of specialized AI agents on a shared memory store, I watched one agent's confabulation become a near-cascade.

Agent A, mid-session, captured a claim about some piece of recent infrastructure state. Agent B opened a session the next morning, read the capture as part of its standard context-loading sweep, and started acting on the false premise. Agent B then made *its own* false attribution about a peer agent's recent work, in turn captured. Two false captures, mutually-supporting, sitting in the corpus like ground truth.

The cascade caught only because agent B happened to verify one of the underlying claims against `git log` rather than re-reading memory. The git history showed nothing — no commits, no branches, no nothing — that matched the claim. The whole chain unwound in twenty minutes.

But the architectural lesson took weeks. **An LLM cannot, in general, distinguish its own past confabulation from grounded recall when it reads from a shared memory store.** The agent that wrote the bad capture had context the reading agent doesn't. If the writing agent was wrong — about a date, a name, a sequence of events, a reference it never actually saw — the reading agent has no signal to challenge it. Compounded across sessions, false claims become load-bearing: agents act on them, cite them, build on them. The corpus develops a pile of mutually-supporting fictions all tracing back to one bad write.

This is the **trust substrate problem**, and it is not solved by trying harder. The naive fixes don't work:

- **Better-trusted writers.** The authenticated writer IS the confabulator. Author authentication doesn't help because nobody is impersonating anyone.
- **Read-time skepticism.** Treating every memory as suggestive rather than authoritative produces noise, not discrimination — every claim becomes equally suspect, including the ones the system actually depends on.
- **Bigger context windows.** The failure is at write-time. Larger contexts let agents read more confabulations, not fewer.

The correct framing: **the memory substrate itself has to carry trust-bearing structure.** Not "everything here is true" — that's unachievable. Rather: "everything here has known provenance, known verification status, and known contradiction posture; you can decide what to trust based on those signals."

We've been operating a substrate built on this principle — Open Brain, the open-source persistent-memory framework — for about a month with a multi-agent system. The architecture that emerged sits on four primitives and six disciplines, with each piece traceable to a specific failure that drove its addition.

## The four primitives

**Provenance fields.** Every captured thought carries the writer's identity (validated against authenticated credentials), the working session it came from, and a list of evidence anchors for any reference-class claim. A reference-class claim with no anchor gets auto-tagged as unverified. The contract isn't that captures must be sourced; it's that callers who omit sources acknowledge the cost in read-time visibility.

**Quarantine filter.** A verification-status enum (`trusted`, `unverified`, `contradicted`, `superseded`, `retracted`, `contested`) on every row. Default reads filter to trusted; callers explicitly opt in to flagged content for audit and incident-response work. The default-on posture is load-bearing — it gives readers a discriminator instead of generic skepticism.

**Contradiction detection.** Reference-class writes are gated against existing knowledge at write-time. The substrate finds the candidate's nearest neighbors by cosine similarity; if a high-similarity neighbor exists *and* the cluster isn't dense enough to produce a false positive, a separate Natural Language Inference service classifies the relationship as ENTAILMENT, NEUTRAL, or CONTRADICTION. Contradiction without evidence is rejected; contradiction with evidence is admitted as `contested`, signaling that two grounded references disagree.

**Burst detection.** A subtle anomaly signal. When an agent generates many long captures inside a short window — synthesis fan-out from a single inflection point — the captures get tagged. Tags don't block writes; they flow into incident-response queries. Bursts during incidents are particularly load-bearing; agents under stress produce long synthesis-class captures that, post-hoc, often turn out to be the load-bearing confabulations.

These four primitives compose at write-time: authenticate, freeze-gate, validate identity, contradiction-check (for references), embed, extract metadata, burst-tag, INSERT. Reads consume the contract simply: authenticate, query, apply quarantine filter (default on), return.

## The six disciplines

Architectural primitives ship as code. Operational disciplines ship as habit, and turn out to be just as load-bearing.

- **Canary probe at session-start.** Every session opens with a search for a permanent deliberately-false reference thought, planted with flagged status. Default search must NOT return it; flagged-content opt-in must return it. Both halves of the contract verified, every session, before the substrate is trusted.
- **Post-capture read-back.** Important writes get re-read by ID immediately after capture. Provenance fields verified intact. Cheap insurance against extractor-layer drift.
- **Ground-truth anchor verification.** Load-bearing temporal claims get verified against external sources — git, filesystem, command output — not against memory. Memory-recursion produces drift; external anchors break the loop.
- **Generate-from-source-not-prior-synthesis.** Recurring artifacts (digests, summaries, wiki pages) regenerate from raw sources, never from the agent's own prior output. Karpathy's compounding-error principle, generalized.
- **Explicit metadata override.** When the LLM-driven metadata extractor shows systematic failure modes (say, reading "no NateBJones" in compliance footers as a person-mention because negation is invisible to the model), callers pass explicit fields to bypass extraction. Defense in depth with the backend deny-list.
- **Pre-pause inbox sweep + rescan-before-blocked.** Before declaring blocked or pausing, sweep external inputs by routing tag. Rescan when revisiting any work-item that's been blocked across sessions or long time gaps.

The disciplines are the gaps the primitives can't cover by themselves — judgment about what is load-bearing, attention to what changed since the last look. Substrates can carry trust-bearing structure. They cannot carry trust-bearing judgment. The disciplines exist because trust is conferred by an attentive operator with the substrate's help, not by the substrate alone.

## Two small incidents

The architecture above didn't arrive whole. It accumulated.

The **founding incident** I described at the top drove the original quality-assurance build-out: provenance fields, quarantine filter, contradiction detection, burst detection — none of those existed before agents were observed mutually-supporting each other's hallucinations. The substrate's design shifted from "trust the writer" to "make trust legible at read-time," and the four primitives are the residue of that shift.

A different class surfaced months later when a routine **platform-update collision** wiped the production cluster. A container-runtime version bump introduced a prerequisite the host operator had deliberately disabled as a hardening posture. The update failed silently; re-enabling Kubernetes through the runtime's UI reset the cluster — wiping the namespace, statefulset, and persistent volume in a single click. Recovery was clean (the prior-night backup had landed cleanly), but the lesson was structural: **vendor updates colliding with deliberate hardening is a recurring class, not a one-off.** The discipline that came out of it was process, not code: vendor-update routing through a coordination layer, with hardening posture checked against vendor prerequisites *before* approval. Sometimes the substrate's trustworthiness depends on what doesn't get clicked.

## What the substrate doesn't yet solve

Honesty about gaps:

- **Evolution vs drift.** Agents carry self-state in structured documents — voice, identity, current commitments. Some of those fields are by-design-evolutionary (commitments change; relationships deepen). Some are anchored. The substrate has no first-class way to distinguish honest evolution from drift toward a default-LLM voice. The discipline depends on operator pattern-watching for now.
- **Adversarial review independence.** When agents review each other's work, both reviewer and writer run on the same model substrate — same blind spots, same failure modes under the same versions. Heterogeneous-source review (a different model family as second reviewer) would be the load-bearing fix; the engineering remains.
- **Single-point-of-failure for institutional memory.** When the substrate is unreachable, every parked agent loses recall. A per-agent on-disk resilience layer is in progress; the right replication strategy for an interactive multi-agent system is still open.
- **Adversarial threat model.** The architecture above assumes well-meaning writers with degraded judgment. A substrate-aware adversary is a different problem with different defenses (forge-resistant provenance, randomized gate parameters, audit independence). For local trusted deployments, that's forward-work; for less-trusted ones, it would deserve its own treatment.

## Why this matters

Multi-agent systems are getting easier to build. Persistent memory for LLM agents is getting cheaper and more capable. The combination produces something useful — agents that accumulate context across sessions, share knowledge, build on each other's work.

It also produces a failure mode that doesn't show up in single-agent benchmarks. One agent's confabulation can become institutional fact for everyone else, with no signal at any individual capture that anything is wrong. The substrate's trust contract is what discriminates "the system actually knows this" from "an agent once said this."

The architectural and operational shape that worked for us isn't novel; it's an accumulation of responses to specific incidents, ground in specific failures. Other systems will have different incidents and need different responses. But the underlying observation generalizes: **shared memory between LLM agents is only as trustworthy as the substrate's read-time contract, and that contract has to be built — code AND habit — not assumed.**

A longer companion paper covers the architecture and incident details in more depth.
