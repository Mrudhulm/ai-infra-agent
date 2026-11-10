# Local AI Infrastructure Agent System (Docker Blueprint)

> A complete microservice architecture, step-by-step build guide, and Docker Compose specification for running a secure infrastructure agent locally connected to your local LLM (Ollama / vLLM / LocalAI).

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
│                            └────────┬───────────────────────────────────────────────┘  │
│                                     │                                                  │
│                    Prompt & Tools   │   Execute Command in Sandbox                     │
│                    ┌────────────────┴─────────────────────────────┐                    │
│                    ▼                                              ▼                    │
│      ┌───────────────────────────┐                  ┌───────────────────────────┐      │
│      │   Local LLM Container     │                  │    Execution Sandbox      │      │
│      │  (Ollama / vLLM / LM Studio)                 │(Terraform, AWS CLI, Git)  │      │
│      └───────────────────────────┘                  └───────────────────────────┘      │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Microservice Container Breakdown

| Microservice Container | Dedicated Role | Base Image / Tool |
| :--- | :--- | :--- |
| `agent-ui` | User dashboard to trigger tasks, view diffs, approve actions | Node.js / React / Next.js |
| `api-gateway` | Exposes REST APIs, receives webhooks, streams logs via SSE | Python FastAPI or Node.js Express |
| `policy-engine` | Evaluates if actions are safe (Observe / Propose / Governed) | Open Policy Agent (OPA) |
| `task-queue` | Dispatches jobs and manages agent output streams | Redis Streams / BullMQ |
| `database` | Stores session history, target repos, and metadata | PostgreSQL (with pgvector) |
| `agent-worker` | Runs the agentic inner loop (Prompt -> LLM -> Tool Call) | Python (LangChain/PydanticAI) or TS |
| `credential-broker` | Injects short-lived credentials into execution sandboxes | Custom Go / Python Microservice |
| `execution-sandbox` | Ephemeral execution workspace with security tools | Debian / Ubuntu + `terraform`, `aws-cli`, `checkov` |
| `local-llm` | Runs local AI model locally (e.g., `qwen2.5-coder`, `llama3.1`) | Ollama (`ollama/ollama`) |

---

## 3. Dedicated Project Directory Structure

Organize your local workspace as a monorepo containing distinct microservices:

```text
local-infra-agent/
├── docker-compose.yml
├── .env.example
├── services/
│   ├── api-gateway/
│   │   ├── Dockerfile
│   │   ├── main.py
│   │   └── requirements.txt
│   ├── policy-engine/
│   │   ├── Dockerfile
│   │   └── policies/
│   │       └── infra_guardrails.rego
│   ├── credential-broker/
│   │   ├── Dockerfile
│   │   └── server.py
│   ├── agent-worker/
│   │   ├── Dockerfile
│   │   ├── agent.py
│   │   └── tools/
│   │       ├── terraform.py
│   │       └── git_ops.py
│   └── execution-sandbox/
│       ├── Dockerfile.sandbox
│       └── scripts/
│           └── entrypoint.sh
└── storage/
    ├── postgres-data/
    ├── redis-data/
    └── workspace-repos/
```

---

## 4. Production-Grade Docker Compose File

Save this file as `docker-compose.yml` in your project root to run the entire system locally:

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
  # 1. Database
  postgres:
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

  # 2. Message Bus & Task Queue
  redis:
    image: redis:7-alpine
    container_name: agent-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - agent-net

  # 3. Local LLM Service (Ollama)
  local-llm:
    image: ollama/ollama:latest
    container_name: agent-local-llm
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    networks:
      - agent-net
    # Enable GPU if available (NVIDIA Docker Toolkit)
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]

  # 4. Policy Engine (OPA)
  policy-engine:
    image: openpolicyagent/opa:latest
    container_name: agent-policy-engine
    ports:
      - "8181:8181"
    command:
      - "run"
      - "--server"
      - "--log-level=info"
    networks:
      - agent-net

  # 5. JIT Credential Broker
  credential-broker:
    build:
      context: ./services/credential-broker
    container_name: agent-cred-broker
    environment:
      - REDIS_URL=redis://agent-redis:6379
      - DB_URL=postgresql://agent_admin:LocalSecurePassword123!@agent-postgres:5432/agent_db
    networks:
      - agent-net

  # 6. API Gateway & Ingestion
  api-gateway:
    build:
      context: ./services/api-gateway
    container_name: agent-api-gateway
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://agent-redis:6379
      - DB_URL=postgresql://agent_admin:LocalSecurePassword123!@agent-postgres:5432/agent_db
    depends_on:
      - postgres
      - redis
    networks:
      - agent-net

  # 7. Stateless Agent Worker
  agent-worker:
    build:
      context: ./services/agent-worker
    container_name: agent-worker
    environment:
      - LLM_API_BASE=http://agent-local-llm:11434/v1
      - LLM_MODEL=qwen2.5-coder:7b
      - REDIS_URL=redis://agent-redis:6379
      - POLICY_ENGINE_URL=http://agent-policy-engine:8181/v1/data/infra/allow
      - CREDENTIAL_BROKER_URL=http://agent-cred-broker:8080
    depends_on:
      - local-llm
      - redis
      - policy-engine
    networks:
      - agent-net
