# GCP Vertex AI Platform Onboarding Guide

This guide walks teams through onboarding AI agents running on GCP Vertex AI to the EAAGF governance framework. It covers Vertex AI Agent Builder API integration, the gateway + SDK deployment pattern, MCP/A2A compatibility shim setup, and GCP-specific conformance profile examples.

For the normative interoperability specification, see [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md). For general agent registration, see [Agent Registration Guide](../agent-registration-guide.md).

---

## Prerequisites

1. A GCP project with Vertex AI enabled.
2. Vertex AI Agent Builder configured for your use case.
3. Your agent's Risk_Tier determined via the [Risk Assessment Guide](../risk-assessment-guide.md).
4. Access to the EAAGF Governance Controller endpoint.
5. A GCP service account configured for the EAAGF gateway.

---

## Architecture Overview

GCP Vertex AI does not natively support MCP or A2A. The EAAGF Platform Adapter for GCP uses a **gateway + SDK deployment pattern** where the EAAGF SDK integrates with the Vertex AI Agent Builder API, and a gateway service handles protocol translation and governance enforcement.

```
┌──────────────────────────────────────────────────┐
│  GCP Vertex AI                                    │
│  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Vertex AI Agent  │  │  Cloud Functions /   │  │
│  │  Builder          │──┤  Cloud Run Hooks     │  │
│  │  (Agent Runtime)  │  │  (EAAGF SDK)         │  │
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

## Step 1: Vertex AI Agent Builder API Integration

### 1.1 Create the Vertex AI Agent

Create your agent using the Vertex AI Agent Builder API with EAAGF metadata labels:

```python
from google.cloud import aiplatform

aiplatform.init(project="your-project-id", location="us-central1")

agent = aiplatform.Agent.create(
    display_name="customer-support-agent",
    description="EAAGF-managed customer support agent",
    model="gemini-1.5-pro",
    tools=[...],
    labels={
        "eaagf-agent-id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "eaagf-risk-tier": "t2",
        "eaagf-owning-team": "customer-success"
    }
)
```

### 1.2 Configure Tool Hooks with EAAGF SDK

Integrate the EAAGF SDK into your agent's tool definitions:

```python
from eaagf_sdk_gcp import GovernanceClient, governed_tool

governance = GovernanceClient(
    gateway_endpoint="https://eaagf-gateway.internal.company.com",
    agent_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    credential_path="/etc/eaagf/credential.json"
)

@governed_tool(
    governance=governance,
    resource="bigquery://project.dataset.customers",
    actions=["SELECT"],
    data_classification="CONFIDENTIAL"
)
def query_customer_data(customer_id: str) -> dict:
    """Query customer data from BigQuery."""
    from google.cloud import bigquery
    client = bigquery.Client()
    query = f"SELECT * FROM `project.dataset.customers` WHERE id = @customer_id"
    job_config = bigquery.QueryJobConfig(
        query_parameters=[bigquery.ScalarQueryParameter("customer_id", "STRING", customer_id)]
    )
    results = client.query(query, job_config=job_config)
    return [dict(row) for row in results]
```

### 1.3 Cloud Function Tool Handlers

For agents using Cloud Functions as tool backends, include the EAAGF SDK:

```python
import functions_framework
from eaagf_sdk_gcp import GovernanceClient, ActionRequest

governance = GovernanceClient(
    gateway_endpoint="https://eaagf-gateway.internal.company.com",
    agent_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890"
)

@functions_framework.http
def agent_tool_handler(request):
    """Cloud Function tool handler with EAAGF governance."""
    
    payload = request.get_json()
    tool_name = payload.get("tool_name")
    parameters = payload.get("parameters", {})
    
    # Evaluate governance policy
    decision = governance.evaluate(ActionRequest(
        action_type="TOOL_CALL",
        target_resource=f"vertex-ai://tools/{tool_name}",
        input_summary=str(parameters)[:1024]
    ))
    
    if decision.outcome == "DENIED":
        return {"error": decision.reason_code}, 403
    
    if decision.outcome == "GATED":
        return {"message": "Pending human approval", "gate_id": decision.gate_id}, 202
    
    # Execute the tool (only if PERMITTED)
    result = execute_tool(tool_name, parameters)
    governance.report_completion(decision.request_id, result)
    
    return result, 200
```

---

## Step 2: Cloud Run Hooks for Lifecycle Events

### 2.1 Pre-Processing Hook

Deploy a Cloud Run service that validates agent identity and session before processing:

```python
from flask import Flask, request, jsonify
from eaagf_sdk_gcp import GovernanceClient

