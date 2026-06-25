# Case Study 3: AI Customer Support Agent

## Google ADK + LangChain Multi-Agent Architecture

---

## Executive Summary

Enterprise customer support teams face a compounding problem: ticket volumes grow faster than headcount, knowledge fragments across wikis and past tickets, and escalation processes lose critical context at every handoff. This case study demonstrates a multi-agent AI support system where **Google ADK** orchestrates specialist agents and manages session state, while **LangChain** powers retrieval-augmented generation, tool chains, and conversational reasoning. The result is a system that resolves common issues autonomously, maintains full context across multi-turn conversations, and packages rich diagnostic summaries when human escalation is necessary.

---

## System Architecture

```
                         Customer Message
                              |
                    +---------v----------+
                    |   Google ADK       |
                    |   Orchestrator     |
                    |  (session state,   |
                    |   execution flow)  |
                    +---------+----------+
                              |
              +-------+-------+-------+-------+
              |       |               |       |
        +-----v--+ +--v--------+ +---v----+ +v----------+
        |Planner | |Knowledge  | |Diag-   | |Escalation |
        |Agent   | |Agent      | |nostic  | |Agent      |
        |        | |(LangChain | |Agent   | |           |
        |        | | RAG)      | |        | |           |
        +--------+ +-----------+ +--------+ +-----------+
                        |
                  +-----v------+
                  | FAISS      |
                  | Vector     |
                  | Store      |
                  +------------+
```

### Data Flow

1. Customer sends a message via chat, email, or API.
2. The **ADK Orchestrator** loads or creates a session, then dispatches the message to the **Planner Agent**.
3. The Planner classifies the issue and selects one or more specialist agents.
4. Specialist agents execute in sequence or parallel (ADK controls flow).
5. Results are aggregated, the session is updated, and a response is returned.
6. If resolution confidence is low or the customer explicitly requests it, the **Escalation Agent** packages full context for a human agent.

---

## Multi-Agent Design

### Planner Agent

**Purpose**: Intent classification and routing.

The Planner inspects the incoming message along with session history and assigns it to the appropriate specialist. It uses a lightweight LLM call with a structured output schema:

```python
class RoutingDecision:
    intent: str          # "login_issue", "password_reset", "billing", "general", "escalation"
    confidence: float    # 0.0 - 1.0
    agents: list[str]    # ordered list of agents to invoke
    reasoning: str       # short explanation for audit trail
```

Routing rules:
- High-confidence single-domain issues go directly to the relevant specialist.
- Ambiguous queries trigger Knowledge Agent first, then Diagnostic if unresolved.
- Explicit frustration signals or third repeated attempt trigger Escalation.

### Knowledge Agent

**Purpose**: Retrieve and synthesize information from the knowledge base.

Built on LangChain's retrieval chain:
- **Document loader**: Ingests support articles, FAQs, and resolved ticket summaries.
- **FAISS vector store**: Embeds documents for semantic search.
- **Retrieval QA chain**: Combines retrieved context with the customer query to generate a grounded answer.

The agent returns both the answer and source documents so the orchestrator can include references.

### Diagnostic Agent

**Purpose**: Automated system checks and incident correlation.

This agent runs programmatic checks rather than LLM inference:
- Queries system health dashboards for active incidents.
- Checks the customer's account status (locked, expired, active).
- Looks up recent deployments or known bugs matching the symptom.
- Returns a structured diagnostic report.

### Escalation Agent

**Purpose**: Human handoff with full context preservation.

When triggered, the Escalation Agent:
1. Summarizes the full conversation history.
2. Attaches diagnostic results and knowledge base articles already surfaced.
3. Assigns a priority level based on issue severity and customer sentiment.
4. Formats the package for the ticketing system (Zendesk, Jira Service Management, etc.).

---

## Google ADK's Role

Google ADK provides the structural backbone:

