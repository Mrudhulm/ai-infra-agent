# Server-Side Control Plane & Dashboard Blueprint

> **Role**: The Server-Side Control Plane is the centralized manager. It hosts the Web Dashboard UI (where clients log in and view live progress), the API Gateway signal router, the Open Policy Agent (OPA) policy engine, the Redis task queue, PostgreSQL state store, Knowledge Service with local RAG Registry, and Multi-Cloud MCP Servers (HashiCorp, AWS, Azure, GCP).

## 1. Server-Side Architecture & Components

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 SERVER-SIDE CONTROL PLANE                                       │
│                                                                                                 │
│  ┌────────────────────────┐         ┌────────────────────────────────────────────────────────┐  │
│  │ Web Dashboard UI       │ ──────► │ API Gateway & Signal Router                            │  │
│  │ (Port 3000 / Next.js)  │ HTTP    │ (Port 8080 / Express / FastAPI)                        │  │
│  └────────────────────────┘         └───────────────────┬────────────────────────────────────┘  │
│                                                         │                                       │
│                                    Evaluates Autonomy   │                                       │
│                                    ┌────────────────────┴───────────────────┐                   │
│                                    ▼                                        ▼                   │
│                      ┌─────────────────────────┐        ┌──────────────────────────────────┐    │
│                      │ Policy Engine (OPA)     │        │ Task Queue                       │    │
│                      │ (Port 8181 / Rego)      │        │ (Port 6379 / Redis Streams)      │    │
│                      └─────────────────────────┘        └───────────┬──────────────────────┘    │
│                                                                     │                           │
│                                                                     ▼                           │
│  ┌──────────────────────────────────────────────────────────────────┴────────────────────────┐  │
│  │ Knowledge Base & Local RAG Registry Engine (Port 8000)                                     │  │
│  │  ├── Indexes local Markdown/PDF Architecture Docs & Compliance Guidelines                 │  │
│  │  └── Vector Store (pgvector / Qdrant) for Fast Hybrid Semantic Search                     │  │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Multi-Cloud MCP Server Registry (Ports 3002-3005)                                          │  │
│  │  ├── HashiCorp MCP (Port 3002): Terraform registry, module specs, HCL docs                 │  │
│  │  ├── AWS MCP (Port 3003): Resource schemas, IAM policies, AWS CloudControl docs            │  │
│  │  ├── Azure MCP (Port 3004): ARM/Bicep specifications, Azure provider docs                 │  │
│  │  └── GCP MCP (Port 3005): Google Cloud API resource schemas & IaC docs                     │  │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                 │
│  ┌──────────────────────────────────────┐       ┌────────────────────────────────────────────┐  │
│  │ PostgreSQL State DB (Port 5432)      │       │ Credential Broker (Port 3001 - JIT STS)    │  │
│  └──────────────────────────────────────┘       └────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

1. **Web Dashboard UI (`agent-ui`)**: Allows users to log in, submit infrastructure tasks, inspect live step-by-step logs via SSE (Server-Sent Events), view side-by-side code diffs, search local Knowledge RAG docs, and track PR statuses.
2. **API Gateway (`api-gateway`)**: Ingests signals (chat prompts, webhooks, alerts), checks tenant auth, normalizes tasks, and streams real-time updates back to the UI.
3. **Knowledge Base & Local RAG Registry (`knowledge-service`)**: Scans and indexes local architecture standards, security rules, and infrastructure code; exposes a search API for local client agents.
4. **Multi-Cloud MCP Server Suite**:
   * **HashiCorp MCP (`mcp-server-hashicorp`)**: Fetches Terraform module docs, provider versions, and HCL specifications.
   * **AWS MCP (`mcp-server-aws`)**: Retrieves AWS CloudControl resource schemas, IAM action lists, and AWS provider details.
   * **Azure MCP (`mcp-server-azure`)**: Retrieves Azure ARM/Bicep specs, resource providers, and Azure RM Terraform docs.
   * **GCP MCP (`mcp-server-gcp`)**: Retrieves GCP Cloud API schemas, service specs, and Google Provider Terraform docs.
5. **Policy Engine (`policy-engine`)**: Uses OPA to enforce autonomy ceilings (`Observe`, `Propose`, `Governed`) and evaluate tool allow/deny lists.
6. **Task Queue (`task-queue`)**: Redis Streams queue managing job distribution and output event fan-out.
7. **Database (`database`)**: PostgreSQL (with `pgvector`) storing user sessions, RAG embeddings, tenant metadata, audit logs, and immutable receipts.
8. **Credential Broker (`credential-broker`)**: Mints short-lived STS/OIDC cloud tokens (AWS STS, Azure Managed Identity, GCP Impersonated Service Accounts) just-in-time when requested by authorized workers.

---

## 2. Server-Side Knowledge Base & Local RAG Engine (`packages/knowledge-service/src/main.py`)

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel
import typing

app = FastAPI(title="Infrastructure Knowledge Base & Local RAG Registry")

# Simulated local document index store (can be backed by pgvector or Qdrant)
LOCAL_RAG_REGISTRY = [
    {
        "id": "doc-001",
        "title": "Corporate AWS S3 Security Standards",
        "category": "security",
        "content": "All S3 buckets MUST have server-side encryption enabled (AES256 or aws:kms) and Public Access Block enabled."
    },
    {
        "id": "doc-002",
        "title": "Multi-Cloud Azure & GCP Naming Conventions",
        "category": "architecture",
        "content": "Azure storage accounts must follow 'st<app><env>' format. GCP buckets must use 'gcp-<env>-<app>-data' format."
    }
]

