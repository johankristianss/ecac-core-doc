# ECAC-Core: Drawbacks and Limitations

An honest assessment of the trade-offs, limitations, and challenges inherent in the ECAC-core approach to deterministic access control.

---

## Table of Contents

1. [Overview](#overview)
2. [Fundamental Trade-offs](#fundamental-trade-offs)
3. [Performance Limitations](#performance-limitations)
4. [Operational Challenges](#operational-challenges)
5. [Security Considerations](#security-considerations)
6. [Implementation Gaps](#implementation-gaps)
7. [Scalability Concerns](#scalability-concerns)
8. [Adoption Barriers](#adoption-barriers)
9. [Comparison Summary](#comparison-summary)

---

## Overview

ECAC-core makes deliberate design choices that enable its core guarantees. However, these choices come with inherent limitations that may make it unsuitable for certain use cases.

```mermaid
graph TB
    subgraph "Design Choices"
        D1[Offline-First]
        D2[Deterministic Replay]
        D3[Deny-Wins Policy]
        D4[Append-Only Log]
    end

    subgraph "Resulting Limitations"
        L1[No Immediate Revocation]
        L2[Full Replay Cost]
        L3[No Partial Recovery]
        L4[Unbounded Storage Growth]
    end

    D1 --> L1
    D2 --> L2
    D3 --> L3
    D4 --> L4

    style L1 fill:#ffcdd2
    style L2 fill:#ffcdd2
    style L3 fill:#ffcdd2
    style L4 fill:#ffcdd2
```

---

## Fundamental Trade-offs

### 1. No Immediate Revocation for Offline Nodes

**The Limitation**: When a credential is revoked, offline nodes continue operating under the old policy until they reconnect and sync.

```mermaid
sequenceDiagram
    participant Admin
    participant Online as Node A (Online)
    participant Offline as Node B (Offline)

    Admin->>Online: Revoke User X's access
    Online->>Online: Revocation effective

    rect rgb(255, 200, 200)
        Note over Offline: Vulnerability Window
        Offline->>Offline: User X still has access
        Offline->>Offline: User X performs writes
        Offline->>Offline: Writes succeed locally
    end

    Note over Offline: Hours/Days later...

    Offline-->>Online: Reconnects

    Note over Offline: Now revocation takes effect<br/>User X's writes skipped on replay
```

**Impact**:
- Sensitive operations may be performed by revoked users
- Time-sensitive revocations (employee termination) have a gap
- Cannot guarantee "access ends at exactly time T"

**Mitigations Available**:
- Short credential validity windows (exp claim)
- Require periodic online check-ins
- Accept this as inherent to offline-first design

---

### 2. Eventual Consistency, Not Strong Consistency

**The Limitation**: Different nodes may temporarily see different states.

```mermaid
graph TB
    subgraph "Time T1"
        A1[Node A: State S1]
        B1[Node B: State S2]
        C1[Node C: State S3]
    end

    subgraph "Time T2 (After Sync)"
        A2[Node A: State S]
        B2[Node B: State S]
        C2[Node C: State S]
    end

    A1 -.->|Eventually| A2
    B1 -.->|Eventually| B2
    C1 -.->|Eventually| C2

    style A1 fill:#ffecb3
    style B1 fill:#ffecb3
    style C1 fill:#ffecb3
    style A2 fill:#c8e6c9
    style B2 fill:#c8e6c9
    style C2 fill:#c8e6c9
```

**Unsuitable For**:
| Use Case | Why It Fails |
|----------|--------------|
| Financial transactions | Need immediate consistency |
| Real-time coordination | Can't tolerate divergence |
| Inventory management | Risk of overselling |
| Reservation systems | Double-booking possible |

---

### 3. Deny-Wins Can Lose User Work

**The Limitation**: When access is revoked, all operations performed under that credential are skipped—even if the user believed they were authorized.

```mermaid
flowchart TD
    subgraph "User's Perspective"
        U1[Perform work offline]
        U2[See success locally]
        U3[Sync with network]
        U4[Work disappears!]
    end

    U1 --> U2
    U2 --> U3
    U3 --> U4

    subgraph "System's Perspective"
        S1[Store operation in log]
        S2[Replay with current policy]
        S3[Credential revoked]
        S4[Skip operation]
    end

    U1 --> S1
    U3 --> S2
    S2 --> S3
    S3 --> S4

    style U4 fill:#ffcdd2
    style S4 fill:#ffcdd2
```

**User Experience Issues**:
- No warning at write time that access may be revoked
- Work done in good faith can be lost
- Difficult to explain to end users

---

### 4. Append-Only Log Never Shrinks

**The Limitation**: Operations are never deleted, only skipped during replay.

```mermaid
graph LR
    subgraph "Log Growth Over Time"
        T1[Day 1<br/>100 ops]
        T2[Month 1<br/>10K ops]
        T3[Year 1<br/>1M ops]
        T4[Year 5<br/>5M ops]
    end

    T1 --> T2 --> T3 --> T4

    subgraph "Storage"
        S1[10 MB]
        S2[1 GB]
        S3[100 GB]
        S4[500 GB]
    end

    T1 --> S1
    T2 --> S2
    T3 --> S3
    T4 --> S4

    style T4 fill:#ffcdd2
    style S4 fill:#ffcdd2
```

**Consequences**:
- Storage costs grow unboundedly
- Cannot comply with "right to be forgotten" (GDPR)
- Historical data accumulates even if no longer needed
- No log compaction or garbage collection

---

## Performance Limitations

### 1. Full Replay Cost

**The Limitation**: To verify state, you must replay the entire log.

```mermaid
graph TB
    subgraph "Replay Complexity"
        OPS[N Operations]
        TIME["O(N) Time"]
        MEM["O(N) Memory for DAG"]
    end

    OPS --> TIME
    OPS --> MEM

    subgraph "At Scale"
        M1[1M ops → seconds]
        M2[100M ops → minutes]
        M3[1B ops → hours]
    end

    style M3 fill:#ffcdd2
```

**Benchmark Reality**:
| Operations | Full Replay Time | Incremental |
|------------|------------------|-------------|
| 10,000 | ~50ms | ~5ms |
| 100,000 | ~500ms | ~50ms |
| 1,000,000 | ~5s | ~500ms |
| 10,000,000 | ~50s | ~5s |

**Mitigations**:
- Checkpoints reduce incremental replay cost
- But full replay still needed for audit verification

---

### 2. No Parallel Replay

**The Limitation**: Topological order must be respected, limiting parallelization.

```mermaid
flowchart LR
    subgraph "Sequential Requirement"
        OP1[Op 1] --> OP2[Op 2]
        OP2 --> OP3[Op 3]
        OP3 --> OP4[Op 4]
    end

    subgraph "Cannot Parallelize"
        P1[Thread 1]
        P2[Thread 2]
        P3[Thread 3]
        P4[Thread 4]
    end

    OP1 --> P1
    OP2 -.->|Must wait| P1
    OP3 -.->|Must wait| P1
    OP4 -.->|Must wait| P1

    style P2 fill:#e0e0e0
    style P3 fill:#e0e0e0
    style P4 fill:#e0e0e0
```

**Why This Matters**:
- Cannot leverage multi-core CPUs effectively
- Replay is inherently single-threaded
- Policy evaluation must be sequential

---

### 3. Cryptographic Overhead

**The Limitation**: Every operation requires signature verification.

```mermaid
graph TB
    subgraph "Per-Operation Cost"
        HASH[BLAKE3 Hash: ~1μs]
        SIG[Ed25519 Verify: ~50μs]
        CBOR[CBOR Decode: ~10μs]
        POLICY[Policy Check: ~5μs]
    end

    TOTAL[Total: ~66μs per op]

    HASH --> TOTAL
    SIG --> TOTAL
    CBOR --> TOTAL
    POLICY --> TOTAL

    subgraph "At Scale"
        OPS[1M operations]
        TIME[~66 seconds<br/>just for crypto]
    end

    TOTAL --> TIME

    style TIME fill:#ffecb3
```

---

### 4. Network Sync Overhead

**The Limitation**: Full operation history must be transferred to new nodes.

```mermaid
sequenceDiagram
    participant New as New Node
    participant Existing as Existing Node

    New->>Existing: Request sync

    rect rgb(255, 235, 200)
        Note over Existing,New: Transfer entire history
        loop For each operation
            Existing->>New: Op CBOR bytes
        end
    end

    Note over New: Must replay all ops<br/>before becoming useful

    New->>New: Full replay
    New->>Existing: Ready to participate
```

**Bootstrap Time**:
| Log Size | Transfer Time | Replay Time | Total |
|----------|---------------|-------------|-------|
| 100 MB | ~10s | ~5s | ~15s |
| 10 GB | ~15min | ~8min | ~23min |
| 1 TB | ~25hr | ~13hr | ~38hr |

---

## Operational Challenges

### 1. Complex Debugging

**The Limitation**: Understanding why state diverged requires replaying and comparing logs.

```mermaid
flowchart TD
    PROBLEM[State mismatch detected]

    PROBLEM --> Q1{Same operations?}
    Q1 -->|No| DIFF1[Find missing ops]
    Q1 -->|Yes| Q2{Same order?}

    Q2 -->|No| DIFF2[Compare topo sort]
    Q2 -->|Yes| Q3{Same policy?}

    Q3 -->|No| DIFF3[Compare credentials]
    Q3 -->|Yes| Q4{Same trust?}

    Q4 -->|No| DIFF4[Compare TrustView]
    Q4 -->|Yes| BUG[Implementation bug]

    style PROBLEM fill:#ffcdd2
    style BUG fill:#ffcdd2
```

**Debugging Challenges**:
- Must have access to full logs from all parties
- Time-consuming to replay and compare
- Requires specialized tooling
- Non-determinism bugs are extremely subtle

---

### 2. No Schema Migration

**The Limitation**: Changing the data model is difficult with an immutable log.

```mermaid
graph TB
    subgraph "Schema V1"
        V1[field: user_name]
    end

    subgraph "Schema V2"
        V2A[field: first_name]
        V2B[field: last_name]
    end

    V1 -->|Migration?| V2A
    V1 -->|Migration?| V2B

    subgraph "Problems"
        P1[Old ops use V1 schema]
        P2[New ops use V2 schema]
        P3[Replay must handle both]
        P4[Cannot rewrite history]
    end

    style P1 fill:#ffecb3
    style P2 fill:#ffecb3
    style P3 fill:#ffecb3
    style P4 fill:#ffcdd2
```

**Options (All Problematic)**:
| Approach | Drawback |
|----------|----------|
| Version fields in ops | Complexity in replay logic |
| Dual-write period | Temporary inconsistency |
| Epoch-based migration | Breaks continuous history |
| Accept tech debt | Schema forever frozen |

---

### 3. Key Management Complexity

**The Limitation**: Losing a signing key is catastrophic; key rotation is complex.

```mermaid
flowchart TD
    subgraph "Key Loss Scenarios"
        L1[Private key compromised]
        L2[Private key lost]
        L3[Key rotation needed]
    end

    L1 --> C1[Attacker can forge ops]
    L2 --> C2[Cannot create new ops]
    L3 --> C3[Must coordinate across all nodes]

    C1 --> R1[Revoke + new identity]
    C2 --> R2[New identity required]
    C3 --> R3[Complex migration]

    style C1 fill:#ffcdd2
    style C2 fill:#ffcdd2
    style C3 fill:#ffecb3
```

**No Built-in Solutions For**:
- Hardware security module integration
- Key escrow or recovery
- Seamless key rotation
- Multi-signature requirements

---

### 4. Credential Lifecycle Management

**The Limitation**: VCs must be carefully managed across their lifecycle.

```mermaid
stateDiagram-v2
    [*] --> Issued
    Issued --> Active: nbf reached
    Active --> Expired: exp reached
    Active --> Revoked: Status list update

    Expired --> [*]
    Revoked --> [*]

    note right of Active
        Window for valid operations
        May be narrow or wide
    end note

    note right of Revoked
        All ops under this VC
        skipped on replay
    end note
```

**Management Burden**:
- Track expiration dates
- Manage status list updates
- Handle renewal workflows
- Coordinate across organizations

---

## Security Considerations

### 1. No Byzantine Fault Tolerance

**The Limitation**: Assumes honest-but-curious participants, not actively malicious ones.

```mermaid
graph TB
    subgraph "Threat Model Coverage"
        T1[Eavesdropping ✅]
        T2[Message tampering ✅]
        T3[Replay attacks ✅]
        T4[Unauthorized access ✅]
    end

    subgraph "NOT Covered"
        N1[Malicious nodes ❌]
        N2[Sybil attacks ❌]
        N3[Collusion ❌]
        N4[DoS attacks ❌]
    end

    style N1 fill:#ffcdd2
    style N2 fill:#ffcdd2
    style N3 fill:#ffcdd2
    style N4 fill:#ffcdd2
```

**Attack Scenarios Not Prevented**:
| Attack | Why Unprotected |
|--------|-----------------|
| Malicious node floods log | No rate limiting |
| Sybil creates fake identities | No proof of identity |
| Colluding nodes fork history | No fork detection |
| DoS via large operations | No size limits enforced |

---

### 2. Confidentiality Is Incomplete

**The Limitation**: The M9 read-control implementation over-redacts, making it effectively unusable.

```mermaid
flowchart TD
    subgraph "Expected Behavior"
        E1[Authorized user]
        E2[Reads encrypted field]
        E3[Sees plaintext]
    end

    E1 --> E2 --> E3

    subgraph "Actual Behavior (M9)"
        A1[Authorized user]
        A2[Reads encrypted field]
        A3[Sees &lt;redacted&gt;]
    end

    A1 --> A2 --> A3

    style E3 fill:#c8e6c9
    style A3 fill:#ffcdd2
```

**Current State**:
- Encryption implemented but broken
- Even authorized users see redacted content
- Confidentiality is not actually provided
- Listed as known limitation

---

### 3. Metadata Leakage

**The Limitation**: Even with encryption, metadata reveals information.

```mermaid
graph TB
    subgraph "Visible to All Nodes"
        M1[Who wrote (author_pk)]
        M2[When (HLC timestamp)]
        M3[What object/field (key)]
        M4[Parent operations]
        M5[Operation size]
    end

    subgraph "Information Leaked"
        L1[Activity patterns]
        L2[Relationship graphs]
        L3[Data structure]
        L4[Collaboration patterns]
    end

    M1 --> L1
    M2 --> L1
    M3 --> L3
    M4 --> L2
    M4 --> L4

    style L1 fill:#ffecb3
    style L2 fill:#ffecb3
    style L3 fill:#ffecb3
    style L4 fill:#ffecb3
```

---

### 4. Trust Bootstrap Problem

**The Limitation**: Someone must initially be trusted to set up the trust hierarchy.

```mermaid
flowchart TD
    START[System Bootstrap]
    START --> Q1{Who has<br/>issuer_admin?}

    Q1 --> CHICKEN[Chicken-and-egg:<br/>Need admin to create admin]

    CHICKEN --> S1[Genesis operation<br/>by system creator]

    S1 --> RISK[Single point of<br/>trust at genesis]

    style CHICKEN fill:#ffecb3
    style RISK fill:#ffcdd2
```

**Bootstrap Challenges**:
- Initial issuer_admin must be trusted
- No distributed key generation
- Genesis operation is a trust anchor
- Compromise of genesis key is catastrophic

---

## Implementation Gaps

### Current Known Limitations

```mermaid
graph TB
    subgraph "Incomplete Features"
        I1[Confidentiality M9<br/>Over-redacts]
        I2[StatusPointer<br/>Not implemented]
        I3[Multi-issuer quorum<br/>Not implemented]
        I4[Key rollover windows<br/>Not implemented]
        I5[Crash-tail repair<br/>Detection only]
    end

    subgraph "Impact"
        IM1[Cannot protect data at rest]
        IM2[Limited revocation granularity]
        IM3[Single issuer dependency]
        IM4[Complex key transitions]
        IM5[Manual recovery needed]
    end

    I1 --> IM1
    I2 --> IM2
    I3 --> IM3
    I4 --> IM4
    I5 --> IM5

    style I1 fill:#ffcdd2
    style I2 fill:#ffecb3
    style I3 fill:#ffecb3
    style I4 fill:#ffecb3
    style I5 fill:#ffecb3
```

---

## Scalability Concerns

### Vertical Scaling Limits

```mermaid
graph TB
    subgraph "Single Node Limits"
        CPU[CPU: Sequential replay]
        MEM[Memory: Full DAG in RAM]
        DISK[Disk: Append-only growth]
        NET[Network: Full history sync]
    end

    subgraph "Bottlenecks"
        B1[Cannot parallelize replay]
        B2[Large DAGs need GBs RAM]
        B3[No compaction possible]
        B4[Bootstrap takes hours]
    end

    CPU --> B1
    MEM --> B2
    DISK --> B3
    NET --> B4

    style B1 fill:#ffcdd2
    style B2 fill:#ffecb3
    style B3 fill:#ffcdd2
    style B4 fill:#ffecb3
```

### Horizontal Scaling Challenges

```mermaid
graph TB
    subgraph "Sharding Challenges"
        S1[Operations reference<br/>parents across shards]
        S2[Policy spans<br/>entire log]
        S3[Credentials may apply<br/>to any shard]
    end

    S1 --> HARD[Cross-shard coordination<br/>defeats purpose]
    S2 --> HARD
    S3 --> HARD

    HARD --> MONO[Must remain<br/>monolithic]

    style MONO fill:#ffcdd2
```

---

## Adoption Barriers

### 1. Complexity for Developers

```mermaid
graph TB
    subgraph "Learning Curve"
        L1[Understand CRDTs]
        L2[Understand HLC]
        L3[Understand DAG ordering]
        L4[Understand VCs]
        L5[Understand deny-wins]
    end

    L1 --> WEEKS[Weeks to months<br/>of learning]
    L2 --> WEEKS
    L3 --> WEEKS
    L4 --> WEEKS
    L5 --> WEEKS

    WEEKS --> BARRIER[High adoption barrier]

    style BARRIER fill:#ffcdd2
```

### 2. Ecosystem Immaturity

```mermaid
graph LR
    subgraph "Missing Ecosystem"
        E1[GUI tools ❌]
        E2[IDE plugins ❌]
        E3[Monitoring dashboards ❌]
        E4[Client libraries ❌]
        E5[Cloud hosting ❌]
    end

    subgraph "Available"
        A1[Rust CLI ✅]
        A2[Core library ✅]
        A3[Documentation ✅]
    end

    style E1 fill:#ffcdd2
    style E2 fill:#ffcdd2
    style E3 fill:#ffcdd2
    style E4 fill:#ffcdd2
    style E5 fill:#ffcdd2
```

### 3. Research Prototype Status

```mermaid
flowchart TD
    STATUS[Research Prototype]

    STATUS --> NOT1[Not production-ready]
    STATUS --> NOT2[No SLA or support]
    STATUS --> NOT3[API may change]
    STATUS --> NOT4[Limited testing]

    NOT1 --> RISK[Business risk<br/>to adopt]
    NOT2 --> RISK
    NOT3 --> RISK
    NOT4 --> RISK

    style STATUS fill:#ffecb3
    style RISK fill:#ffcdd2
```

---

## Comparison Summary

### When NOT to Use ECAC-Core

```mermaid
graph TB
    subgraph "Poor Fit Use Cases"
        UC1[High-frequency trading]
        UC2[Real-time gaming]
        UC3[Social media feeds]
        UC4[E-commerce inventory]
        UC5[Banking transactions]
        UC6[GDPR-compliant systems]
    end

    UC1 --> R1[Too slow]
    UC2 --> R2[Eventual consistency]
    UC3 --> R3[Scale issues]
    UC4 --> R4[Need strong consistency]
    UC5 --> R5[Need immediate finality]
    UC6 --> R6[Cannot delete data]

    style R1 fill:#ffcdd2
    style R2 fill:#ffcdd2
    style R3 fill:#ffcdd2
    style R4 fill:#ffcdd2
    style R5 fill:#ffcdd2
    style R6 fill:#ffcdd2
```

### Summary Table

| Limitation | Severity | Workaround Available |
|------------|----------|---------------------|
| No immediate revocation | High | Short credential windows |
| Eventual consistency | Medium | Accept for use case |
| Append-only growth | High | None |
| Sequential replay | Medium | Checkpoints |
| No BFT | High | Trusted participants |
| Incomplete confidentiality | Critical | None (broken) |
| Bootstrap complexity | Medium | Careful planning |
| Schema migration | High | Version fields |
| Debugging complexity | Medium | Better tooling |
| Ecosystem immaturity | High | Build internally |

---

## Conclusion

ECAC-core is a **research prototype** with deliberate limitations that stem from its design goals. It is **not suitable** for:

- Systems requiring immediate consistency
- High-throughput applications
- GDPR-compliant data management
- Byzantine fault-tolerant deployments
- Production systems without extensive testing

It **is suitable** for research, exploration, and scenarios where:

- Offline operation is more important than immediate revocation
- Auditability is more important than performance
- Determinism is more important than flexibility
- The limitations are acceptable trade-offs

```mermaid
pie title Suitability Assessment
    "Suitable Use Cases" : 30
    "Marginal (with workarounds)" : 25
    "Not Suitable" : 45
```

---

## References

- [Architecture Documentation](./ARCHITECTURE.md)
- [Vision and Use Cases](./VISION.md)
- [Advantages](./ADVANTAGES.md)
