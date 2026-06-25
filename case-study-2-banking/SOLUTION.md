# Case Study 2: AI Customer Support Agent for Banking

## Executive Summary

Banks handle millions of customer service interactions daily — balance inquiries, transaction disputes, card management, and fraud reports. The majority of these are repetitive Tier-0 queries that consume agent time while customers wait in queues. This case study designs a secure, AI-powered customer support agent that automates routine banking inquiries, enforces strict identity verification for sensitive operations, and escalates high-risk scenarios to human agents. GitHub Copilot is used throughout development to accelerate secure code generation, suggest compliance patterns, and produce comprehensive test coverage.

---

## System Architecture

```
Customer (Web/Mobile Chat)
        │
        ▼
┌──────────────────────┐
│   Secure Session      │  ← TLS 1.3, session token, timeout
│   Manager             │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│   Intent Classifier   │  ← NLP-based intent detection
│   + Risk Scorer       │     Assigns LOW / MEDIUM / HIGH
└──────┬───────────────┘
       │
       ├─── LOW ──────► Automated Response (balance, tx history)
       │
       ├─── MEDIUM ───► Identity Verification → Automated Action
       │                 (card unblock, dispute filing)
       │
       └─── HIGH ─────► Human Escalation Queue
                         (large transfers, fraud, account closure)
```

### Core Components

| Component | Responsibility |
|---|---|
| **Secure Session Manager** | TLS termination, JWT session tokens, idle timeout (5 min), MFA challenge |
| **Intent Classifier** | Maps user input to banking intents with confidence scores |
| **Risk Scorer** | Assigns risk tier based on intent + transaction amount + account flags |
| **Identity Verification Module** | Validates customer identity via security questions, OTP, or biometric |
| **Tool Executor** | Calls core banking APIs (read-only for low-risk, write for medium) |
| **Escalation Handler** | Routes high-risk cases to human agents with full conversation context |
| **Audit Logger** | Immutable log of every action, PII-masked |

---

## GitHub Copilot's Role

GitHub Copilot accelerates development across every layer of this system. Below are specific examples of how Copilot assists.

### 1. API Boilerplate Generation

Writing a comment like:

```python
# FastAPI endpoint for checking account balance, requires JWT auth
```

Copilot suggests:

```python
@app.get("/api/v1/accounts/{account_id}/balance")
async def get_balance(
    account_id: str,
    token: str = Depends(verify_jwt_token)
):
    account = await account_service.get_account(account_id)
    if account is None:
        raise HTTPException(status_code=404, detail="Account not found")
    return {"account_id": mask_account_id(account_id), "balance": account.balance}
```

### 2. Secure Coding Patterns

Prompt: `# Sanitize user input to prevent SQL injection and XSS`

Copilot suggests:

```python
import bleach
from sqlalchemy import text

def sanitize_input(user_input: str) -> str:
    cleaned = bleach.clean(user_input, tags=[], strip=True)
    if len(cleaned) > 500:
        raise ValueError("Input exceeds maximum length")
    return cleaned

def get_transactions(account_id: str, db: Session):
    stmt = text("SELECT * FROM transactions WHERE account_id = :acct_id")
    return db.execute(stmt, {"acct_id": account_id}).fetchall()
```

### 3. Validation Logic

Prompt: `# Validate account ID format: 10-digit numeric string`

```python
import re

def validate_account_id(account_id: str) -> bool:
    return bool(re.match(r'^\d{10}$', account_id))
```

### 4. Unit Test Generation

Prompt: `# Test risk classification for various banking intents`

Copilot generates:

```python
def test_balance_check_is_low_risk():
    assert classify_risk("check_balance") == "LOW"

def test_card_unblock_is_medium_risk():
    assert classify_risk("card_unblock") == "MEDIUM"

def test_large_transfer_is_high_risk():
    assert classify_risk("large_transfer") == "HIGH"

def test_unknown_intent_defaults_to_high():
    assert classify_risk("unknown_intent") == "HIGH"
```

