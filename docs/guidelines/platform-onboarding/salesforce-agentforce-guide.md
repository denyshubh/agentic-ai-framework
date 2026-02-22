# Salesforce AgentForce Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on Salesforce AgentForce to the EAAGF governance framework. It covers AgentForce Command Center API integration, native MCP and A2A support configuration, the gateway deployment pattern, and Salesforce-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. A Salesforce org with AgentForce enabled.
2. Access to the AgentForce Command Center API.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. Salesforce Connected App configured for OAuth 2.0 integration with the enterprise IdP.

---

## Architecture Overview

Salesforce AgentForce provides **native MCP and A2A support**, making it one of the most straightforward platforms to onboard. The EAAGF Platform Adapter for Salesforce uses a **gateway deployment pattern** where the Governance Controller sits between AgentForce and external resources, intercepting actions via the AgentForce Command Center API.

```
┌──────────────────────────────────────────────────┐
│  Salesforce AgentForce                            │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Agent Runtime    │  │  AgentForce Command  │  │
│  │  (AgentForce)     │──┤  Center API          │  │
│  │                   │  │  (Native MCP + A2A)  │  │
│  └──────────────────┘  └──────────┬───────────┘  │
└───────────────────────────────────┼──────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │  EAAGF Gateway        │
                         │  (Platform Adapter)   │
                         │  ┌────────────────┐   │
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

## Step 1: AgentForce Command Center API Integration

### 1.1 Register the EAAGF Gateway as a Connected App

Create a Salesforce Connected App that the EAAGF gateway uses to authenticate with the AgentForce Command Center API:

1. Navigate to **Setup → App Manager → New Connected App**.
2. Configure OAuth settings:
   - **Callback URL:** `https://eaagf-gateway.internal.company.com/oauth/callback`
   - **Selected OAuth Scopes:** `api`, `agentforce_api`, `refresh_token`
3. Enable **Client Credentials Flow** for server-to-server communication.
4. Note the `client_id` and `client_secret` for gateway configuration.

### 1.2 Configure the Command Center Webhook

Register the EAAGF gateway as a webhook listener in the AgentForce Command Center so that all agent actions are routed through governance:

```json
{
  "webhook_name": "EAAGF Governance Gateway",
  "endpoint": "https://eaagf-gateway.internal.company.com/v1/salesforce/actions",
  "events": [
    "agent.action.requested",
    "agent.action.completed",
    "agent.delegation.requested",
    "agent.lifecycle.changed"
  ],
  "authentication": {
    "type": "oauth2_client_credentials",
    "token_endpoint": "https://login.salesforce.com/services/oauth2/token",
    "client_id": "{{CONNECTED_APP_CLIENT_ID}}",
    "client_secret": "{{CONNECTED_APP_CLIENT_SECRET}}"
  }
}
```

### 1.3 Action Interception

The gateway intercepts every `agent.action.requested` event, translates it into an EAAGF normalized action request, and forwards it to the Governance Controller for policy evaluation. The gateway enforces the PERMIT/DENY/GATE decision before allowing the action to proceed in Salesforce.

---

## Step 2: Native MCP Configuration

Salesforce AgentForce supports MCP natively. Configure MCP connections through the AgentForce Command Center.

### 2.1 Register Enterprise MCP Servers

In the AgentForce Command Center, register the MCP servers your agent needs:

```json
{
  "mcp_connections": [
    {
      "server_uri": "mcp://enterprise-catalog/salesforce-crm",
      "display_name": "Salesforce CRM Tools",
      "authentication": {
        "type": "oauth2",
        "token_endpoint": "https://auth.internal.company.com/token"
      },
      "eaagf_governance": {
        "route_through_gateway": true,
        "gateway_endpoint": "https://eaagf-gateway.internal.company.com/v1/mcp"
      }
    }
  ]
}
```

### 2.2 MCP Tool Manifest

AgentForce automatically discovers available tools from registered MCP servers. The EAAGF gateway validates that each MCP server is listed in the enterprise MCP directory before allowing connections:

```json
{
  "tools": [
    {
      "name": "account_read",
      "description": "Read Salesforce Account records",
      "input_schema": { "type": "object", "properties": { "account_id": { "type": "string" } } },
      "output_schema": { "type": "object" },
      "required_permissions": ["salesforce://sobject/Account:READ"],
      "max_data_classification": "CONFIDENTIAL"
    }
  ]
}
```

### 2.3 Governance Routing

All MCP tool calls are routed through the EAAGF gateway by setting `route_through_gateway: true`. This ensures:

- The Governance Controller validates the agent's identity and permissions.
- The MCP server is checked against the enterprise MCP directory.
- Rate limiting and data classification controls are enforced.
- Audit events are emitted for every tool call.

---

## Step 3: Native A2A Configuration

Salesforce AgentForce supports A2A natively. Configure agent-to-agent delegation through the Command Center.

