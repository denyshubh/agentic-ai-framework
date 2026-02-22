# Credential Lifecycle Flow

## Overview

This document describes the complete lifecycle of Agent_Identity credentials within the EAAGF, from initial issuance through active monitoring, automatic rotation at 80% TTL, revocation on task completion, and revocation on agent decommission.

Credentials are the mechanism by which agents prove their identity to the Governance_Controller and Platform Adapters. The credential lifecycle is tightly coupled to the agent's Risk_Tier, which determines TTL limits and session behavior.

### Applicable Requirements

| Requirement | Description |
|---|---|
| 1.3 | Issue a unique Agent_Identity credential (X.509 or OAuth 2.0) to each registered agent |
| 1.4 | Automatically rotate credentials at 80% TTL without agent downtime |
| 1.5 | Revoke all credentials within 60 seconds of agent decommission |
| 3.2 | Issue scope-bound tokens with TTL not exceeding tier-based maximums |
| 3.6 | Immediately revoke all session-scoped credentials on task completion or timeout |

---

## Credential Lifecycle State Diagram

```mermaid
stateDiagram-v2
    [*] --> ISSUED : Agent registered<br/>or credential rotated

    ISSUED --> ACTIVE : Credential activated

    ACTIVE --> ROTATION_PENDING : 80% TTL reached<br/>(Req 1.4)
    ROTATION_PENDING --> ACTIVE : New credential issued<br/>Old credential revoked

    ACTIVE --> REVOKED_SESSION : Task completed<br/>or timed out<br/>(Req 3.6)
    ACTIVE --> REVOKED_DECOMMISSION : Agent decommissioned<br/>within 60 seconds<br/>(Req 1.5)
    ACTIVE --> REVOKED_EMERGENCY : Emergency stop executed

    ROTATION_PENDING --> REVOKED_DECOMMISSION : Agent decommissioned<br/>during rotation
    ROTATION_PENDING --> REVOKED_EMERGENCY : Emergency stop<br/>during rotation

    REVOKED_SESSION --> [*]
    REVOKED_DECOMMISSION --> [*]
    REVOKED_EMERGENCY --> [*]

    note right of ACTIVE
        Credential is valid.
        Agent can perform
        governed actions.
    end note

    note right of ROTATION_PENDING
        New credential being issued.
        Old credential still valid
        until new one is active.
    end note
```

---

## Credential Issuance Flow (Requirements 1.3, 3.2)

The following diagram shows the credential issuance process, which occurs at agent registration and after each rotation.

```mermaid
flowchart TD
    START([Credential issuance<br/>triggered]) --> SOURCE{Trigger source?}

    SOURCE -->|Agent registration<br/>Req 1.3| REG[Agent successfully<br/>registered in Agent_Registry]
    SOURCE -->|Credential rotation<br/>Req 1.4| ROT[80% TTL threshold<br/>reached on current credential]
    SOURCE -->|Session credential<br/>request<br/>Req 3.2| SESS[Agent requests<br/>task session credential]

    REG --> CRED_TYPE{Credential type?}
    ROT --> CRED_TYPE
    SESS --> SCOPE_BIND

    CRED_TYPE -->|X.509| X509[Issue X.509 certificate<br/>Include agent UUID in SAN<br/>Default TTL: 90 days]
    CRED_TYPE -->|OAuth 2.0| OAUTH[Issue OAuth 2.0 token<br/>Include agent UUID in sub claim]

    X509 --> BIND[Bind credential to<br/>agent UUID v4]
    OAUTH --> BIND

    BIND --> RECORD[Record credential:<br/>credential_type, credential_id,<br/>issued_at, expires_at]
    RECORD --> EMIT[Emit CREDENTIAL_ISSUED<br/>audit event]
    EMIT --> DONE([Credential active])

    SCOPE_BIND[Scope-bind token to task<br/>Req 3.2]
    SCOPE_BIND --> TTL_CHECK{Agent Risk_Tier?}

    TTL_CHECK -->|T1 / T2| TTL_LONG[TTL ≤ 3600 seconds<br/>1 hour maximum]
    TTL_CHECK -->|T3 / T4| TTL_SHORT[TTL ≤ 900 seconds<br/>15 minutes maximum]

    TTL_LONG --> SCOPE[Scope credential to:<br/>• Authorized resource set<br/>• Authorized action set<br/>• Task context]
    TTL_SHORT --> SCOPE

    SCOPE --> RECORD
```

