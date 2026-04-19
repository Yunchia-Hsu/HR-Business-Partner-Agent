# Virtual HR Business Partner Agent

A conversational AI agent that supports employees with day-to-day HR needs — answering questions on holidays, benefits, and career development, and providing basic coaching on performance conversations.

---

## Overview

HR information is distributed across multiple internal sources (Confluence, Google Drive, policy documents, HR systems). This agent retrieves and combines insights from these sources to deliver accurate, context-aware responses via Telegram.

The system uses a **hybrid retrieval approach**:

- **FAQ fast path** — 80% of common questions are answered directly from a pre-indexed FAQ store, without invoking the LLM. This improves response speed, reduces cost, and lowers the risk of hallucination.
- **Full RAG pipeline** — less common or more complex questions are routed to an AI Agent that retrieves relevant chunks from the full HR document store and generates a grounded response.

---

## Architecture

[Interactive Process Diagram](process_diagram.html)

![n8n Workflow Node](n8n_node.png)

```
QUERY FLOW (real-time)

Telegram message
       ↓
Pinecone vector search (namespace: hr-faq)
       ↓
Score ≥ 0.65?
  ├── YES → Format FAQ reply → Send Telegram reply
  └── NO  → AI Agent (Simple Memory + Pinecone hr-docs + OpenAI GPT-4)
                  ↓
             Answer questions with RAG
                  ↓
             Send Telegram reply

INDEXING FLOW (offline / triggered on file update)

Manual Trigger / Google Drive fileUpdated
       ↓
Search ICEYE Google Drive folder
       ↓
File type?
  ├── FAQ  → Download → Extract text → Split into Q&A pairs → Insert to Pinecone (hr-faq)
  └── DOCS → Download → Extract text → Recursive Text Splitter → Insert to Pinecone (hr-docs)
```

---

## Key Components

| Component              | Tool                    | Purpose                                             |
| ---------------------- | ----------------------- | --------------------------------------------------- |
| Workflow orchestration | n8n                     | Connects all components, handles routing logic      |
| Vector database        | Pinecone                | Stores and retrieves embeddings for FAQ and HR docs |
| Embeddings             | OpenAI text-embedding-3 | Converts text to semantic vectors                   |
| LLM                    | OpenAI GPT-4            | Generates grounded answers from retrieved context   |
| Session memory         | n8n Simple Memory       | Maintains conversation context within a session     |
| Chat interface         | Telegram Bot            | Employee-facing interface                           |
| Document source        | Google Drive            | Stores HR policy documents and FAQ file             |

---

## Knowledge Base

The agent is trained on four HR documents:

- `leave_policy.txt` — Annual leave, sick leave, parental leave, bereavement leave, study leave
- `benefits.txt` — Healthcare, learning budget, remote work policy, lunch and commuting benefits
- `career.txt` — Performance review cycle, career levels, promotion process, coaching guidance
- `hr_faq.txt` — 35 pre-written Q&A pairs covering the most common HR questions

---

## Design Decisions

**Why hybrid retrieval?**
Most HR questions are repetitive and well-defined (e.g. "how many days of annual leave do I get?"). Pre-indexing these as Q&A pairs means the system can return verified answers instantly, without LLM inference. Only ambiguous or compound questions need the full RAG pipeline.

**Why separate Pinecone namespaces?**
FAQ documents and HR policy documents require different retrieval strategies. FAQ pairs are short and self-contained — a high similarity score to a FAQ chunk is a strong signal. Policy documents are longer and benefit from chunk-level retrieval with context. Separating namespaces avoids interference between the two retrieval modes.

**Why 0.65 as the FAQ score threshold?**
A threshold of 0.65 balances precision and recall for this use case. Below this value, the question is too different from any FAQ entry to trust a direct answer. Above it, the semantic match is strong enough that the FAQ answer is likely correct. This value should be tuned based on observed failures during evaluation.

**Why n8n for orchestration?**
n8n enables rapid prototyping of the full pipeline without sacrificing architectural clarity. Each node in the workflow corresponds directly to a component in the system design. For a production implementation, the same architecture would be replicated in TypeScript using LangChain, with proper error handling, retries, and observability.

**Escalation behaviour**
The system prompt instructs the agent to escalate to a human HR Business Partner for sensitive topics (disciplinary, termination, mental health). The agent should never fabricate a policy it cannot find in the knowledge base.

---

## What Is Out of Scope

- **Authentication** — the prototype does not verify employee identity. Production would require SSO integration.
- **Real-time HR system data** — leave balances and personalised data are not pulled live. The agent answers based on policy, not personal records.
- **Multi-language support** — the knowledge base is in English only.
- **Deployment infrastructure** — no production deployment is included in this prototype.

---

## Evaluation

The agent can be evaluated on three dimensions:

**Faithfulness** — Does the answer reference only information present in the retrieved documents? Hallucinated policy details are the primary failure mode.

**Answer relevance** — Does the answer actually address what the employee asked? Partial answers (missing a key condition) count as failures.

**Escalation rate** — What proportion of questions does the agent escalate to human HR? Too high means the knowledge base is insufficient. Too low means the agent may be overreaching on sensitive topics.

A set of 10 test cases is included in `hr_agent_test_cases.txt`, covering simple fact retrieval, multi-condition queries, cross-document retrieval, coaching questions, boundary conditions, and out-of-scope escalation.

Tool: **Ragas** can be used to automate faithfulness and relevance scoring at scale.

---