```

---

## 5. End-to-End Workflow Execution Sequence

Here is how a request like **"Create an S3 bucket with encryption enabled"** flows securely through your microservices:

```
[ User Input / Webhook ]
           │
           ▼
1. API Gateway ──────────────► 2. Policy Engine
   (Accepts Request)              (Checks rule: Is user allowed to request code generation?)
                                         │
                                         ▼ Allowed
                               3. Redis Task Queue
                                  (Pushes task payload)
                                         │
                                         ▼ Worker Claims Task
                               4. Agent Worker
                                  │
                                  ├─► Calls Local LLM (Ollama) to draft HCL Terraform code
                                  │
                                  ├─► Requests Short-Lived Token from Credential Broker
                                  │
                                  └─► Spawns Ephemeral Execution Sandbox
                                         │
                                         ▼ Executes inside Sandbox
                               5. Execution Sandbox
                                  │
                                  ├─► 1. Writes `vpc.tf` file
                                  ├─► 2. Runs `terraform validate`
                                  ├─► 3. Runs `checkov` security scan
                                  └─► 4. Runs `terraform plan -json`
                                         │
                                         ▼ If validation passes (Zero Drift)
                               6. Git Service
                                  (Pushes branch & opens Pull Request for human review)
```

---

## 6. Step-by-Step Build & Setup Guide

### Step 1: Clone & Prepare Structure
Create your local folders:
```bash
mkdir -p local-infra-agent/services/{api-gateway,policy-engine,credential-broker,agent-worker,execution-sandbox}
cd local-infra-agent
```

### Step 2: Download Local LLM Model
Start Ollama and download a coding model optimized for infrastructure (e.g., `qwen2.5-coder` or `llama3.1`):
```bash
docker compose up -d local-llm
docker exec -it agent-local-llm ollama pull qwen2.5-coder:7b
```

### Step 3: Configure OPA Safety Rules
Create `services/policy-engine/infra_guardrails.rego`:
```rego
package infra.allow

default allow = false

# Allow read & plan tasks by default
allow {
    input.action == "terraform_plan"
}

# Deny direct apply commands — force PR workflow
deny {
    input.action == "terraform_apply"
}
```

### Step 4: Build Agent Worker Loop
In `services/agent-worker/agent.py`, configure the OpenAI/LangChain client to point to your local LLM container:
```python
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.getenv("LLM_API_BASE", "http://agent-local-llm:11434/v1"),
    api_key="ollama" # Dummy key required for local server
)

def generate_iac(prompt: str):
    response = client.chat.completions.create(
        model=os.getenv("LLM_MODEL", "qwen2.5-coder:7b"),
        messages=[
            {"role": "system", "content": "You are an Infrastructure-as-Code agent. Output valid Terraform HCL only."},
            {"role": "user", "content": prompt}
        ]
    )
    return response.choices[0].message.content
```

### Step 5: Start the Full Stack
Launch all dedicated microservices locally:
```bash
docker compose up --build -d
```

### Step 6: Verify Stack Health
Check that all containers are active:
```bash
docker compose ps
```
You can now send POST requests to `http://localhost:8000` to trigger your agent, which will generate code using your **local LLM** and execute sandboxed validation steps locally.
