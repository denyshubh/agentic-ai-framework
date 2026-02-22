# Databricks Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on Databricks to the EAAGF governance framework. It covers MLflow Model Serving integration, Unity Catalog event hooks, the Kubernetes sidecar deployment pattern, MCP/A2A compatibility shim setup, and Databricks-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. A Databricks workspace with MLflow Model Serving enabled.
2. Unity Catalog configured for your workspace.
3. A Kubernetes cluster (or Databricks-managed K8s) for sidecar deployment.
4. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
5. Access to the EAAGF Governance Controller endpoint.

---

## Architecture Overview

Databricks does not natively support MCP or A2A. The EAAGF Platform Adapter for Databricks uses a **Kubernetes sidecar deployment pattern** where a governance sidecar container runs alongside the agent's MLflow Model Serving endpoint. The sidecar intercepts all agent actions, translates them into EAAGF-normalized requests, and enforces governance decisions.

```
┌─────────────────────────────────────────────────┐
│  Kubernetes Pod                                  │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │  MLflow Model     │  │  EAAGF Governance    │ │
│  │  Serving Endpoint │◄─┤  Sidecar             │ │
│  │  (Agent Runtime)  │  │  ┌────────────────┐  │ │
│  │                   │  │  │ MCP Shim        │  │ │
│  │                   │  │  │ A2A Shim        │  │ │
│  │                   │  │  │ Identity Proxy  │  │ │
│  │                   │  │  │ Telemetry Fwd   │  │ │
│  └──────────────────┘  │  └────────────────┘  │ │
│                         └──────────┬───────────┘ │
└────────────────────────────────────┼─────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │  Governance Controller│
                          │  (Policy Engine,      │
                          │   Agent Registry,     │
                          │   Telemetry Emitter)  │
                          └───────────────────────┘
```

---

## Step 1: MLflow Model Serving Integration

### 1.1 Register Your Agent Endpoint

Your Databricks agent runs as an MLflow Model Serving endpoint. The EAAGF sidecar intercepts requests to and from this endpoint.

Configure your MLflow model to expose a standard prediction interface:

```python
# mlflow_agent_model.py
import mlflow

class EAAGFAgentModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        """
        Agent entry point. The EAAGF sidecar intercepts
        this call and applies governance checks before
        the agent processes the input.
        """
        # Agent reasoning logic here
        return agent_response
```

### 1.2 Configure the Serving Endpoint

When creating the MLflow Model Serving endpoint, set the environment variables that the EAAGF sidecar uses to connect to the Governance Controller:

```json
{
  "name": "my-agent-endpoint",
  "config": {
    "served_models": [
      {
        "model_name": "my-agent",
        "model_version": "1",
        "workload_size": "Small",
        "environment_vars": {
          "EAAGF_AGENT_ID": "{{secrets/eaagf/agent-id}}",
          "EAAGF_GOVERNANCE_ENDPOINT": "https://governance.internal.company.com",
          "EAAGF_CREDENTIAL_PATH": "{{secrets/eaagf/credential}}"
        }
      }
    ]
  }
}
```

---

## Step 2: Unity Catalog Event Hooks

Unity Catalog provides event hooks that the EAAGF sidecar uses to enforce data governance controls at the catalog level.

### 2.1 Configure Audit Logging

Enable Unity Catalog audit logging so that all data access events are captured and forwarded to the EAAGF Telemetry Emitter:

```sql
-- Enable audit logging for the workspace
-- (Typically configured via Databricks account admin)
-- The EAAGF sidecar reads these events from the audit log table

-- Verify audit log access
SELECT * FROM system.access.audit
WHERE action_name IN ('getTable', 'readTable', 'writeTable')
  AND user_identity.email LIKE '%agent%'
ORDER BY event_time DESC
LIMIT 10;
```

### 2.2 Data Classification Tags

Use Unity Catalog tags to map data assets to EAAGF classification levels. The sidecar resolves these tags at runtime:

