# Azure AI Foundry Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on Azure AI Foundry to the EAAGF governance framework. It covers AI Foundry SDK integration, Azure API Management configuration, native MCP support, A2A compatibility shim setup, and Azure-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. An Azure subscription with Azure AI Foundry provisioned.
2. Azure API Management (APIM) instance deployed.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. Azure AD app registration configured for the EAAGF gateway.

---

## Architecture Overview

Azure AI Foundry provides **native MCP support** but requires an **A2A compatibility shim** for agent-to-agent delegation. The EAAGF Platform Adapter uses Azure API Management as the governance enforcement point, with the AI Foundry SDK providing in-process governance hooks.

```
┌──────────────────────────────────────────────────┐
│  Azure AI Foundry                                 │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  AI Foundry Agent │  │  Azure API           │  │
│  │  (Agent Runtime)  │──┤  Management (APIM)   │  │
│  │  (AI Foundry SDK) │  │  (Policy Enforcement)│  │
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

## Step 1: AI Foundry SDK Integration

### 1.1 Install the EAAGF SDK Extension

The EAAGF SDK integrates with the Azure AI Foundry SDK to provide governance hooks:

```bash
pip install azure-ai-foundry eaagf-sdk-azure
```

### 1.2 Configure the Agent with Governance Hooks

```python
from azure.ai.foundry import AIFoundryClient
from azure.identity import DefaultAzureCredential
from eaagf_sdk_azure import GovernanceMiddleware

# Initialize AI Foundry client
credential = DefaultAzureCredential()
client = AIFoundryClient(
    endpoint="https://your-foundry.api.azureml.ms",
    credential=credential
)

# Attach EAAGF governance middleware
governance = GovernanceMiddleware(
    gateway_endpoint="https://eaagf-gateway.internal.company.com",
    agent_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    platform="AZURE"
)

# Create agent with governance
agent = client.agents.create(
    name="customer-insights-agent",
    model="gpt-4o",
    instructions="You are a customer insights agent. All actions are governed by EAAGF.",
    tools=[...],
    middleware=[governance]  # EAAGF governance middleware
)
```

### 1.3 Governance-Wrapped Tool Definitions

Define tools with EAAGF governance annotations:

```python
from eaagf_sdk_azure import governed_tool

@governed_tool(
    resource="cosmos://customer-db/insights",
    actions=["READ"],
    data_classification="CONFIDENTIAL"
)
def query_customer_insights(customer_id: str) -> dict:
    """Query customer insights from Cosmos DB."""
    # The governance middleware automatically:
    # 1. Evaluates the action against EAAGF policies
    # 2. Creates a Context_Compartment for CONFIDENTIAL data
    # 3. Emits audit events
    # 4. Blocks if policy denies
    return cosmos_client.query(customer_id)
```

---

## Step 2: Azure API Management Configuration

Azure API Management serves as the network-level governance enforcement point for all agent API calls.

### 2.1 Import the EAAGF Policy Fragment

Add the EAAGF governance policy to your APIM instance:

```xml
<!-- EAAGF Governance Policy Fragment -->
<fragment>
  <inbound>
    <!-- Extract EAAGF agent identity from request -->
    <set-variable name="eaagf-agent-id"
      value="@(context.Request.Headers.GetValueOrDefault("X-EAAGF-Agent-Id", ""))" />
    <set-variable name="eaagf-risk-tier"
      value="@(context.Request.Headers.GetValueOrDefault("X-EAAGF-Risk-Tier", ""))" />

    <!-- Forward to EAAGF gateway for policy evaluation -->
    <send-request mode="new" response-variable-name="govDecision" timeout="5">
      <set-url>https://eaagf-gateway.internal.company.com/v1/azure/evaluate</set-url>
      <set-method>POST</set-method>
      <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
      </set-header>
      <set-body>@{
        return new JObject(
          new JProperty("agent_id", context.Variables["eaagf-agent-id"]),
          new JProperty("action_type", context.Request.Method),
          new JProperty("target_resource", context.Request.Url.Path),
          new JProperty("risk_tier", context.Variables["eaagf-risk-tier"])
        ).ToString();
      }</set-body>
    </send-request>

    <!-- Enforce governance decision -->
    <choose>
      <when condition="@(((IResponse)context.Variables["govDecision"]).Body.As<JObject>()["outcome"].ToString() == "DENIED")">
        <return-response>
          <set-status code="403" reason="EAAGF Policy Denied" />
          <set-body>@{
            var decision = ((IResponse)context.Variables["govDecision"]).Body.As<JObject>();
            return decision.ToString();
          }</set-body>
        </return-response>
      </when>
    </choose>
  </inbound>
