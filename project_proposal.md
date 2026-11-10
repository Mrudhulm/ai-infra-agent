# Comprehensive Project Proposal: Enterprise Multi-Cloud AI Infrastructure Agent System

## Executive Summary

Modern cloud engineering requires balancing security, policy compliance, and rapid execution across multiple cloud providers (AWS, Azure, GCP) and Infrastructure as Code (IaC) tooling (HashiCorp Terraform/OpenTofu). This proposal outlines the architecture, design, and implementation plan for a **100% Privacy-Preserving, Self-Hosted Infrastructure AI Agent System**.

The platform operates on a strict **two-tier architecture**:
1. **Client-Side / Local Execution Runner**: Operates locally on developer workstations or isolated execution sandboxes. It runs local LLMs (via Ollama), connects to official multi-cloud Model Context Protocol (MCP) servers for live schema lookups, and generates Pull Requests (PRs).
2. **Server-Side Control Plane & Dashboard**: Serves as the centralized control tower. It hosts the user dashboard, manages authentication, evaluates Open Policy Agent (OPA) security ceilings, queues tasks via Redis, and maintains a local Retrieval-Augmented Generation (RAG) Knowledge Registry.

Crucially, the system enforces a **PR-First Change Control Model**: the AI agent **never** executes direct production deployments (`terraform apply`). Every change is delivered as a verified Git Pull Request for human review and automated CI validation.

---

## 1. System Vision & Key Capabilities

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   WEB DASHBOARD UI (HTTP 3000)                                    │
│  - User Authentication & Session Control                                                          │
│  - Natural Language Prompt Interface                                                              │
│  - Live Step-by-Step Telemetry & SSE Log Streaming                                                │
│  - Side-by-Side Diff Inspection & PR Approval Trigger                                            │
└─────────────────────────────────┬─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              SERVER-SIDE CONTROL PLANE & KNOWLEDGE                                │
│  ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────────────┐  │
│  │ API Gateway (HTTP 8080) │──►│ OPA Guardrails (8181)   │──►│ Redis Queue & Dispatch (6379)   │  │
│  └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Local Knowledge Service & RAG Registry (HTTP 8000)                                           │  │
│  │ Indexes local architecture guides, security policies, and organizational IaC conventions   │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬─────────────────────────────────────────────────────────────────┘
                                  │ Task Stream
                                  ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT-SIDE LOCAL EXECUTION SANDBOX                                  │
│  ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────────────┐  │
│  │ Local LLM (Ollama)      │   │ Execution Sandbox       │   │ Multi-Cloud MCP Servers         │  │
│  │ model: qwen2.5-coder:7b │   │ (terraform, checkov)    │   │ - HashiCorp (Registry & Specs)  │  │
│  └─────────────────────────┘   └─────────────────────────┘   │ - AWS (Resource Schemas)        │  │
│                                                              │ - Azure (ARM / Bicep Schemas)   │  │
│                                                              │ - GCP (GDM & Terraform Specs)   │  │
│                                                              └─────────────────────────────────┘  │
│                                                                                                   │
│  Output: Local Git Feature Branch ──► Pushes to Git Provider ──► Creates Pull Request (PR)        │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Core System Principles
- **100% Private Inference**: Powered by local Ollama (`qwen2.5-coder:7b` or `llama3.1`). Zero external API calls or token costs.
- **Multi-Cloud Awareness**: Integrates dedicated MCP servers for **HashiCorp**, **AWS**, **Azure**, and **GCP** so the agent can query accurate provider documentation before writing code.
- **Local RAG Integration**: Built-in Knowledge Base indexing internal security rules, naming conventions, and golden IaC modules.
- **Zero Direct Cloud Mutations**: All actions write code to feature branches and open PRs. Production deployment is owned by human-approved CI/CD pipelines.

---

## 2. Structural Split: Client-Side vs. Server-Side

To maintain complete security isolation, the system cleanly separates concerns:

