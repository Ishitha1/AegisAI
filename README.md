# AegisAI: A Policy Enforcement Gateway for AI Customer-Support Agents

AegisAI sits between an AI customer-support chatbot and a company's business tools. It checks every proposed response and every proposed action against company policy **before** anything reaches the customer or gets executed.

Each interaction ends in one of four decisions: **ALLOW**, **MODIFY**, **BLOCK** or **ESCALATE**. Every decision is written to an audit store.

---

## Author

1. Ishitha S - 230701117
2. Hemashri U - 230701114
3. Ishwari Rajmohan - 230701118
4. Jaya Bharathi M - 230701126

**Department**: Computer Science and Engineering

**Institution**: Rajalakshmi Engineering College

---

## The Problem

AI customer-support agents can make mistakes that carry financial, legal and privacy risk:

- Promising a refund above the company's limit
- Revealing another customer's order details or personal information
- Promising a delivery date without checking the order system
- Approving compensation, cancellations or account changes without authorization
- Failing to escalate fraud complaints or legal threats

Most systems find these problems after the chatbot has already responded or acted. AegisAI focuses on **pre-execution enforcement**: detect the violation, then stop or repair it before damage happens.

**Example.** A customer asks for a ₹3,000 refund. The chatbot proposes "I can certainly help you process a refund of ₹3000." Company policy requires approval above ₹500. AegisAI blocks the refund action, replaces the response with a safe one, and records an incident.

---

## Architecture

```
Customer Message
       ↓
  [Input Guard]
       ↓
[Agent A: Customer-Support Agent (Qwen3-8B)]
       ↓ (proposes response & optional business action)
[AegisAI Gateway (deterministic engine & priority pipeline)]
       ↓
   Decision:
   ├── ALLOW    → [Agent B: Operations Agent] → [Tool Interceptor] → Tool Execution
   ├── MODIFY   → [Remediation Agent]
   ├── BLOCK    → [Safe Response Generation]
   └── ESCALATE → [Human Escalation Queue]
       ↓
  [Audit Store (PostgreSQL / SQLite)]
       ↓
Final Safe Customer Response
```

### Design principles

- **Proposal-only.** Agent A can only propose actions as structured JSON. It has no direct permission to run tools.
- **Authorization envelope.** Agent B receives only the allowed tool, its parameters and the decision ID.
- **Pre-execution interception.** The `ToolInterceptor` checks the tool name and every argument against the authorization envelope immediately before invocation. Missing or modified arguments block execution.
- **Fail-closed, audit-first.** Every evaluation is recorded, and high-severity violations create reviewable incidents.

---

## Built-In Policies

| Policy ID | Severity | Rule |
| --- | --- | --- |
| `REFUND_LIMIT_001` | HIGH | Refunds above ₹500 require manager authorization |
| `CUSTOMER_VERIFICATION_001` | HIGH | Customer identity must be verified before order details or financial operations |
| `DELIVERY_VERIFICATION_001` | MEDIUM | No specific delivery-date promises unless confirmed by a verified order lookup |
| `PII_PROTECTION_001` | CRITICAL | Card numbers, CVVs, passwords, full phone numbers and cross-customer data are blocked |
| `HIGH_RISK_ESCALATION_001` | CRITICAL | Fraud complaints, legal threats and consumer-court mentions are escalated immediately |

Policies are defined in YAML and loaded into the policy engine at runtime.

---

## Decision Output

Every evaluation returns a structured record. Example of a blocked refund:

```json
{
  "decision": "BLOCK",
  "severity": "HIGH",
  "policy_id": "REFUND_LIMIT_001",
  "reason": "Refund amount of ₹3,000.00 exceeds automatic threshold of ₹500.00 and lacks manager authorization.",
  "proposed_action": {
    "tool_name": "issue_refund",
    "arguments": { "order_id": "ORD-101", "amount": 3000 }
  },
  "approved_action": null,
  "requires_human_review": true,
  "tool_executed": false,
  "agent_b_called": false
}
```

---

## Tech Stack

| Layer | Technology | Role |
| --- | --- | --- |
| Model | Qwen3-8B via Ollama | Agent A: generates responses and proposes actions |
| Orchestration | LangGraph, `langchain-ollama` | Workflow state and ALLOW/MODIFY/BLOCK/ESCALATE routing |
| Backend | Python, FastAPI, Pydantic v2 | API, gateway, policy engine, tool interceptor |
| Database | PostgreSQL / SQLite, SQLAlchemy 2.x, Alembic | Audit events, incidents, policies, approvals |
| Frontend | React, TypeScript, Vite, Tailwind CSS | Dashboard |
| Testing | Pytest, HTTPX, TestClient | Backend tests |
| Deployment | Docker, Docker Compose | Containerized setup |