### Credential TTL Rules by Risk Tier

| Risk Tier | Maximum Credential TTL | Default Value | Session Behavior |
|---|---|---|---|
| T1 (Informational) | ≤ 3600 seconds | 3600 seconds (1 hour) | Revoke on task completion |
| T2 (Transactional) | ≤ 3600 seconds | 3600 seconds (1 hour) | Revoke on task completion |
| T3 (Autonomous) | ≤ 900 seconds | 900 seconds (15 minutes) | Revoke on task completion or timeout |
| T4 (Critical) | ≤ 900 seconds | 900 seconds (15 minutes) | Revoke on task completion or timeout |

---

## Automatic Rotation Flow (Requirement 1.4)

When a credential reaches 80% of its configured TTL, the Agent_Registry automatically rotates it without requiring agent downtime.

```mermaid
flowchart TD
    A[Credential active<br/>TTL monitoring] --> B{80% TTL<br/>reached?}
    B -->|No| A
    B -->|Yes| C[Initiate automatic<br/>rotation<br/>Req 1.4]

    C --> D[Issue new credential<br/>with fresh TTL]
    D --> E{New credential<br/>issued successfully?}

    E -->|Yes| F[Activate new credential]
    F --> G[Verify new credential<br/>is active and functional]
    G --> H[Revoke old credential]
    H --> I[Emit CREDENTIAL_ROTATED<br/>audit event<br/>old_id → new_id]
    I --> J([New credential active<br/>No agent downtime])

    E -->|No| K[Emit CREDENTIAL_ROTATION_FAILED<br/>audit event]
    K --> L[Notify owning team]
    L --> M[Old credential remains active<br/>until expiry or manual intervention]
    M --> N{Retry rotation?}
    N -->|Yes| C
    N -->|No, credential expired| O[Agent must<br/>re-authenticate]
```

### Rotation Guarantees

- The new credential is activated before the old credential is revoked (no gap in validity).
- Rotation does not require agent downtime or restart.
- The agent continues operating with the existing credential until the new one is confirmed active.
- If rotation fails, the owning team is notified and the old credential remains valid until its natural expiry.

---

## Session Credential Revocation Flow (Requirement 3.6)

When an agent's task completes or times out, all session-scoped credentials are immediately revoked.

```mermaid
flowchart TD
    A[Agent task execution] --> B{Task status?}

    B -->|Completed successfully| C[Task completion event]
    B -->|Timed out| D[Task timeout event]
    B -->|Failed| E[Task failure event]

    C --> F[Initiate session<br/>credential revocation<br/>within 1 second<br/>Req 3.6]
    D --> F
    E --> F

    F --> G[Identify all session-scoped<br/>credentials for this task]
    G --> H[Revoke primary<br/>session credential]
    H --> I[Revoke derived /<br/>delegated credentials]
    I --> J[Propagate revocation<br/>to all Platform Adapters]

    J --> K{T3 or T4 agent?}
    K -->|Yes| L[Also check for<br/>TTL-expired credentials<br/>that need cleanup]
    K -->|No| M[Emit SESSION_CREDENTIAL_REVOKED<br/>audit event per credential]

    L --> M

    M --> N{All credentials<br/>revoked?}
    N -->|Yes| O([Session cleanup complete])
    N -->|No| P[Retry revocation<br/>Log failure]
    P --> N
```

### Revocation Scope

All session-scoped credentials associated with the task are revoked, including:
- The primary session credential issued for the task
- Any derived or delegated credentials issued during the task (e.g., credentials passed to sub-tasks or delegated agents)
- Platform-specific tokens issued by Platform Adapters for the task

After revocation, any attempt to use the revoked credentials is rejected by all EAAGF components.

---

## Decommission Credential Revocation Flow (Requirement 1.5)

When an agent is decommissioned, ALL associated credentials must be revoked within 60 seconds.