### 3.1 Enable A2A Delegation

```json
{
  "a2a_config": {
    "enabled": true,
    "agent_card_signing": {
      "key_source": "salesforce_certificate_store",
      "certificate_name": "eaagf-agent-signing-cert"
    },
    "governance_gateway": "https://eaagf-gateway.internal.company.com/v1/a2a",
    "max_delegation_depth": 3,
    "delegation_timeout_seconds": 300
  }
}
```

### 3.2 Agent Card Integration

AgentForce automatically generates and signs Agent Cards for A2A delegations using the configured signing certificate. The Agent Card includes:

- The delegating agent's `agent_id` from the EAAGF Agent Registry.
- The agent's current `risk_tier`.
- The delegation scope (permitted actions, resources, and maximum risk tier).

The EAAGF gateway validates the Agent Card and enforces combined Risk_Tier limits before permitting the delegation.

---

## Step 4: Gateway Deployment

### 4.1 Gateway Configuration

Deploy the EAAGF Salesforce gateway with the following configuration:

```yaml
# eaagf-salesforce-gateway.yaml
gateway:
  platform: "SALESFORCE"
  governance_controller_endpoint: "https://governance.internal.company.com"
  
  salesforce:
    instance_url: "https://your-org.my.salesforce.com"
    api_version: "v60.0"
    connected_app:
      client_id: "${SALESFORCE_CLIENT_ID}"
      client_secret: "${SALESFORCE_CLIENT_SECRET}"
    command_center:
      webhook_secret: "${WEBHOOK_SECRET}"

  mcp:
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300

  a2a:
    enabled: true
    agent_card_validation: true

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-salesforce-gateway"

  health:
    port: 8081
    path: "/health"
```

### 4.2 Deploy the Gateway

The gateway runs as a containerized service in your enterprise Kubernetes cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eaagf-salesforce-gateway
  namespace: eaagf-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eaagf-salesforce-gateway
  template:
    metadata:
      labels:
        app: eaagf-salesforce-gateway
    spec:
      containers:
        - name: gateway
          image: eaagf-registry.internal.company.com/salesforce-gateway:latest
          ports:
            - containerPort: 8080
            - containerPort: 8081
          envFrom:
            - secretRef:
                name: eaagf-salesforce-secrets
          volumeMounts:
            - name: gateway-config
              mountPath: /etc/eaagf
      volumes:
        - name: gateway-config
          configMap:
            name: eaagf-salesforce-gateway-config
```

---

## Step 5: Salesforce-Specific Conformance Profile Examples

### T1 Salesforce Agent (Read-Only CRM Query)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "crm-query-agent"
  version: "1.0.0"
  owning_team: "sales-ops"
  platform: "SALESFORCE"
spec:
  risk_tier: "T1"
  capabilities:
    - TOOL_CALL
    - DATA_READ
  declared_permissions:
    - resource: "salesforce://sobject/Account"
      actions: ["READ"]
    - resource: "salesforce://sobject/Opportunity"
      actions: ["READ"]
    - resource: "salesforce://sobject/Contact"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/salesforce-crm"
  data_classifications_accessed:
    - "INTERNAL"
  oversight_mode: "HUMAN_IN_LOOP"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "crm-read-context"
  protocols_supported:
    - "MCP_1_0"
```

### T2 Salesforce Agent (CRM Update Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "crm-update-agent"
  version: "2.1.0"
  owning_team: "sales-operations"
  platform: "SALESFORCE"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "salesforce://sobject/Account"
      actions: ["READ", "UPDATE"]
    - resource: "salesforce://sobject/Opportunity"
      actions: ["READ", "UPDATE", "CREATE"]
    - resource: "salesforce://sobject/Case"
      actions: ["READ", "CREATE"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/salesforce-crm"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "crm-operations-context"
  geographic_constraints:
    - "US"
    - "EU"
  protocols_supported:
    - "MCP_1_0"
```

### T3 Salesforce Agent (Multi-Agent Service Orchestrator)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "service-orchestrator-agent"
  version: "1.0.0"
  owning_team: "customer-success"
  platform: "SALESFORCE"
spec:
  risk_tier: "T3"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION
  declared_permissions:
    - resource: "salesforce://sobject/Case"
      actions: ["READ", "UPDATE", "CREATE"]
    - resource: "salesforce://sobject/Account"
      actions: ["READ"]
    - resource: "salesforce://sobject/Knowledge"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/salesforce-crm"
    - "mcp://enterprise-catalog/salesforce-knowledge"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "service-orchestration-context"
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
- [ ] AgentForce Command Center webhook is registered and receiving events.
- [ ] MCP connections are routed through the EAAGF gateway (`route_through_gateway: true`).
- [ ] A2A delegation is configured with Agent Card signing certificate.
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] Connected App OAuth scopes are limited to the minimum required.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