---

## Features and Status

| Feature | Status |
| --- | --- |
| Inline policy enforcement gateway | Completed |
| Tool and action authorization | Completed |
| Audit event and incident store | Completed |
| Policy management | Completed |
| Customer support simulator | Completed |
| Docker Compose deployment | Completed |
| Test suite | Completed |
| Response remediation | Partial (LLM rewriting in progress) |
| Human escalation workflow | Completed |
| Dashboard and analytics | Completed |
| Delivery-date verification | Completed |
| PII protection | Completed |
| Fraud / legal escalation | Partial |
| Multi-turn context checking | Partial |
| Human approval workflow | Partial |
| Shadow mode | Partial |
| Evaluation dataset | Partial |
| Metrics dashboard | Partial |
| Replay mode | Not started |

---

## Getting Started

### Prerequisites

- Python 3 and Node.js
- [Ollama](https://ollama.com) with the Qwen3-8B model pulled
- Docker (optional, for Compose)

Copy `.env.example` to `.env` and adjust the values.

### Run the backend

```bash
cd backend
uvicorn app.main:app --reload --port 8000
```

### Run the frontend

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`.

### Run with Docker Compose

```bash
docker-compose up --build
```

### Run the tests

```bash
cd backend
python -m pytest -v
```

---

## API Endpoints

| Method | Endpoint | Purpose |
| --- | --- | --- |
| POST | `/api/chat` | Run the full pipeline on a customer message |
| GET | `/api/audits` | Audit trail with filtering and pagination |
| GET | `/api/audits/{event_id}` | Full audit event |
| GET | `/api/incidents` | Security and policy incident queue |
| GET | `/api/policies` | Registered policies and active versions |
| POST | `/api/policies` | Draft a new policy |
| PUT | `/api/policies/{policy_id}` | Create a new policy version |
| POST | `/api/approvals/{decision_id}/approve` | Manager approval |
| POST | `/api/approvals/{decision_id}/reject` | Manager rejection |
| GET | `/api/health` | Service health and active policy count |

---

## Demonstration Scenarios

1. **₹3,000 refund without manager approval:** `BLOCK` (`REFUND_LIMIT_001`). Agent B is not called, the refund tool does not run, and an incident is created.
2. **₹400 refund for a verified customer:** `ALLOW`. Agent B is called with an authorization envelope and the refund executes.
3. **Unverified access to order details:** `BLOCK` (`CUSTOMER_VERIFICATION_001`).
4. **Unverified delivery-date promise:** `MODIFY` (`DELIVERY_VERIFICATION_001`). The response is rewritten to ask for a verified order ID.
5. **Fraud or legal threat:** `ESCALATE` (`HIGH_RISK_ESCALATION_001`). A high-severity incident is queued for human review.
6. **Tampered tool arguments:** `BLOCK`. The interceptor halts the modified call.

---

## Roadmap

- Multi-turn context checking
- Evaluation dataset
- Human approval workflow
- LLM-based response remediation
- Advanced dashboard analytics
- Shadow mode, metrics dashboard and replay mode

---

## SDG Alignment

The project aligns with the following United Nations Sustainable Development Goals:

### SDG 09 — Industry, Innovation and Infrastructure

Promotes responsible AI innovation through independent policy validation, tool authorization, continuous monitoring, and audit mechanisms for secure AI integration.

### SDG 16 — Peace, Justice and Strong Institutions

Promotes accountable and transparent AI operations through consistent policy enforcement, controlled tool execution, explainable `ALLOW`, `MODIFY`, `BLOCK`, and `ESCALATE` decisions, and traceable audit logging.

---

## Disclaimer

This repository represents an academic final-year project.

AegisAI is developed as an independent AI policy enforcement and security compliance platform for the ShopSphere application. The implemented policies, security controls, tool authorization, audit mechanisms and evaluation results are intended for academic research, testing, and demonstration purposes.

**Limitations:** The current implementation is limited to the defined ShopSphere environment, configured policies, supported tools, and test scenarios. It does not guarantee complete detection or prevention of all security threats, policy violations, or AI-related risks and should not be considered a production-ready security or compliance solution.

