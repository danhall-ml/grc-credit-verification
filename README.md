# GRC Credit Verification

Part 2 repository for the GRC assignment: Credit Verification Pipeline Design.

This document is the main design narrative. The dataflow diagram remains separate in `docs/dataflow.md`.

## What This Repo Contains

- Design narrative and operating model (this README)
- Dataflow diagram (`docs/dataflow.md`)
- Verification record schema (`schemas/ranch_verification_record.schema.yaml`)
- Example verification record (`schemas/ranch_verification_record.example.json`)

## Design Objective

The easiest way to explain this system is to follow one ranch through one reporting period.

At the end of a period, GRC needs to answer a simple question: did this ranch meet the requirements for credit issuance, and if so, at what tier? In the current state, this answer is assembled by hand from several systems. The goal here is to turn that manual process into a repeatable flow that still keeps people in control of final decisions.

For each ranch and period, the system gathers evidence, checks whether the evidence is complete enough to evaluate, recommends a Good/Better/Best tier, and routes the case to review when needed. The output is a final verification record that explains what happened, why it happened, and who approved it.

## Architecture and Flow

The core requirement is consistency. If the same data and same rule version are used, the result should not change. Every recommendation points to specific evidence rather than broad summaries. Missing or conflicting data is made explicit and routed for human review. Reruns are safe, and older evidence remains intact so a decision can be reconstructed later.

The pipeline follows the same path a reviewer already follows mentally, but in a structured sequence:

1. Collect source data from PastureMap, Salesforce, soil lab systems, hardware feeds, and uploaded documents (GMP/LSA).
2. Save raw evidence snapshots without overwrite so the exact inputs are preserved.
3. Normalize snapshots into one internal case structure with stable `ranch_id` mapping.
4. Run consistency and completeness checks to decide whether auto-evaluation is allowed.
5. Evaluate tier claims and recommend the highest satisfied tier.
6. Route to human review for approval/rejection/override, and store the final verification record.

The hard part is not tier math. The hard part is evidence quality across mixed formats. Some evidence is tabular, some is geospatial, and some is embedded in PDFs. Because of that, the system handles extraction and reconciliation first, then decision logic.

## Document Handling and Reconciliation

One of the main technical problems in this system is that the evidence does not arrive in one clean format. Important facts are spread across PDFs, GeoJSON files, and source-system records, and those sources do not necessarily agree on identifiers, dates, parcel definitions, or naming. That means the hard work happens before tier logic even starts.

In practice, the system needs to do two things well. First, it needs to extract the small set of facts that actually matter for verification. Second, it needs to merge those facts into one stable ranch-period case without pretending the source data is cleaner than it really is.

PDFs are especially difficult because key facts often live inside tables, scans, or inconsistent layouts rather than in clean structured fields. In practice, that means the system may only be able to extract a small set of decision-relevant facts automatically and leave the rest for a reviewer to read directly. GeoJSON and map-based data introduce a different problem: location checks have to line up with the land listed in the documents. Source systems add another layer of complexity because the same ranch or parcel may appear under slightly different names or IDs. If those pieces do not align, the system should not silently smooth that over. It should surface the mismatch and route the case for human review.

That is why the design treats evidence capture, extraction, and reconciliation as first-class parts of the pipeline. The goal is not just to collect files. The goal is to turn messy evidence into a stable case that can be evaluated consistently and explained later.

## Example Case Walkthrough

For one ranch-period case, the system pulls source snapshots and stores them as immutable evidence. It then resolves ranch identity, aligns period coverage, and checks location consistency between map boundaries, soil samples, and listed parcels.

If identifiers, dates, and geometry all align, the case moves to tier evaluation. If they do not align, the case is flagged for review with explicit reasons.

When the case is evaluable, the engine computes a tier recommendation and records:

- which claims passed,
- which claims failed,
- which evidence supported each claim,
- which rule version produced the recommendation.

The reviewer then finalizes the case (approve, reject, request info, or override). That final action is stored as an append-only audit event.

## Checks and Tiering

Most painful verification issues are completeness issues, not math issues. That is why checks are a separate gate.