</fragment>
```

### 2.2 Apply the Policy to Agent APIs

Apply the EAAGF policy fragment to all APIs that agents access:

```xml
<policies>
  <inbound>
    <base />
    <include-fragment fragment-id="eaagf-governance" />
  </inbound>
  <outbound>
    <base />
  </outbound>
</policies>
```

### 2.3 Rate Limiting via APIM

Configure APIM rate limiting aligned with EAAGF tier limits:

```xml
<rate-limit-by-key
  calls="@(context.Variables["eaagf-risk-tier"].ToString().StartsWith("T3") ||
           context.Variables["eaagf-risk-tier"].ToString().StartsWith("T4") ? 20 : 100)"
  renewal-period="60"
  counter-key="@(context.Variables["eaagf-agent-id"])" />
```

---

## Step 3: Native MCP Configuration

Azure AI Foundry supports MCP natively. Configure MCP connections through the AI Foundry SDK.

### 3.1 Register MCP Servers

```python
from azure.ai.foundry import MCPServerConfig

mcp_servers = [
    MCPServerConfig(
        uri="mcp://enterprise-catalog/azure-cosmos",
        display_name="Azure Cosmos DB",
        authentication={
            "type": "managed_identity",
            "resource": "https://cosmos.azure.com"
        },
        governance_routing={
            "enabled": True,
            "gateway_endpoint": "https://eaagf-gateway.internal.company.com/v1/mcp"
        }
    ),
    MCPServerConfig(
        uri="mcp://enterprise-catalog/azure-search",
        display_name="Azure AI Search",
        authentication={
            "type": "managed_identity",
            "resource": "https://search.azure.com"
        },
        governance_routing={
            "enabled": True,
            "gateway_endpoint": "https://eaagf-gateway.internal.company.com/v1/mcp"
        }
    )
]

agent = client.agents.create(
    name="customer-insights-agent",
    model="gpt-4o",
    mcp_servers=mcp_servers,
    middleware=[governance]
)
```

### 3.2 Governance Routing for MCP

All MCP tool calls are routed through the EAAGF gateway by enabling `governance_routing`. The gateway validates each call against the enterprise MCP directory and the agent's Conformance_Profile.

---

## Step 4: A2A Compatibility Shim

Azure AI Foundry does not natively support A2A. The EAAGF gateway provides a compatibility shim.

### 4.1 A2A Delegation via SDK

```python
from eaagf_sdk_azure import GovernanceMiddleware

# Delegate a sub-task to another agent
delegation_result = governance.delegate_task(
    target_agent_id="target-agent-uuid",
    sub_task_description="Generate compliance report from audit data",
    permitted_actions=["TOOL_CALL", "DATA_READ"],
    permitted_resources=["cosmos://audit-db/compliance-records"],
    max_risk_tier="T2"
)

if delegation_result.status == "COMPLETED":
    report = delegation_result.result
elif delegation_result.status == "DENIED":
    print(f"Delegation denied: {delegation_result.reason_code}")
```

### 4.2 Gateway A2A Configuration

```yaml
# eaagf-azure-gateway.yaml (a2a section)
a2a_shim:
  enabled: true
  version: "A2A_1_0"
  agent_card_signing:
    key_vault_uri: "https://eaagf-keyvault.vault.azure.net/"
    key_name: "agent-card-signing-key"
  delegation_timeout_seconds: 300
  max_delegation_depth: 3