### 5. PII Masking

Prompt: `# Mask account number showing only last 4 digits`

```python
def mask_account_id(account_id: str) -> str:
    return "X" * (len(account_id) - 4) + account_id[-4:]
```

---

## Security & Compliance

### Data Encryption

- **In Transit**: TLS 1.3 for all API communication. Certificate pinning on mobile clients.
- **At Rest**: AES-256 encryption for stored customer data. Encryption keys managed via AWS KMS or Azure Key Vault.

### PII Masking in Logs

All logs are processed through a PII scrubber before storage:

- Account numbers → `XXXXXX1234`
- SSN → `XXX-XX-XXXX`
- Phone numbers → `(XXX) XXX-1234`
- Email → `s***@example.com`

No raw PII is ever written to application logs or sent to the LLM provider.

### Audit Trails

Every agent action generates an immutable audit record:

```json
{
  "timestamp": "2026-06-25T14:30:00Z",
  "session_id": "sess_abc123",
  "customer_id": "MASKED",
  "action": "check_balance",
  "risk_level": "LOW",
  "outcome": "success",
  "verification_performed": false
}
```

### Session Security

- **Idle timeout**: 5 minutes of inactivity terminates the session.
- **Absolute timeout**: 30-minute maximum session duration.
- **MFA required** for medium-risk operations (OTP via SMS/email or biometric).

### Compliance

- SOC 2 Type II audit logging
- PCI DSS compliance for card-related operations
- GDPR/CCPA data minimization — only retrieve data needed for the current query

---

## Component Design

### Intent Classifier with Risk Scoring

| Intent | Risk Level | Requires Verification | Auto-Resolvable |
|---|---|---|---|
| `check_balance` | LOW | No | Yes |
| `transaction_history` | LOW | No | Yes |
| `pending_transaction` | LOW | No | Yes |
| `card_status` | LOW | No | Yes |
| `card_unblock` | MEDIUM | Yes (OTP) | Yes |
| `dispute_transaction` | MEDIUM | Yes (OTP) | Yes |
| `large_transfer` | HIGH | Yes (MFA) | No — escalate |
| `account_closure` | HIGH | Yes (MFA) | No — escalate |
| `suspicious_activity` | HIGH | Yes | No — escalate |

### Identity Verification Module

Three-tier verification:
1. **Basic**: Customer ID + last 4 of account number (for low-risk, optional)
2. **Standard**: OTP sent to registered phone/email (for medium-risk)
3. **Enhanced**: MFA + security questions (for high-risk before escalation)

### Transaction Lookup

- Read-only access to core banking transaction API
- Filters by date range, status (pending/completed/failed), amount
- Returns masked merchant names and amounts

### Card Management

- Check card status (active/blocked/expired)
- Request card unblock (requires standard verification)
- Report lost/stolen card (triggers immediate block + escalation)

### Escalation Handler

When escalation triggers:
1. Summarize the conversation for the human agent
2. Transfer session with full context
3. Notify customer of estimated wait time
4. Log escalation reason and timestamp

---

## Workflow

```
Customer initiates secure chat session
        │
        ▼
Authenticate session (JWT + device fingerprint)
        │
        ▼
Customer sends message
        │
        ▼
Intent Classification + Risk Scoring
        │
        ├── LOW RISK
        │   └── Execute tool directly → Return response
        │
        ├── MEDIUM RISK
        │   ├── Trigger identity verification (OTP)
        │   ├── Verification passed? → Execute tool → Return response
        │   └── Verification failed? → Lock after 3 attempts → Escalate
        │
        └── HIGH RISK
            ├── Collect context from customer
            ├── Trigger enhanced verification
            └── Route to human agent with conversation summary
```

---

## Tier-0 vs Tier-1 Query Handling

### Tier-0 (Fully Automated)

Queries that require no human involvement:

- "What's my account balance?" — Direct API lookup
- "Show me my last 5 transactions" — Transaction query with formatting
- "Is my card active?" — Card status check
- "Why is my transaction still pending?" — Status lookup + standard explanation

**Target**: 70-80% of all incoming queries should resolve at Tier-0.

### Tier-1 (AI-Assisted with Verification)

Queries that require identity verification but can still be automated:

- "Unblock my debit card" — OTP verification + API call
- "I want to dispute a charge" — Verification + dispute form creation
- "Update my phone number" — Enhanced verification + profile update

**Target**: 15-20% of queries resolve at Tier-1.

### Human Escalation (Remaining 5-10%)

- Fraud reports with active compromise
- Wire transfers above threshold
- Account closure requests
- Complex disputes requiring judgment

---

## Evaluation Metrics

| Metric | Target | Measurement |
|---|---|---|
| **Tier-0 Resolution Rate** | > 75% | Queries resolved without human intervention |
| **Average Handle Time** | < 90 seconds | Time from query to resolution for Tier-0 |
| **Customer Satisfaction (CSAT)** | > 4.2 / 5.0 | Post-interaction survey |
| **Intent Classification Accuracy** | > 95% | Against labeled test set |
| **False Escalation Rate** | < 5% | Escalations that could have been automated |
| **Security Incident Rate** | 0 | Unauthorized data access or actions |
| **Identity Verification Success** | > 90% | Legitimate customers passing verification |
| **Uptime** | 99.9% | System availability |

---

## Sample Interactions

### 1. Pending Transaction Inquiry (LOW risk)

> **Customer**: Why is my card transaction at Amazon still pending?
>
> **Agent**: I can help you with that. Let me look up your recent transactions.
>
> I found a pending transaction:
> - **Merchant**: Amazon
> - **Amount**: $47.99
> - **Date**: June 23, 2026
> - **Status**: Pending
>
> Pending transactions typically settle within 2-3 business days. The merchant has authorized the charge but hasn't finalized it yet. If it remains pending beyond June 26, please contact us again and we'll investigate further.

### 2. Blocked Card (MEDIUM risk)

> **Customer**: My debit card is blocked. I need it unblocked right away.
>
> **Agent**: I understand the urgency. For your security, I need to verify your identity before making changes to your card. I've sent a one-time verification code to your registered phone number ending in **1234**. Please enter the code.
>
> **Customer**: 847291
>
> **Agent**: Verified successfully. I can see your card ending in **5678** was blocked on June 24 due to multiple incorrect PIN attempts. I've submitted an unblock request. Your card will be active within 15 minutes. If you didn't make those PIN attempts, please let me know and I'll escalate this to our fraud team.

### 3. Balance Check (LOW risk)

> **Customer**: What's my account balance?
>
> **Agent**: Here's your current account summary:
>
> - **Account**: XXXXXX4321
> - **Available Balance**: $3,247.50
> - **Current Balance**: $3,295.49
>
> The difference of $47.99 is due to a pending transaction. Would you like to see your recent transactions?

### 4. Suspicious Activity Report (HIGH risk — Escalation)

> **Customer**: I see a suspicious transaction on my account. There's a $500 charge from a store I've never visited.
>
> **Agent**: I take this very seriously. Let me gather some details to help our fraud team investigate:
>
> - Can you confirm which transaction looks suspicious? I see a $500.00 charge at "ElectroMart - Miami, FL" on June 24.
>
> **Customer**: Yes, that one. I've never been to Miami.
>
> **Agent**: Thank you for confirming. I'm taking the following immediate steps:
> 1. **Temporarily blocking your card** to prevent further unauthorized charges
> 2. **Flagging the transaction** for investigation
> 3. **Connecting you with a fraud specialist** who can assist with the dispute and issue a replacement card
>
> A fraud specialist will be with you shortly (estimated wait: under 2 minutes). Your case reference number is **FR-2026-84721**. Please keep this for your records.
