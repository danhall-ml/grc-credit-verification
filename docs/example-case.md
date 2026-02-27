# Credit Verification Pipeline Design

## Objective

The easiest way to explain this system is to follow one ranch through one reporting period.

At the end of a period, GRC needs to answer a simple question: did this ranch meet the requirements for credit issuance, and if so, at what tier? In the current state, this answer is assembled by hand from several systems. We need to construct a proposed pipeline that turns that manual process into a repeatable flow that still keeps people in control of final decisions.

For each ranch and period, we would like for the system to gather evidence, check whether the evidence is complete enough to evaluate, recommend a Good/Better/Best tier, and then send the case to review when needed. The output is a final verification record that explains what happened, why it happened, and who approved it.

## Architecture

The most important factor in the design of this system is consistency. If the same data and same rule version are used, the result should not change. Every recommendation ought to point to specific evidence rather than broad summaries. Missing or conflicting data should not be silently ignored; it should be made explicit and routed for human review. Reruns should be safe, and older evidence should remain intact so a decision can be reconstructed later.

The architecture follows the same path a reviewer already follows mentally, but in a structured way. The main difference is that the pipeline has to deal with a real-world constraint that humans handle implicitly: the evidence is fragmented across formats and systems, and it is rarely cleanly aligned on identifiers, dates, and boundaries.

First, source data is collected from PastureMap, Salesforce, soil lab systems, hardware feeds, and uploaded documents like GMP and LSA. Before we can even talk about tier logic, we need to be clear about what “collecting evidence” means in practice, because the shape of that data drives the rest of the system.

The hard part is that the evidence comes in different forms: PDFs, map files (GeoJSON), and rows from systems like Salesforce, labs, PastureMap, and hardware. So the first step is simply saving an exact copy of whatever we receive each time we pull it, so we can always show what we used later. For PDFs we only record a few basic facts we need to make decisions (which ranch it applies to, what dates it covers, and for LSA the parcel list and acres), and the rest is left for the reviewer to read. The map files are used for simple location checks: soil samples and paddocks should fall within the land listed in the LSA. If any of these pieces don’t line up—names/IDs don’t match, dates don’t match, or locations fall outside the listed land—the case is flagged for human review instead of being auto-approved.

Once that raw evidence is saved, the next steps are about turning it into a stable “case” that can be evaluated the same way every time.

Second, those snapshots need normalized into one internal structure. This is where the system resolves mismatched source identifiers, links records to one stable `ranch_id`, and creates a consistent set of evidence records.

Third, the pipeline has to run checks for consistency and completeness. These checks should not decide tier directly. They should instead answer whether the case is safe to evaluate automatically. If a blocking check fails, the case ought to be marked for review with explicit reasons.

Fourth, if the case is complete enough, the tier engine should evaluate rule claims and recommend the highest satisfied tier. It should also record which claims passed, which failed, and why.

## Checks and Tiering

In practice, most painful verification issues are completeness issues, not math issues. That is why this stage is treated as its own gate.

The system checks whether LSA data exists and is usable, whether boundary geometry is valid, whether GMP is present and period-relevant, whether soil sampling evidence is complete, whether Better-tier activity signals are present, and whether Best-tier hardware coverage is sufficient.

When these checks fail in a blocking way, the engine does not force a recommendation. It routes the case to reviewers with a plain explanation of what is missing or conflicting.

Tiering is cumulative by design. Good is foundational, Better builds on Good, and Best builds on Better. The engine assigns the highest tier whose full claim set is satisfied by current evidence.

If there are unresolved conflicts across systems, the recommendation is still generated, but flagged for review rather than silently finalized. The goal is to keep automation useful without pretending it can settle ambiguous evidence on its own.

Every evaluation stores the rule version, rule hash, claim outcomes, evidence references, and rationale text so the decision can be replayed later.

## Verification Record and Audit

The Ranch Verification Record is the final artifact for a ranch-period decision. It includes identity and period fields, protocol fields, automated recommendation, final reviewer outcome, completeness results, claim results, conflicts, evidence references, provenance metadata, and a full audit log.

The state moves from draft to review-ready, then to a final review state such as approved, rejected, requested-info, or approved-with-override. Finalization does not rewrite history; new events are appended.

If an external reviewer asks, "Why was this ranch approved?", the system should answer without manual reconstruction. The record shows the decision, the evidence used, the exact rule version, and the human actions taken.

Because snapshots are immutable and the rule version is pinned, the same case can be replayed months later with the same inputs.

## Review Workflow

Reviewers should receive the case with all automated findings attached. They then approve, reject, request more info, or override. Their actions should be saved as append-only audit events.

The review experience should feel like a case file. A reviewer should open one page and immediately see: recommended tier, blocking checks, conflicts, and linked evidence. From there, the reviewer takes one action: approve, reject, request info, or override. Overrides require a reason code and note. That is enforced by workflow and schema so exceptions are explainable later.

## Edge Cases and Operations

A few edge cases matter enough to define immediately.

When source data is partial, the case is held for review rather than auto-passed. When systems disagree, the conflict is surfaced explicitly and requires adjudication. When tier signals change mid-period, the record keeps both timing context and final reviewer interpretation. When documents arrive late, the original state remains visible and the late evidence is added with timestamped review action.

Policy owners control the rule definitions and thresholds. Platform/ML Ops controls connectors, orchestration, storage, deployment, and monitoring. Changes to rules are tested on fixture cases, rolled out in stages, and promoted once behavior is confirmed.