```mermaid
flowchart TD
    A([Agent decommission<br/>event received]) --> B[Start 60-second<br/>revocation window<br/>Req 1.5]

    B --> C[Identify ALL credentials<br/>associated with agent]

    C --> D[Primary Agent_Identity<br/>credential<br/>X.509 or OAuth 2.0]
    C --> E[Active session-scoped<br/>credentials]
    C --> F[Pending rotation<br/>credentials]
    C --> G[Delegated credentials<br/>issued to other agents]

    D --> H[Revoke credential]
    E --> H
    F --> H
    G --> H

    H --> I[Publish revocation status<br/>to all connected<br/>Platform Adapters]

    I --> J[Emit CREDENTIAL_REVOKED<br/>audit event per credential]

    J --> K{All revocations<br/>completed within<br/>60 seconds?}

    K -->|Yes| L([Decommission credential<br/>revocation complete])
    K -->|No| M[Escalate: notify<br/>AI Governance Team]
    M --> N[Continue revocation<br/>until complete]
    N --> L
```

### Decommission Revocation Guarantees

| Property | Value |
|---|---|
| Revocation window | 60 seconds from decommission event |
| Credential scope | ALL credentials (primary, session, rotation, delegated) |
| Platform propagation | Revocation published to all connected Platform Adapters within the 60-second window |
| Post-revocation enforcement | Any use of revoked credentials is rejected by all EAAGF components |
| Audit trail | CREDENTIAL_REVOKED event emitted for each revoked credential |

---

## Complete Credential Lifecycle Diagram

The following diagram shows the full credential lifecycle integrating issuance, monitoring, rotation, and all revocation paths.

```mermaid
flowchart TD
    REG([Agent registered]) --> ISSUE[Issue Agent_Identity<br/>credential<br/>Req 1.3]
    ISSUE --> ACTIVE[Credential ACTIVE]

    ACTIVE --> MON{Monitor TTL}

    MON -->|80% TTL reached| ROTATE[Auto-rotate<br/>Req 1.4]
    ROTATE --> NEW[Issue new credential]
    NEW --> REVOKE_OLD[Revoke old credential]
    REVOKE_OLD --> ACTIVE

    MON -->|TTL not reached| CHECK_EVENTS{Check events}

    CHECK_EVENTS -->|Task completed<br/>or timed out| REVOKE_SESSION[Revoke session<br/>credentials<br/>within 1 second<br/>Req 3.6]
    REVOKE_SESSION --> SESSION_DONE[Emit SESSION_CREDENTIAL_REVOKED]

    CHECK_EVENTS -->|Agent decommissioned| REVOKE_ALL[Revoke ALL credentials<br/>within 60 seconds<br/>Req 1.5]
    REVOKE_ALL --> DECOM_DONE[Emit CREDENTIAL_REVOKED<br/>per credential]

    CHECK_EVENTS -->|Emergency stop| REVOKE_EMERGENCY[Revoke ALL credentials<br/>immediately]
    REVOKE_EMERGENCY --> EMERGENCY_DONE[Emit EMERGENCY_STOP<br/>audit event]

    CHECK_EVENTS -->|No event| MON
```

---

## Audit Event Coverage

| Event | Trigger | Key Fields |
|---|---|---|
| `CREDENTIAL_ISSUED` | New credential issued at registration or rotation | agent_id, credential_id, credential_type, ttl, issued_at |
| `CREDENTIAL_ROTATED` | Automatic rotation at 80% TTL | agent_id, old_credential_id, new_credential_id, rotation_timestamp |
| `CREDENTIAL_ROTATION_FAILED` | Rotation attempt failed | agent_id, credential_id, failure_reason, timestamp |
| `SESSION_CREDENTIAL_ISSUED` | Task session credential issued | agent_id, credential_id, task_id, scope, ttl_seconds |
| `SESSION_CREDENTIAL_REVOKED` | Session credential revoked on task completion/timeout | agent_id, credential_id, task_id, revocation_reason |
| `CREDENTIAL_REVOKED` | Credential revoked on decommission or emergency stop | agent_id, credential_id, revocation_reason, timestamp |

---

## Cross-References

- [Agent Identity and Registration Standard](../eaagf-specification/02-agent-identity-standard.md) — Credential issuance, rotation, and decommission revocation rules
- [Authorization Standard](../eaagf-specification/04-authorization-standard.md) — Credential TTL rules and session revocation
- [Observability Standard](../eaagf-specification/05-observability-standard.md) — Audit event schema
- [Agent Registration Flow](./agent-registration-flow.md) — Credential issuance during registration
- [Human Oversight Flow](./human-oversight-flow.md) — Emergency stop credential revocation
