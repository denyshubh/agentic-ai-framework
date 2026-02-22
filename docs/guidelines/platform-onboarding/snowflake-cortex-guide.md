# Snowflake Cortex Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on Snowflake Cortex to the EAAGF governance framework. It covers Cortex Agent API integration, Snowpark hooks, the gateway deployment pattern, MCP/A2A compatibility shim setup, and Snowflake-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. A Snowflake account with Cortex AI features enabled.
2. Snowpark configured for your account.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. A Snowflake security integration configured for OAuth 2.0 with the enterprise IdP.

---

## Architecture Overview

Snowflake Cortex does not natively support MCP or A2A. The EAAGF Platform Adapter for Snowflake uses a **gateway deployment pattern** where an external gateway service intercepts Cortex Agent API calls via Snowpark external functions and applies governance controls.

```
┌──────────────────────────────────────────────────┐
│  Snowflake                                        │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Cortex Agent     │  │  Snowpark External   │  │
│  │  (Agent Runtime)  │──┤  Function Hooks      │  │
│  │                   │  │                      │  │
│  └──────────────────┘  └──────────┬───────────┘  │
└───────────────────────────────────┼──────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  EAAGF Gateway        │
                         │  (Platform Adapter)   │
                         │  ┌────────────────┐   │
                         │  │ MCP Shim        │   │
                         │  │ A2A Shim        │   │
                         │  │ Identity Proxy  │   │
                         │  │ Telemetry Fwd   │   │
                         │  └────────────────┘   │
                         └──────────┬────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  Governance Controller│
                         └───────────────────────┘
```

---

## Step 1: Cortex Agent API Integration

### 1.1 Create the Cortex Agent

Define your Cortex agent using the Cortex Agent API:

```sql
CREATE OR REPLACE CORTEX AGENT my_agent
  MODEL = 'llama3.1-70b'
  TOOLS = (
    my_search_tool,
    my_analyst_tool
  )
  COMMENT = 'EAAGF-managed agent: sales-analysis-agent';
```

### 1.2 Configure the EAAGF Security Integration

Create a Snowflake security integration that allows the EAAGF gateway to authenticate:

```sql
CREATE OR REPLACE SECURITY INTEGRATION eaagf_gateway_integration
  TYPE = API_AUTHENTICATION
  AUTH_TYPE = OAUTH2
  OAUTH_CLIENT_ID = '{{EAAGF_GATEWAY_CLIENT_ID}}'
  OAUTH_CLIENT_SECRET = '{{EAAGF_GATEWAY_CLIENT_SECRET}}'
  OAUTH_TOKEN_ENDPOINT = 'https://auth.internal.company.com/token'
  ENABLED = TRUE;
```

### 1.3 Create the API Integration for the Gateway

```sql
CREATE OR REPLACE API INTEGRATION eaagf_gateway_api
  API_PROVIDER = 'PRIVATE'
  API_ALLOWED_PREFIXES = ('https://eaagf-gateway.internal.company.com/')
  ENABLED = TRUE;
```

---

## Step 2: Snowpark External Function Hooks

Snowpark external functions serve as the bridge between Cortex agents and the EAAGF gateway. Every agent action is routed through an external function that calls the gateway for governance evaluation.

### 2.1 Create the Governance Hook Function

```sql
CREATE OR REPLACE EXTERNAL FUNCTION eaagf_governance_hook(
  agent_id VARCHAR,
  action_type VARCHAR,
  target_resource VARCHAR,
  input_summary VARCHAR
)
RETURNS VARIANT
API_INTEGRATION = eaagf_gateway_api
HEADERS = (
  'x-eaagf-platform' = 'SNOWFLAKE',
  'x-eaagf-correlation-id' = '{correlation_id}'
)
AS 'https://eaagf-gateway.internal.company.com/v1/snowflake/evaluate';
```

### 2.2 Create Governance-Wrapped Tool Functions

Wrap each Cortex agent tool with a governance check:

```sql
-- Original tool function
CREATE OR REPLACE FUNCTION sales_data_query(query_text VARCHAR)
RETURNS TABLE (result VARIANT)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'SalesDataQuery.run'
AS $$
class SalesDataQuery:
    def run(self, query_text):
        # Query logic here
        pass
$$;

-- Governance-wrapped version
CREATE OR REPLACE FUNCTION governed_sales_data_query(
  agent_id VARCHAR,
  query_text VARCHAR
)
RETURNS TABLE (result VARIANT)
AS
$$
  -- Step 1: Check governance
  WITH governance_decision AS (
    SELECT eaagf_governance_hook(
      agent_id,
      'TOOL_CALL',
      'snowflake://analytics/sales',
      LEFT(query_text, 1024)
    ) AS decision
  )
  -- Step 2: Execute only if permitted
  SELECT * FROM TABLE(sales_data_query(query_text))
  WHERE (SELECT decision:outcome::STRING FROM governance_decision) = 'PERMITTED'
$$;
```

### 2.3 Event Table for Audit Logging

Configure a Snowflake event table to capture agent actions for EAAGF telemetry:

```sql
CREATE OR REPLACE EVENT TABLE eaagf_agent_events
  DATA_RETENTION_TIME_IN_DAYS = 2555  -- 7 years
  CHANGE_TRACKING = TRUE;

-- Grant the gateway access to write events
GRANT INSERT ON EVENT TABLE eaagf_agent_events TO ROLE eaagf_gateway_role;
```

---

## Step 3: Gateway Deployment

### 3.1 Gateway Configuration