| Capability | How It's Used |
|---|---|
| **Agent Teams** | Groups specialist agents into a callable team with defined execution order |
| **Execution Flow** | Controls sequential vs. parallel agent invocation; handles branching based on Planner output |
| **Session State** | Persists conversation history, diagnostic results, and routing decisions across turns |
| **Tool Registry** | Registers LangChain chains and custom functions as callable tools within the ADK framework |
| **Callbacks** | Pre/post-execution hooks for logging, metrics, and guardrails |

ADK manages the lifecycle: which agents run, in what order, with what inputs, and how their outputs merge back into the session.

---

## LangChain's Role

LangChain handles the intelligence layer:

| Capability | How It's Used |
|---|---|
| **RAG Pipeline** | Document loading, chunking, embedding, FAISS indexing, retrieval QA |
| **Tool Chains** | Wraps diagnostic functions and API calls as LangChain tools |
| **Prompt Templates** | Standardized prompts for classification, summarization, and response generation |
| **Conversation Memory** | `ConversationBufferMemory` feeds prior turns into each agent's context window |
| **Output Parsers** | Structured output parsing for routing decisions and diagnostic reports |

---

## Session Management

Sessions track:
- **Conversation history**: All customer and agent messages.
- **Agent trace**: Which agents ran, in what order, with what results.
- **Diagnostic state**: Results of system checks, cached to avoid redundant calls.
- **Escalation flag**: Whether the conversation has been marked for handoff.

Context is maintained across turns by the ADK session store. When a customer sends a follow-up message, the orchestrator loads the full session and passes relevant context to whichever agent handles the new message. This means the Knowledge Agent can see that it already suggested "clear your cache" and avoid repeating itself.

---

## Scalability

### Concurrent Conversation Handling

- Each conversation gets its own session object; there is no shared mutable state between sessions.
- The ADK orchestrator is stateless per request -- it loads session from the store, processes, and writes back.
- FAISS vector store is read-only at serving time (rebuilt offline when knowledge base updates).

### Agent Pooling

- LLM calls are the bottleneck. The system uses async execution so multiple conversations can have in-flight LLM calls simultaneously.
- Diagnostic checks (API calls, DB lookups) run concurrently with LLM-based agents when the execution plan allows it.

### Throughput Targets

| Metric | Target |
|---|---|
| Median response latency | < 3 seconds |
| Concurrent sessions | 500+ |
| Auto-resolution rate | 60-70% of Tier 1 issues |
| Escalation context completeness | 95%+ (all diagnostics and history included) |

---

## Structured Escalation

### Escalation Triggers

1. **Confidence threshold**: Planner or Knowledge Agent confidence drops below 0.4.
2. **Repeat attempts**: Customer has asked about the same issue 3+ times without resolution.
3. **Sentiment detection**: Frustration or anger signals in customer message.
4. **Explicit request**: Customer says "talk to a human" or equivalent.
5. **Diagnostic failure**: System checks reveal an issue that requires manual intervention.

### Escalation Package

```json
{
  "priority": "high",
  "category": "login_failure",
  "summary": "Customer unable to log in after cache clear and password reset. Diagnostic shows no active incidents. Account status is active. Likely client-specific environment issue.",
  "conversation_history": ["..."],
  "diagnostics": {
    "system_health": "all_clear",
    "account_status": "active",
    "known_incidents": "none_matching",
    "steps_attempted": ["cache_clear", "password_reset"]
  },
  "knowledge_articles_referenced": ["KB-1042", "KB-1015"],
  "customer_sentiment": "frustrated",
  "session_id": "sess_abc123"
}
```

---

## Evaluation Metrics

| Metric | Description | Target |
|---|---|---|
| **Auto-Resolution Rate** | Percentage of conversations resolved without human intervention | 60-70% |
| **First-Response Relevance** | Human-rated relevance of the first agent response (1-5 scale) | >= 4.0 |
| **Escalation Quality** | Percentage of escalations where the human agent had sufficient context to act | >= 95% |
| **Mean Turns to Resolution** | Average number of back-and-forth messages before resolution | <= 4 |
| **Retrieval Precision** | Percentage of retrieved knowledge base articles that are relevant | >= 80% |
| **Routing Accuracy** | Percentage of queries routed to the correct specialist agent | >= 90% |
| **Customer Satisfaction (CSAT)** | Post-conversation rating | >= 4.2 / 5 |
| **Latency (p95)** | 95th percentile response time | < 5 seconds |

