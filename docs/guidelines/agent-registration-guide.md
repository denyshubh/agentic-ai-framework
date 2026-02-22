# Agent Registration Guide

This guide walks product teams through the process of registering AI agents with the EAAGF Agent Registry. It covers preparing the agent manifest, submitting for registration, understanding the response, and troubleshooting common errors.

For the normative specification, see [Agent Identity and Registration Standard](../eaagf-specification/02-agent-identity-standard.md). For the registration flow diagram, see [Agent Registration Flow](../flows/agent-registration-flow.md).

---

## Prerequisites

Before registering an agent, ensure you have:

1. A registered team identity in the enterprise directory (your `owning_team` value).
2. A completed risk assessment for your agent. See the [Risk Assessment Guide](./risk-assessment-guide.md) if you have not yet determined your agent's Risk_Tier.
3. A list of resources your agent needs to access, with specific actions (e.g., `SELECT`, `READ`, `UPDATE`).
4. Knowledge of which MCP servers and egress endpoints your agent requires.
5. Your target deployment platform identified (DATABRICKS, SALESFORCE, SNOWFLAKE, COPILOT_STUDIO, AWS, AZURE, or GCP).

---

## Step 1: Prepare the Agent Manifest

The agent manifest is a YAML file that declares your agent's identity, capabilities, permissions, and governance requirements. The Agent Registry validates this manifest against the EAAGF schema before accepting the registration.

### Manifest Structure

Every manifest has three top-level sections:

```yaml
apiVersion: eaagf/v1       # Always "eaagf/v1"
kind: AgentManifest         # Always "AgentManifest" for single-agent registration
metadata:                   # Agent identity fields
  name: ""
  version: ""
  owning_team: ""
  platform: ""
spec:                       # Agent governance specification
  risk_tier: ""
  capabilities: []
  declared_permissions: []
  oversight_mode: ""
  max_session_duration_seconds: 0
  max_actions_per_minute: 0
  protocols_supported: []
```

### Required Fields Reference

| Field | Location | Type | Description |
|---|---|---|---|
| `apiVersion` | root | string | Must be `eaagf/v1` |
| `kind` | root | string | Must be `AgentManifest` |
| `name` | `metadata` | string | Human-readable agent name (1–256 characters) |
| `version` | `metadata` | semver | Semantic version (e.g., `1.0.0`) |
| `owning_team` | `metadata` | string | Your registered team name |
| `platform` | `metadata` | enum | One of: `DATABRICKS`, `SALESFORCE`, `SNOWFLAKE`, `COPILOT_STUDIO`, `AWS`, `AZURE`, `GCP` |
| `risk_tier` | `spec` | enum | One of: `T1`, `T2`, `T3`, `T4` |
| `capabilities` | `spec` | array | At least one of: `TOOL_CALL`, `DATA_READ`, `DATA_WRITE`, `AGENT_DELEGATION`, `EXTERNAL_CONNECTION` |
| `declared_permissions` | `spec` | array | Resource-action pairs your agent needs |
| `oversight_mode` | `spec` | enum | One of: `FULL_AUTO`, `SUPERVISED`, `APPROVAL_REQUIRED`, `HUMAN_IN_LOOP` |
| `max_session_duration_seconds` | `spec` | integer | Credential TTL. Default: 3600 (T1/T2), 900 (T3/T4) |
| `max_actions_per_minute` | `spec` | integer | Rate limit. Default: 100 (T1/T2), 20 (T3/T4) |
| `protocols_supported` | `spec` | array | At least `MCP_1_0`. Add `A2A_1_0` if delegating to other agents |

### Optional Fields