app = Flask(__name__)
governance = GovernanceClient(
    gateway_endpoint="https://eaagf-gateway.internal.company.com",
    agent_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890"
)

@app.route("/pre-process", methods=["POST"])
def pre_process():
    """Validate EAAGF identity before agent processes input."""
    payload = request.get_json()
    agent_id = payload.get("agent_id")
    
    identity_valid = governance.validate_identity(agent_id)
    if not identity_valid:
        return jsonify({
            "allowed": False,
            "reason": "IDENTITY_UNREGISTERED"
        }), 403
    
    rate_ok = governance.check_rate_limit(agent_id)
    if not rate_ok:
        return jsonify({
            "allowed": False,
            "reason": "RATE_LIMIT_EXCEEDED"
        }), 429
    
    return jsonify({"allowed": True}), 200
```

### 2.2 Audit Logging via Cloud Logging

Configure Cloud Logging to capture agent events and forward them to the EAAGF Telemetry Emitter:

```python
import google.cloud.logging
from eaagf_sdk_gcp import TelemetryForwarder

# Initialize Cloud Logging
logging_client = google.cloud.logging.Client()
logger = logging_client.logger("eaagf-agent-events")

# Forward to EAAGF telemetry
telemetry = TelemetryForwarder(
    otlp_endpoint="https://otel-collector.internal.company.com:4317",
    service_name="eaagf-gcp-agent"
)

def log_agent_event(event_type, agent_id, details):
    """Log an agent event to both Cloud Logging and EAAGF telemetry."""
    entry = {
        "eaagf.agent.id": agent_id,
        "eaagf.action.type": event_type,
        "eaagf.action.outcome": details.get("outcome"),
        "eaagf.agent.platform": "GCP"
    }
    logger.log_struct(entry, severity="INFO")
    telemetry.emit(entry)
```

---

## Step 3: Gateway + SDK Deployment

### 3.1 Gateway Configuration

```yaml
# eaagf-gcp-gateway.yaml
gateway:
  platform: "GCP"
  governance_controller_endpoint: "https://governance.internal.company.com"

  gcp:
    project_id: "your-project-id"
    region: "us-central1"
    vertex_ai:
      agent_endpoint_prefix: "projects/your-project-id/locations/us-central1/agents/"
    authentication:
      type: "service_account"
      key_path: "/etc/eaagf/gcp-service-account.json"

  mcp_shim:
    enabled: true
    version: "MCP_1_0"
    enterprise_directory_endpoint: "https://mcp-directory.internal.company.com/v1"
    refresh_interval_seconds: 300
    gcp_mappings:
      - gcp_operation: "bigquery.query"
        mcp_tool: "mcp://enterprise-catalog/gcp-bigquery"
      - gcp_operation: "storage.objects.get"
        mcp_tool: "mcp://enterprise-catalog/gcp-storage"
      - gcp_operation: "vertex-ai.agent.invoke"
        mcp_tool: "mcp://enterprise-catalog/vertex-ai-agent"

  a2a_shim:
    enabled: true
    version: "A2A_1_0"
    agent_card_signing:
      kms_key_name: "projects/your-project-id/locations/global/keyRings/eaagf/cryptoKeys/agent-card-signing"

  telemetry:
    otlp_endpoint: "https://otel-collector.internal.company.com:4317"
    service_name: "eaagf-gcp-gateway"
```

### 3.2 Deploy the Gateway

Deploy as a Cloud Run service or on the enterprise Kubernetes cluster (GKE):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eaagf-gcp-gateway
  namespace: eaagf-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eaagf-gcp-gateway
  template:
    metadata:
      labels:
        app: eaagf-gcp-gateway
    spec:
      serviceAccountName: eaagf-gcp-gateway-sa
      containers:
        - name: gateway
          image: eaagf-registry.internal.company.com/gcp-gateway:latest
          ports:
            - containerPort: 8080
            - containerPort: 8081
          envFrom:
            - secretRef:
                name: eaagf-gcp-secrets
          volumeMounts:
            - name: gateway-config
              mountPath: /etc/eaagf
      volumes:
        - name: gateway-config
          configMap:
            name: eaagf-gcp-gateway-config
```

---

## Step 4: MCP Compatibility Shim

### 4.1 How the MCP Shim Works

1. The Vertex AI agent invokes a tool (Cloud Function or Cloud Run endpoint).
2. The EAAGF SDK in the tool handler translates the call into an MCP tool call request.
3. The SDK forwards the MCP request to the gateway for policy evaluation.
4. If permitted, the gateway routes the call to the target MCP server or back to the GCP service.
5. The response is returned to the tool handler.