@app.get("/health")
def health():
    return {"status": "healthy", "service": "knowledge-rag-registry"}

@app.get("/context/rag-search")
def search_local_knowledge(q: str = Query(..., description="Query for local architecture/security docs")):
    """Searches local architecture documents and returns compact Markdown for LLM prompt insertion."""
    results = [doc for doc in LOCAL_RAG_REGISTRY if any(k in doc["content"].lower() for k in q.lower().split())]
    
    if not results:
        results = LOCAL_RAG_REGISTRY  # Fallback to general guidelines
        
    markdown_output = "# Local Knowledge Base & Architecture Standards\n\n"
    for item in results:
        markdown_output += f"### {item['title']}\n{item['content']}\n\n"
        
    return {"markdown": markdown_output, "count": len(results)}
```

---

## 3. Multi-Cloud MCP Server Implementation (`packages/mcp-server/src/server.js`)

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.get('/health', (req, res) => res.json({ status: 'healthy', mcpSuite: ['hashicorp', 'aws', 'azure', 'gcp'] }));

// HashiCorp MCP: Fetch Terraform Provider Documentation & Registry Spec
app.post('/mcp/hashicorp/search-registry', (req, res) => {
  const { provider, resource } = req.body;
  res.json({
    provider: provider || 'aws',
    resource: resource || 'aws_s3_bucket',
    docUrl: `https://registry.terraform.io/providers/hashicorp/${provider}/latest/docs/resources/${resource}`,
    schema: {
      required: ["bucket"],
      optional: ["acl", "force_destroy", "server_side_encryption_configuration"]
    }
  });
});

// AWS MCP: Get AWS CloudControl Resource Schema
app.post('/mcp/aws/resource-schema', (req, res) => {
  const { resourceType } = req.body;
  res.json({
    resourceType: resourceType || 'AWS::S3::Bucket',
    documentation: 'https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html',
    requiredProperties: [],
    readOnlyProperties: ["/properties/Arn", "/properties/DomainName"]
  });
});

// Azure MCP: Get Azure ARM / Terraform Provider Spec
app.post('/mcp/azure/resource-schema', (req, res) => {
  const { resourceType } = req.body;
  res.json({
    resourceType: resourceType || 'azurerm_storage_account',
    documentation: 'https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account',
    requiredProperties: ["name", "resource_group_name", "location", "account_tier", "account_replication_type"]
  });
});

// GCP MCP: Get GCP Resource Spec
app.post('/mcp/gcp/resource-schema', (req, res) => {
  const { resourceType } = req.body;
  res.json({
    resourceType: resourceType || 'google_storage_bucket',
    documentation: 'https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket',
    requiredProperties: ["name", "location"]
  });
});

app.listen(3002, () => console.log('Multi-Cloud MCP Server Suite listening on port 3002'));
```

---

## 4. Server-Side Policy Guardrail (`policies/infra_guardrails.rego`)

```rego
package infra.authz

default allow = false

# Allow read-only & doc lookup operations
allow {
    input.action == "read"
}
allow {
    input.action == "mcp_doc_lookup"
}
allow {
    input.action == "rag_doc_search"
}

# Allow plan and PR creation operations (Propose ceiling)
allow {
    input.action == "terraform_plan"
}
allow {
    input.action == "git_create_pr"
}

# HARD DENY: Direct terraform apply on production infrastructure is strictly forbidden
deny {
    input.action == "terraform_apply"
    input.environment == "production"
}
```

---

## 5. End-to-End Multi-Cloud Execution Summary

```
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ 1. USER PROMPT IN DASHBOARD (http://localhost:3000)                                   │
 │    "Configure AWS S3 bucket with encryption & create Azure Storage Account in main.tf" │
 └────────────────────────────────────────────────┬───────────────────────────────────────┘
                                                  │
                                                  ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ 2. API GATEWAY & POLICY ENGINE (OPA Check @ Port 8181)                                 │
 │    - Autonomy Ceiling: PROPOSE (PR only) -> Approved                                   │
 └────────────────────────────────────────────────┬───────────────────────────────────────┘
                                                  │
                                                  ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ 3. AGENT WORKER & LOCAL LLM RUNNER (Ollama @ Port 11434)                               │
 │    - Step A: Fetches internal security rules from Local RAG Engine (Port 8000)        │
 │    - Step B: Fetches HashiCorp, AWS, & Azure specs from Multi-Cloud MCP (Port 3002)    │
 │    - Step C: Generates formatted, compliant Terraform HCL                              │
 └────────────────────────────────────────────────┬───────────────────────────────────────┘
                                                  │
                                                  ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ 4. EXECUTION SANDBOX (Docker Ephemeral Container)                                      │
 │    - Runs `checkov` linting scan -> PASSED                                             │
 │    - Runs `terraform plan` -> PASSED (Zero unexpected drift)                           │
 └────────────────────────────────────────────────┬───────────────────────────────────────┘
                                                  │
                                                  ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │ 5. GITOPS CHANGE CONTROL & DASHBOARD STREAMING                                         │
 │    - Pushes feature branch & opens Git Pull Request (PR)                               │
 │    - Streams execution logs, diff view, & PR link live to Dashboard UI (Port 3000)    │
 └────────────────────────────────────────────────────────────────────────────────────────┘
```