---

## Sample Interactions

### Scenario 1: Login Failure Diagnosis

```
Customer: My application fails every time I try to log in.

[Planner Agent] -> intent: login_issue, agents: [KnowledgeAgent, DiagnosticAgent]

[KnowledgeAgent] Retrieved: KB-1042 "Login Troubleshooting Guide"
[DiagnosticAgent] System health: OK | Account status: active | Known incidents: none

Agent: I'm sorry you're having trouble logging in. Here are a few things to try:
1. Clear your browser cache and cookies
2. Try an incognito/private browsing window
3. Ensure you're using the correct login URL: app.example.com/login
4. Check that your caps lock is off

Our systems are currently healthy with no known login issues. Could you try
these steps and let me know if the problem persists?
```

### Scenario 2: Multi-Turn Troubleshooting (Context Maintained)

```
Customer: I tried clearing cache but it still doesn't work.

[Planner Agent] -> Session loaded, sees prior login_issue context
  -> agents: [KnowledgeAgent, DiagnosticAgent]

[KnowledgeAgent] Retrieved: KB-1015 "Advanced Login Issues"
  (avoids repeating cache suggestion -- already in session history)
[DiagnosticAgent] Re-checks: no new incidents

Agent: Thank you for trying that. Since clearing the cache didn't help, let's
try a few more things:
1. Try a different browser entirely (Chrome, Firefox, or Edge)
2. Disable any browser extensions, especially ad blockers or VPNs
3. Check if you can access other pages on app.example.com

Also -- are you seeing any specific error message when the login fails?
That would help me narrow this down.
```

### Scenario 3: Escalation with Full Context

```
Customer: This issue is still not resolved. I need to talk to someone.

[Planner Agent] -> Escalation triggers: explicit request + 3rd attempt
  -> agents: [EscalationAgent]

[EscalationAgent] Packages:
  - Full 3-turn conversation history
  - Diagnostic results (system healthy, account active)
  - Steps attempted: cache clear, suggested browser switch
  - Priority: high
  - Sentiment: frustrated

Agent: I completely understand your frustration, and I'm sorry we haven't been
able to resolve this yet. I'm connecting you with a support specialist now.

I've prepared a full summary of our conversation, including the steps you've
already tried and our diagnostic results, so you won't need to repeat anything.

A specialist will be with you shortly. Your reference number is ESC-7291.
```

### Scenario 4: Simple Knowledge Retrieval (New Session)

```
Customer: How do I reset my password?

[Planner Agent] -> intent: password_reset, confidence: 0.95
  -> agents: [KnowledgeAgent]

[KnowledgeAgent] Retrieved: KB-1003 "Password Reset Guide"

Agent: To reset your password:
1. Go to app.example.com/login
2. Click "Forgot Password?" below the login form
3. Enter the email address associated with your account
4. Check your inbox for a reset link (check spam/junk if you don't see it)
5. Click the link and set a new password

The reset link expires after 24 hours. If you run into any issues, let me know!
```

---

## Technology Summary

| Component | Technology | Purpose |
|---|---|---|
| Agent Orchestration | Google ADK | Team management, execution flow, session persistence |
| Retrieval & Reasoning | LangChain | RAG pipeline, tool chains, prompt engineering |
| Vector Store | FAISS | Semantic search over knowledge base |
| Embeddings | OpenAI text-embedding-ada-002 | Document and query embedding |
| LLM | GPT-4 / Gemini | Classification, generation, summarization |
| Session Store | Redis / Firestore | Persistent session state |
| Monitoring | LangSmith | Trace logging, evaluation, debugging |