```sql
-- Tag tables with EAAGF data classification
ALTER TABLE analytics.sales.customers
SET TAGS ('eaagf_classification' = 'CONFIDENTIAL');

ALTER TABLE analytics.public.product_catalog
SET TAGS ('eaagf_classification' = 'PUBLIC');

ALTER TABLE finance.restricted.transactions
SET TAGS ('eaagf_classification' = 'RESTRICTED');
```

The sidecar queries these tags before each data access and enforces the classification controls defined in the [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md).

### 2.3 Row-Level and Column-Level Security

For Confidential and Restricted data, configure Unity Catalog row filters and column masks that the sidecar enforces:

```sql
-- Create a row filter function for agent access
CREATE FUNCTION analytics.governance.agent_row_filter(region STRING)
RETURN IF(region IN ('US', 'EU'), true, false);

-- Apply the filter to a table
ALTER TABLE analytics.sales.customers
SET ROW FILTER analytics.governance.agent_row_filter ON (region);
```

---

## Step 3: Kubernetes Sidecar Deployment

The EAAGF governance sidecar runs as a container alongside your agent in the same Kubernetes pod. It intercepts all network traffic and applies governance controls.

### 3.1 Sidecar Container Configuration

Add the EAAGF sidecar to your agent's Kubernetes pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: databricks-agent-pod
  labels:
    eaagf.io/managed: "true"
    eaagf.io/agent-id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    eaagf.io/risk-tier: "T2"
spec:
  containers:
    # Your agent container
    - name: agent
      image: your-registry/databricks-agent:1.0.0
      ports:
        - containerPort: 8080
      env:
        - name: EAAGF_SIDECAR_URL
          value: "http://localhost:9090"

    # EAAGF governance sidecar
    - name: eaagf-sidecar
      image: eaagf-registry.internal.company.com/governance-sidecar:latest
      ports:
        - containerPort: 9090
      env:
        - name: EAAGF_AGENT_ID
          valueFrom:
            secretKeyRef:
              name: eaagf-credentials
              key: agent-id
        - name: EAAGF_GOVERNANCE_ENDPOINT
          value: "https://governance.internal.company.com"
        - name: EAAGF_PLATFORM
          value: "DATABRICKS"
        - name: EAAGF_MCP_SHIM_ENABLED
          value: "true"
        - name: EAAGF_A2A_SHIM_ENABLED
          value: "true"
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "200m"
      livenessProbe:
        httpGet:
          path: /health
          port: 9090
        initialDelaySeconds: 5
        periodSeconds: 10
```

### 3.2 Network Policy

Configure a Kubernetes NetworkPolicy to ensure all agent egress traffic routes through the sidecar:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: eaagf-agent-egress
spec:
  podSelector:
    matchLabels:
      eaagf.io/managed: "true"
  policyTypes:
    - Egress
  egress:
    # Allow traffic to the Governance Controller
    - to:
        - namespaceSelector:
            matchLabels:
              name: eaagf-system
      ports:
        - port: 443
    # Allow traffic to approved egress endpoints
    # (managed by the sidecar's allowlist)
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - port: 443
```

---

## Step 4: MCP Compatibility Shim

Databricks does not natively support MCP. The EAAGF sidecar includes an MCP compatibility shim that translates between Databricks-native tool calls and MCP protocol.

### 4.1 How the MCP Shim Works

1. Your agent makes a Databricks-native tool call (e.g., via MLflow Model Serving or Databricks SDK).
2. The sidecar intercepts the call and translates it into an MCP tool call request.
3. The sidecar forwards the MCP request to the Governance Controller for policy evaluation.
4. If permitted, the sidecar routes the call to the target MCP server (or translates back to the Databricks-native API).
5. The response is translated back to the Databricks-native format and returned to the agent.

### 4.2 MCP Shim Configuration

Configure the MCP shim in the sidecar's configuration file:

```yaml
# eaagf-sidecar-config.yaml
mcp_shim:
  enabled: true
  version: "MCP_1_0"
  tool_manifest_path: "/etc/eaagf/tool-manifest.json"
  enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
  refresh_interval_seconds: 300
  databricks_mappings:
    # Map Databricks-native operations to MCP tool calls
    - databricks_operation: "mlflow.deployments.predict"
      mcp_tool: "mcp://enterprise-catalog/databricks-inference"
    - databricks_operation: "spark.sql"
      mcp_tool: "mcp://enterprise-catalog/snowflake-query"
    - databricks_operation: "dbutils.fs.ls"
      mcp_tool: "mcp://enterprise-catalog/databricks-files"
```

### 4.3 Declaring MCP Servers in Your Manifest

In your agent manifest, list the MCP servers your agent needs:

```yaml
spec:
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/databricks-jobs"
    - "mcp://enterprise-catalog/databricks-files"
  protocols_supported:
    - "MCP_1_0"
```

---

## Step 5: A2A Compatibility Shim

If your Databricks agent delegates tasks to other agents, the A2A compatibility shim handles the protocol translation.

### 5.1 A2A Shim Configuration

```yaml
# eaagf-sidecar-config.yaml (continued)
a2a_shim:
  enabled: true
  version: "A2A_1_0"
  agent_card_signing_key_path: "/etc/eaagf/signing-key.pem"
  delegation_timeout_seconds: 300
  max_delegation_depth: 3
```

### 5.2 Delegating to Other Agents

When your Databricks agent needs to delegate a sub-task, it calls the sidecar's delegation endpoint:

```python
import requests

delegation_request = {
    "target_agent_id": "target-agent-uuid",
    "sub_task_description": "Run anomaly detection on Q4 sales data",
    "permitted_actions": ["TOOL_CALL", "DATA_READ"],
    "permitted_resources": ["snowflake://analytics/sales/q4_data"],
    "max_risk_tier": "T2"
}

response = requests.post(
    "http://localhost:9090/a2a/delegate",
    json=delegation_request
)
```

The sidecar automatically:
- Creates and signs an Agent Card with the delegating agent's identity.
- Validates the combined Risk_Tier does not exceed the task context maximum.
- Routes the delegation through the Governance Controller for policy evaluation.
- Emits an `A2A_DELEGATION` audit event.

---

## Step 6: Databricks-Specific Conformance Profile

### T2 Databricks Agent Example

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "sales-forecast-agent"
  version: "1.2.0"
  owning_team: "revenue-analytics"
  platform: "DATABRICKS"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "snowflake://analytics/sales/forecast"
      actions: ["SELECT", "INSERT"]
    - resource: "databricks://workspace/jobs/forecast-pipeline"
      actions: ["RUN", "MONITOR"]
    - resource: "unity-catalog://analytics.sales.customers"
      actions: ["SELECT"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/databricks-jobs"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "sales-analytics-context"
  geographic_constraints:
    - "US"
    - "EU"
  protocols_supported:
    - "MCP_1_0"
```

### T3 Databricks Agent Example (Autonomous Pipeline Orchestrator)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "data-pipeline-orchestrator"
  version: "3.0.0"
  owning_team: "data-engineering"
  platform: "DATABRICKS"
spec:
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
    - resource: "unity-catalog://analytics.events.*"
      actions: ["SELECT", "INSERT"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/databricks-jobs"
    - "mcp://enterprise-catalog/databricks-files"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "analytics-pipeline-context"
  geographic_constraints:
    - "US"
  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

---

## Validation Checklist

Before requesting promotion to STAGING, verify:

- [ ] Agent manifest passes schema validation against the EAAGF conformance schema.
- [ ] EAAGF sidecar is deployed and healthy (`/health` endpoint returns 200).
- [ ] MCP shim correctly translates Databricks-native tool calls to MCP format.
- [ ] Unity Catalog tables are tagged with `eaagf_classification` labels.
- [ ] Network policy restricts egress to approved endpoints only.
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] For T3/T4: A2A shim is configured and Agent Card signing key is provisioned.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
