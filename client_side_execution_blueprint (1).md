# Client-Side Architecture & Execution Blueprint

> **Role**: The Client-Side layer acts as the local agent runner, execution sandbox, and developer interface. It hosts the local SDK/CLI tools, executes local IaC commands (`terraform`, `checkov`, `git`), queries multi-cloud MCP servers (HashiCorp, AWS, Azure, GCP), interfaces with the local Ollama LLM, and creates secure Git Pull Requests without holding permanent cloud secrets.

---

## 1. Client-Side Responsibilities & Workflow

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENT-SIDE ENVIRONMENT                                        │
│                                                                                                  │
│  ┌─────────────────────────┐         ┌────────────────────────────────────────────────────────┐  │
│  │ Local SDK / CLI Runner  │ ──────► │ Local Execution Sandbox (Docker / CLI)                 │  │
│  │ (Claude SDK / Ollama)   │         └───────────────────────────┬────────────────────────────┘  │
│  └────────────┬────────────┘                                     │                               │
│               │ Pulls Skills                                     │ Executes Multi-Cloud Tools    │
│               ▼                                                  ▼                               │
│  ┌─────────────────────────┐         ┌────────────────────────────────────────────────────────┐  │
│  │ Skills (`SKILL.md`)     │         │ Dedicated MCP Client Hub                               │  │
│  └─────────────────────────┘         │  ├── HashiCorp MCP (Modules, Registry, State Docs)      │  │
│                                      │  ├── AWS MCP (Resource Schemas, IAM, Docs)              │  │
│                                      │  ├── Azure MCP (ARM/Bicep Schemas, Docs)                │  │
│                                      │  └── GCP MCP (GDM/Terraform Docs, API Schemas)          │  │
│                                      └───────────────────────────┬────────────────────────────┘  │
│                                                                  │                               │
│                                                                  ▼                               │
│                                      ┌────────────────────────────────────────────────────────┐  │
│                                      │ Git PR Engine (Branch -> Commit -> Push -> Open PR)   │  │
│                                      └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

1. **Local Reasoning & Tool Execution**: Drives the inner agentic loop (`LLM -> Tool Use -> Execute -> Output -> LLM`) using local Ollama (`qwen2.5-coder:7b`) or an agent SDK.
2. **Multi-Cloud MCP Tooling**: Queries official MCP servers for HashiCorp, AWS, Azure, and GCP to fetch live resource schemas, provider specs, and documentation.
3. **Local Knowledge RAG Integration**: Pulls local policy docs and architectural guides from the server-side Knowledge RAG Registry.
4. **Sandboxed Command Execution**: Runs commands such as `terraform fmt`, `tflint`, `checkov`, and `terraform plan` inside a controlled, isolated workspace.
5. **Git PR Authoring**: Pushes generated infrastructure changes to a feature branch and creates a Pull Request (PR) for human review. It **never** runs `terraform apply` directly.

---

## 2. Directory Structure (Client SDK / Workspace)

```text
client-workspace/
├── .env.local
├── skills/
│   ├── terraform-plan/
│   │   └── SKILL.md
│   ├── cloud-scan/
│   │   └── SKILL.md
│   └── multicloud-lookup/
│       └── SKILL.md
├── src/
│   ├── agent_runner.py        # Inner loop runner connecting Ollama, RAG & MCP tools
│   ├── mcp_clients/           # Multi-Cloud MCP Client connectors
│   │   ├── hashicorp_mcp.py   # Registry lookup, provider docs & TF schemas
│   │   ├── aws_mcp.py         # AWS resource specs, IAM & Boto3 docs
│   │   ├── azure_mcp.py       # Azure ARM / Bicep & Provider docs
│   │   └── gcp_mcp.py         # GCP Terraform & Resource API specs
│   └── sandbox_executor.py    # Executes sandboxed bash/IaC commands
└── workspace/
    └── infra-repo/            # Local Terraform repository targeted for remediation
        └── main.tf
```

---

## 3. Client SKILL Specification (`skills/terraform-plan/SKILL.md`)

```markdown
---
name: terraform-plan
description: Safely format, lint, plan, and create PRs for multi-cloud Terraform infrastructure changes using HashiCorp, AWS, Azure, and GCP MCP documentation servers.
allowed_tools: 
  - "mcp__hashicorp__search_registry"
  - "mcp__aws__get_resource_schema"
  - "mcp__azure__get_bicep_resource_schema"
  - "mcp__gcp__get_terraform_provider_doc"
  - "mcp__infra-tools__terraform_plan"
  - "mcp__infra-tools__checkov_scan"
---

# Multi-Cloud Terraform Plan & PR Authoring Skill

## Execution Constraints
- **Doc Lookup First**: Query HashiCorp, AWS, Azure, or GCP MCP tools to verify provider syntax and mandatory security attributes BEFORE generating Terraform code.
- **Validation First**: ALWAYS run `terraform fmt` and `checkov` before generating a plan.
- **Strict Change Control**: NEVER run `terraform apply`. All modifications MUST be output as a Git feature branch and submitted as a Pull Request.
- **Iteration Cap**: Limit plan correction iterations to 10 attempts per session. If unresolvable drift remains, submit a Draft PR with the diff attached.
```

---

## 4. Multi-Cloud Client Agent Runner Script (`src/agent_runner.py`)

```python
import os
import requests
import json

# Local Ollama LLM endpoint & Server-side Knowledge RAG
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://localhost:11434/api/generate")
RAG_SERVICE_URL = os.getenv("RAG_SERVICE_URL", "http://localhost:8000/context/rag-search")
MODEL = "qwen2.5-coder:7b"

def query_local_rag(query: str):
    """Fetch internal architecture guidelines from Server Knowledge Base RAG Registry."""
    try:
        res = requests.get(RAG_SERVICE_URL, params={"q": query})
        return res.json().get("markdown", "")
    except Exception as e:
        return f"RAG Query Offline: {e}"

def run_local_agent_step(prompt: str, skill_context: str, repo_path: str):
    # 1. Retrieve internal RAG knowledge
    rag_docs = query_local_rag(prompt)

    full_prompt = f"""
    [SYSTEM INSTRUCTIONS]
    You are a Multi-Cloud Local Infrastructure Agent (AWS, Azure, GCP, HashiCorp).
    Adhere strictly to the rules and skills below:
    {skill_context}
    
    [INTERNAL RAG KNOWLEDGE BASE]
    {rag_docs}
    
    [TASK]
    {prompt}
    
    [TARGET REPOSITORY]
    {repo_path}
    """
    
    payload = {
        "model": MODEL,
        "prompt": full_prompt,
        "stream": False
    }
    
    response = requests.post(OLLAMA_URL, json=payload)
    return response.json().get("response", "")

if __name__ == "__main__":
    prompt = "Scan main.tf in /workspace/infra-repo and add S3 server-side encryption and an Azure Storage account using latest HashiCorp provider standards."
    with open("skills/terraform-plan/SKILL.md", "r") as f:
        skill = f.read()
    
    print("Executing local multi-cloud agent reasoning...")
    result = run_local_agent_step(prompt, skill, "/workspace/infra-repo")
    print("\n--- Agent Response ---")
    print(result)
```