# Case Study 1: E-Commerce AI Customer Support Agent — Solution

## Executive Summary

E-commerce support teams face high ticket volumes across order tracking, returns, refunds, and product inquiries. Most queries are repetitive and answerable from structured data (order database) or unstructured knowledge (FAQs, policies). This solution builds a RAG-based conversational agent that autonomously resolves routine queries, retrieves real-time order data, and escalates complex or sensitive cases to human agents — reducing average response time from minutes to seconds while maintaining customer satisfaction.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Customer Interface                     │
│              (Chat widget / API endpoint)                │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  LangChain Agent                         │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Intent      │  │   Entity     │  │  Escalation  │  │
│  │ Classifier    │→ │ Extractor    │→ │  Handler     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                  │                  │          │
│         ▼                  ▼                  ▼          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Tool Router (ReAct)                  │   │
│  └──────┬──────────────┬───────────────┬────────────┘   │
└─────────┼──────────────┼───────────────┼────────────────┘
          │              │               │
          ▼              ▼               ▼
   ┌────────────┐ ┌────────────┐ ┌────────────────┐
   │ FAISS      │ │ Order      │ │ Return/Refund  │
   │ Vector     │ │ Database   │ │ Actions        │
   │ Store      │ │ (Mock)     │ │                │
   │ (FAQs)     │ │            │ │                │
   └────────────┘ └────────────┘ └────────────────┘
```

### Core Components

- **LLM**: OpenAI GPT-4 (via LangChain) — handles intent classification, entity extraction, and response generation in a single ReAct loop
- **Vector Store**: FAISS — stores embedded FAQ/policy documents for semantic retrieval
- **Embeddings**: OpenAI `text-embedding-ada-002` — converts FAQ text into vectors
- **Agent Framework**: LangChain's tool-calling agent — orchestrates tool selection and response generation
- **Mock Database**: Python dict simulating an order management system

---

## Component Design

### 1. Intent Classifier
Built into the agent's system prompt. The LLM identifies the customer's intent from these categories:
- `order_tracking` — "Where is my order?"
- `return_request` — "I want to return this item"
- `refund_inquiry` — "When will I get my refund?"
- `policy_question` — "What's your return policy?"
- `complaint` — frustrated or angry customer
- `escalation` — complex issue requiring human help

### 2. Entity Extractor
The LLM extracts structured entities from the query:
- Order IDs (pattern: `ORD-XXXX`)
- Product names
- Customer sentiment (positive/neutral/negative/angry)

### 3. Knowledge Retriever
A FAISS-backed retrieval tool that:
- Accepts a natural language query
- Returns the top-3 most relevant FAQ passages
- Uses cosine similarity over OpenAI embeddings

### 4. Response Generator
The agent's LLM synthesizes a response by combining:
- Retrieved knowledge base passages
- Order data from the mock database
- Conversational context
- Brand voice guidelines from the system prompt

### 5. Escalation Handler
Triggers escalation to a human agent when:
- Customer sentiment is highly negative (detected via prompt)
- The agent cannot resolve after 2 tool calls
- The query involves billing disputes, legal threats, or safety concerns
- Confidence is low (the LLM expresses uncertainty)

---

## Workflow

```
Customer Query
      │
      ▼
┌─────────────────┐
│ Intent           │
│ Classification   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Entity           │
│ Extraction       │──→ order_id, product, sentiment
└────────┬────────┘
         │
         ├──→ [order_tracking / return_request]
         │         │
         │         ▼
         │    ┌──────────────┐
         │    │ Order DB     │──→ order details
         │    │ Lookup       │
         │    └──────────────┘
         │
         ├──→ [policy_question / refund_inquiry]
         │         │
         │         ▼
         │    ┌──────────────┐
         │    │ FAISS        │──→ relevant FAQ passages
         │    │ Retrieval    │
         │    └──────────────┘
         │
         ├──→ [complaint / escalation]
         │         │
         │         ▼
         │    ┌──────────────┐
         │    │ Escalation   │──→ transfer to human agent
         │    │ Handler      │
         │    └──────────────┘
         │
         ▼
┌─────────────────┐
│ Response         │
│ Generation       │──→ Final answer to customer
└─────────────────┘
```

---

## Key Design Decisions and Trade-offs

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **FAISS over Pinecone/Chroma** | Zero infrastructure, runs in-memory, fast for small FAQ sets (<10K docs) | Not suitable for millions of documents; no persistence without explicit save/load |
| **Single ReAct agent vs. multi-agent** | Simpler to debug and deploy; one LLM call chain handles all intents | Less modular; harder to specialize per intent at scale |
| **OpenAI embeddings** | High quality, easy integration with LangChain | Vendor lock-in; cost per embedding call |
| **Tool-based architecture** | Agent decides which tools to call; easy to add new capabilities | LLM may call wrong tool; requires good tool descriptions |
| **Escalation via prompt engineering** | No separate classifier needed; LLM detects frustration naturally | Less deterministic than a dedicated sentiment model |

---

## Handling Peak Loads

For production deployment beyond this prototype:

1. **Async Processing**: Use `asyncio` with LangChain's async interfaces to handle concurrent conversations
2. **Queue-Based Architecture**: Place incoming queries in a message queue (Redis/RabbitMQ); worker processes consume and respond
3. **Caching**: Cache FAISS retrieval results for common queries (LRU cache with 1-hour TTL)
4. **Rate Limiting**: Cap per-customer request rate to prevent abuse
5. **Pre-computed Embeddings**: Embed FAQ updates in batch, not per-request

```
[Customer] → [API Gateway] → [Redis Queue] → [Worker Pool] → [LLM API]
                                                    │
                                              [FAISS Cache]