The system checks whether:

- LSA data exists and is usable,
- boundary geometry is valid,
- GMP is present and relevant to the period,
- soil sampling evidence is complete,
- Better-tier activity signals are present,
- Best-tier hardware coverage is sufficient.

When blocking checks fail, the engine does not force a recommendation. It routes the case to review with a plain explanation of what is missing or conflicting.

Tiering is cumulative by design: Good is foundational, Better builds on Good, and Best builds on Better. The engine assigns the highest tier whose full claim set is satisfied by current evidence.

## Verification Record and Audit

The Ranch Verification Record is the final artifact for a ranch-period decision. It includes identity and period fields, protocol fields, automated recommendation, final reviewer outcome, completeness results, claim results, conflicts, evidence references, provenance metadata, and a full audit log.

The state moves from draft to review-ready, then to a final review state such as approved, rejected, requested-info, or approved-with-override. Finalization does not rewrite history; new events are appended.

If someone asks months later why a ranch was approved, the record can answer directly because it stores the decision, the evidence used, the exact rule version, and the human actions taken.

## Tooling and Storage

Raw evidence (PDFs, GeoJSON, source exports) should be stored in immutable object storage paths so prior pulls are never overwritten.

Normalized records, verification records, and identifier mappings should be stored in a relational database. Geospatial support is important because parcel and sample-location checks are central.

Pipeline execution can start with a scheduled container task. If workflow complexity grows, it can move to a full orchestrator later. Reviewers use a small internal application backed by the same database, with evidence links and append-only reviewer actions.

One concrete AWS-oriented implementation would look like this.

Raw evidence such as PDFs, GeoJSON, and source exports would be stored in S3. Each pull would write to a new time-stamped prefix so prior evidence is never overwritten.

Normalized records, verification records, and identifier mappings would live in Postgres on RDS, with PostGIS enabled to store parcel boundaries, paddocks, and sample locations and to run containment checks directly.

Pipeline execution could initially be handled by an ECS scheduled task that runs periodically. That job would pull source data, write raw snapshots to S3, update normalized records in RDS, and evaluate any ranch-period combinations that are ready for verification. If the workflow grows more complex over time, this could later move into Airflow on MWAA, but that is not required for an initial implementation.

Reviewers would interact with a small internal application, for example a service running on ECS, backed by the same RDS database. The UI would display the verification record, link out to S3 evidence, and write reviewer actions as append-only audit events.

## Additional Thoughts on Document Extraction

One thing that becomes clear when designing a system like this is that the real difficulty is often not in the verification rules themselves. The harder problem is getting the evidence into a usable shape, especially when important details are buried in PDFs or spread across multiple systems.

Because of that, I think it is reasonable to consider a bounded use of an LLM for document extraction. The role would not be to decide whether a ranch should be approved, and it would not replace the verification rules or human review. The role would be much narrower: help pull specific fields from difficult documents so the rest of the pipeline can evaluate them more consistently.

If that approach were used, I would still keep the original evidence, store exactly what was extracted, and validate those extracted fields before using them in case evaluation. In other words, the LLM would only help with turning messy documents into structured inputs. It would not be the system making the final verification decision.

Regardless of whether that approach is used, the main outcome of the design stays the same: every case should follow the same path, every decision should point back to specific evidence, and reviewers should remain in control whenever something is unclear.

## Review Workflow

Reviewers receive the case with automated findings attached. They approve, reject, request more information, or override. Overrides require a reason code and note so exceptions are explainable later.

The interface should feel like a case file: recommended tier, blocking checks, conflicts, and evidence links in one view.

## Edge Cases and Operations

When source data is partial, the case is held for review rather than auto-passed. When systems disagree, the conflict is surfaced explicitly and requires adjudication. When tier signals change mid-period, the record keeps timing context plus final reviewer interpretation. When documents arrive late, the original state remains visible and late evidence is added with timestamped review actions.

Policy owners control rule definitions and thresholds. Platform/ML Ops controls connectors, orchestration, storage, deployment, and monitoring. Rule changes are tested on fixture cases, rolled out in stages, and promoted when behavior is confirmed.