| Field | Location | Type | Description |
|---|---|---|---|
| `approved_mcp_servers` | `spec` | array | MCP server URIs from the enterprise MCP directory |
| `approved_egress_endpoints` | `spec` | array | Allowlisted outbound endpoints (exact hostnames or wildcard patterns) |
| `data_classifications_accessed` | `spec` | array | Data classification levels: `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| `context_compartments` | `spec` | array | Context_Compartment identifiers for data isolation |
| `geographic_constraints` | `spec` | array | Geographic regions for data residency (e.g., `EU`, `US`) |

---

## Step 2: Choose the Right Values for Your Tier

Your Risk_Tier determines several mandatory governance parameters. Use the table below to set the correct values:

| Parameter | T1 | T2 | T3 | T4 |
|---|---|---|---|---|
| `oversight_mode` | `HUMAN_IN_LOOP` | `SUPERVISED` | `APPROVAL_REQUIRED` | `APPROVAL_REQUIRED` |
| `max_session_duration_seconds` | ≤ 3600 | ≤ 3600 | ≤ 900 | ≤ 900 |
| `max_actions_per_minute` | ≤ 100 | ≤ 100 | ≤ 20 | ≤ 20 |

You may set stricter values (lower TTL, lower rate limit, more restrictive oversight) but you cannot relax them below the tier minimum. For example, a T3 agent cannot set `max_session_duration_seconds: 3600` — the maximum is 900.

T4 agents cannot use `FULL_AUTO` oversight mode without explicit AI Governance Team authorization.

---

## Step 3: Submit for Registration

### Single Agent Registration

Submit your manifest to the Agent Registry API:

```bash
curl -X POST https://agent-registry.internal.company.com/v1/agents \
  -H "Content-Type: application/yaml" \
  -H "Authorization: Bearer $TEAM_TOKEN" \
  --data-binary @agent-manifest.yaml
```

### Bulk Registration (Up to 1,000 Agents)

For bulk registration, wrap multiple agent manifests in an `AgentManifestList`:

```yaml
apiVersion: eaagf/v1
kind: AgentManifestList
metadata:
  submitted_by: "your-team-name"
  submission_timestamp: "2025-07-14T10:00:00Z"
items:
  - apiVersion: eaagf/v1
    kind: AgentManifest
    metadata:
      name: "agent-one"
      version: "1.0.0"
      owning_team: "your-team"
      platform: "DATABRICKS"
    spec:
      # ... full spec
  - apiVersion: eaagf/v1
    kind: AgentManifest
    metadata:
      name: "agent-two"
      version: "1.0.0"
      owning_team: "your-team"
      platform: "SALESFORCE"
    spec:
      # ... full spec
  # Up to 1,000 items
```

Submit the bulk manifest:

```bash
curl -X POST https://agent-registry.internal.company.com/v1/agents/bulk \
  -H "Content-Type: application/yaml" \
  -H "Authorization: Bearer $TEAM_TOKEN" \
  --data-binary @bulk-manifest.yaml
