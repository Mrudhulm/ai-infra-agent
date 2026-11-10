# Project Proposal & Local Implementation Guide
## Self-Hosted, Autonomous Infrastructure AI Agent Platform

---

## 1. Executive Summary & Project Proposal

### 1.1 Overview
This project proposes the deployment of a **100% Local, Secure AI Infrastructure Agent System**. The platform enables non-expert and expert users alike to interact with an AI agent via an intuitive Web Dashboard to manage, scan, plan, and optimize local infrastructure code (such as Terraform, Docker, and Kubernetes) safely.

By using **Ollama** for local LLM inference alongside containerized microservices (Task Queue, Policy Engine, Knowledge Base, Credential Broker, and Execution Sandbox), all data, code, and prompts remain completely on your local machine with zero external cloud dependencies or API costs.

### 1.2 Core System Goals
- **100% Privacy & Zero API Cost:** Powered by Ollama (`qwen2.5-coder` or `llama3.1`) running locally.
- **Interactive User Dashboard:** A web-based UI (`http://localhost:3000`) for login, prompting, real-time log streaming, and diff verification.
- **Strict Policy Guardrails:** Autonomous ceilings enforced via Open Policy Agent (OPA) to prevent accidental execution of destructive commands.
- **Just-In-Time Credentials & Isolated Sandboxing:** Code formatting, linting, scanning, and plan generation occur inside ephemeral Docker sandboxes.

---

