# Local AI Infrastructure Agent System (Docker Blueprint)

> A complete microservice architecture, step-by-step build guide, and Docker Compose specification for running a secure infrastructure agent locally connected to your local LLM (Ollama / vLLM / LocalAI), complete with Skills (`SKILL.md`), Data/Knowledge Plane serialization, and Model Context Protocol (MCP) servers.

---

## 1. High-Level Architecture (Local Microservice Stack)

Every component runs in its own isolated Docker container. Your local LLM handles reasoning, while a dedicated Credential Broker and Sandboxed Runner execute infrastructure commands safely.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 DOCKER VIRTUAL NETWORK                                 │
│                                                                                        │
│  ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────────────────┐   │
│  │   UI / Portal   │──────▶│   API Gateway   │──────▶│         Policy Engine       │   │
│  │ (Next.js / Port)│ HTTP  │ (FastAPI/Node)  │ HTTP  │  (OPA / Rule Evaluator)     │   │
│  └─────────────────┘       └────────┬────────┘       └──────────────┬──────────────┘   │
│                                     │                               │                  │
│                                     │ Task Dispatch                 │ Allowed Action   │
│                                     ▼                               ▼                  │
│                            ┌─────────────────┐             ┌────────────────────────┐  │
│                            │ Task Queue & DB │             │   JIT Credential       │  │
│                            │(Redis + Postgres│             │      Broker            │  │
│                            └────────┬────────┘             └───────────┬────────────┘  │
│                                     │                                  │               │
│                                     │ Claim Task                       │ Issue Token   │
│                                     ▼                                  ▼               │
│                            ┌────────────────────────────────────────────────────────┐  │
│                            │                    Agent Worker                        │  │
│                            │   (Drives LLM Loop & Orchestrates Execution)           │  │
│                            └────────┬───────────────────────────────────────┬───────┘  │
│                                     │                                       │          │
│                    Prompt & Tools   │ Execute via MCP                       │ Reads    │
│                    ┌────────────────┴──────────────┐         ┌──────────────┴────────┐ │
│                    ▼                               ▼         ▼                       │ │
│      ┌───────────────────────────┐   ┌───────────────────────────┐   ┌───────────────┐ │
│      │   Local LLM Container     │   │    MCP Server (Tools)     │   │ SKILL.md      │ │
│      │  (Ollama / vLLM / LM Studio)   │ (Model Context Protocol)  │   │ (Progressive) │ │
│      └───────────────────────────┘   └───────────────────────────┘   └───────────────┘ │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Microservice Container Breakdown

| Microservice Container | Dedicated Role | Base Image / Tool | Port / Volume |
| :--- | :--- | :--- | :--- |
| `agent-ui` | User dashboard to trigger tasks, view diffs, approve actions | Node.js / React / Next.js | `3000:3000` |
| `api-gateway` | Exposes REST APIs, receives webhooks, streams logs via SSE | Python FastAPI or Node.js | `8080:8080` |
| `policy-engine` | Evaluates if actions are safe (Observe / Propose / Governed) | Open Policy Agent (OPA) | `8181:8181` (`./policies`) |
| `task-queue` | Dispatches jobs and manages agent output streams | Redis Streams / BullMQ | `6379:6379` (`redis-data`) |
| `database` | Stores session history, target repos, and metadata | PostgreSQL (with pgvector) | `5432:5432` (`postgres-data`) |
| `knowledge-service`| Scans cloud assets and IaC repos, serializes to Markdown | Python FastAPI (Data Plane) | `8000:8000` |
| `mcp-server-infra`| Exposes typed infrastructure tools over HTTP/stdio | Node.js (`@modelcontextprotocol`) | `3002:3002` |
| `credential-broker`| Injects short-lived credentials into execution sandboxes | Custom Go / Python Microservice | `3001:3001` |
| `agent-worker` | Runs the agentic inner loop (Prompt -> LLM -> Tool Call) | Python / TS + Agent SDK | Internal Worker |
| `execution-sandbox`| Ephemeral execution workspace with security tools | Debian / Ubuntu + `terraform`, `aws-cli` | Ephemeral (`tmpfs`) |
| `local-llm` | Runs local AI model locally (`qwen2.5-coder`, `llama3.1`) | Ollama (`ollama/ollama`) | `11434:11434` (`ollama-data`) |
| `otel-collector` | Aggregates OpenTelemetry spans, metrics, and Action Trails | Otel Collector Contrib | `4317:4317` / `4318:4318` |

---

## 3. Dedicated Project Directory Structure

```
infra-agent-system/
├── docker-compose.yml
├── .env.example
├── README.md
├── policies/
│   └── infra_guardrails.rego
├── skills/
│   └── terraform-plan/
│       └── SKILL.md
├── packages/
│   ├── api-gateway/
│   ├── credential-broker/
│   ├── knowledge-service/      # Data Plane Engine (Chapter 6)
│   ├── mcp-server/             # Model Context Protocol Server (Chapter 3)
│   └── worker/                 # Inner/Outer Loop Execution Worker
└── sandboxes/
    └── docker/
        ├── Dockerfile.sandbox
        └── entrypoint.sh
```

---

## 4. Skills Architecture (`SKILL.md`) Example

Create `skills/terraform-plan/SKILL.md`:

```markdown
---
name: terraform-plan
description: Execute terraform plan and validate IaC changes safely. Use when writing or verifying Terraform files.
allowed_tools: ["mcp__infra-tools__terraform_plan", "mcp__infra-tools__checkov_scan"]
---

# Terraform Plan & Validation Skill

## Execution Rules
- ALWAYS run `terraform fmt` and `checkov` before running plan.
- NEVER execute `terraform apply` directly. Changes MUST flow through a Pull Request.
- Maximum 10 plan iteration attempts per session. If drift remains after 10 turns, open a Draft PR marked `[not converged]`.
```

---

## 5. Production-Grade Docker Compose File

Save this file as `docker-compose.yml` in your project root:

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

  knowledge-service:
    build: ./packages/knowledge-service
    container_name: agent-knowledge
    ports:
      - "8000:8000"
    networks:
      - agent-net

  credential-broker:
    build: ./packages/credential-broker
    container_name: agent-cred-broker
    ports:
      - "3001:3001"
    environment:
      - REDIS_URL=redis://agent-redis:6379
      - DB_URL=postgresql://agent_admin:LocalSecurePassword123!@agent-postgres:5432/agent_db
    networks:
      - agent-net

  mcp-server-infra:
    build: ./packages/mcp-server
    container_name: agent-mcp-server
    ports:
      - "3002:3002"
    networks:
      - agent-net

  api-gateway:
    build: ./packages/api-gateway
    container_name: agent-api-gateway
    ports:
      - "8080:8080"
    environment:
      - REDIS_URL=redis://agent-redis:6379
      - DB_URL=postgresql://agent_admin:LocalSecurePassword123!@agent-postgres:5432/agent_db
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
      - KNOWLEDGE_SERVICE_URL=http://agent-knowledge:8000
      - MCP_SERVER_URL=http://agent-mcp-server:3002
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

## 6. Step-by-Step Local Deployment

1. **Create directories**:
   ```bash
   mkdir -p infra-agent-system/{policies,skills/terraform-plan,packages/{api-gateway,credential-broker,knowledge-service,mcp-server,worker}}
   cd infra-agent-system
   ```
2. **Download local LLM model**:
   ```bash
   docker compose up -d local-llm
   docker exec -it agent-local-llm ollama pull qwen2.5-coder:7b
   ```
3. **Start the complete stack**:
   ```bash
   docker compose up --build -d
   ```
4. **Verify container health**:
   ```bash
   docker compose ps
   ```