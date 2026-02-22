# Conformance Profile Examples

> Companion to [`conformance-profile-template.json`](./conformance-profile-template.json).
> Each example below shows a valid Conformance Profile for a different capability combination.

## Example 1: Read-Only Tool Agent (T1)

Minimal profile — agent only reads data via MCP tools. No writes, no delegation, no external connections.

```json
{
  "schema_version": "1.0",
  "agent_id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "capabilities": ["TOOL_CALL", "DATA_READ"],
  "declared_permissions": [
    { "resource": "salesforce://sobject/Knowledge__kav", "actions": ["READ"] }
  ],
  "approved_mcp_servers": ["mcp://enterprise-catalog/salesforce-knowledge"],
  "approved_egress_endpoints": [],
  "data_classifications_accessed": ["PUBLIC"],
  "oversight_mode": "HUMAN_IN_LOOP",
  "max_session_duration_seconds": 3600,
  "max_actions_per_minute": 100,
  "context_compartments": [],
  "geographic_constraints": [],
  "protocols_supported": ["MCP_1_0"]
}
```

## Example 2: Read + Write Agent with Compartments (T2)

Agent reads and writes internal/confidential data. Requires a context compartment for confidential access.

```json
{
  "schema_version": "1.0",
  "agent_id": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "capabilities": ["TOOL_CALL", "DATA_READ", "DATA_WRITE"],
  "declared_permissions": [
    { "resource": "snowflake://analytics/sales/forecast", "actions": ["SELECT", "INSERT"] },
    { "resource": "salesforce://sobject/Opportunity", "actions": ["READ"] }
  ],
  "approved_mcp_servers": [
    "mcp://enterprise-catalog/snowflake-query",
    "mcp://enterprise-catalog/salesforce-crm"
  ],
  "approved_egress_endpoints": [],
  "data_classifications_accessed": ["INTERNAL", "CONFIDENTIAL"],
  "oversight_mode": "SUPERVISED",
  "max_session_duration_seconds": 3600,
  "max_actions_per_minute": 100,
  "context_compartments": ["sales-analytics-context"],
  "geographic_constraints": ["US", "EU"],
  "protocols_supported": ["MCP_1_0"]
}
```

## Example 3: Autonomous Agent with Delegation (T3)

Agent operates autonomously, delegates sub-tasks to other agents via A2A, and accesses confidential data.

```json
{
  "schema_version": "1.0",
  "agent_id": "c3d4e5f6-a7b8-4c9d-0e1f-2a3b4c5d6e7f",
  "capabilities": ["TOOL_CALL", "DATA_READ", "DATA_WRITE", "AGENT_DELEGATION"],
  "declared_permissions": [
    { "resource": "azure://sql/hr-db/employees", "actions": ["SELECT"] },
    { "resource": "azure://blob/compliance-reports", "actions": ["READ", "WRITE"] }
  ],
  "approved_mcp_servers": [
    "mcp://enterprise-catalog/azure-sql",
    "mcp://enterprise-catalog/azure-blob"
  ],
  "approved_egress_endpoints": [],
  "data_classifications_accessed": ["INTERNAL", "CONFIDENTIAL"],
  "oversight_mode": "APPROVAL_REQUIRED",
  "max_session_duration_seconds": 900,
  "max_actions_per_minute": 20,
  "context_compartments": ["hr-compliance-context"],
  "geographic_constraints": ["EU"],
  "protocols_supported": ["MCP_1_0", "A2A_1_0"]
}
```

## Example 4: Critical Agent with External Connections (T4)

Full capability set — reads/writes restricted data, delegates to agents, and connects to external systems. Strictest governance controls.

```json
{
  "schema_version": "1.0",
  "agent_id": "d4e5f6a7-b8c9-4d0e-1f2a-3b4c5d6e7f80",
  "capabilities": ["TOOL_CALL", "DATA_READ", "DATA_WRITE", "AGENT_DELEGATION", "EXTERNAL_CONNECTION"],
  "declared_permissions": [
    { "resource": "aws://dynamodb/trade-orders", "actions": ["READ", "WRITE", "DELETE"] },
    { "resource": "aws://s3/market-data-restricted", "actions": ["READ"] },
    { "resource": "snowflake://finance/positions/current", "actions": ["SELECT", "INSERT", "UPDATE"] }
  ],
  "approved_mcp_servers": [
    "mcp://enterprise-catalog/aws-dynamodb",
    "mcp://enterprise-catalog/aws-s3",
    "mcp://enterprise-catalog/snowflake-query"
  ],
  "approved_egress_endpoints": [
    "api.marketdata-provider.com",
    "fix.exchange-gateway.com"
  ],
  "data_classifications_accessed": ["CONFIDENTIAL", "RESTRICTED"],
  "oversight_mode": "APPROVAL_REQUIRED",
  "max_session_duration_seconds": 900,
  "max_actions_per_minute": 20,
  "context_compartments": ["trading-restricted-context", "market-data-context"],
  "geographic_constraints": ["US"],
  "protocols_supported": ["MCP_1_0", "A2A_1_0"]
}
```

## Validation Quick Reference

| Rule | Condition |
|---|---|
| `schema_version` | Must be `"1.0"` |
| `agent_id` | Valid UUID v4 |
| `capabilities` | At least one value from allowed set |
| `AGENT_DELEGATION` in capabilities | `protocols_supported` must include `A2A_1_0` |
| `TOOL_CALL` in capabilities | `approved_mcp_servers` must be non-empty |
| `EXTERNAL_CONNECTION` in capabilities | `approved_egress_endpoints` must be non-empty |
| `CONFIDENTIAL` or `RESTRICTED` in data classifications | `context_compartments` must be non-empty |
| T4 agent | `oversight_mode` cannot be `FULL_AUTO` |
| T1/T2 `max_session_duration_seconds` | ≤ 3600 |
| T3/T4 `max_session_duration_seconds` | ≤ 900 |
| T1/T2 `max_actions_per_minute` | ≤ 100 |
| T3/T4 `max_actions_per_minute` | ≤ 20 |