### 4.2 Declaring MCP Servers in Your Manifest

```yaml
spec:
  approved_mcp_servers:
    - "mcp://enterprise-catalog/gcp-bigquery"
    - "mcp://enterprise-catalog/gcp-storage"
    - "mcp://enterprise-catalog/vertex-ai-agent"
  protocols_supported:
    - "MCP_1_0"
```

---

## Step 5: A2A Compatibility Shim

### 5.1 Delegating to Other Agents

When a Vertex AI agent needs to delegate a sub-task, the EAAGF SDK provides a delegation method:

```python
delegation_result = governance.delegate_task(
    target_agent_id="target-agent-uuid",
    sub_task_description="Generate quarterly revenue report from BigQuery data",
    permitted_actions=["TOOL_CALL", "DATA_READ"],
    permitted_resources=["bigquery://project.dataset.revenue"],
    max_risk_tier="T2"
)

if delegation_result.status == "COMPLETED":
    report = delegation_result.result
elif delegation_result.status == "DENIED":
    print(f"Delegation denied: {delegation_result.reason_code}")
```

The gateway automatically:
- Creates and signs an Agent Card using Cloud KMS.
- Validates combined Risk_Tier limits.
- Routes the delegation through the Governance Controller.
- Emits an `A2A_DELEGATION` audit event.

---

## Step 6: GCP-Specific Conformance Profile Examples

### T2 GCP Agent (Data Analytics Agent)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "data-analytics-agent"
  version: "1.0.0"
  owning_team: "business-intelligence"
  platform: "GCP"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "bigquery://project.analytics.sales"
      actions: ["SELECT"]
    - resource: "bigquery://project.analytics.reports"
      actions: ["SELECT", "INSERT"]
    - resource: "storage://analytics-bucket/reports"
      actions: ["READ", "WRITE"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/gcp-bigquery"
    - "mcp://enterprise-catalog/gcp-storage"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "analytics-context"
  geographic_constraints:
    - "US"
  protocols_supported:
    - "MCP_1_0"
```

### T3 GCP Agent (ML Pipeline Orchestrator)

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "ml-pipeline-orchestrator"
  version: "2.0.0"
  owning_team: "ml-engineering"
  platform: "GCP"
spec:
  risk_tier: "T3"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
    - AGENT_DELEGATION
  declared_permissions:
    - resource: "bigquery://project.ml.training_data"
      actions: ["SELECT"]
    - resource: "bigquery://project.ml.predictions"
      actions: ["SELECT", "INSERT"]
    - resource: "storage://ml-models-bucket/models"
      actions: ["READ", "WRITE"]
    - resource: "vertex-ai://pipelines/training"
      actions: ["RUN", "MONITOR"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/gcp-bigquery"
    - "mcp://enterprise-catalog/gcp-storage"
    - "mcp://enterprise-catalog/vertex-ai-agent"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "APPROVAL_REQUIRED"
  max_session_duration_seconds: 900
  max_actions_per_minute: 20
  context_compartments:
    - "ml-pipeline-context"
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
- [ ] Cloud Function / Cloud Run tool handlers include the EAAGF SDK.
- [ ] MCP shim correctly translates GCP-native calls to MCP format.
- [ ] GCP service account follows least-privilege (only permissions needed for the agent's declared scope).
- [ ] Agent_Identity credential is valid and auto-rotation is configured.
- [ ] Audit events appear in the Governance Controller's telemetry backend.
- [ ] For T3/T4: A2A delegation is configured via the EAAGF SDK with Cloud KMS signing.
- [ ] Vertex AI agent is labeled with `eaagf-agent-id` and `eaagf-risk-tier`.

---

## Cross-References

- [Interoperability Standard](../../eaagf-specification/07-interoperability-standard.md) — Normative specification for MCP, A2A, and Platform Adapters
- [Agent Registration Guide](../agent-registration-guide.md) — General registration process
- [Risk Assessment Guide](../risk-assessment-guide.md) — Determining your agent's Risk_Tier
- [Data Governance Standard](../../eaagf-specification/08-data-governance-standard.md) — Data classification and compartment rules
- [Authorization Standard](../../eaagf-specification/04-authorization-standard.md) — Least-privilege and credential TTL rules
- [Agent Lifecycle Flow](../../flows/agent-lifecycle-flow.md) — Lifecycle state transitions
- [Glossary](../../reference/glossary.md) — EAAGF terminology