```

---

## Step 5: Gateway Deployment

### 5.1 Gateway Configuration

```yaml
# eaagf-azure-gateway.yaml
gateway:
  platform: "AZURE"
  governance_controller_endpoint: "https://governance.internal.company.com"

  azure:
    tenant_id: "${AZURE_TENANT_ID}"
    subscription_id: "${AZURE_SUBSCRIPTION_ID}"
    ai_foundry:
      endpoint: "https://your-foundry.api.azureml.ms"
    apim:
      endpoint: "https://your-apim.azure-api.net"
      subscription_key: "${APIM_SUBSCRIPTION_KEY}"
    authentication:
      type: "managed_identity"

  mcp:
    native_support: true
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300

  a2a_shim:
    enabled: true
    version: "A2A_1_0"
    agent_card_signing:
      key_vault_uri: "https://eaagf-keyvault.vault.azure.net/"
      key_name: "agent-card-signing-key"

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-azure-gateway"
    azure_monitor:
      connection_string: "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
```

### 5.2 Deploy the Gateway

Deploy as an Azure Container App or on the enterprise Kubernetes cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eaagf-azure-gateway
  namespace: eaagf-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eaagf-azure-gateway
  template:
    metadata:
      labels:
        app: eaagf-azure-gateway
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: eaagf-azure-gateway-sa
      containers:
        - name: gateway
          image: eaagf-registry.internal.company.com/azure-gateway:latest
          ports:
            - containerPort: 8080
            - containerPort: 8081
          envFrom:
            - secretRef:
                name: eaagf-azure-secrets
          volumeMounts:
            - name: gateway-config
              mountPath: /etc/eaagf
      volumes:
        - name: gateway-config
          configMap:
            name: eaagf-azure-gateway-config
```

---

## Step 6: Azure-Specific Conformance Profile Examples

### T1 Azure Agent (Knowledge Base Assistant)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "kb-assistant-agent"
  version: "1.0.0"
  owning_team: "knowledge-management"
  platform: "AZURE"
spec:
  risk_tier: "T1"
  capabilities:
    - TOOL_CALL
    - DATA_READ
  declared_permissions:
    - resource: "azure-search://knowledge-index"
      actions: ["QUERY"]
    - resource: "blob://knowledge-base/articles"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/azure-search"
  data_classifications_accessed:
    - "PUBLIC"
    - "INTERNAL"
  oversight_mode: "HUMAN_IN_LOOP"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  protocols_supported:
    - "MCP_1_0"
```

### T2 Azure Agent (Customer Insights Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "customer-insights-agent"
  version: "2.0.0"
  owning_team: "customer-analytics"
  platform: "AZURE"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "cosmos://customer-db/profiles"
      actions: ["READ"]
    - resource: "cosmos://customer-db/insights"
      actions: ["READ", "WRITE"]
    - resource: "azure-search://customer-index"
      actions: ["QUERY"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/azure-cosmos"
    - "mcp://enterprise-catalog/azure-search"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "customer-insights-context"
  geographic_constraints:
    - "EU"
  protocols_supported:
    - "MCP_1_0"
```

### T3 Azure Agent (Compliance Reporting Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "compliance-reporting-agent"
  version: "1.0.0"
  owning_team: "compliance"
  platform: "AZURE"
spec:
  risk_tier: "T3"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION
  declared_permissions:
    - resource: "cosmos://audit-db/compliance-records"
      actions: ["READ"]
    - resource: "blob://compliance-reports/output"
      actions: ["READ", "WRITE"]
    - resource: "sql://governance-db/agent-registry"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/azure-cosmos"
    - "mcp://enterprise-catalog/azure-blob"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "compliance-reporting-context"
  geographic_constraints:
    - "EU"
    - "US"
  protocols_supported:
    - "MCP_1_0"
    - "A2A_1_0"
```

---

## Validation Checklist

Before requesting promotion to STAGING, verify:

- [ ] Agent manifest passes schema validation against the EAAGF conformance schema.
- [ ] EAAGF gateway is deployed and healthy (`/health` endpoint returns 200).
- [ ] AI Foundry SDK governance middleware is attached to the agent.
- [ ] Azure API Management EAAGF policy fragment is applied to all agent APIs.
- [ ] MCP connections are routed through the EAAGF gateway.
- [ ] A2A compatibility shim is configured (if delegation is needed).
- [ ] Managed identity has minimum required permissions.
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] APIM rate limiting is configured per EAAGF tier limits.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
