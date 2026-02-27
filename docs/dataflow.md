# Credit Verification Dataflow

This is the same flow described in `README.md`, shown in a simple step-by-step diagram.
Path: collect evidence -> normalize records -> run checks -> suggest tier -> human review -> final record.

## System Flow

```mermaid
flowchart LR
  subgraph SRC["Source Systems"]
    PM["PastureMap"]
    SF["Salesforce"]
    SL["Soil Lab APIs"]
    HW["Hardware Feeds"]
    DOC["GMP / LSA Documents"]
  end

  subgraph ING["Collect and Save Evidence"]
    CONN["Pull Source Data"]
    RAW["Raw Evidence Files"]
    IDX["Snapshot Log"]
  end

  subgraph NORM["Normalize Case Data"]
    CAN["Normalize Records"]
    REC["Match IDs and Locations"]
    CLAIM["Build Check Inputs"]
  end

  subgraph VER["Checks and Tier Suggestion"]
    GATE["Completeness Checks"]
    TIER["Tier Rules (Good / Better / Best)"]
    RECOUT["Case Draft"]
    PACK["Evidence References"]
  end

  subgraph REV["Human Review"]
    QUEUE["Review Queue"]
    CASE["Reviewer View"]
    DEC["Approve / Reject / Request Info / Override"]
    AUD["Audit Log"]
  end

  subgraph OUT["Outputs"]
    FINAL["Final Verification Record"]
    API["Read API"]
    CREDIT["Credit Workflow Input"]
  end

  PM --> CONN
  SF --> CONN
  SL --> CONN
  HW --> CONN
  DOC --> CONN

  CONN --> RAW
  CONN --> IDX

  RAW --> CAN
  IDX --> CAN
  CAN --> REC
  REC --> CLAIM

  CLAIM --> GATE
  GATE -->|pass| TIER
  GATE -->|blocker| QUEUE

  TIER --> RECOUT
  CLAIM --> PACK
  REC --> PACK
  RECOUT --> QUEUE
  PACK --> QUEUE

  QUEUE --> CASE
  CASE --> DEC
  DEC --> AUD
  DEC --> FINAL

  FINAL --> API
  FINAL --> CREDIT
```

## Decision Branches

1. If a blocking check fails, the case goes straight to human review.
2. If checks pass, the system suggests the best tier it can support.
3. Reviewer finalizes case:
   - `APPROVED`
   - `REJECTED`
   - `REQUESTED_INFO`
   - `APPROVED_WITH_OVERRIDE` (requires reason code and note)

## What Each Stage Produces

1. Collection stage:
   - immutable source snapshots
   - connector metadata
   - snapshot hashes
2. Normalization stage:
   - normalized case records
   - reconciliation metrics
   - check inputs
3. Checks and tier stage:
   - gate results
   - tier suggestion
   - evidence references
4. Review stage:
   - final status
   - reviewer action metadata
   - immutable audit events

## Versioning and Replay Anchors

Each final verification record stores:

- `policy_version` and policy hash
- all source `snapshot_id` values and payload hashes
- connector version metadata
- reconciliation rule version
- evidence reference manifest hash

With those anchors, the same ranch-period decision can be replayed and explained during audit.