```

Bulk registration is atomic per-agent: if one agent fails validation, the others still register. The response includes per-agent status.

---

## Step 4: Understand the Registration Response

### Successful Registration

A successful registration returns the assigned `agent_id` (UUID v4), issued credentials, and lifecycle state:

```json
{
  "status": "SUCCESS",
  "agent_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "lifecycle_state": "DEVELOPMENT",
  "risk_tier": "T2",
  "identity": {
    "credential_type": "OAUTH2",
    "credential_id": "cred-xyz-123",
    "issued_at": "2025-07-14T10:00:00Z",
    "expires_at": "2025-07-14T11:00:00Z"
  }
}
```

All newly registered agents start in the `DEVELOPMENT` lifecycle state. To promote to `STAGING`, you need a passing conformance test suite run. See [Agent Lifecycle Flow](../flows/agent-lifecycle-flow.md) for transition requirements.

### Bulk Registration Response

```json
{
  "status": "PARTIAL_SUCCESS",
  "summary": {
    "total_submitted": 5,
    "total_succeeded": 4,
    "total_failed": 1
  },
  "results": [
    { "name": "agent-one", "status": "SUCCESS", "agent_id": "..." },
    { "name": "agent-two", "status": "FAILED", "error_code": "CONFORMANCE_PROFILE_INVALID", "details": "..." }
  ]
}
```

---

## Step 5: After Registration

Once registered, your agent:

1. Has a UUID v4 identity that is used in all audit events and policy evaluations.
2. Has an Agent_Identity credential (X.509 or OAuth 2.0) that it must present for every governed action.
3. Is in `DEVELOPMENT` state — it can be tested but cannot process production data.
4. Will have its credential automatically rotated when it reaches 80% of its TTL.

### Next Steps

- Run the Conformance_Test_Suite against your agent to validate it meets EAAGF requirements.
- Complete a security scan (SAST, dependency vulnerability scan) before requesting promotion to STAGING.
- For T3/T4 agents, request AI Governance Team approval for production deployment.

---

## Troubleshooting Common Errors

### CLASSIFICATION_REQUIRED

```json
{
  "error_code": "CLASSIFICATION_REQUIRED",
  "message": "Agent must complete risk classification before deployment.",
  "agent_id": "a1b2c3d4-...",
  "recommended_action": "Complete the risk classification questionnaire."
}
```

**Cause:** The `risk_tier` field is missing from your manifest, or the classification engine could not determine a tier from the declared capabilities.

**Fix:**
1. Complete the [Risk Assessment Guide](./risk-assessment-guide.md) to determine your agent's tier.
2. Add the `risk_tier` field to your manifest's `spec` section.
3. Ensure your `capabilities`, `data_classifications_accessed`, and `oversight_mode` are consistent with the declared tier.

### CONFORMANCE_PROFILE_INVALID

```json
{
  "error_code": "CONFORMANCE_PROFILE_INVALID",
  "message": "Conformance_Profile does not conform to the EAAGF schema.",
  "details": [
    { "path": "$.spec.capabilities", "error": "Array must contain at least 1 item" },
    { "path": "$.spec.protocols_supported[0]", "error": "Value 'MCP_2_0' is not in enum [MCP_1_0, A2A_1_0]" }
  ]
}
```

**Cause:** One or more fields in your manifest do not match the expected schema.

**Common issues:**
- Empty `capabilities` array — you must declare at least one capability.
- Invalid enum values — check that `platform`, `risk_tier`, `oversight_mode`, `capabilities`, and `protocols_supported` use the exact allowed values.
- Missing `protocols_supported` — at least `MCP_1_0` is required.
- Invalid semver in `version` — must follow `MAJOR.MINOR.PATCH` format (e.g., `1.0.0`).

### REQUIRED_FIELD_MISSING

```json
{
  "error_code": "REQUIRED_FIELD_MISSING",
  "message": "Required field 'owning_team' is missing.",
  "path": "$.metadata.owning_team"
}
```

**Cause:** A required field is absent from your manifest.

**Fix:** Add the missing field. Refer to the Required Fields Reference table in Step 1.

### MANIFEST_SCHEMA_INVALID (Bulk Registration)

```json
{
  "error_code": "MANIFEST_SCHEMA_INVALID",
  "message": "Bulk manifest does not conform to the AgentManifestList schema.",
  "details": "Expected 'kind: AgentManifestList', got 'kind: AgentManifest'"
}
```

**Cause:** The bulk manifest wrapper is malformed.

**Fix:** Ensure your bulk manifest uses `kind: AgentManifestList` and wraps individual manifests in the `items` array.

---

## Annotated Manifest Examples

### T1 — Informational Agent (Internal FAQ Bot)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "internal-faq-bot"
  version: "1.0.0"
  owning_team: "employee-experience"
  platform: "COPILOT_STUDIO"
spec:
  # T1: Read-only, non-sensitive data, human approves every action
  risk_tier: "T1"

  # Read-only capabilities
  capabilities:
    - TOOL_CALL
    - DATA_READ

  # Only reads from the internal knowledge base
  declared_permissions:
    - resource: "sharepoint://sites/knowledge-base/articles"
      actions: ["READ"]

  # MCP server for knowledge base access
  approved_mcp_servers:
    - "mcp://enterprise-catalog/sharepoint-kb"

  # No outbound connections needed
  approved_egress_endpoints: []

  # Only accesses public/internal data
  data_classifications_accessed:
    - "PUBLIC"
    - "INTERNAL"

  # T1 default: human approves every response
  oversight_mode: "HUMAN_IN_LOOP"

  # T1 credential TTL: 1 hour
  max_session_duration_seconds: 3600

  # T1 rate limit: 100 actions/minute
  max_actions_per_minute: 100

  # No sensitive data compartment needed
  context_compartments: []

  # No geographic constraints for public data
  geographic_constraints: []

  # MCP for tool connections
  protocols_supported:
    - "MCP_1_0"
```

### T2 — Transactional Agent (CRM Update Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "crm-update-agent"
  version: "2.1.0"
  owning_team: "sales-operations"
  platform: "SALESFORCE"