```yaml
# eaagf-snowflake-gateway.yaml
gateway:
  platform: "SNOWFLAKE"
  governance_controller_endpoint: "https://governance.internal.company.com"

  snowflake:
    account: "your-account.snowflakecomputing.com"
    warehouse: "EAAGF_GOVERNANCE_WH"
    database: "EAAGF_GOVERNANCE"
    schema: "PUBLIC"
    authentication:
      type: "oauth2"
      client_id: "${SNOWFLAKE_CLIENT_ID}"
      client_secret: "${SNOWFLAKE_CLIENT_SECRET}"
      token_endpoint: "https://your-account.snowflakecomputing.com/oauth/token-request"

  mcp_shim:
    enabled: true
    version: "MCP_1_0"
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300
    snowflake_mappings:
      - snowflake_operation: "CORTEX_AGENT_TOOL_CALL"
        mcp_tool: "mcp://enterprise-catalog/snowflake-query"
      - snowflake_operation: "EXTERNAL_FUNCTION_CALL"
        mcp_tool: "mcp://enterprise-catalog/snowflake-external"

  a2a_shim:
    enabled: true
    version: "A2A_1_0"
    agent_card_signing_key_path: "/etc/eaagf/signing-key.pem"

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-snowflake-gateway"
```

### 3.2 Deploy the Gateway

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eaagf-snowflake-gateway
  namespace: eaagf-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eaagf-snowflake-gateway
  template:
    metadata:
      labels:
        app: eaagf-snowflake-gateway
    spec:
      containers:
        - name: gateway
          image: eaagf-registry.internal.company.com/snowflake-gateway:latest
          ports:
            - containerPort: 8080
            - containerPort: 8081
          envFrom:
            - secretRef:
                name: eaagf-snowflake-secrets
          volumeMounts:
            - name: gateway-config
              mountPath: /etc/eaagf
      volumes:
        - name: gateway-config
          configMap:
            name: eaagf-snowflake-gateway-config
```

---

## Step 4: MCP Compatibility Shim

### 4.1 How the MCP Shim Works

1. The Cortex agent invokes a tool via the Cortex Agent API.
2. The Snowpark external function hook routes the call to the EAAGF gateway.
3. The gateway's MCP shim translates the Snowflake-native call into an MCP tool call request.
4. The Governance Controller evaluates the MCP request against policies.
5. If permitted, the gateway routes the call to the target MCP server or back to Snowflake.
6. The response is translated back to the Snowflake-native format.

### 4.2 Declaring MCP Servers in Your Manifest

```yaml
spec:
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/snowflake-cortex"
  protocols_supported:
    - "MCP_1_0"
```

---

## Step 5: A2A Compatibility Shim

### 5.1 Delegating to Other Agents

When a Cortex agent needs to delegate a sub-task, it calls the gateway's A2A endpoint via a Snowpark external function:

```sql
CREATE OR REPLACE EXTERNAL FUNCTION eaagf_delegate_task(
  target_agent_id VARCHAR,
  sub_task_description VARCHAR,
  permitted_actions ARRAY,
  permitted_resources ARRAY,
  max_risk_tier VARCHAR
)
RETURNS VARIANT
API_INTEGRATION = eaagf_gateway_api
AS 'https://eaagf-gateway.internal.company.com/v1/a2a/delegate';
```

Usage from within a Cortex agent tool:

```sql
SELECT eaagf_delegate_task(
  'target-agent-uuid',
  'Analyze Q4 revenue trends from CRM data',
  ARRAY_CONSTRUCT('TOOL_CALL', 'DATA_READ'),
  ARRAY_CONSTRUCT('salesforce://sobject/Opportunity'),
  'T2'
);
```

The gateway automatically creates and signs the Agent Card, validates combined Risk_Tier limits, and emits audit events.

---

## Step 6: Snowflake-Specific Conformance Profile Examples

### T2 Snowflake Agent (Analytics Query Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "analytics-query-agent"
  version: "1.0.0"
  owning_team: "business-intelligence"
  platform: "SNOWFLAKE"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "snowflake://analytics/sales/revenue"
      actions: ["SELECT"]
    - resource: "snowflake://analytics/sales/forecasts"
      actions: ["SELECT", "INSERT"]
    - resource: "snowflake://analytics/reports/output"
      actions: ["INSERT"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "analytics-query-context"
  geographic_constraints:
    - "US"
  protocols_supported:
    - "MCP_1_0"
```

### T3 Snowflake Agent (Data Pipeline Orchestrator)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "cortex-pipeline-agent"
  version: "2.0.0"
  owning_team: "data-engineering"
  platform: "SNOWFLAKE"
spec:
  risk_tier: "T3"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION
  declared_permissions:
    - resource: "snowflake://warehouse/raw/events"
      actions: ["SELECT"]
    - resource: "snowflake://warehouse/processed/events"
      actions: ["SELECT", "INSERT", "UPDATE"]
    - resource: "snowflake://warehouse/staging/temp"
      actions: ["SELECT", "INSERT", "DELETE"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/snowflake-cortex"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "data-pipeline-context"
  geographic_constraints:
    - "US"
    - "EU"
  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

---

## Validation Checklist

Before requesting promotion to STAGING, verify:

- [ ] Agent manifest passes schema validation against the EAAGF conformance schema.
- [ ] EAAGF gateway is deployed and healthy (`/health` endpoint returns 200).
- [ ] Snowpark external function hooks are created and functional.
- [ ] API integration and security integration are configured correctly.
- [ ] MCP shim correctly translates Snowflake-native calls to MCP format.
- [ ] Event table is created with 7-year retention for audit logging.
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] For T3/T4: A2A delegation external function is configured.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
