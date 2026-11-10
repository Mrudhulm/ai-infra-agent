# Server-Side Control Plane & Dashboard Blueprint

> **Role**: The Server-Side Control Plane is the centralized manager. It hosts the Web Dashboard UI (where clients log in and view live progress), the API Gateway signal router, the Open Policy Agent (OPA) policy engine, the Redis task queue, PostgreSQL state store, and JIT Credential Broker.

---

## 1. Server-Side Architecture & Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            SERVER-SIDE CONTROL PLANE                        │
│                                                                             │
│  ┌────────────────────────┐        ┌─────────────────────────────────────┐  │
│  │ Web Dashboard UI       │ ─────► │ API Gateway & Signal Router         │  │
│  │ (Port 3000 / Next.js)  │ HTTP   │ (Port 8080 / Express / FastAPI)     │  │
│  └────────────────────────┘        └──────────────────┬──────────────────┘  │
│                                                       │                     │
│                                  Evaluates Autonomy   │                     │
│                                  ┌────────────────────┴──────────────────┐  │
│                                  ▼                                       ▼  │
│                    ┌─────────────────────────┐       ┌───────────────────┐  │
│                    │ Policy Engine (OPA)     │       │ Task Queue        │  │
│                    │ (Port 8181 / Rego)      │       │ (Port 6379/Redis) │  │
│                    └─────────────────────────┘       └─────────┬─────────┘  │
│                                                                │            │
│                                                                ▼            │
│                                                      ┌───────────────────┐  │
│                                                      │ PostgreSQL DB     │  │
│                                                      │ (Port 5432)       │  │
│                                                      └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

1. **Web Dashboard UI (`agent-ui`)**: Allows users to log in, submit infrastructure tasks, inspect live step-by-step logs via SSE (Server-Sent Events), view side-by-side code diffs, and track PR statuses.
2. **API Gateway (`api-gateway`)**: Ingests signals (chat prompts, webhooks, alerts), checks tenant auth, normalizes tasks, and streams real-time updates back to the UI.
3. **Policy Engine (`policy-engine`)**: Uses OPA to enforce autonomy ceilings (`Observe`, `Propose`, `Governed`) and evaluate tool allow/deny lists.
4. **Message Bus & Task Queue (`task-queue`)**: Redis Streams queue managing job distribution and output event fan-out.
5. **Database (`database`)**: PostgreSQL database storing user sessions, tenant metadata, audit logs, and immutable receipts.
6. **Credential Broker (`credential-broker`)**: Mints short-lived STS/OIDC cloud tokens just-in-time when requested by authorized workers.

---

## 2. Server-Side Policy Guardrail (`policies/infra_guardrails.rego`)

```rego
package infra.authz

default allow = false

# Allow read-only operations (Observe ceiling)
allow {
    input.action == "read"
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

## 3. Server-Side API Gateway Routing Example (`packages/api-gateway/src/index.js`)

```javascript
const express = require('express');
const { createClient } = require('redis');

const app = express();
app.use(express.json());

const redis = createClient({ url: process.env.REDIS_URL || 'redis://localhost:6379' });
redis.connect();

// Task Dispatch Endpoint
app.post('/api/v1/sessions', async (req, res) => {
  const { prompt, repo, priority = 'normal' } = req.body;
  
  const task = {
    sessionId: `sess-${Date.now()}`,
    prompt,
    repo,
    priority,
    status: 'QUEUED',
    createdAt: new Date().toISOString()
  };

  // Push to Redis Stream dispatch queue
  await redis.xAdd('task:dispatch:infra', '*', {
    payload: JSON.stringify(task)
  });

  res.status(202).json({
    message: 'Task accepted and queued',
    sessionId: task.sessionId
  });
});

// Real-Time SSE Log Streaming Endpoint for Dashboard UI
app.get('/api/v1/sessions/:sessionId/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const { sessionId } = req.params;
  
  // Stream events from Redis to the Dashboard
  const subscriber = redis.duplicate();
  await subscriber.connect();
  
  await subscriber.subscribe(`session:${sessionId}:events`, (message) => {
    res.write(`data: ${message}\n\n`);
  });

  req.on('close', () => subscriber.disconnect());
});

app.listen(8080, () => console.log('API Gateway listening on port 8080'));
```