spec:
  # T2: Reads independently, human-in-loop for writes, internal data
  risk_tier: "T2"

  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE

  declared_permissions:
    - resource: "salesforce://sobject/Account"
      actions: ["READ", "UPDATE"]
    - resource: "salesforce://sobject/Opportunity"
      actions: ["READ", "UPDATE"]
    - resource: "salesforce://sobject/Contact"
      actions: ["READ"]

  approved_mcp_servers:
    - "mcp://enterprise-catalog/salesforce-crm"

  approved_egress_endpoints:
    - "api.internal.company.com"

  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"

  # T2 default: gates on write operations
  oversight_mode: "SUPERVISED"

  # T2 credential TTL: 1 hour
  max_session_duration_seconds: 3600

  # T2 rate limit: 100 actions/minute
  max_actions_per_minute: 100

  context_compartments:
    - "crm-operations-context"

  geographic_constraints:
    - "US"
    - "EU"

  protocols_supported:
    - "MCP_1_0"
```

### T3 — Autonomous Agent (Data Pipeline Orchestrator)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "data-pipeline-orchestrator"
  version: "3.0.0"
  owning_team: "data-engineering"
  platform: "DATABRICKS"
spec:
  # T3: Autonomous multi-step, confidential data, multi-step writes
  risk_tier: "T3"

  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION

  declared_permissions:
    - resource: "snowflake://warehouse/analytics/raw_events"
      actions: ["SELECT"]
    - resource: "snowflake://warehouse/analytics/processed_events"
      actions: ["SELECT", "INSERT", "UPDATE"]
    - resource: "databricks://workspace/jobs/etl-pipeline"
      actions: ["RUN", "MONITOR"]

  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/databricks-jobs"

  approved_egress_endpoints:
    - "api.internal.company.com"

  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"

  # T3 default: gates on all non-trivial actions
  oversight_mode: "APPROVAL_REQUIRED"

  # T3 credential TTL: 15 minutes (mandatory maximum)
  max_session_duration_seconds: 900

  # T3 rate limit: 20 actions/minute (mandatory maximum)
  max_actions_per_minute: 20

  context_compartments:
    - "analytics-pipeline-context"

  geographic_constraints:
    - "US"

  # MCP for tools, A2A for delegating to sub-agents
  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

### T4 — Critical Agent (Payment Processing Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "payment-processing-agent"
  version: "2.0.1"
  owning_team: "treasury-operations"
  platform: "AWS"
spec:
  # T4: Restricted data, external systems, financial transactions
  risk_tier: "T4"

  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - EXTERNAL_CONNECTION

  declared_permissions:
    - resource: "rds://finance/payments/transactions"
      actions: ["SELECT", "INSERT", "UPDATE"]
    - resource: "s3://finance-restricted/audit-reports"
      actions: ["READ"]

  approved_mcp_servers:
    - "mcp://enterprise-catalog/payment-gateway"
    - "mcp://enterprise-catalog/finance-db"

  approved_egress_endpoints:
    - "payments.provider.com"
    - "api.internal.company.com"

  data_classifications_accessed:
    - "CONFIDENTIAL"
    - "RESTRICTED"

  # T4 default: APPROVAL_REQUIRED (cannot be FULL_AUTO without AI Gov Team authorization)
  oversight_mode: "APPROVAL_REQUIRED"

  # T4 credential TTL: 15 minutes (mandatory maximum)
  max_session_duration_seconds: 900

  # T4 rate limit: 20 actions/minute (mandatory maximum)
  max_actions_per_minute: 20

  context_compartments:
    - "payment-processing-context"
    - "finance-restricted-context"

  geographic_constraints:
    - "US"

  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

---

## Cross-References

- [Agent Identity and Registration Standard](../eaagf-specification/02-agent-identity-standard.md) — Normative specification
- [Risk Assessment Guide](./risk-assessment-guide.md) — How to determine your agent's Risk_Tier
- [Agent Registration Flow](../flows/agent-registration-flow.md) — Registration flow diagram
- [Credential Lifecycle Flow](../flows/credential-lifecycle-flow.md) — Credential rotation and revocation
- [Agent Lifecycle Flow](../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../reference/glossary.md) — EAAGF terminology