### 2.1 Server-Side System (Control Tower)
- **Web Dashboard (`agent-ui`)**: Responsive web console for non-technical and technical users to log in, issue prompts, view real-time traces, and audit completed runs.
- **API Gateway (`api-gateway`)**: Accepts incoming prompts, manages session tokens, and validates inputs against OPA policy.
- **Policy Engine (`policy-engine`)**: Open Policy Agent container enforcing strict autonomy ceilings (`Observe`, `Propose`, `Governed`).
- **Knowledge Service (`knowledge-service`)**: RAG engine exposing an endpoint (`/context/rag-search`) to supply organizational context to workers.
- **Task Broker (`task-queue` & `database`)**: Redis Streams handling task queues and PostgreSQL storing structured audit receipts.

### 2.2 Client-Side System (Local Worker & Tools)
- **Local LLM Engine (`local-llm`)**: Ollama service executing model inference directly on host hardware.
- **Skill Engine (`SKILL.md`)**: Progressive disclosure manifests defining allowed tools, prompt context, and execution constraints.
- **Multi-Cloud MCP Hub (`mcp-servers`)**:
  - `HashiCorp MCP`: Search Terraform Registry, fetch module inputs, check state schemas.
  - `AWS MCP`: Fetch AWS resource schemas, IAM policy constraints, Boto3 docs.
  - `Azure MCP`: Query ARM/Bicep resource schemas, provider specs.
  - `GCP MCP`: Search Google Cloud Provider resource syntax and API schemas.
- **Sandboxed Workspace (`sandboxes/docker`)**: Isolated container for running `terraform fmt`, `tflint`, `checkov`, and `git` commands safely.

---

## 3. End-to-End Operational Workflow

```
[User Prompts UI] ──► [API Gateway] ──► [OPA Policy Check] ──► [Redis Task Queue]
                                                                      │
                                                                      ▼
[Git PR Created] ◄── [Sandbox Checks] ◄── [Local LLM + MCP + RAG] ◄── [Client Worker]
        │
        ▼
[Human Reviews & Merges PR] ──► [CI/CD Pipeline Executes terraform apply]
```

1. **Prompt Ingestion**: A user logs into `http://localhost:3000` and submits a task: *"Scan my main.tf file, add server-side encryption to the S3 bucket using AWS provider specs, and add an Azure blob storage container per our company standards."*
2. **Policy Verification**: The API Gateway forwards the task payload to the OPA policy engine. OPA verifies that the target autonomy mode is `Propose` and permits task creation.
3. **Queueing & Claiming**: The job is pushed to the Redis stream `task:dispatch:background-agent`. A client-side worker claims the task.
4. **Knowledge Retrieval**: The worker queries the server-side Knowledge Service RAG endpoint for internal organizational standards.
5. **Multi-Cloud MCP Inspection**: The worker queries the HashiCorp, AWS, and Azure MCP tools to fetch the correct Terraform block structures and security parameters.
6. **Code Generation & Sandboxed Validation**: The worker runs inside an ephemeral Docker container:
   - Modifies `main.tf`
   - Executes `terraform fmt`
   - Executes `checkov` security scanning
   - Runs `terraform plan` to verify zero unexpected drift
7. **PR Submission**: The worker creates a feature branch (`agent/remediation-s3-azure-123`), commits the changes, pushes to Git, and opens a Pull Request.
8. **Real-Time Streaming**: Throughout execution, every tool execution, log line, and diff preview streams to the Dashboard via Server-Sent Events (SSE).
9. **Human Merge & Build**: The human engineer inspects the PR in Git or via the Dashboard link, approves it, and merges it. The standard CI pipeline executes `terraform apply`.

---

## 4. Technology & Infrastructure Matrix

### 4.1 Software & Framework Inventory

