# Microsoft Copilot Studio Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on Microsoft Copilot Studio to the EAAGF governance framework. It covers Power Platform connector integration, native MCP support, A2A compatibility shim setup, the gateway deployment pattern, and Copilot Studio-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. A Microsoft 365 tenant with Copilot Studio enabled.
2. Power Platform environment provisioned for your team.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. Azure AD app registration configured for the EAAGF gateway.

---

## Architecture Overview

Microsoft Copilot Studio provides **native MCP support** but requires an **A2A compatibility shim** for agent-to-agent delegation. The EAAGF Platform Adapter uses a **gateway deployment pattern** where a Power Platform custom connector routes agent actions through the Governance Controller.

```
┌──────────────────────────────────────────────────┐
│  Microsoft Copilot Studio                         │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Copilot Agent    │  │  Power Platform      │  │
│  │  (Agent Runtime)  │──┤  Custom Connector    │  │
│  │                   │  │  (Native MCP)        │  │
│  └──────────────────┘  └──────────┬───────────┘  │
└───────────────────────────────────┼──────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  EAAGF Gateway        │
                         │  (Platform Adapter)   │
                         │  ┌────────────────┐   │
                         │  │ A2A Shim        │   │
                         │  │ Identity Proxy  │   │
                         │  │ Policy Enforcer │   │
                         │  │ Telemetry Fwd   │   │
                         │  └────────────────┘   │
                         └──────────┬────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  Governance Controller│
                         └───────────────────────┘
```

---

## Step 1: Power Platform Connector Integration

### 1.1 Create the EAAGF Custom Connector

Create a Power Platform custom connector that routes Copilot Studio agent actions through the EAAGF gateway:

1. Navigate to **Power Platform Admin Center → Custom Connectors → New Custom Connector**.
2. Configure the connector:

```json
{
  "connectorName": "EAAGF Governance Gateway",
  "host": "eaagf-gateway.internal.company.com",
  "basePath": "/v1/copilot-studio",
  "schemes": ["https"],
  "authentication": {
    "type": "oauth2",
    "oauth2Settings": {
      "identityProvider": "aad",
      "clientId": "{{AZURE_AD_APP_CLIENT_ID}}",
      "clientSecret": "{{AZURE_AD_APP_CLIENT_SECRET}}",
      "authorizationUrl": "https://login.microsoftonline.com/{{TENANT_ID}}/oauth2/v2.0/authorize",
      "tokenUrl": "https://login.microsoftonline.com/{{TENANT_ID}}/oauth2/v2.0/token",
      "scopes": "api://eaagf-gateway/.default"
    }
  }
}
```

### 1.2 Define Connector Actions

Define the governance actions the connector exposes to Copilot Studio:

```json
{
  "actions": {
    "EvaluateAction": {
      "operationId": "evaluateAction",
      "summary": "Evaluate an agent action against EAAGF governance policies",
      "parameters": [
        { "name": "agent_id", "type": "string", "required": true },
        { "name": "action_type", "type": "string", "required": true },
        { "name": "target_resource", "type": "string", "required": true },
        { "name": "input_summary", "type": "string", "required": true }
      ],
      "responses": {
        "200": {
          "schema": {
            "type": "object",
            "properties": {
              "outcome": { "type": "string", "enum": ["PERMITTED", "DENIED", "GATED"] },
              "reason_code": { "type": "string" }
            }
          }
        }
      }
    }
  }
}
```

### 1.3 Configure the Copilot Agent to Use the Connector

In Copilot Studio, add the EAAGF custom connector as a plugin action:

1. Open your Copilot agent in Copilot Studio.
2. Navigate to **Actions → Add an action → Custom connector**.
3. Select the **EAAGF Governance Gateway** connector.
4. Map the agent's action parameters to the connector's input fields.

---

## Step 2: Native MCP Configuration

Copilot Studio supports MCP natively. Configure MCP connections through the Copilot Studio agent settings.

### 2.1 Register MCP Servers

In Copilot Studio, add MCP server connections:

1. Navigate to **Agent Settings → Connections → MCP Servers**.
2. Add each enterprise MCP server your agent needs:

```json
{
  "mcp_server_uri": "mcp://enterprise-catalog/sharepoint-kb",
  "display_name": "SharePoint Knowledge Base",
  "authentication": {
    "type": "azure_ad",
    "tenant_id": "{{TENANT_ID}}",
    "client_id": "{{MCP_CLIENT_ID}}"
  },
  "governance_routing": {
    "enabled": true,
    "gateway_endpoint": "https://eaagf-gateway.internal.company.com/v1/mcp"
  }
}
```

### 2.2 Governance Routing for MCP

All MCP tool calls are routed through the EAAGF gateway by enabling `governance_routing`. This ensures the Governance Controller validates every tool call against the enterprise MCP directory and the agent's Conformance_Profile.

---

## Step 3: A2A Compatibility Shim

Copilot Studio does not natively support A2A. The EAAGF gateway provides a compatibility shim.

### 3.1 A2A Shim via Custom Connector

Add an A2A delegation action to the EAAGF custom connector:

```json
{
  "actions": {
    "DelegateTask": {
      "operationId": "delegateTask",
      "summary": "Delegate a sub-task to another agent via A2A protocol",
      "parameters": [
        { "name": "target_agent_id", "type": "string", "required": true },
        { "name": "sub_task_description", "type": "string", "required": true },
        { "name": "permitted_actions", "type": "array", "required": true },
        { "name": "permitted_resources", "type": "array", "required": true },
        { "name": "max_risk_tier", "type": "string", "required": true }
      ],
      "responses": {
        "200": {
          "schema": {
            "type": "object",
            "properties": {
              "delegation_id": { "type": "string" },
              "status": { "type": "string" },
              "result": { "type": "object" }
            }
          }
        }
      }
    }
  }
}
```

### 3.2 Using A2A Delegation in Copilot Studio

In your Copilot agent's topic flow, add a **Call an action** node that invokes the `DelegateTask` action:

1. Create a new topic or edit an existing one.
2. Add a **Call an action** node → select **EAAGF Governance Gateway → DelegateTask**.
3. Map the delegation parameters from the conversation context.

The gateway automatically:
- Creates and signs an Agent Card with the Copilot agent's identity.
- Validates combined Risk_Tier limits.
- Routes the delegation through the Governance Controller.
- Returns the delegation result to the Copilot agent.

---

## Step 4: Gateway Deployment

### 4.1 Gateway Configuration

```yaml
# eaagf-copilot-studio-gateway.yaml
gateway:
  platform: "COPILOT_STUDIO"
  governance_controller_endpoint: "https://governance.internal.company.com"

  copilot_studio:
    tenant_id: "${AZURE_TENANT_ID}"
    environment_id: "${POWER_PLATFORM_ENV_ID}"
    authentication:
      type: "azure_ad"
      client_id: "${AZURE_AD_APP_CLIENT_ID}"
      client_secret: "${AZURE_AD_APP_CLIENT_SECRET}"

  mcp:
    native_support: true
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300

  a2a_shim:
    enabled: true
    version: "A2A_1_0"
    agent_card_signing_key_path: "/etc/eaagf/signing-key.pem"

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-copilot-studio-gateway"
```

### 4.2 Deploy the Gateway

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eaagf-copilot-studio-gateway
  namespace: eaagf-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eaagf-copilot-studio-gateway
  template:
    metadata:
      labels:
        app: eaagf-copilot-studio-gateway
    spec:
      containers:
        - name: gateway
          image: eaagf-registry.internal.company.com/copilot-studio-gateway:latest
          ports:
            - containerPort: 8080
            - containerPort: 8081
          envFrom:
            - secretRef:
                name: eaagf-copilot-studio-secrets
          volumeMounts:
            - name: gateway-config
              mountPath: /etc/eaagf
      volumes:
        - name: gateway-config
          configMap:
            name: eaagf-copilot-studio-gateway-config
```

---

## Step 5: Copilot Studio-Specific Conformance Profile Examples

### T1 Copilot Studio Agent (Internal FAQ Bot)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "internal-faq-bot"
  version: "1.0.0"
  owning_team: "employee-experience"
  platform: "COPILOT_STUDIO"
spec:
  risk_tier: "T1"
  capabilities:
    - TOOL_CALL
    - DATA_READ
  declared_permissions:
    - resource: "sharepoint://sites/knowledge-base/articles"
      actions: ["READ"]
    - resource: "graph://users/me/profile"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/sharepoint-kb"
  data_classifications_accessed:
    - "PUBLIC"
    - "INTERNAL"
  oversight_mode: "HUMAN_IN_LOOP"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  protocols_supported:
    - "MCP_1_0"
```

### T2 Copilot Studio Agent (IT Helpdesk Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "it-helpdesk-agent"
  version: "2.0.0"
  owning_team: "it-support"
  platform: "COPILOT_STUDIO"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "servicenow://incident"
      actions: ["READ", "CREATE", "UPDATE"]
    - resource: "graph://users"
      actions: ["READ"]
    - resource: "sharepoint://sites/it-kb/articles"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/servicenow"
    - "mcp://enterprise-catalog/sharepoint-kb"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "it-helpdesk-context"
  geographic_constraints:
    - "US"
    - "EU"
  protocols_supported:
    - "MCP_1_0"
```

### T3 Copilot Studio Agent (HR Process Automation)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "hr-process-agent"
  version: "1.0.0"
  owning_team: "human-resources"
  platform: "COPILOT_STUDIO"
spec:
  risk_tier: "T3"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION
  declared_permissions:
    - resource: "workday://employees"
      actions: ["READ", "UPDATE"]
    - resource: "sharepoint://sites/hr-policies"
      actions: ["READ"]
    - resource: "graph://users"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/workday-hr"
    - "mcp://enterprise-catalog/sharepoint-kb"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "hr-process-context"
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
- [ ] Power Platform custom connector is created and tested.
- [ ] MCP connections are routed through the EAAGF gateway.
- [ ] A2A compatibility shim is configured via the custom connector (if delegation is needed).
- [ ] Azure AD app registration has minimum required permissions.
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