```

---

## Escalation Logic

The agent escalates when any of these conditions are met:

| Condition | Detection Method | Action |
|-----------|-----------------|--------|
| High frustration | Keywords: "furious", "lawyer", "BBB", "sue" + sentiment analysis in prompt | Immediate escalation with context summary |
| Confidence below threshold | Agent outputs "I'm not sure" or similar hedging | Escalate with attempted resolution attached |
| Sensitive topic | Billing dispute > $500, safety issue, legal mention | Escalate with priority flag |
| Repeated failure | Same customer asks 3+ times on same issue | Escalate with full conversation history |
| Explicit request | "Let me speak to a human" | Immediate escalation |

---

## Evaluation Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Automated Resolution Rate** | > 70% | Queries resolved without escalation / total queries |
| **Average Response Time** | < 5 seconds | Time from query to first response |
| **CSAT Score** | > 4.0 / 5.0 | Post-conversation survey |
| **Escalation Rate** | < 20% | Escalated conversations / total conversations |
| **Hallucination Rate** | < 2% | Responses containing fabricated information (sampled audit) |
| **First Contact Resolution** | > 65% | Issues resolved in single conversation |

---

## Sample Interactions

### 1. Order Tracking
```
Customer: Where is my order ORD-1234?
Agent:    [Calls lookup_order("ORD-1234")]
          Your order ORD-1234 is currently "shipped" and is expected to
          arrive by 2024-01-20. It contains: Wireless Headphones, Phone Case.
          Would you like the tracking link?
```

### 2. Return Request
```
Customer: I want to return the shoes I bought
Agent:    I'd be happy to help with your return! Could you provide your
          order ID so I can look it up?
Customer: It's ORD-5678
Agent:    [Calls lookup_order("ORD-5678")]
          [Calls search_knowledge_base("return policy")]
          I found your order for Running Shoes. Our return policy allows
          returns within 30 days of delivery for a full refund. Your order
          was delivered on Jan 10, so you're within the return window.
          Would you like me to initiate the return?
Customer: Yes please
Agent:    [Calls initiate_return("ORD-5678")]
          Done! Your return for order ORD-5678 has been initiated. You'll
          receive a prepaid shipping label via email within 24 hours.
          Once we receive the item, your refund will be processed in 5-7
          business days.
```

### 3. Policy Question
```
Customer: What's your refund policy?
Agent:    [Calls search_knowledge_base("refund policy")]
          Our refund policy: Once we receive your returned item and verify
          its condition, refunds are processed within 5-7 business days.
          Refunds are issued to the original payment method. For items
          damaged during shipping, we offer immediate replacement or full
          refund — no return needed.
```

### 4. Escalation — Frustrated Customer
```
Customer: I've been waiting 2 weeks and no one is helping me. This is
          unacceptable and I want to speak to a manager!
Agent:    I completely understand your frustration, and I sincerely
          apologize for the delay and poor experience. No one should have
          to wait this long for help.

          I'm escalating your case to a senior support specialist right
          now. They will contact you within 1 hour with a resolution.
          I've flagged this as high priority.

          [ESCALATED — Reason: Customer frustration + repeated unresolved
          issue + explicit request for manager. Priority: HIGH]
```

---

## Code Architecture

The companion notebook (`agent.ipynb`) implements this solution with the following structure:

1. **Mock Data Setup** — Sample orders dict simulating a real order database
2. **Knowledge Base** — FAQ strings covering returns, shipping, refunds, and general policies
3. **FAISS Vector Store** — Embeds FAQs and provides semantic search via LangChain's `FAISS.from_texts()`
4. **Tool Definitions** — `lookup_order` and `initiate_return` as LangChain `@tool` functions, plus a retrieval tool wrapping the FAISS store
5. **Agent Construction** — LangChain `create_tool_calling_agent` with a system prompt encoding brand voice, escalation rules, and response guidelines
6. **Demo Conversations** — Four sample queries exercising each major path (tracking, returns, policy, escalation)

The architecture is intentionally simple to serve as a clear demonstration. Production enhancements would include persistent vector storage, real database integration, conversation memory across sessions, and monitoring/observability.
