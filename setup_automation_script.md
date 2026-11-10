#!/usr/bin/env bash
set -e

echo ""
echo "========================================================="
echo "  Building Complete Local Infrastructure Agent Monorepo"
echo "========================================================="
echo ""

# 1. Create Directory Hierarchy
echo "[1/4] Creating all folder and subfolder paths..."
mkdir -p infra-agent-system/policies
mkdir -p infra-agent-system/skills/terraform-plan
mkdir -p infra-agent-system/packages/api-gateway/src
mkdir -p infra-agent-system/packages/credential-broker/src
mkdir -p infra-agent-system/packages/knowledge-service/src
mkdir -p infra-agent-system/packages/mcp-server/src
mkdir -p infra-agent-system/packages/worker/src
mkdir -p infra-agent-system/sandboxes/docker

cd infra-agent-system

# 2. Write Skills and Policy Guardrails
echo "[2/4] Writing SKILL.md and OPA policy guardrails..."

cat << 'EOF' > skills/terraform-plan/SKILL.md
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
EOF

cat << 'EOF' > policies/infra_guardrails.rego
package infra.authz

default allow = false

# Allow read actions by default under Observe/Propose
allow {
    input.action == "read"
}

# Deny destroy or apply directly without PR gateway
deny {
    input.action == "terraform_apply"
}
EOF

# 3. Write All Microservice Files & Dockerfiles
echo "[3/4] Generating Dockerfiles, package manifests, and code..."

# --- API Gateway ---
cat << 'EOF' > packages/api-gateway/Dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY src ./src
EXPOSE 8080
CMD ["node", "src/index.js"]
EOF

cat << 'EOF' > packages/api-gateway/package.json
{
  "name": "api-gateway",
  "version": "1.0.0",
  "main": "src/index.js",
  "dependencies": {
    "express": "^4.19.2",
    "redis": "^4.6.13",
    "pg": "^8.11.3"
  }
}
EOF

cat << 'EOF' > packages/api-gateway/src/index.js
const express = require('express');
const app = express();
app.use(express.json());

app.get('/health', (req, res) => res.json({ status: 'healthy', service: 'api-gateway' }));

app.post('/api/v1/sessions', (req, res) => {
  res.status(202).json({ sessionId: 'sess-local-123', status: 'dispatched', prompt: req.body.prompt || 'default' });
});

app.listen(8080, () => console.log('API Gateway running on port 8080'));
EOF

# --- Knowledge Service ---
cat << 'EOF' > packages/knowledge-service/Dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install fastapi uvicorn
COPY src ./src
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

cat << 'EOF' > packages/knowledge-service/src/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "healthy", "service": "knowledge-service"}

@app.get("/context/resource-graph")
def get_resource_graph(repo: str):
    markdown_output = f"# Resource Graph for {repo}\n- **aws_s3_bucket.data**: Active S3 bucket needing encryption compliance."
    return {"markdown": markdown_output}
EOF

# --- Credential Broker ---
cat << 'EOF' > packages/credential-broker/Dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY src ./src
EXPOSE 3001
CMD ["node", "src/index.js"]
EOF

cat << 'EOF' > packages/credential-broker/package.json
{
  "name": "credential-broker",
  "version": "1.0.0",
  "main": "src/index.js",
  "dependencies": {
    "express": "^4.19.2"
  }
}
EOF

cat << 'EOF' > packages/credential-broker/src/index.js
const express = require('express');
const app = express();
app.use(express.json());

app.get('/health', (req, res) => res.json({ status: 'healthy', service: 'credential-broker' }));

app.post('/get-cloud-credentials', (req, res) => {
  res.json({ token: 'mock-sts-short-lived-token-xyz', expiresIn: 3600 });
});

app.listen(3001, () => console.log('Credential Broker running on port 3001'));
EOF

# --- MCP Server ---
cat << 'EOF' > packages/mcp-server/Dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY src ./src
EXPOSE 3002
CMD ["node", "src/server.js"]
EOF

cat << 'EOF' > packages/mcp-server/package.json
{
  "name": "mcp-server",
  "version": "1.0.0",
  "main": "src/server.js",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0",
    "express": "^4.19.2"
  }
}
EOF

cat << 'EOF' > packages/mcp-server/src/server.js
const express = require('express');
const app = express();
app.use(express.json());

app.get('/health', (req, res) => res.json({ status: 'healthy', service: 'mcp-server' }));

app.post('/mcp/tools/terraform_plan', (req, res) => {
  res.json({ success: true, driftResources: [], message: 'Plan executed successfully with zero drift.' });
});

app.listen(3002, () => console.log('MCP Server running on port 3002'));
EOF

# --- Agent Worker ---
cat << 'EOF' > packages/worker/Dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY src ./src
CMD ["node", "src/worker.js"]
EOF

cat << 'EOF' > packages/worker/package.json
{
  "name": "agent-worker",
  "version": "1.0.0",
  "main": "src/worker.js",
  "dependencies": {
    "redis": "^4.6.13",
    "node-fetch": "^3.3.2"
  }
}
EOF

cat << 'EOF' > packages/worker/src/worker.js
console.log('Agent Worker started and listening to dispatch streams...');
setInterval(() => {
  // Worker heartbeat / poll loop simulation
}, 30000);
EOF

# 4. Write Docker Compose File
echo "[4/4] Writing docker-compose.yml orchestration specification..."

cat << 'EOF' > docker-compose.yml
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
    depends_on:
      - database
      - task-queue
    networks:
      - agent-net

  agent-worker:
    build: ./packages/worker
    container_name: agent-worker
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./skills:/skills
    depends_on:
      - local-llm
      - task-queue
      - policy-engine
    networks:
      - agent-net
EOF

echo ""
echo "========================================================="
echo "  Setup Completed Successfully!"
echo "========================================================="
echo "  To launch your local infrastructure agent stack:"
echo "    1. cd infra-agent-system"
echo "    2. docker compose up --build -d"
echo "    3. docker exec -it agent-local-llm ollama pull qwen2.5-coder:7b"
echo "========================================================="
echo ""