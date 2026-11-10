# Client-Side Architecture & Execution Blueprint

> **Role**: The Client-Side layer acts as the local agent runner, execution sandbox, and developer interface. It hosts the local SDK/CLI tools, executes local IaC commands (`terraform`, `checkov`, `git`), interfaces with the local Ollama LLM, and creates secure Git Pull Requests without holding permanent cloud secrets.

---

## 1. Client-Side Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CLIENT-SIDE ENVIRONMENT                          │
│                                                                             │
│  ┌─────────────────────────┐        ┌────────────────────────────────────┐  │
│  │ Local SDK / CLI Runner  │ ─────► │ Local Execution Sandbox            │  │
│  │ (Claude SDK / Ollama)   │        │ (Docker Container / Local CLI)     │  │
│  └────────────┬────────────┘        └─────────────────┬──────────────────┘  │
│               │                                       │                     │
│               │ Pulls Skills & Prompt                 │ Executes Tools      │
│               ▼                                       ▼                     │
│  ┌─────────────────────────┐        ┌────────────────────────────────────┐  │
│  │ Skills (`SKILL.md`)     │        │ Tools (`terraform plan`, `git`)    │  │
│  └─────────────────────────┘        └─────────────────┬──────────────────┘  │
│                                                       │                     │
│                                                       ▼                     │
│                                     ┌────────────────────────────────────┐  │
│                                     │ Opens Git Pull Request (PR)        │  │
│                                     └────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

1. **Local Reasoning & Tool Execution**: Drives the inner agentic loop (`LLM -> Tool Use -> Execute -> Output -> LLM`) using local Ollama (`qwen2.5-coder:7b`) or an agent SDK.
2. **Context & Skill Loading**: Reads local directory files, loads `SKILL.md` rules, and injects IaC context into the prompt window.
3. **Sandboxed Command Execution**: Runs commands such as `terraform fmt`, `tflint`, `checkov`, and `terraform plan` inside a controlled, isolated workspace.
4. **Git PR Authoring**: Pushes generated infrastructure changes to a feature branch and creates a Pull Request (PR) for human review. It **never** runs `terraform apply` directly.

---

## 2. Directory Structure (Client SDK / Workspace)

```text
client-workspace/
├── .env.local
├── skills/
│   ├── terraform-plan/
│   │   └── SKILL.md
│   └── cloud-scan/
│       └── SKILL.md
├── src/
│   ├── agent_runner.py        # Inner loop runner connecting Ollama & MCP tools
│   ├── mcp_client.py          # Invokes local MCP infrastructure tools
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
description: Safely format, lint, plan, and create PRs for Terraform infrastructure changes.
allowed_tools: ["mcp__infra-tools__terraform_plan", "mcp__infra-tools__checkov_scan"]
---

# Terraform Plan & PR Authoring Skill

## Execution Constraints
- **Validation First**: ALWAYS run `terraform fmt` and `checkov` before generating a plan.
- **Strict Change Control**: NEVER run `terraform apply`. All modifications MUST be output as a Git feature branch and submitted as a Pull Request.
- **Iteration Cap**: Limit plan correction iterations to 10 attempts per session. If unresolvable drift remains, submit a Draft PR with the diff attached.
```

---

## 4. Client Agent Runner Script (`src/agent_runner.py`)

```python
import os
import requests

# Local Ollama LLM endpoint
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://localhost:11434/api/generate")
MODEL = "qwen2.5-coder:7b"

def run_local_agent_step(prompt: str, skill_context: str, repo_path: str):
    full_prompt = f"""
    [SYSTEM INSTRUCTIONS]
    You are a local Infrastructure Agent. Adhere strictly to the rules below:
    {skill_context}
    
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
    prompt = "Scan main.tf in /workspace/infra-repo and fix any unencrypted S3 bucket resources."
    with open("skills/terraform-plan/SKILL.md", "r") as f:
        skill = f.read()
    
    print("Executing local agent reasoning...")
    result = run_local_agent_step(prompt, skill, "/workspace/infra-repo")
    print("\n--- Agent Response ---")
    print(result)
```