## 2. Target User Experience (Dashboard Workflow)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            LOCAL WEB DASHBOARD                              │
│                            (http://localhost:3000)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. User Logs In  ──►  2. Submits Infrastructure Request                    │
│                        (e.g., "Scan my Terraform files for S3 security")    │
│                                                                             │
│  3. Real-Time Execution Updates:                                            │
│     ├── [Policy Engine] Check permissions -> PASSED (Propose Mode)         │
│     ├── [Knowledge Base] Retreiving IaC code context...                     │
│     ├── [Local LLM] Reasoning via qwen2.5-coder:7b...                       │
│     ├── [MCP Server] Running `terraform fmt` & `checkov` in Sandbox...     │
│     └── [Output Stream] Generating proposed changes & Git PR link          │
│                                                                             │
│  4. User inspects side-by-side Diff & approves PR                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. System Requirements & Software Inventory

### 3.1 Prerequisites on Host Machine
- **Operating System:** macOS, Linux, or Windows (with WSL2).
- **RAM:** Minimum 16 GB (32 GB recommended for running local LLMs and containers smoothly).
- **Disk Space:** At least 20 GB free.

### 3.2 Required Installed Software
1. **Docker Desktop** (or Docker Engine + Docker Compose)
2. **Ollama** (for local AI inference)
3. **Git** and **Bash** (or Git Bash on Windows)

### 3.3 Microservice Container Architecture

| Container Name | Base Software | Host Port | Role |
| :--- | :--- | :--- | :--- |
| `agent-ui` | Node.js (React / Next.js) | `3000:3000` | Web portal login, chat interface, SSE log display |
| `api-gateway` | Express / FastAPI | `8080:8080` | Normalizes signals, validates sessions, routes requests |
| `policy-engine` | Open Policy Agent (OPA) | `8181:8181` | Enforces autonomy ceilings (`Observe`, `Propose`) |
| `task-queue` | Redis Streams | `6379:6379` | Manages async task dispatch and event streams |
| `database` | PostgreSQL 16 | `5432:5432` | Stores metadata, user credentials, action receipts |
| `knowledge-service` | Python FastAPI | `8000:8000` | Serializes infrastructure assets into compact Markdown |
| `credential-broker` | Node.js | `3001:3001` | Issues short-lived mock/local execution tokens |
| `mcp-server-infra` | Node.js (MCP SDK) | `3002:3002` | Exposes typed infrastructure tools over stdio/HTTP |
| `agent-worker` | Node.js / Python | Internal | Orchestrates the inner reasoning loop with local LLM |
| `local-llm` | Ollama Container | `11434:11434` | Serves local AI model (`qwen2.5-coder:7b`) |

---

## 4. Beginner Step-by-Step Implementation Guide

### Phase 1: Install Ollama & Pull the AI Model
1. Download and install **Ollama** from [https://ollama.com](https://ollama.com).
2. Open your terminal and pull the local coding LLM:
   ```bash
   ollama pull qwen2.5-coder:7b
   ```
3. Test that Ollama is responding locally:
   ```bash
   curl http://localhost:11434/api/generate -d '{"model": "qwen2.5-coder:7b", "prompt": "Hello!"}'
   ```

---

### Phase 2: Project Setup & Monorepo Directory Creation
Run the following commands in your terminal to create the required directory structure:

```bash
mkdir -p infra-agent-system/{policies,skills/terraform-plan,packages/{api-gateway,credential-broker,knowledge-service,mcp-server,worker},sandboxes/docker}
cd infra-agent-system
```

---

### Phase 3: Create Core System Files

#### 1. Define Guardrail Policy (`policies/infra_guardrails.rego`)
```rego
package infra.authz

default allow = false

# Allow read & propose operations
allow {
    input.action == "read"
}
allow {
    input.action == "terraform_plan"
}

# Block direct production modifications
deny {
    input.action == "terraform_apply"
}
```

#### 2. Define Agent Skill (`skills/terraform-plan/SKILL.md`)
```markdown
---
name: terraform-plan
description: Execute terraform plan and validate IaC changes safely.
allowed_tools: ["mcp__infra-tools__terraform_plan", "mcp__infra-tools__checkov_scan"]
---

# Terraform Plan & Validation Skill

## Execution Rules
1. ALWAYS run checkov linting before running plan.
2. NEVER execute direct terraform apply; output must be presented as a Pull Request diff.
3. Maximum 10 plan iterations per session.
```

#### 3. Define Docker Orchestration (`docker-compose.yml`)
```yaml
version: '3.8'

networks:
  agent-net:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  ollama-data:

services:
  database:
    image: postgres:16-alpine
    container_name: agent-postgres
    environment:
      POSTGRES_DB: agent_db
      POSTGRES_USER: agent_admin
      POSTGRES_PASSWORD: LocalSecurePassword123!
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - agent-net

  task-queue:
    image: redis:7-alpine
    container_name: agent-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - agent-net

  local-llm:
    image: ollama/ollama:latest
    container_name: agent-local-llm
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    networks:
      - agent-net

  policy-engine:
    image: openpolicyagent/opa:latest
    container_name: agent-policy-engine
    ports:
      - "8181:8181"
    volumes:
      - ./policies:/policies
    command: ["run", "--server", "--addr=0.0.0.0:8181", "/policies"]
    networks:
      - agent-net

  api-gateway:
    build: ./packages/api-gateway
    container_name: agent-api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - database
      - task-queue
    networks:
      - agent-net

  agent-worker:
    build: ./packages/worker
    container_name: agent-worker
    environment:
      - LLM_API_BASE=http://agent-local-llm:11434/v1
      - LLM_MODEL=qwen2.5-coder:7b
      - REDIS_URL=redis://agent-redis:6379
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./skills:/skills
    depends_on:
      - local-llm
      - task-queue
      - policy-engine
    networks:
      - agent-net
```

---

### Phase 4: Spin Up the Stack & Verify
1. Launch all microservice containers in background mode:
   ```bash
   docker compose up --build -d
   ```
2. Verify all 10 containers show status `Up`:
   ```bash
   docker compose ps
   ```
3. Open your browser and navigate to:
   ```text
   http://localhost:3000
   ```
4. Log into the local dashboard, type a prompt (e.g., *"Scan my local Terraform code for security vulnerabilities"*), and watch real-time execution outputs streamed live on your dashboard!

---

## 5. Summary & Key Milestones

- [x] Complete architecture proposal with 6 security planes.
- [x] Private local LLM engine setup using Ollama.
- [x] Monorepo folder layout and container configurations.
- [x] OPA policies & `SKILL.md` progressive disclosure integration.
- [x] Web dashboard UI interface setup for end-user interaction.