| Layer | Component | Software / Framework | Port | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Frontend** | `agent-ui` | React / Next.js | `3000` | User login, chat interface, SSE stream viewer |
| **Backend API** | `api-gateway` | Node.js / Express | `8080` | Auth, signal router, session management |
| **Policy** | `policy-engine` | Open Policy Agent (OPA) | `8181` | Enforces Rego security guardrails |
| **Task Queue** | `task-queue` | Redis 7 (Streams) | `6379` | Asynchronous task distribution |
| **Database** | `database` | PostgreSQL 16 | `5432` | Persists session records and task status |
| **Knowledge RAG**| `knowledge-service`| Python FastAPI + Vector Store | `8000` | Serves internal IaC & policy RAG search |
| **Local LLM** | `local-llm` | Ollama (`qwen2.5-coder:7b`) | `11434` | Private local LLM inference engine |
| **MCP HashiCorp** | `mcp-hashicorp` | Node.js `@modelcontextprotocol/sdk` | Internal | Terraform Registry & state lookup |
| **MCP AWS** | `mcp-aws` | Python / Node.js MCP Server | Internal | AWS resource schemas & IAM lookup |
| **MCP Azure** | `mcp-azure` | Python / Node.js MCP Server | Internal | Azure ARM/Bicep schema lookup |
| **MCP GCP** | `mcp-gcp` | Python / Node.js MCP Server | Internal | GCP resource schemas & API specs |
| **Worker** | `agent-worker` | Node.js / Python | Internal | Orchestrates inner loop & sandboxes |

---

## 5. File & Manifest Specifications

### 5.1 OPA Guardrails Policy (`policies/infra_guardrails.rego`)
```rego
package infra.authz

default allow = false

# Allow read, document lookup, and plan operations
allow {
    input.action == "read"
}
allow {
    input.action == "mcp_lookup"
}
allow {
    input.action == "terraform_plan"
}
allow {
    input.action == "create_pr"
}

# Explicitly DENY direct production execution
deny {
    input.action == "terraform_apply"
}
deny {
    input.action == "destroy"
}
```

### 5.2 Multi-Cloud Skill Definition (`skills/terraform-plan/SKILL.md`)
```markdown
---
name: terraform-plan
description: Multi-cloud IaC remediation skill using HashiCorp, AWS, Azure, and GCP MCP servers.
allowed_tools:
  - "mcp__hashicorp__search_registry"
  - "mcp__aws__get_resource_schema"
  - "mcp__azure__get_bicep_resource_schema"
  - "mcp__gcp__get_terraform_provider_doc"
  - "mcp__infra-tools__terraform_plan"
  - "mcp__infra-tools__checkov_scan"
---

# Multi-Cloud Terraform Plan & PR Skill

## Mandatory Rules
1. ALWAYS perform RAG lookup for internal compliance rules first.
2. Query the respective Cloud MCP server (AWS, Azure, GCP, HashiCorp) to verify schema specifications before drafting code.
3. Formats and lints code using `terraform fmt` and `checkov` inside the sandbox environment.
4. NEVER attempt direct apply. Output MUST be delivered as a Pull Request.
```

---

## 6. Project Roadmap & Milestone Deliverables

```
Phase 1: Foundation (Weeks 1-2)
  ├── Setup Docker Compose monorepo
  ├── Deploy local Ollama with qwen2.5-coder:7b
  └── Deploy Redis & PostgreSQL state stores

Phase 2: Core Control Plane & Guardrails (Weeks 3-4)
  ├── Implement API Gateway & OPA Policy Engine
  ├── Implement Knowledge Service RAG Registry
  └── Build Next.js Dashboard UI with SSE event streaming

Phase 3: Multi-Cloud MCP & Sandbox Integration (Weeks 5-6)
  ├── Integrate HashiCorp, AWS, Azure, and GCP MCP servers
  ├── Build ephemeral Docker execution sandbox container
  └── Implement client-side worker agent inner loop

Phase 4: End-to-End Testing & Hardening (Weeks 7-8)
  ├── Test full prompt -> RAG -> MCP -> Sandbox -> Git PR flow
  ├── Verify strict OPA blocking of direct apply commands
  └── Finalize documentation & developer guide
```

---

## 7. Conclusion & Next Steps

This proposal provides a complete blueprint for an enterprise-grade, privacy-first infrastructure AI agent system. By isolating the **Client-Side execution sandbox** from the **Server-Side control tower**, leveraging **local LLMs**, integrating **multi-cloud MCP servers**, and enforcing a **PR-First change policy**, organizations can safely automate cloud infrastructure management without risking security breaches or unwanted cloud mutations.