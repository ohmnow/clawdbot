# Deep Dive: Technical Implementation
## Dual-Model SaaS Routing System — Clawdbot-as-a-Service

> **Document version:** 0.1-draft · **Last updated:** 2025-07-09
> **Status:** Architecture specification · Pre-implementation

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [API Design](#2-api-design)
3. [Request Lifecycle](#3-request-lifecycle)
4. [Usage Tracking & Credit System](#4-usage-tracking--credit-system)
5. [Streaming Architecture](#5-streaming-architecture)
6. [Tool Calling Abstraction](#6-tool-calling-abstraction)
7. [Error Handling & Fallback](#7-error-handling--fallback)
8. [Observability](#8-observability)
9. [Tech Stack Recommendation](#9-tech-stack-recommendation)
10. [MVP vs v2 Roadmap](#10-mvp-vs-v2-roadmap)

---

## 1. System Architecture

### 1.1 High-Level Component Diagram

```
                           ┌─────────────────────────────────────────────────────────┐
                           │                     INTERNET                            │
                           └────────────────────────┬────────────────────────────────┘
                                                    │
                                                    ▼
                           ┌────────────────────────────────────────────────────────┐
                           │               EDGE / CDN (Cloudflare)                  │
                           │          WAF · Rate Limiting · DDoS Protection         │
                           └────────────────────────┬───────────────────────────────┘
                                                    │
                                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY (Kong / Traefik)                            │
│                                                                                      │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ REST Handler  │  │ WS Handler    │  │ Auth Middleware│  │ Rate Limiter           │  │
│  │ /v1/chat/*    │  │ /v1/stream    │  │ JWT + API Key │  │ (per-user, per-tier)   │  │
│  └──────┬───────┘  └──────┬────────┘  └──────┬───────┘  └──────────┬─────────────┘  │
│         └──────────────────┴─────────────────┬┘                     │                │
└─────────────────────────────────────────────┬┼─────────────────────┘                 │
                                              ││                                       │
                                              ▼▼                                       │
               ┌──────────────────────────────────────────────────────────┐             │
               │                    AUTH / BILLING SERVICE                │             │
               │                                                          │             │
               │  • Validate JWT / API key                                │             │
               │  • Check subscription tier (Starter/Pro/Business)        │             │
               │  • Check Opus credit balance                             │             │
               │  • Attach user context to request                        │             │
               └─────────────────────────┬────────────────────────────────┘             │
                                         │                                              │
                                         ▼                                              │
               ┌──────────────────────────────────────────────────────────┐             │
               │                   REQUEST ROUTER                         │             │
               │                                                          │             │
               │  ┌───────────────────────────────────────────────────┐   │             │
               │  │           Classification Engine                    │   │             │
               │  │  • Keyword / intent heuristics (fast path)        │   │             │
               │  │  • Lightweight classifier (distilled model)       │   │             │
               │  │  • User-override header (X-Model-Preference)      │   │             │
               │  │  • Credit availability check                      │   │             │
               │  └───────────┬────────────────────┬──────────────────┘   │             │
               │              │                    │                       │             │
               │        ┌─────▼─────┐        ┌────▼──────┐               │             │
               │        │  FLASH    │        │  OPUS     │               │             │
               │        │  (70-85%) │        │  (15-30%) │               │             │
               │        └─────┬─────┘        └────┬──────┘               │             │
               │              │                    │                       │             │
               └──────────────┼────────────────────┼───────────────────────┘             │
                              │                    │                                     │
                  ┌───────────▼────────┐  ┌───────▼──────────┐                          │
                  │  FLASH INFERENCE   │  │  OPUS INFERENCE   │                          │
                  │  SERVICE           │  │  SERVICE           │                          │
                  │                    │  │                    │                          │
                  │  Gemini 2.0 Flash  │  │  Claude Opus 4.5  │                          │
                  │  via Google AI API │  │  via Anthropic API │                          │
                  │                    │  │                    │                          │
                  │  • Tool adapter    │  │  • Tool adapter    │                          │
                  │  • Token counter   │  │  • Token counter   │                          │
                  │  • Retry logic     │  │  • Retry logic     │                          │
                  └───────────┬────────┘  └───────┬──────────┘                          │
                              │                    │                                     │
                              └────────┬───────────┘                                    │
                                       │                                                │
                                       ▼                                                │
               ┌──────────────────────────────────────────────────────────┐             │
               │               ESCALATION HANDLER                         │             │
               │                                                          │             │
               │  • Confidence scorer (did Flash hedge / ask to retry?)   │             │
               │  • Complexity detector (code errors, contradictions)     │             │
               │  • Auto-escalate to Opus if Flash confidence < threshold │             │
               │  • Buffer Flash partial response for context             │             │
               └─────────────────────────┬────────────────────────────────┘             │
                                         │                                              │
                                         ▼                                              │
               ┌──────────────────────────────────────────────────────────┐             │
               │                RESPONSE MERGER                           │             │
               │                                                          │             │
               │  • Normalize output format (Gemini → unified schema)     │             │
               │  • Merge tool call results                               │             │
               │  • Attach metadata (model used, tokens, latency)         │             │
               │  • Stream or batch response to client                    │             │
               └─────────────────────────┬────────────────────────────────┘             │
                                         │                                              │
                              ┌──────────▼──────────┐                                   │
                              │   USAGE TRACKER      │                                   │
                              │                      │                                   │
                              │  • Opus credits ±1   │                                   │
                              │  • Token metering    │                                   │
                              │  • Cost attribution  │                                   │
                              │  • Async via queue   │                                   │
                              └──────────┬───────────┘                                  │
                                         │                                              │
                ┌────────────────────────┬┴────────────────────────┐                    │
                ▼                        ▼                         ▼                     │
    ┌───────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐           │
    │    PostgreSQL      │  │       Redis           │  │    ClickHouse /     │           │
    │                    │  │                        │  │    TimescaleDB      │           │
    │  • Users           │  │  • Session cache       │  │                    │           │
    │  • Subscriptions   │  │  • Rate limit counters │  │  • Request logs    │           │
    │  • Credit balances │  │  • Opus credit cache   │  │  • Token usage     │           │
    │  • Billing events  │  │  • Feature flags       │  │  • Cost analytics  │           │
    └────────────────────┘  └────────────────────────┘  └────────────────────┘           │
                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Responsibilities

| Component | Responsibility | Scaling Strategy |
|-----------|---------------|------------------|
| **API Gateway** | TLS termination, routing, auth delegation, rate limiting | Horizontal (stateless) |
| **Auth/Billing Service** | JWT validation, subscription checks, Stripe integration | Horizontal + Redis cache |
| **Request Router** | Classify intent, pick model, enforce credit limits | Horizontal (stateless) |
| **Flash Inference** | Gemini API calls, tool adaptation, streaming | Horizontal, queue-buffered |
| **Opus Inference** | Anthropic API calls, tool adaptation, streaming | Horizontal, queue-buffered |
| **Escalation Handler** | Confidence scoring, Flash→Opus promotion | Inline (same process as router) |
| **Response Merger** | Format normalization, metadata attachment | Inline |
| **Usage Tracker** | Credit decrement, token metering, cost logging | Async via Redis Streams |
| **PostgreSQL** | Source of truth for users, subs, credits | Primary + read replica |
| **Redis** | Hot cache, rate limits, session state, pub/sub | Sentinel or Cluster |
| **ClickHouse** | Analytics, usage dashboards, cost reporting | Single node (MVP) |

### 1.3 Network Topology (MVP — Single VPS → Production)

```
MVP (Single Host)                          Production (Multi-Node)
┌──────────────────────┐                   ┌────────────────────────────┐
│  Docker Compose      │                   │  Kubernetes (GKE / EKS)   │
│                      │                   │                            │
│  ┌────────────────┐  │                   │  ┌──────┐  ┌──────┐       │
│  │ FastAPI (all   │  │                   │  │Router│  │Router│  ...  │
│  │ services in    │  │        →          │  │ pod  │  │ pod  │       │
│  │ one process)   │  │                   │  └──────┘  └──────┘       │
│  ├────────────────┤  │                   │  ┌──────┐  ┌──────┐       │
│  │ PostgreSQL     │  │                   │  │Flash │  │Opus  │       │
│  ├────────────────┤  │                   │  │worker│  │worker│       │
│  │ Redis          │  │                   │  └──────┘  └──────┘       │
│  └────────────────┘  │                   │  Managed PG · Redis Cluster│
└──────────────────────┘                   └────────────────────────────┘
```

---

## 2. API Design

### 2.1 Base URL & Versioning

```
Production:  https://api.clawdbot.com/v1
Staging:     https://api-staging.clawdbot.com/v1
```

### 2.2 Authentication

Two auth methods supported:

```
# API Key (server-to-server)
Authorization: Bearer cb_live_sk_xxxxxxxxxxxxxxxxxxxx

# JWT (web/mobile clients, obtained via /auth/login)
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
```

### 2.3 Core Endpoints

#### 2.3.1 Chat Completion (Synchronous)

```yaml
POST /v1/chat/completions
Content-Type: application/json

# Request Body
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Explain quantum entanglement simply." }
  ],
  "conversation_id": "conv_abc123",       # optional, for multi-turn
  "model_preference": "auto",             # "auto" | "flash" | "opus"
  "tools": [...],                         # optional tool definitions
  "max_tokens": 4096,                     # optional
  "temperature": 0.7                      # optional
}

# Response (200 OK)
{
  "id": "msg_xYz789",
  "conversation_id": "conv_abc123",
  "model": "gemini-2.0-flash",            # which model actually handled it
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Quantum entanglement is like..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 42,
    "completion_tokens": 156,
    "total_tokens": 198,
    "opus_credits_used": 0,
    "opus_credits_remaining": 200
  },
  "meta": {
    "routing_reason": "simple_explanation",
    "latency_ms": 823,
    "escalated": false
  }
}
```

#### 2.3.2 Chat Completion (Streaming via SSE)

```yaml
POST /v1/chat/completions
Content-Type: application/json

{
  "messages": [...],
  "stream": true
}

# Response: text/event-stream
# --- Stream events ---

event: message.start
data: {"id":"msg_xYz789","model":"gemini-2.0-flash"}

event: content.delta
data: {"delta":"Quantum "}

event: content.delta
data: {"delta":"entanglement "}

event: content.delta
data: {"delta":"is like..."}

# If escalation occurs mid-stream:
event: escalation.start
data: {"reason":"low_confidence","from":"gemini-2.0-flash","to":"claude-opus-4-5"}

event: content.delta
data: {"delta":"[Opus continues] Let me give you a more precise explanation..."}

event: message.end
data: {"finish_reason":"stop","usage":{...},"meta":{...}}
```

#### 2.3.3 WebSocket (Real-time Bidirectional)

```yaml
GET /v1/stream
Upgrade: websocket
Sec-WebSocket-Protocol: clawdbot-v1

# --- Client → Server ---
{
  "type": "message.send",
  "conversation_id": "conv_abc123",
  "content": "Write me a Python function for merge sort",
  "model_preference": "auto"
}

# --- Server → Client ---
{ "type": "message.start", "id": "msg_001", "model": "gemini-2.0-flash" }
{ "type": "content.delta", "delta": "```python\n" }
{ "type": "content.delta", "delta": "def merge_sort(arr):\n" }
...
{ "type": "message.end", "usage": {...} }

# --- Client can also send tool results ---
{
  "type": "tool.result",
  "tool_call_id": "tc_abc",
  "result": { "status": "success", "output": "File saved." }
}
```

#### 2.3.4 Usage & Account

```yaml
GET /v1/usage
# Returns current billing period usage

{
  "period": { "start": "2025-07-01", "end": "2025-07-31" },
  "tier": "pro",
  "opus": {
    "used": 142,
    "limit": 600,
    "remaining": 458
  },
  "flash": {
    "requests": 1847,
    "tokens_in": 234000,
    "tokens_out": 512000
  },
  "total_requests": 1989,
  "estimated_cost_to_us": "$4.23"   # admin-only field
}

GET /v1/usage/history?days=30
# Daily breakdown

GET /v1/account
# Subscription details, billing info
```

#### 2.3.5 Tool Execution (Standalone)

```yaml
POST /v1/tools/execute
Content-Type: application/json

{
  "tool": "web_search",
  "parameters": {
    "query": "latest SpaceX launch",
    "count": 5
  },
  "conversation_id": "conv_abc123"  # optional context
}

# Response
{
  "tool_call_id": "tc_001",
  "tool": "web_search",
  "result": { "results": [...] },
  "model_used": "gemini-2.0-flash",
  "usage": { "opus_credits_used": 0 }
}
```

### 2.4 OpenAPI Spec (Abbreviated)

```yaml
openapi: "3.1.0"
info:
  title: Clawdbot SaaS API
  version: "1.0.0"
  description: Dual-model AI assistant API with intelligent routing
servers:
  - url: https://api.clawdbot.com/v1

paths:
  /chat/completions:
    post:
      operationId: createChatCompletion
      summary: Create a chat completion (sync or streaming)
      security:
        - BearerAuth: []
        - ApiKeyAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ChatCompletionRequest"
      responses:
        "200":
          description: Chat completion response
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ChatCompletionResponse"
            text/event-stream:
              schema:
                type: string
        "402":
          description: Opus credits exhausted
        "429":
          description: Rate limit exceeded

  /usage:
    get:
      operationId: getUsage
      summary: Get current period usage
      security:
        - BearerAuth: []
      responses:
        "200":
          description: Usage summary

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
    ApiKeyAuth:
      type: apiKey
      in: header
      name: Authorization

  schemas:
    ChatCompletionRequest:
      type: object
      required: [messages]
      properties:
        messages:
          type: array
          items:
            $ref: "#/components/schemas/Message"
        conversation_id:
          type: string
        model_preference:
          type: string
          enum: [auto, flash, opus]
          default: auto
        stream:
          type: boolean
          default: false
        tools:
          type: array
          items:
            $ref: "#/components/schemas/ToolDefinition"
        max_tokens:
          type: integer
          maximum: 32768
        temperature:
          type: number
          minimum: 0
          maximum: 2

    Message:
      type: object
      required: [role, content]
      properties:
        role:
          type: string
          enum: [system, user, assistant, tool]
        content:
          type: string
        tool_calls:
          type: array
          items:
            $ref: "#/components/schemas/ToolCall"
        tool_call_id:
          type: string

    ChatCompletionResponse:
      type: object
      properties:
        id:
          type: string
        conversation_id:
          type: string
        model:
          type: string
        choices:
          type: array
          items:
            type: object
            properties:
              index: { type: integer }
              message: { $ref: "#/components/schemas/Message" }
              finish_reason: { type: string }
        usage:
          $ref: "#/components/schemas/Usage"
        meta:
          type: object
          properties:
            routing_reason: { type: string }
            latency_ms: { type: integer }
            escalated: { type: boolean }

    Usage:
      type: object
      properties:
        prompt_tokens: { type: integer }
        completion_tokens: { type: integer }
        total_tokens: { type: integer }
        opus_credits_used: { type: integer }
        opus_credits_remaining: { type: integer }
```

### 2.5 Rate Limits by Tier

| Tier | Requests/min | Concurrent Streams | Max Tokens/Request |
|------|-------------|-------------------|-------------------|
| **Starter** ($19) | 30 | 3 | 8,192 |
| **Pro** ($49) | 60 | 10 | 16,384 |
| **Business** ($99) | 120 | 25 | 32,768 |

Rate limit headers on every response:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1720540800
X-Opus-Credits-Remaining: 458
```

---

## 3. Request Lifecycle

### 3.1 Full Flow Diagram

```
    User sends message
           │
           ▼
    ┌──────────────┐
    │ API Gateway   │──── TLS termination, basic validation
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐     ┌──────────────┐
    │ Auth Service  │────▶│ Redis Cache   │  Check cached JWT / API key
    └──────┬───────┘     └──────────────┘
           │
           │  User context attached:
           │  {user_id, tier, opus_remaining, rate_limit_ok}
           ▼
    ┌──────────────┐
    │ Rate Limiter  │──── Check sliding window counter in Redis
    └──────┬───────┘
           │
           │  Pass? ──── No ──▶ 429 Too Many Requests
           │
           ▼
    ┌──────────────────────────────────────────────────┐
    │                REQUEST ROUTER                     │
    │                                                    │
    │  Step 1: Check model_preference header             │
    │    ├── "flash"  → force Flash                      │
    │    ├── "opus"   → check credits, force Opus        │
    │    └── "auto"   → continue to classification       │
    │                                                    │
    │  Step 2: Fast heuristic check (< 1ms)              │
    │    • Message length < 50 chars + simple intent     │
    │      → Flash                                        │
    │    • Contains "analyze", "compare", "debug",       │
    │      "write a detailed", "review this code"        │
    │      → Opus candidate                               │
    │    • Tool call only (web_search, file_read)        │
    │      → Flash                                        │
    │                                                    │
    │  Step 3: Lightweight classifier (< 5ms)            │
    │    • TF-IDF + logistic regression on message       │
    │    • Features: length, vocabulary complexity,       │
    │      question type, conversation depth,             │
    │      presence of code, multi-step reasoning cues   │
    │    • Output: complexity_score (0.0 - 1.0)          │
    │                                                    │
    │  Step 4: Route decision                            │
    │    • score < 0.4  → Flash                          │
    │    • score ≥ 0.4 AND opus_credits > 0 → Opus      │
    │    • score ≥ 0.4 AND opus_credits = 0 → Flash     │
    │      (with header: X-Would-Use-Opus: true)         │
    │                                                    │
    └────────────────┬──────────────┬────────────────────┘
                     │              │
              ┌──────▼──────┐ ┌────▼────────┐
              │ Flash Path   │ │ Opus Path    │
              └──────┬───────┘ └────┬────────┘
                     │              │
                     ▼              ▼
              ┌──────────────────────────────┐
              │    MODEL INFERENCE SERVICE    │
              │                              │
              │  1. Convert to model-native  │
              │     format (tool adapter)    │
              │  2. Call model API           │
              │  3. Stream/collect response  │
              │  4. Convert back to unified  │
              │     format                   │
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │    CONFIDENCE CHECK           │
              │    (Flash path only)          │
              │                              │
              │  Scan response for:          │
              │  • "I'm not sure"            │
              │  • "I cannot" + complex task │
              │  • Contradictions            │
              │  • Very short response to    │
              │    complex query             │
              │  • Code with obvious errors  │
              │                              │
              │  confidence < 0.5?           │
              │    └── ESCALATE TO OPUS      │
              └──────────┬──────┬────────────┘
                         │      │
                    OK   │      │ Escalate
                         │      │
                         │      ▼
                         │  ┌──────────────────────┐
                         │  │ ESCALATION HANDLER    │
                         │  │                       │
                         │  │ 1. Check Opus credits │
                         │  │ 2. Bundle original    │
                         │  │    query + Flash      │
                         │  │    attempt as context  │
                         │  │ 3. Call Opus           │
                         │  │ 4. Replace or merge   │
                         │  │    response           │
                         │  └──────────┬────────────┘
                         │             │
                         └──────┬──────┘
                                │
                                ▼
              ┌──────────────────────────────┐
              │    RESPONSE MERGER            │
              │                              │
              │  • Normalize to unified      │
              │    ChatCompletionResponse     │
              │  • Attach model metadata     │
              │  • Attach usage counters     │
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │    USAGE TRACKER (async)      │
              │                              │
              │  → Redis XADD to stream      │
              │  → Worker processes:          │
              │    • Decrement Opus credits   │
              │    • Log tokens to analytics  │
              │    • Update billing counters  │
              └──────────────┬───────────────┘
                             │
                             ▼
                      Return to client
```

### 3.2 Routing Decision Matrix

| Signal | Flash | Opus | Notes |
|--------|-------|------|-------|
| Simple Q&A, greetings | ✅ | | "What's the weather?" |
| Tool execution (search, read) | ✅ | | Pure tool orchestration |
| Translation | ✅ | | Straightforward |
| Summarization (< 2K tokens) | ✅ | | Short summaries |
| Code generation (simple) | ✅ | | Single function, boilerplate |
| Multi-step analysis | | ✅ | "Compare these 3 approaches…" |
| Code review / debugging | | ✅ | Needs deep reasoning |
| Creative writing (long-form) | | ✅ | Quality-sensitive |
| Complex reasoning / math | | ✅ | Chain-of-thought needed |
| Summarization (> 5K tokens) | | ✅ | Long document analysis |
| User explicitly requests Opus | | ✅ | `model_preference: opus` |
| User out of Opus credits | ✅ | | Fallback with notice |

### 3.3 Router Implementation

```python
# router/classifier.py

from enum import Enum
from dataclasses import dataclass
import re
from typing import Optional

class ModelChoice(str, Enum):
    FLASH = "flash"
    OPUS = "opus"

@dataclass
class RoutingDecision:
    model: ModelChoice
    reason: str
    confidence: float          # 0-1, how confident we are in the routing
    complexity_score: float    # 0-1, estimated task complexity
    forced: bool = False       # user explicitly chose model

# Keywords and patterns that suggest complex tasks
OPUS_SIGNALS = [
    r"\banalyze\b.*\b(in detail|thoroughly|deeply)\b",
    r"\bcompare\b.*\b(and contrast|versus|differences)\b",
    r"\bdebug\b.*\b(this|my|the)\b.*\b(code|error|issue)\b",
    r"\breview\b.*\b(code|architecture|design|approach)\b",
    r"\bwrite\b.*\b(essay|article|story|poem|detailed)\b",
    r"\bexplain\b.*\b(why|how).*\b(works|happens|differs)\b",
    r"\brefactor\b",
    r"\boptimize\b.*\b(performance|query|algorithm)\b",
    r"\bprove\b.*\b(that|mathematically|formally)\b",
    r"\bdesign\b.*\b(system|architecture|schema|database)\b",
]

FLASH_SIGNALS = [
    r"^(hi|hello|hey|thanks|ok|sure|yes|no)\b",
    r"\b(search|find|look up|google)\b",
    r"\b(what is|who is|when did|where is)\b",
    r"\b(translate|convert)\b",
    r"\b(summarize|tldr|summary)\b.{0,100}$",  # short summarization
    r"\b(run|execute|call)\b.*\b(tool|function|command)\b",
]

class RequestRouter:
    def __init__(self, classifier_model=None):
        """
        classifier_model: Optional sklearn/ONNX model for
                         complexity scoring. None = heuristics only.
        """
        self.classifier = classifier_model
        self._compile_patterns()

    def _compile_patterns(self):
        self.opus_patterns = [re.compile(p, re.IGNORECASE) for p in OPUS_SIGNALS]
        self.flash_patterns = [re.compile(p, re.IGNORECASE) for p in FLASH_SIGNALS]

    def route(
        self,
        message: str,
        conversation_depth: int,
        user_preference: Optional[str],
        opus_credits_remaining: int,
        has_tools: bool,
    ) -> RoutingDecision:
        """Main routing logic."""

        # Step 1: User override
        if user_preference == "opus":
            if opus_credits_remaining <= 0:
                return RoutingDecision(
                    model=ModelChoice.FLASH,
                    reason="opus_requested_but_no_credits",
                    confidence=1.0,
                    complexity_score=0.5,
                    forced=True,
                )
            return RoutingDecision(
                model=ModelChoice.OPUS,
                reason="user_explicit_request",
                confidence=1.0,
                complexity_score=0.5,
                forced=True,
            )
        if user_preference == "flash":
            return RoutingDecision(
                model=ModelChoice.FLASH,
                reason="user_explicit_request",
                confidence=1.0,
                complexity_score=0.2,
                forced=True,
            )

        # Step 2: Fast heuristics
        flash_score = sum(1 for p in self.flash_patterns if p.search(message))
        opus_score = sum(1 for p in self.opus_patterns if p.search(message))

        # Short messages (< 30 chars) are almost always Flash
        if len(message) < 30 and opus_score == 0:
            return RoutingDecision(
                model=ModelChoice.FLASH,
                reason="short_simple_message",
                confidence=0.95,
                complexity_score=0.1,
            )

        # Tool-only requests → Flash
        if has_tools and opus_score == 0:
            return RoutingDecision(
                model=ModelChoice.FLASH,
                reason="tool_execution",
                confidence=0.9,
                complexity_score=0.2,
            )

        # Step 3: Complexity scoring
        complexity = self._compute_complexity(
            message, conversation_depth, flash_score, opus_score
        )

        # Step 4: Route decision
        threshold = 0.4

        if complexity >= threshold and opus_credits_remaining > 0:
            return RoutingDecision(
                model=ModelChoice.OPUS,
                reason=f"complexity_score_{complexity:.2f}",
                confidence=min(complexity + 0.3, 1.0),
                complexity_score=complexity,
            )
        else:
            return RoutingDecision(
                model=ModelChoice.FLASH,
                reason=(
                    f"below_threshold_{complexity:.2f}"
                    if complexity < threshold
                    else "no_opus_credits"
                ),
                confidence=max(1.0 - complexity, 0.5),
                complexity_score=complexity,
            )

    def _compute_complexity(
        self,
        message: str,
        conversation_depth: int,
        flash_score: int,
        opus_score: int,
    ) -> float:
        """
        Compute a 0-1 complexity score.
        Uses lightweight features. Replace with ML model in v2.
        """
        score = 0.0
        words = message.split()
        word_count = len(words)

        # Length factor (longer = more complex)
        if word_count > 200:
            score += 0.3
        elif word_count > 100:
            score += 0.2
        elif word_count > 50:
            score += 0.1

        # Pattern matches
        score += opus_score * 0.15
        score -= flash_score * 0.1

        # Code presence (triple backticks, indentation patterns)
        if "```" in message or message.count("\n    ") > 3:
            score += 0.15

        # Question complexity (multiple question marks, "and also")
        if message.count("?") > 2:
            score += 0.1
        if re.search(r"\b(and also|additionally|furthermore|moreover)\b", message):
            score += 0.1

        # Conversation depth (deeper = more likely to need reasoning)
        if conversation_depth > 10:
            score += 0.1
        elif conversation_depth > 5:
            score += 0.05

        # Unique vocabulary (proxy for complexity)
        unique_ratio = len(set(w.lower() for w in words)) / max(word_count, 1)
        if unique_ratio > 0.8:
            score += 0.1

        return max(0.0, min(1.0, score))
```

---

## 4. Usage Tracking & Credit System

### 4.1 Database Schema (PostgreSQL)

```sql
-- =============================================================
-- Core tables for the credit and usage tracking system
-- =============================================================

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255),                    -- NULL for OAuth-only users
    name            VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    stripe_customer_id VARCHAR(255) UNIQUE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- Subscription tiers (source of truth for plan config)
CREATE TABLE tiers (
    id              VARCHAR(50) PRIMARY KEY,          -- 'starter', 'pro', 'business'
    name            VARCHAR(100) NOT NULL,
    price_cents     INTEGER NOT NULL,                 -- 1900, 4900, 9900
    opus_credits    INTEGER NOT NULL,                 -- 200, 600, 1500
    rate_limit_rpm  INTEGER NOT NULL,                 -- 30, 60, 120
    max_concurrent  INTEGER NOT NULL,                 -- 3, 10, 25
    max_tokens      INTEGER NOT NULL,                 -- 8192, 16384, 32768
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO tiers VALUES
    ('starter',  'Starter',  1900,  200,  30,  3, 8192,  NOW()),
    ('pro',      'Pro',      4900,  600,  60, 10, 16384, NOW()),
    ('business', 'Business', 9900, 1500, 120, 25, 32768, NOW());

-- Subscriptions (one active per user)
CREATE TABLE subscriptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    tier_id             VARCHAR(50) NOT NULL REFERENCES tiers(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
                        -- 'active', 'past_due', 'cancelled', 'trialing'
    stripe_subscription_id VARCHAR(255) UNIQUE,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    cancelled_at        TIMESTAMPTZ,

    CONSTRAINT uq_user_active_sub UNIQUE (user_id, status)
        -- Only one active subscription per user (enforced via partial index below)
);

CREATE UNIQUE INDEX idx_one_active_sub_per_user
    ON subscriptions(user_id)
    WHERE status = 'active';

-- Credit balances (hot table, updated frequently)
CREATE TABLE credit_balances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    subscription_id     UUID NOT NULL REFERENCES subscriptions(id),
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ NOT NULL,
    opus_credits_total  INTEGER NOT NULL,             -- allocated for this period
    opus_credits_used   INTEGER NOT NULL DEFAULT 0,
    opus_credits_bonus  INTEGER NOT NULL DEFAULT 0,   -- purchased add-ons
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_user_period UNIQUE (user_id, period_start)
);

-- Audit every credit change
CREATE TABLE credit_transactions (
    id              BIGSERIAL PRIMARY KEY,
    user_id         UUID NOT NULL REFERENCES users(id),
    balance_id      UUID NOT NULL REFERENCES credit_balances(id),
    delta           INTEGER NOT NULL,                 -- +1 for refund, -1 for usage
    reason          VARCHAR(50) NOT NULL,
                    -- 'opus_usage', 'escalation_usage', 'period_reset',
                    -- 'bonus_purchase', 'admin_adjustment', 'refund'
    request_id      VARCHAR(100),                     -- links to the API request
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_credit_tx_user_time ON credit_transactions(user_id, created_at DESC);

-- Per-request usage log (high-volume, consider partitioning)
CREATE TABLE request_log (
    id              BIGSERIAL,
    request_id      VARCHAR(100) NOT NULL,
    user_id         UUID NOT NULL,
    conversation_id VARCHAR(100),
    model_used      VARCHAR(50) NOT NULL,             -- 'gemini-2.0-flash', 'claude-opus-4-5'
    routed_by       VARCHAR(50) NOT NULL,             -- 'heuristic', 'classifier', 'user_override', 'escalation'
    complexity_score REAL,
    prompt_tokens   INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    total_tokens    INTEGER NOT NULL,
    opus_credit_charged BOOLEAN NOT NULL DEFAULT FALSE,
    latency_ms      INTEGER NOT NULL,
    escalated       BOOLEAN NOT NULL DEFAULT FALSE,
    escalation_reason VARCHAR(100),
    status          VARCHAR(20) NOT NULL,             -- 'success', 'error', 'timeout'
    error_code      VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE request_log_2025_07 PARTITION OF request_log
    FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
CREATE TABLE request_log_2025_08 PARTITION OF request_log
    FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');

CREATE INDEX idx_request_log_user ON request_log(user_id, created_at DESC);
CREATE INDEX idx_request_log_model ON request_log(model_used, created_at DESC);

-- API keys
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    key_hash        VARCHAR(255) NOT NULL UNIQUE,     -- SHA-256 of the key
    key_prefix      VARCHAR(20) NOT NULL,             -- 'cb_live_sk_xxxx' for display
    name            VARCHAR(100),                     -- user-given name
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ                       -- NULL = no expiry
);
```

### 4.2 Credit Check Flow (Hot Path)

The credit check is on the **critical path** of every potentially-Opus request. It must be fast.

```python
# credits/service.py

import redis.asyncio as redis
from datetime import datetime, timezone
import asyncpg

class CreditService:
    """
    Two-tier credit check:
      1. Redis (hot cache, ~0.2ms) — authoritative for checks
      2. PostgreSQL (cold storage) — authoritative for totals, synced async
    
    Redis key: credits:{user_id}:{period_start}
    Redis value: hash { total, used, bonus }
    Redis TTL: period_end + 1 day
    """

    def __init__(self, redis_pool: redis.Redis, pg_pool: asyncpg.Pool):
        self.redis = redis_pool
        self.pg = pg_pool

    def _cache_key(self, user_id: str, period_start: str) -> str:
        return f"credits:{user_id}:{period_start}"

    async def get_remaining(self, user_id: str, period_start: str) -> int:
        """Get remaining Opus credits. Returns -1 if no active subscription."""
        key = self._cache_key(user_id, period_start)

        # Try Redis first
        cached = await self.redis.hgetall(key)
        if cached:
            total = int(cached[b"total"]) + int(cached[b"bonus"])
            used = int(cached[b"used"])
            return total - used

        # Cache miss — load from PG and populate Redis
        row = await self.pg.fetchrow("""
            SELECT opus_credits_total, opus_credits_used, opus_credits_bonus,
                   period_end
            FROM credit_balances
            WHERE user_id = $1 AND period_start = $2
        """, user_id, period_start)

        if not row:
            return -1

        # Populate cache
        ttl_seconds = int((row["period_end"] - datetime.now(timezone.utc)).total_seconds()) + 86400
        await self.redis.hset(key, mapping={
            "total": row["opus_credits_total"],
            "used": row["opus_credits_used"],
            "bonus": row["opus_credits_bonus"],
        })
        await self.redis.expire(key, max(ttl_seconds, 3600))

        return (row["opus_credits_total"] + row["opus_credits_bonus"]) - row["opus_credits_used"]

    async def consume_credit(self, user_id: str, period_start: str, request_id: str) -> bool:
        """
        Atomically consume one Opus credit.
        Returns True if credit was consumed, False if insufficient.
        
        Uses Redis EVAL (Lua script) for atomicity.
        """
        key = self._cache_key(user_id, period_start)

        lua_script = """
        local total = tonumber(redis.call('HGET', KEYS[1], 'total') or 0)
        local bonus = tonumber(redis.call('HGET', KEYS[1], 'bonus') or 0)
        local used  = tonumber(redis.call('HGET', KEYS[1], 'used') or 0)
        
        if used < (total + bonus) then
            redis.call('HINCRBY', KEYS[1], 'used', 1)
            return 1
        else
            return 0
        end
        """

        result = await self.redis.eval(lua_script, 1, key)

        if result == 1:
            # Async write-behind to PostgreSQL
            await self._enqueue_credit_update(user_id, period_start, request_id)
            return True
        return False

    async def _enqueue_credit_update(self, user_id: str, period_start: str, request_id: str):
        """Push credit update to Redis Stream for async PG write."""
        await self.redis.xadd("credit_updates", {
            "user_id": user_id,
            "period_start": period_start,
            "request_id": request_id,
            "delta": "-1",
            "reason": "opus_usage",
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })

    async def refund_credit(self, user_id: str, period_start: str, request_id: str, reason: str):
        """Refund a credit (e.g., model error, timeout)."""
        key = self._cache_key(user_id, period_start)
        await self.redis.hincrby(key, "used", -1)
        await self.redis.xadd("credit_updates", {
            "user_id": user_id,
            "period_start": period_start,
            "request_id": request_id,
            "delta": "1",
            "reason": reason,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })
```

### 4.3 Credit Update Worker

```python
# workers/credit_worker.py

import asyncio
import redis.asyncio as redis
import asyncpg

async def credit_update_worker(redis_pool: redis.Redis, pg_pool: asyncpg.Pool):
    """
    Consumes credit_updates stream and writes to PostgreSQL.
    Runs as a background task / separate process.
    """
    last_id = "0-0"

    while True:
        try:
            # Block for up to 5 seconds waiting for new entries
            entries = await redis_pool.xread(
                {"credit_updates": last_id},
                count=100,
                block=5000,
            )

            if not entries:
                continue

            for stream, messages in entries:
                async with pg_pool.acquire() as conn:
                    async with conn.transaction():
                        for msg_id, data in messages:
                            user_id = data[b"user_id"].decode()
                            period_start = data[b"period_start"].decode()
                            delta = int(data[b"delta"])
                            reason = data[b"reason"].decode()
                            request_id = data.get(b"request_id", b"").decode()

                            # Update balance
                            await conn.execute("""
                                UPDATE credit_balances
                                SET opus_credits_used = opus_credits_used - $1,
                                    updated_at = NOW()
                                WHERE user_id = $2 AND period_start = $3
                            """, delta, user_id, period_start)
                            # Note: delta is negative for usage, so subtracting
                            # a negative = incrementing used

                            # Audit log
                            await conn.execute("""
                                INSERT INTO credit_transactions
                                    (user_id, balance_id, delta, reason, request_id)
                                SELECT $1, cb.id, $2, $3, $4
                                FROM credit_balances cb
                                WHERE cb.user_id = $1 AND cb.period_start = $5
                            """, user_id, delta, reason, request_id, period_start)

                            last_id = msg_id

            # Trim processed entries (keep last 10K for debugging)
            await redis_pool.xtrim("credit_updates", maxlen=10000, approximate=True)

        except Exception as e:
            print(f"Credit worker error: {e}")
            await asyncio.sleep(1)
```

### 4.4 Period Reset (Cron Job)

```python
# workers/period_reset.py
# Runs daily at 00:05 UTC via cron / APScheduler

async def reset_credits_for_renewed_subscriptions(pg_pool):
    """
    When a subscription period ends and renews (Stripe webhook confirms payment),
    create a new credit_balances row for the new period.
    """
    rows = await pg_pool.fetch("""
        SELECT s.id as subscription_id, s.user_id, s.tier_id,
               s.current_period_end as new_period_start,
               t.opus_credits
        FROM subscriptions s
        JOIN tiers t ON t.id = s.tier_id
        WHERE s.status = 'active'
          AND s.current_period_end <= NOW()
          AND NOT EXISTS (
              SELECT 1 FROM credit_balances cb
              WHERE cb.user_id = s.user_id
                AND cb.period_start = s.current_period_end
          )
    """)

    for row in rows:
        new_period_end = row["new_period_start"] + timedelta(days=30)
        await pg_pool.execute("""
            INSERT INTO credit_balances
                (user_id, subscription_id, period_start, period_end,
                 opus_credits_total, opus_credits_used, opus_credits_bonus)
            VALUES ($1, $2, $3, $4, $5, 0, 0)
        """, row["user_id"], row["subscription_id"],
             row["new_period_start"], new_period_end, row["opus_credits"])
```

### 4.5 Overage Handling Strategies

```
Strategy 1: HARD STOP (MVP)
┌─────────────────────────────────────┐
│ Credits exhausted → Flash only      │
│ Response includes:                  │
│   X-Opus-Credits-Remaining: 0       │
│   X-Would-Use-Opus: true            │
│   Body hint: "Upgrade for Opus..."  │
└─────────────────────────────────────┘

Strategy 2: SOFT OVERAGE (v2)
┌─────────────────────────────────────┐
│ Allow 10% overage buffer            │
│ Charge $0.15 per extra Opus call    │
│ Notify user at 80%, 100%, 110%      │
│ Hard stop at 120%                   │
└─────────────────────────────────────┘

Strategy 3: CREDIT PACKS (v2)
┌─────────────────────────────────────┐
│ Users can buy 50-pack for $5        │
│ Never expires, rolls over           │
│ Used after monthly allocation       │
└─────────────────────────────────────┘
```

---

## 5. Streaming Architecture

### 5.1 The Streaming Challenge

Both models support streaming, but the challenge is the **escalation case**: Flash starts streaming, the system detects low confidence mid-response, and needs to switch to Opus.

### 5.2 Streaming Flow (Normal Case)

```
Client                    Server                   Model API
  │                         │                         │
  │── POST /chat (stream) ─▶│                         │
  │                         │── Route to Flash ──────▶│
  │                         │                         │
  │◀── event: msg.start ───│                         │
  │                         │◀── chunk "Hello " ─────│
  │◀── event: delta ───────│                         │
  │                         │◀── chunk "world!" ─────│
  │◀── event: delta ───────│                         │
  │                         │◀── [DONE] ─────────────│
  │◀── event: msg.end ─────│                         │
  │                         │                         │
```

### 5.3 Escalation During Streaming

Three strategies, ordered by complexity:

#### Strategy A: Buffer & Replace (MVP — Recommended)

Don't stream Flash responses for the first N tokens. Wait for a confidence signal, then decide.

```
Client                    Server                   Flash              Opus
  │                         │                        │                  │
  │── POST /chat (stream) ─▶│                        │                  │
  │                         │── call Flash ─────────▶│                  │
  │                         │                        │                  │
  │  (BUFFERING - no        │◀── chunk 1 ───────────│                  │
  │   stream to client yet) │◀── chunk 2 ───────────│                  │
  │                         │◀── chunk 3 ───────────│                  │
  │                         │                        │                  │
  │                         │── confidence check ────│                  │
  │                         │   score = 0.3 (LOW!)   │                  │
  │                         │                        │                  │
  │                         │── ESCALATE: call Opus ─┼─────────────────▶│
  │                         │   (discard Flash buf)  │                  │
  │                         │                        │                  │
  │◀── event: msg.start ───│  (model: opus)         │                  │
  │                         │◀── chunk 1 ────────────┼──────────────────│
  │◀── event: delta ───────│                        │                  │
  │                         │◀── chunk 2 ────────────┼──────────────────│
  │◀── event: delta ───────│                        │                  │
  │◀── event: msg.end ─────│                        │                  │
```

```python
# streaming/buffer_strategy.py

from typing import AsyncIterator
import asyncio

BUFFER_TOKEN_THRESHOLD = 50  # Buffer first 50 tokens before streaming
CONFIDENCE_CHECK_THRESHOLD = 0.5

async def stream_with_escalation(
    request,
    router,
    flash_service,
    opus_service,
    credit_service,
    confidence_checker,
) -> AsyncIterator[dict]:
    """
    Buffer initial Flash tokens, check confidence, then either
    stream the buffer + continue, or escalate to Opus.
    """
    routing = router.route(request)
    
    if routing.model == "opus":
        # Direct to Opus — no buffering needed
        async for event in opus_service.stream(request):
            yield event
        return

    # Flash path with potential escalation
    buffer = []
    buffer_text = ""
    flash_stream = flash_service.stream(request)

    # Phase 1: Buffer initial tokens
    async for event in flash_stream:
        if event["type"] == "content.delta":
            buffer.append(event)
            buffer_text += event["delta"]
            
            if len(buffer_text.split()) >= BUFFER_TOKEN_THRESHOLD:
                break
        elif event["type"] == "message.start":
            # Hold this event — don't send yet
            buffer.insert(0, event)
            continue

    # Phase 2: Confidence check on buffered content
    confidence = confidence_checker.score(
        query=request.messages[-1]["content"],
        response_so_far=buffer_text,
    )

    if confidence < CONFIDENCE_CHECK_THRESHOLD and await credit_service.has_credits(request.user_id):
        # ESCALATE to Opus
        # Cancel the Flash stream
        await flash_stream.aclose()

        yield {"type": "message.start", "model": "claude-opus-4-5", "escalated": True}
        
        # Feed original query + Flash attempt as context to Opus
        enhanced_messages = request.messages + [
            {
                "role": "assistant",
                "content": f"[Previous attempt was insufficient: {buffer_text}]",
            },
            {
                "role": "user", 
                "content": "Please provide a better, more thorough response to my original question.",
            },
        ]

        async for event in opus_service.stream(request.with_messages(enhanced_messages)):
            yield event
    else:
        # Flash confidence is fine — flush buffer and continue streaming
        yield {"type": "message.start", "model": "gemini-2.0-flash"}
        
        for event in buffer:
            if event["type"] == "content.delta":
                yield event

        # Continue streaming remaining Flash output
        async for event in flash_stream:
            yield event
```

#### Strategy B: Parallel Start (v2 — Lower Latency for Opus)

For requests near the routing threshold, start both models and cancel the loser.

```
Client                    Server              Flash              Opus
  │                         │                   │                  │
  │── POST /chat (stream) ─▶│                   │                  │
  │                         │── call BOTH ──────▶│                  │
  │                         │── call BOTH ──────┼─────────────────▶│
  │                         │                   │                  │
  │  (wait for first        │◀── chunk ─────────│                  │
  │   tokens from both)     │◀── chunk ─────────┼──────────────────│
  │                         │                   │                  │
  │                         │── pick winner ────│                  │
  │                         │── cancel loser ───│                  │
  │                         │                   │                  │
  │◀── stream winner ──────│                   │                  │
```

**Cost implication:** This burns Opus tokens even when Flash wins. Only viable for borderline requests (~5% of traffic). We'd need a separate credit policy.

#### Strategy C: Notification-Based (v2 — Best UX)

Stream Flash immediately, but if escalation triggers, notify the client and let them choose.

```
event: content.delta
data: {"delta": "Here's my answer..."}

event: escalation.available
data: {"reason": "This response may benefit from deeper analysis",
       "action": "reply with 'use opus' to re-generate with Opus"}
```

### 5.4 SSE Implementation

```python
# api/streaming.py

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
import json

router = APIRouter()

@router.post("/v1/chat/completions")
async def chat_completions(request: Request, body: ChatCompletionRequest):
    if body.stream:
        return StreamingResponse(
            event_stream(request, body),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering
            },
        )
    else:
        return await sync_completion(request, body)


async def event_stream(request: Request, body: ChatCompletionRequest):
    """Generate SSE events."""
    try:
        async for event in stream_with_escalation(
            request=body,
            router=app.state.router,
            flash_service=app.state.flash,
            opus_service=app.state.opus,
            credit_service=app.state.credits,
            confidence_checker=app.state.confidence,
        ):
            event_type = event.pop("type", "content.delta")
            data = json.dumps(event)
            yield f"event: {event_type}\ndata: {data}\n\n"

        # Final usage event
        yield f"event: message.end\ndata: {json.dumps(usage_summary)}\n\n"

    except asyncio.CancelledError:
        # Client disconnected
        pass
    except Exception as e:
        error_event = {"error": str(e), "code": "internal_error"}
        yield f"event: error\ndata: {json.dumps(error_event)}\n\n"
```

### 5.5 WebSocket Implementation

```python
# api/websocket.py

from fastapi import WebSocket, WebSocketDisconnect

@router.websocket("/v1/stream")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept(subprotocol="clawdbot-v1")

    user = await authenticate_websocket(websocket)
    if not user:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    try:
        while True:
            data = await websocket.receive_json()

            if data["type"] == "message.send":
                # Stream response back
                async for event in process_message(user, data):
                    await websocket.send_json(event)

            elif data["type"] == "tool.result":
                # Handle tool result from client
                await handle_tool_result(user, data)

            elif data["type"] == "ping":
                await websocket.send_json({"type": "pong"})

    except WebSocketDisconnect:
        pass
```

---

## 6. Tool Calling Abstraction

### 6.1 The Problem

Gemini and Claude use fundamentally different tool calling formats:

**Gemini (Google AI):**
```json
{
  "function_declarations": [
    {
      "name": "web_search",
      "description": "Search the web",
      "parameters": {
        "type": "object",
        "properties": {
          "query": { "type": "string" }
        },
        "required": ["query"]
      }
    }
  ]
}
```

**Claude (Anthropic):**
```json
{
  "tools": [
    {
      "name": "web_search",
      "description": "Search the web",
      "input_schema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "Search query" }
        },
        "required": ["query"]
      }
    }
  ]
}
```

Their response formats differ too — Gemini returns `functionCall` blocks, Claude returns `tool_use` content blocks.

### 6.2 Unified Tool Interface

```python
# tools/schema.py

from dataclasses import dataclass, field
from typing import Any, Optional
from enum import Enum

@dataclass
class ToolParameter:
    name: str
    type: str                          # "string", "integer", "boolean", "array", "object"
    description: str
    required: bool = True
    enum: Optional[list] = None
    items: Optional[dict] = None       # For array types
    default: Optional[Any] = None

@dataclass
class ToolDefinition:
    """Unified tool definition. Model adapters convert to/from this."""
    name: str
    description: str
    parameters: list[ToolParameter]
    category: str = "general"          # "general", "browser", "code", "data"

    def to_json_schema(self) -> dict:
        """Convert to JSON Schema (shared base format)."""
        properties = {}
        required = []
        for param in self.parameters:
            prop = {"type": param.type, "description": param.description}
            if param.enum:
                prop["enum"] = param.enum
            if param.items:
                prop["items"] = param.items
            properties[param.name] = prop
            if param.required:
                required.append(param.name)
        return {
            "type": "object",
            "properties": properties,
            "required": required,
        }

@dataclass
class ToolCall:
    """Unified tool call (from model output)."""
    id: str
    name: str
    arguments: dict[str, Any]

@dataclass 
class ToolResult:
    """Unified tool result (to feed back to model)."""
    tool_call_id: str
    name: str
    result: Any
    is_error: bool = False
    error_message: Optional[str] = None
```

### 6.3 Model Adapters

```python
# tools/adapters.py

from abc import ABC, abstractmethod
from typing import Any

class ModelToolAdapter(ABC):
    """Abstract adapter for converting between unified and model-specific formats."""

    @abstractmethod
    def format_tools(self, tools: list[ToolDefinition]) -> Any:
        """Convert unified tool defs to model-specific format."""
        ...

    @abstractmethod
    def parse_tool_calls(self, model_response: Any) -> list[ToolCall]:
        """Extract tool calls from model response."""
        ...

    @abstractmethod
    def format_tool_results(self, results: list[ToolResult]) -> Any:
        """Format tool results for feeding back to model."""
        ...

    @abstractmethod
    def format_messages(self, messages: list[dict]) -> Any:
        """Convert unified message format to model-specific format."""
        ...


class GeminiAdapter(ModelToolAdapter):
    """Adapter for Gemini 2.0 Flash (Google AI API)."""

    def format_tools(self, tools: list[ToolDefinition]) -> list[dict]:
        return [{
            "function_declarations": [
                {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.to_json_schema(),
                }
                for tool in tools
            ]
        }]

    def parse_tool_calls(self, response) -> list[ToolCall]:
        """Parse Gemini's response for function calls."""
        calls = []
        for candidate in response.candidates:
            for part in candidate.content.parts:
                if hasattr(part, "function_call") and part.function_call:
                    fc = part.function_call
                    calls.append(ToolCall(
                        id=f"gemini_tc_{hash(fc.name)}_{len(calls)}",
                        name=fc.name,
                        arguments=dict(fc.args),
                    ))
        return calls

    def format_tool_results(self, results: list[ToolResult]) -> list[dict]:
        """Format as Gemini function response parts."""
        return [{
            "function_response": {
                "name": r.name,
                "response": {"result": r.result} if not r.is_error else {"error": r.error_message},
            }
        } for r in results]

    def format_messages(self, messages: list[dict]) -> list[dict]:
        """Convert to Gemini's content format."""
        contents = []
        for msg in messages:
            if msg["role"] == "system":
                # Gemini uses system_instruction separately
                continue
            role = "user" if msg["role"] == "user" else "model"
            contents.append({
                "role": role,
                "parts": [{"text": msg["content"]}],
            })
        return contents

    def get_system_instruction(self, messages: list[dict]) -> Optional[str]:
        """Extract system message for Gemini's system_instruction field."""
        for msg in messages:
            if msg["role"] == "system":
                return msg["content"]
        return None


class ClaudeAdapter(ModelToolAdapter):
    """Adapter for Claude Opus 4.5 (Anthropic API)."""

    def format_tools(self, tools: list[ToolDefinition]) -> list[dict]:
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.to_json_schema(),
            }
            for tool in tools
        ]

    def parse_tool_calls(self, response) -> list[ToolCall]:
        """Parse Claude's response for tool_use blocks."""
        calls = []
        for block in response.content:
            if block.type == "tool_use":
                calls.append(ToolCall(
                    id=block.id,
                    name=block.name,
                    arguments=block.input,
                ))
        return calls

    def format_tool_results(self, results: list[ToolResult]) -> list[dict]:
        """Format as Claude tool_result content blocks."""
        return [
            {
                "type": "tool_result",
                "tool_use_id": r.tool_call_id,
                "content": str(r.result) if not r.is_error else r.error_message,
                "is_error": r.is_error,
            }
            for r in results
        ]

    def format_messages(self, messages: list[dict]) -> list[dict]:
        """Convert to Claude's message format."""
        # Claude handles system separately
        return [
            {"role": msg["role"], "content": msg["content"]}
            for msg in messages
            if msg["role"] != "system"
        ]

    def get_system_prompt(self, messages: list[dict]) -> Optional[str]:
        for msg in messages:
            if msg["role"] == "system":
                return msg["content"]
        return None
```

### 6.4 Tool Registry

```python
# tools/registry.py

class ToolRegistry:
    """Central registry of all available tools."""

    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}
        self._handlers: dict[str, callable] = {}
        self._model_overrides: dict[str, str] = {}  # tool_name → required_model

    def register(
        self,
        definition: ToolDefinition,
        handler: callable,
        requires_model: Optional[str] = None,
    ):
        """Register a tool with its handler function."""
        self._tools[definition.name] = definition
        self._handlers[definition.name] = handler
        if requires_model:
            self._model_overrides[definition.name] = requires_model

    def get_tools_for_model(self, model: str) -> list[ToolDefinition]:
        """Get tools compatible with a given model."""
        return [
            tool for name, tool in self._tools.items()
            if self._model_overrides.get(name) in (None, model)
        ]

    async def execute(self, tool_call: ToolCall) -> ToolResult:
        """Execute a tool call and return the result."""
        handler = self._handlers.get(tool_call.name)
        if not handler:
            return ToolResult(
                tool_call_id=tool_call.id,
                name=tool_call.name,
                result=None,
                is_error=True,
                error_message=f"Unknown tool: {tool_call.name}",
            )

        try:
            result = await handler(**tool_call.arguments)
            return ToolResult(
                tool_call_id=tool_call.id,
                name=tool_call.name,
                result=result,
            )
        except Exception as e:
            return ToolResult(
                tool_call_id=tool_call.id,
                name=tool_call.name,
                result=None,
                is_error=True,
                error_message=str(e),
            )


# Registration example
registry = ToolRegistry()

registry.register(
    definition=ToolDefinition(
        name="web_search",
        description="Search the web using Brave Search API",
        parameters=[
            ToolParameter(name="query", type="string", description="Search query"),
            ToolParameter(name="count", type="integer", description="Number of results", required=False, default=5),
        ],
        category="browser",
    ),
    handler=web_search_handler,
)
```

### 6.5 Tool Execution Loop

```python
# inference/tool_loop.py

MAX_TOOL_ITERATIONS = 10

async def run_with_tools(
    model_service,       # FlashService or OpusService
    adapter: ModelToolAdapter,
    messages: list[dict],
    tools: list[ToolDefinition],
    registry: ToolRegistry,
) -> dict:
    """
    Run model inference with tool calling loop.
    Handles multi-step tool use (model calls tool, gets result,
    calls another tool, etc.)
    """
    formatted_tools = adapter.format_tools(tools)
    conversation = adapter.format_messages(messages)
    all_tokens = {"prompt": 0, "completion": 0}

    for iteration in range(MAX_TOOL_ITERATIONS):
        response = await model_service.generate(
            messages=conversation,
            tools=formatted_tools,
        )

        all_tokens["prompt"] += response.usage.prompt_tokens
        all_tokens["completion"] += response.usage.completion_tokens

        # Check for tool calls
        tool_calls = adapter.parse_tool_calls(response)

        if not tool_calls:
            # No tool calls — model is done
            return {
                "response": response,
                "tokens": all_tokens,
                "tool_iterations": iteration,
            }

        # Execute all tool calls (in parallel where possible)
        results = await asyncio.gather(*[
            registry.execute(tc) for tc in tool_calls
        ])

        # Feed results back to model
        formatted_results = adapter.format_tool_results(results)
        conversation.extend(formatted_results)

    # Exceeded max iterations
    return {
        "response": response,
        "tokens": all_tokens,
        "tool_iterations": MAX_TOOL_ITERATIONS,
        "truncated": True,
    }
```

---

## 7. Error Handling & Fallback

### 7.1 Failure Modes & Recovery

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ERROR HANDLING DECISION TREE                         │
│                                                                             │
│  Request arrives                                                            │
│    │                                                                        │
│    ├── Auth fails ──▶ 401 Unauthorized                                      │
│    ├── Rate limited ──▶ 429 + Retry-After header                           │
│    ├── Credits exhausted + Opus requested ──▶ 402 Payment Required         │
│    │                                                                        │
│    ├── Router succeeds ──▶ Route to Flash or Opus                          │
│    │     │                                                                  │
│    │     ├── Flash API call                                                │
│    │     │     ├── Success ──▶ Return response                             │
│    │     │     ├── 429 (Google rate limit) ──▶ Retry 3x with backoff      │
│    │     │     │     └── Still failing ──▶ Fallback to Opus (if credits)   │
│    │     │     │                     └── No credits ──▶ 503 + queue        │
│    │     │     ├── 500/502/503 (Google outage) ──▶ Retry 2x               │
│    │     │     │     └── Still failing ──▶ Fallback to Opus               │
│    │     │     ├── Timeout (>30s) ──▶ Return partial + error               │
│    │     │     └── Invalid response ──▶ Retry 1x, then 500               │
│    │     │                                                                  │
│    │     ├── Opus API call                                                 │
│    │     │     ├── Success ──▶ Return response                             │
│    │     │     ├── 429 (Anthropic rate limit) ──▶ Retry 3x with backoff   │
│    │     │     │     └── Still failing ──▶ Fallback to Flash + refund     │
│    │     │     ├── 500/502/503 (Anthropic outage) ──▶ Retry 2x           │
│    │     │     │     └── Still failing ──▶ Fallback to Flash + refund     │
│    │     │     ├── 529 (Overloaded) ──▶ Queue + retry in 30s             │
│    │     │     ├── Timeout (>60s) ──▶ Return partial + error + refund     │
│    │     │     └── Invalid response ──▶ Retry 1x, then 500 + refund      │
│    │     │                                                                  │
│    │     └── Both models down ──▶ 503 Service Unavailable                  │
│    │           Body: "Our AI providers are experiencing issues.             │
│    │                  We're on it. Try again in a few minutes."            │
│    │           + Incident alert triggered                                   │
│    │                                                                        │
│    └── Router fails ──▶ Default to Flash (safe fallback)                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Retry Logic

```python
# core/retry.py

import asyncio
import random
from typing import TypeVar, Callable
from functools import wraps

T = TypeVar("T")

class RetryConfig:
    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 30.0,
        exponential_base: float = 2.0,
        jitter: bool = True,
        retryable_status_codes: set = None,
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
        self.retryable_status_codes = retryable_status_codes or {429, 500, 502, 503, 529}

FLASH_RETRY = RetryConfig(max_retries=3, base_delay=0.5, max_delay=10.0)
OPUS_RETRY = RetryConfig(max_retries=2, base_delay=1.0, max_delay=30.0)

async def retry_with_backoff(
    func: Callable,
    config: RetryConfig,
    *args, **kwargs,
) -> T:
    """Execute function with exponential backoff retry."""
    last_exception = None

    for attempt in range(config.max_retries + 1):
        try:
            return await func(*args, **kwargs)
        except ModelAPIError as e:
            last_exception = e
            
            if e.status_code not in config.retryable_status_codes:
                raise  # Non-retryable error

            if attempt == config.max_retries:
                break  # Exhausted retries

            # Calculate delay with exponential backoff + jitter
            delay = min(
                config.base_delay * (config.exponential_base ** attempt),
                config.max_delay,
            )
            if config.jitter:
                delay = delay * (0.5 + random.random())

            # Special case: 429 with Retry-After header
            if e.status_code == 429 and e.retry_after:
                delay = max(delay, e.retry_after)

            await asyncio.sleep(delay)

    raise last_exception


class ModelAPIError(Exception):
    def __init__(self, status_code: int, message: str, retry_after: float = None):
        self.status_code = status_code
        self.message = message
        self.retry_after = retry_after
        super().__init__(f"Model API error {status_code}: {message}")
```

### 7.3 Circuit Breaker

```python
# core/circuit_breaker.py

import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"           # Normal operation
    OPEN = "open"               # Failing, rejecting requests
    HALF_OPEN = "half_open"     # Testing if service recovered

class CircuitBreaker:
    """
    Circuit breaker for model API calls.
    Opens after N failures in a window, re-tests periodically.
    """

    def __init__(
        self,
        name: str,
        failure_threshold: int = 5,     # failures before opening
        recovery_timeout: float = 60.0,  # seconds before half-open
        success_threshold: int = 2,      # successes before closing
        window_size: float = 120.0,      # failure counting window
    ):
        self.name = name
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.window_size = window_size
        
        self.state = CircuitState.CLOSED
        self.failures: list[float] = []  # timestamps
        self.last_failure_time: float = 0
        self.half_open_successes: int = 0

    def can_execute(self) -> bool:
        """Check if we should attempt a call."""
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_successes = 0
                return True
            return False
        
        # HALF_OPEN: allow limited requests
        return True

    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.half_open_successes += 1
            if self.half_open_successes >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failures.clear()

    def record_failure(self):
        now = time.time()
        self.failures.append(now)
        self.last_failure_time = now

        # Clean old failures outside window
        cutoff = now - self.window_size
        self.failures = [t for t in self.failures if t > cutoff]

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            return

        if len(self.failures) >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage
flash_breaker = CircuitBreaker("flash", failure_threshold=5, recovery_timeout=30)
opus_breaker = CircuitBreaker("opus", failure_threshold=3, recovery_timeout=60)
```

### 7.4 Fallback Matrix

| Primary | Fails | Fallback | Credit Impact | User Notice |
|---------|-------|----------|---------------|-------------|
| Flash | All retries fail | Opus (if credits) | Yes, -1 credit | Header: `X-Fallback: opus` |
| Flash | All retries fail | None (no credits) | No | 503 with ETA |
| Opus | All retries fail | Flash | Refund credit | Header: `X-Fallback: flash` + quality warning |
| Opus | Timeout > 60s | Flash (partial) | Refund credit | Degraded response notice |
| Both | Down | None | No | 503 + incident alert |
| Router | Classification fails | Flash (default) | No | Silent fallback |

### 7.5 Timeout Configuration

```python
# config/timeouts.py

TIMEOUTS = {
    "flash": {
        "connect": 5.0,          # seconds to establish connection
        "first_token": 10.0,     # seconds to first response token
        "total": 30.0,           # total response time
        "streaming_idle": 15.0,  # max gap between stream chunks
    },
    "opus": {
        "connect": 5.0,
        "first_token": 15.0,     # Opus is slower to start
        "total": 120.0,          # Allow longer for complex tasks
        "streaming_idle": 30.0,
    },
    "router": {
        "classification": 0.05,  # 50ms max for routing decision
    },
    "tool_execution": {
        "web_search": 10.0,
        "web_fetch": 15.0,
        "code_execution": 30.0,
        "default": 10.0,
    },
}
```

---

## 8. Observability

### 8.1 Three Pillars

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY STACK                            │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │    LOGGING     │  │    METRICS     │  │    TRACING             │   │
│  │               │  │               │  │                       │   │
│  │  Structured   │  │  Prometheus   │  │  OpenTelemetry        │   │
│  │  JSON logs    │  │  counters,    │  │  distributed traces   │   │
│  │  → Loki /     │  │  histograms,  │  │  → Jaeger / Tempo     │   │
│  │    stdout     │  │  gauges       │  │                       │   │
│  │               │  │  → Grafana    │  │  Request lifecycle    │   │
│  │  Every request│  │               │  │  with spans for each  │   │
│  │  with context │  │  Real-time    │  │  component             │   │
│  └───────────────┘  │  dashboards   │  └───────────────────────┘   │
│                     └───────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Structured Log Format

```python
# observability/logging.py

import structlog
import time
from contextlib import asynccontextmanager

logger = structlog.get_logger()

@asynccontextmanager
async def request_context(request_id: str, user_id: str):
    """Bind request context to all log entries in this scope."""
    bound = logger.bind(
        request_id=request_id,
        user_id=user_id,
    )
    start = time.monotonic()
    try:
        yield bound
    finally:
        elapsed = (time.monotonic() - start) * 1000
        bound.info("request.completed", latency_ms=round(elapsed, 2))

# Example log output (JSON):
# {
#   "timestamp": "2025-07-09T14:23:01.234Z",
#   "level": "info",
#   "event": "model.inference",
#   "request_id": "req_abc123",
#   "user_id": "usr_def456",
#   "model": "gemini-2.0-flash",
#   "routing_reason": "simple_explanation",
#   "complexity_score": 0.23,
#   "prompt_tokens": 142,
#   "completion_tokens": 389,
#   "latency_ms": 823,
#   "escalated": false,
#   "tool_calls": 0,
#   "status": "success"
# }
```

### 8.3 Key Metrics

```python
# observability/metrics.py

from prometheus_client import Counter, Histogram, Gauge

# === REQUEST METRICS ===

REQUEST_TOTAL = Counter(
    "clawdbot_requests_total",
    "Total requests",
    ["model", "tier", "status"],
)
# Labels: model=flash|opus, tier=starter|pro|business, status=success|error|timeout

REQUEST_LATENCY = Histogram(
    "clawdbot_request_latency_seconds",
    "Request latency in seconds",
    ["model", "tier"],
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0, 120.0],
)

ROUTING_DECISIONS = Counter(
    "clawdbot_routing_decisions_total",
    "Routing decisions by model and reason",
    ["model", "reason"],
)

# === ESCALATION METRICS ===

ESCALATIONS_TOTAL = Counter(
    "clawdbot_escalations_total",
    "Flash→Opus escalations",
    ["reason"],
)
# reason=low_confidence|user_request|tool_failure

ESCALATION_RATE = Gauge(
    "clawdbot_escalation_rate",
    "Rolling escalation rate (5min window)",
)

# === CREDIT METRICS ===

OPUS_CREDITS_CONSUMED = Counter(
    "clawdbot_opus_credits_consumed_total",
    "Opus credits consumed",
    ["tier", "reason"],
)
# reason=direct_route|escalation|user_override

OPUS_CREDITS_REMAINING = Gauge(
    "clawdbot_opus_credits_remaining",
    "Remaining Opus credits per user (sampled)",
    ["tier"],
)

CREDIT_EXHAUSTION_EVENTS = Counter(
    "clawdbot_credit_exhaustion_total",
    "Times users hit zero Opus credits",
    ["tier"],
)

# === COST METRICS ===

MODEL_COST_DOLLARS = Counter(
    "clawdbot_model_cost_dollars_total",
    "Estimated cost in USD by model",
    ["model"],
)

TOKEN_USAGE = Counter(
    "clawdbot_tokens_total",
    "Tokens consumed by model and direction",
    ["model", "direction"],  # direction=input|output
)

# === QUALITY METRICS ===

CONFIDENCE_SCORES = Histogram(
    "clawdbot_confidence_score",
    "Response confidence scores",
    ["model"],
    buckets=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0],
)

# === SYSTEM HEALTH ===

CIRCUIT_BREAKER_STATE = Gauge(
    "clawdbot_circuit_breaker_state",
    "Circuit breaker state (0=closed, 1=half_open, 2=open)",
    ["model"],
)

MODEL_API_ERRORS = Counter(
    "clawdbot_model_api_errors_total",
    "Model API errors by type",
    ["model", "error_type"],  # error_type=rate_limit|server_error|timeout
)
```

### 8.4 Dashboard Layout (Grafana)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CLAWDBOT OPERATIONS DASHBOARD                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│  │  Requests/min   │  │  P95 Latency    │  │  Error Rate     │            │
│  │     247 ▲12%    │  │  Flash: 0.8s    │  │    0.3% ✅      │            │
│  │                 │  │  Opus:  2.1s    │  │                 │            │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Request Volume (stacked area — Flash blue, Opus purple)           │    │
│  │  ████████████████████████████████████████████░░░░░░░░░░░           │    │
│  │  ████████████████████████████████████████████░░░░░░░░░░░░          │    │
│  │  ████████████████████████████████████████░░░░░░░░░░░░░░            │    │
│  │  ▲ 12:00   13:00   14:00   15:00   16:00   17:00                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌──────────────────────────┐  ┌──────────────────────────────────────┐    │
│  │  Model Split (donut)     │  │  Latency Distribution (histogram)    │    │
│  │                          │  │                                      │    │
│  │     ┌────────┐           │  │  Flash:  ▓▓▓▓▓▓▓▓▓▓▓▓░░░            │    │
│  │   ╔═╤════════╤═╗        │  │  Opus:   ░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓░         │    │
│  │   ║ │ 78% F  │ ║        │  │          0s   1s   2s   5s   10s     │    │
│  │   ║ │ 22% O  │ ║        │  │                                      │    │
│  │   ╚═╧════════╧═╝        │  │                                      │    │
│  └──────────────────────────┘  └──────────────────────────────────────┘    │
│                                                                             │
│  ┌──────────────────────────┐  ┌──────────────────────────────────────┐    │
│  │  Cost Today              │  │  Escalation Rate                     │    │
│  │                          │  │                                      │    │
│  │  Flash:  $12.34          │  │  Current: 4.2%                       │    │
│  │  Opus:   $48.91          │  │  ─────────── target: 5% ──────────  │    │
│  │  Total:  $61.25          │  │  ▓▓▓▓▓░░░░░░░░░░                    │    │
│  │  Revenue: $892.00        │  │                                      │    │
│  │  Margin:  93.1% ✅       │  │  Alert if > 10%                     │    │
│  └──────────────────────────┘  └──────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Credit Usage Heatmap (users approaching limits)                   │    │
│  │                                                                     │    │
│  │  Starter (200):  ████████░░ 156 users, 23 at >80%                  │    │
│  │  Pro (600):      ██████░░░░ 89 users, 12 at >80%                   │    │
│  │  Business (1500): ████░░░░░░ 34 users, 4 at >80%                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌──────────────────────────┐  ┌──────────────────────────────────────┐    │
│  │  Circuit Breakers        │  │  Active Alerts                       │    │
│  │                          │  │                                      │    │
│  │  Flash:  🟢 CLOSED       │  │  ⚠️  Opus P95 > 5s (14:22)         │    │
│  │  Opus:   🟢 CLOSED       │  │  ✅  Flash error rate normal        │    │
│  │  Router: 🟢 HEALTHY      │  │  ✅  All circuit breakers closed    │    │
│  └──────────────────────────┘  └──────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.5 Alert Rules

```yaml
# alerts/rules.yml (Prometheus alerting rules)

groups:
  - name: clawdbot_critical
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(clawdbot_requests_total{status="error"}[5m]))
          / sum(rate(clawdbot_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for 2 minutes"

      - alert: ModelDown
        expr: clawdbot_circuit_breaker_state > 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker OPEN for {{ $labels.model }}"

      - alert: HighOpusLatency
        expr: |
          histogram_quantile(0.95,
            rate(clawdbot_request_latency_seconds_bucket{model="opus"}[5m])
          ) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Opus P95 latency above 10s"

  - name: clawdbot_business
    rules:
      - alert: HighEscalationRate
        expr: clawdbot_escalation_rate > 0.10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Escalation rate above 10% — router may need tuning"

      - alert: CostAnomaly
        expr: |
          rate(clawdbot_model_cost_dollars_total[1h]) > 50
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Hourly model cost exceeding $50 — possible abuse or misconfiguration"

      - alert: MassCrediExhaustion
        expr: |
          sum(rate(clawdbot_credit_exhaustion_total[1h])) > 20
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "Many users hitting credit limits — consider plan adjustment"
```

### 8.6 Cost Tracking

```python
# observability/cost.py

# Token pricing (as of July 2025 — update periodically)
PRICING = {
    "gemini-2.0-flash": {
        "input_per_1m": 0.10,    # $0.10 per 1M input tokens
        "output_per_1m": 0.40,   # $0.40 per 1M output tokens
    },
    "claude-opus-4-5": {
        "input_per_1m": 15.00,   # $15.00 per 1M input tokens
        "output_per_1m": 75.00,  # $75.00 per 1M output tokens
    },
}

def estimate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    """Estimate cost in USD for a single request."""
    pricing = PRICING.get(model)
    if not pricing:
        return 0.0

    input_cost = (input_tokens / 1_000_000) * pricing["input_per_1m"]
    output_cost = (output_tokens / 1_000_000) * pricing["output_per_1m"]
    return round(input_cost + output_cost, 6)

# Example costs per request:
#   Flash: 1K in + 500 out = $0.0003  (~$0.03 per 100 requests)
#   Opus:  1K in + 500 out = $0.0525  (~$5.25 per 100 requests)
#
# At target mix (80% Flash / 20% Opus):
#   100 requests = 80 × $0.0003 + 20 × $0.0525 = $0.024 + $1.05 = $1.074
#   1000 requests/user/month → ~$10.74 cost per user
#
# Revenue per user (Pro @ $49):
#   Margin = $49 - $10.74 = $38.26 (78% gross margin) ✅
```

---

## 9. Tech Stack Recommendation

### 9.1 Stack Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        TECH STACK                                │
├──────────────┬───────────────────┬───────────────────────────────┤
│ Layer        │ Technology        │ Why                           │
├──────────────┼───────────────────┼───────────────────────────────┤
│ Language     │ Python 3.12+      │ Best AI/ML ecosystem, async   │
│ Framework    │ FastAPI           │ Async, OpenAPI, typing, fast  │
│ ASGI Server  │ Uvicorn           │ Production-ready, async       │
│ Database     │ PostgreSQL 16     │ Reliable, JSON support, CTEs  │
│ Cache/Queue  │ Redis 7+          │ Streams, Lua scripts, fast    │
│ Analytics    │ TimescaleDB       │ Time-series on PostgreSQL     │
│              │ (or ClickHouse)   │                               │
│ Auth         │ JWT (PyJWT)       │ Stateless, standard           │
│ Payments     │ Stripe            │ Webhooks, subscriptions       │
│ ORM          │ SQLAlchemy 2.0    │ Async support, migrations     │
│ Migrations   │ Alembic           │ Standard for SQLAlchemy       │
│ HTTP Client  │ httpx             │ Async, HTTP/2, streaming      │
│ Validation   │ Pydantic v2       │ Fast, built into FastAPI      │
│ Task Queue   │ Redis Streams     │ Simple, no extra infra (MVP)  │
│              │ (→ Celery v2)     │ Upgrade if needed             │
│ Monitoring   │ Prometheus        │ Industry standard             │
│ Dashboards   │ Grafana           │ Flexible, free                │
│ Logging      │ structlog → stdout│ JSON structured, 12-factor    │
│ Tracing      │ OpenTelemetry     │ Vendor-neutral                │
│ Containers   │ Docker + Compose  │ Simple deployment (MVP)       │
│ Orchestration│ K8s (v2)          │ Production scaling            │
│ Reverse Proxy│ Caddy             │ Auto-TLS, simple config       │
│ CDN/WAF      │ Cloudflare        │ Free tier works for MVP       │
│ CI/CD        │ GitHub Actions    │ Free for private repos        │
│ Hosting      │ Hetzner (MVP)     │ Cheap, EU-based               │
│              │ → GCP/AWS (v2)    │ Global scale                  │
└──────────────┴───────────────────┴───────────────────────────────┘
```

### 9.2 Python Dependencies

```toml
# pyproject.toml

[project]
name = "clawdbot-saas"
version = "0.1.0"
requires-python = ">=3.12"

dependencies = [
    # Web framework
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "websockets>=12.0",
    
    # Database
    "asyncpg>=0.29.0",
    "sqlalchemy[asyncio]>=2.0",
    "alembic>=1.13",
    
    # Cache & Queue
    "redis[hiredis]>=5.0",
    
    # AI Model APIs
    "anthropic>=0.40.0",
    "google-genai>=1.0.0",
    
    # HTTP client
    "httpx>=0.27",
    
    # Auth
    "pyjwt[crypto]>=2.8",
    "passlib[bcrypt]>=1.7",
    
    # Payments
    "stripe>=10.0",
    
    # Observability
    "prometheus-client>=0.20",
    "structlog>=24.0",
    "opentelemetry-api>=1.25",
    "opentelemetry-sdk>=1.25",
    "opentelemetry-instrumentation-fastapi>=0.46",
    
    # Utilities
    "pydantic>=2.8",
    "pydantic-settings>=2.4",
    "python-multipart>=0.0.9",
]
```

### 9.3 Docker Compose (MVP)

```yaml
# docker-compose.yml

version: "3.9"

services:
  # ─────────────────────────────────────────────
  # Application
  # ─────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://clawdbot:secret@postgres:5432/clawdbot
      - REDIS_URL=redis://redis:6379/0
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - GOOGLE_AI_API_KEY=${GOOGLE_AI_API_KEY}
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
      - JWT_SECRET=${JWT_SECRET}
      - ENVIRONMENT=production
      - LOG_LEVEL=info
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Background workers (credit updates, analytics)
  worker:
    build:
      context: .
      dockerfile: Dockerfile
    command: python -m clawdbot.workers.main
    environment:
      - DATABASE_URL=postgresql+asyncpg://clawdbot:secret@postgres:5432/clawdbot
      - REDIS_URL=redis://redis:6379/0
      - ENVIRONMENT=production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"

  # ─────────────────────────────────────────────
  # Data Stores
  # ─────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: clawdbot
      POSTGRES_USER: clawdbot
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations/init.sql:/docker-entrypoint-initdb.d/01-init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U clawdbot"]
      interval: 5s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512M

  redis:
    image: redis:7-alpine
    command: >
      redis-server
        --maxmemory 256mb
        --maxmemory-policy allkeys-lru
        --appendonly yes
        --appendfsync everysec
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 300M

  # ─────────────────────────────────────────────
  # Observability
  # ─────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.53.0
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./observability/alerts:/etc/prometheus/alerts
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
    deploy:
      resources:
        limits:
          memory: 256M

  grafana:
    image: grafana/grafana:11.1.0
    volumes:
      - grafana_data:/var/lib/grafana
      - ./observability/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./observability/grafana/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    deploy:
      resources:
        limits:
          memory: 256M

  # ─────────────────────────────────────────────
  # Reverse Proxy
  # ─────────────────────────────────────────────
  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
  caddy_data:
  caddy_config:
```

### 9.4 Dockerfile

```dockerfile
# Dockerfile

FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

FROM python:3.12-slim

WORKDIR /app

# Runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .

EXPOSE 8000

# Run with uvicorn
CMD ["uvicorn", "clawdbot.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "4", \
     "--loop", "uvloop", \
     "--http", "httptools", \
     "--access-log"]
```

### 9.5 Caddyfile

```
# Caddyfile

api.clawdbot.com {
    reverse_proxy api:8000

    # WebSocket support
    @websocket {
        header Connection *Upgrade*
        header Upgrade websocket
    }
    reverse_proxy @websocket api:8000

    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
    }

    # Rate limiting at edge (backup)
    rate_limit {
        zone api_zone {
            key {remote_host}
            events 100
            window 1m
        }
    }
}
```

### 9.6 FastAPI Application Entry Point

```python
# clawdbot/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
import asyncpg
import redis.asyncio as aioredis
import structlog

from clawdbot.api import chat, usage, auth, webhooks
from clawdbot.core.config import settings
from clawdbot.services.router import RequestRouter
from clawdbot.services.flash import FlashService
from clawdbot.services.opus import OpusService
from clawdbot.services.credits import CreditService
from clawdbot.tools.registry import create_tool_registry
from clawdbot.observability.metrics import setup_prometheus

logger = structlog.get_logger()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup/shutdown lifecycle."""
    logger.info("Starting Clawdbot SaaS API", environment=settings.ENVIRONMENT)

    # Initialize database pool
    app.state.pg_pool = await asyncpg.create_pool(
        dsn=settings.DATABASE_URL.replace("+asyncpg", ""),
        min_size=5,
        max_size=20,
    )

    # Initialize Redis
    app.state.redis = aioredis.from_url(
        settings.REDIS_URL,
        decode_responses=False,
    )

    # Initialize services
    app.state.router = RequestRouter()
    app.state.flash = FlashService(api_key=settings.GOOGLE_AI_API_KEY)
    app.state.opus = OpusService(api_key=settings.ANTHROPIC_API_KEY)
    app.state.credits = CreditService(
        redis_pool=app.state.redis,
        pg_pool=app.state.pg_pool,
    )
    app.state.tool_registry = create_tool_registry()

    logger.info("All services initialized")

    yield

    # Shutdown
    await app.state.pg_pool.close()
    await app.state.redis.close()
    logger.info("Shutdown complete")


app = FastAPI(
    title="Clawdbot SaaS API",
    version="1.0.0",
    lifespan=lifespan,
)

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Prometheus metrics endpoint
setup_prometheus(app)

# Routes
app.include_router(auth.router, prefix="/v1/auth", tags=["auth"])
app.include_router(chat.router, prefix="/v1", tags=["chat"])
app.include_router(usage.router, prefix="/v1", tags=["usage"])
app.include_router(webhooks.router, prefix="/webhooks", tags=["webhooks"])


@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.0.0"}


@app.get("/ready")
async def ready(request: Request):
    """Readiness check — verify all dependencies."""
    try:
        await request.app.state.pg_pool.fetchval("SELECT 1")
        await request.app.state.redis.ping()
        return {"status": "ready"}
    except Exception as e:
        return {"status": "not_ready", "error": str(e)}, 503
```

---

## 10. MVP vs v2 Roadmap

### 10.1 MVP (Weeks 1-6)

**Goal:** Working system with core routing, credits, and basic streaming. Deploy to first 10 beta users.

```
┌─────────────────────────────────────────────────────────────────┐
│  MVP — 6 WEEKS                                                  │
│                                                                 │
│  Week 1-2: Foundation                                           │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ ✅ FastAPI project skeleton                         │        │
│  │ ✅ PostgreSQL schema + migrations (Alembic)         │        │
│  │ ✅ Redis setup                                      │        │
│  │ ✅ Docker Compose (all services)                    │        │
│  │ ✅ User registration + JWT auth                     │        │
│  │ ✅ API key generation                               │        │
│  │ ✅ Stripe subscription integration                  │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 3-4: Core Intelligence                                    │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ ✅ Request Router (heuristic only — no ML)          │        │
│  │ ✅ Flash inference service (Gemini API)             │        │
│  │ ✅ Opus inference service (Anthropic API)           │        │
│  │ ✅ Unified tool adapter (Flash + Opus)              │        │
│  │ ✅ Basic tool calling loop (max 5 iterations)       │        │
│  │ ✅ Credit check + consumption (Redis + PG)          │        │
│  │ ✅ POST /v1/chat/completions (sync)                 │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 5: Streaming + Polish                                     │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ ✅ SSE streaming (both models)                      │        │
│  │ ✅ Buffer & Replace escalation (Strategy A)         │        │
│  │ ✅ Basic retry logic (3x with backoff)              │        │
│  │ ✅ Flash↔Opus fallback (basic)                      │        │
│  │ ✅ Usage endpoint (GET /v1/usage)                   │        │
│  │ ✅ Rate limiting (Redis sliding window)             │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 6: Deploy + Observe                                       │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ ✅ Prometheus metrics (basic set)                   │        │
│  │ ✅ Structured logging (structlog)                   │        │
│  │ ✅ Grafana dashboard (1 overview panel)             │        │
│  │ ✅ Deploy to Hetzner VPS                            │        │
│  │ ✅ Caddy reverse proxy + TLS                        │        │
│  │ ✅ Smoke tests + basic integration tests            │        │
│  │ ✅ Beta invite system (10 users)                    │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  DEFERRED from MVP:                                             │
│  ❌ ML-based classifier (use heuristics instead)                │
│  ❌ WebSocket endpoint                                          │
│  ❌ Confidence-based auto-escalation                            │
│  ❌ Circuit breaker pattern                                     │
│  ❌ Parallel Start escalation strategy                          │
│  ❌ Credit packs / overage billing                              │
│  ❌ Multi-region deployment                                     │
│  ❌ Admin dashboard UI                                          │
│  ❌ OpenTelemetry tracing                                       │
│  ❌ Conversation persistence (multi-turn memory)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 v2 (Weeks 7-14)

```
┌─────────────────────────────────────────────────────────────────┐
│  v2 — WEEKS 7-14                                                │
│                                                                 │
│  Week 7-8: Quality & Intelligence                               │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ 🔜 Train ML classifier on routing logs              │        │
│  │ 🔜 Confidence scoring on Flash responses            │        │
│  │ 🔜 Auto-escalation (Flash→Opus on low confidence)   │        │
│  │ 🔜 A/B testing framework (Flash vs Opus quality)    │        │
│  │ 🔜 Conversation persistence (Redis + PG)            │        │
│  │ 🔜 Multi-turn context management                    │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 9-10: Reliability & Scale                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ 🔜 Circuit breaker implementation                   │        │
│  │ 🔜 WebSocket endpoint (/v1/stream)                  │        │
│  │ 🔜 OpenTelemetry distributed tracing                │        │
│  │ 🔜 Full Grafana dashboard suite (4+ dashboards)     │        │
│  │ 🔜 Alerting rules (PagerDuty / Slack integration)   │        │
│  │ 🔜 Load testing (Locust, target: 500 req/s)        │        │
│  │ 🔜 Database read replica                            │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 11-12: Business Features                                  │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ 🔜 Credit packs (buy extra Opus credits)            │        │
│  │ 🔜 Soft overage (10% buffer, charge per-call)       │        │
│  │ 🔜 Usage analytics dashboard (customer-facing)      │        │
│  │ 🔜 Email notifications (80%/100% credit alerts)     │        │
│  │ 🔜 Admin panel (user management, analytics)         │        │
│  │ 🔜 Team/organization support                        │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Week 13-14: Scale Prep                                         │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ 🔜 Kubernetes migration (GKE or EKS)                │        │
│  │ 🔜 Horizontal auto-scaling                          │        │
│  │ 🔜 Redis Cluster (from standalone)                  │        │
│  │ 🔜 CDN for static assets (docs, dashboard)          │        │
│  │ 🔜 SOC 2 Type I preparation                        │        │
│  │ 🔜 Public launch readiness                          │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.3 v3+ (Future)

```
┌─────────────────────────────────────────────────────────────────┐
│  v3+ — FUTURE                                                   │
│                                                                 │
│  🔮 Fine-tuned routing model (trained on our traffic data)      │
│  🔮 User feedback loop (thumbs up/down → improve routing)       │
│  🔮 Model-specific prompt optimization                          │
│  🔮 Additional models (Claude Sonnet for mid-tier, Gemini Pro)  │
│  🔮 Self-hosted model option (for enterprise)                   │
│  🔮 Plugin / custom tool marketplace                            │
│  🔮 Multi-region deployment (US + EU + APAC)                    │
│  🔮 Enterprise tier (SSO, SLA, dedicated capacity)              │
│  🔮 Batch processing API (async, bulk jobs)                     │
│  🔮 Fine-tuning as a service (custom model per customer)        │
│  🔮 SOC 2 Type II certification                                │
│  🔮 HIPAA compliance tier                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.4 Decision Log

| Decision | Choice | Rationale | Revisit When |
|----------|--------|-----------|-------------|
| **Router** | Heuristics first | ML needs training data we don't have yet | 1K+ requests/day |
| **Streaming** | SSE first, WS later | SSE is simpler, covers 90% of use cases | Users ask for real-time |
| **Escalation** | Buffer & Replace | Simplest to implement correctly | Latency complaints |
| **Database** | Single PG instance | Overkill to shard at MVP scale | 100K+ users |
| **Queue** | Redis Streams | No extra infra, good enough for credit updates | 10K+ msg/sec |
| **Hosting** | Single Hetzner VPS | Cheapest, we're cost-constrained | 50+ concurrent users |
| **Auth** | JWT + API keys | Standard, stateless, both server + client use | Enterprise SSO needs |
| **Payments** | Stripe only | Industry standard, good webhooks | International expansion |
| **Analytics** | PG queries first | Avoid extra infra (ClickHouse) until needed | Query time > 5s |
| **CI/CD** | GitHub Actions | Free, integrated, sufficient | Need canary deploys |

---

## Appendices

### A. Project Structure

```
clawdbot-saas/
├── clawdbot/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app entry point
│   ├── api/
│   │   ├── __init__.py
│   │   ├── chat.py                # Chat completion endpoints
│   │   ├── usage.py               # Usage & billing endpoints
│   │   ├── auth.py                # Authentication endpoints
│   │   ├── webhooks.py            # Stripe webhooks
│   │   └── websocket.py           # WebSocket endpoint (v2)
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py              # Settings (env vars)
│   │   ├── deps.py                # FastAPI dependencies
│   │   ├── security.py            # JWT + API key validation
│   │   ├── retry.py               # Retry logic
│   │   └── circuit_breaker.py     # Circuit breaker
│   ├── models/                    # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── subscription.py
│   │   └── credit.py
│   ├── schemas/                   # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── chat.py
│   │   ├── usage.py
│   │   └── auth.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── router.py              # Request classification + routing
│   │   ├── flash.py               # Gemini 2.0 Flash service
│   │   ├── opus.py                # Claude Opus 4.5 service
│   │   ├── credits.py             # Credit management
│   │   ├── escalation.py          # Confidence checking + escalation
│   │   └── merger.py              # Response normalization
│   ├── streaming/
│   │   ├── __init__.py
│   │   ├── sse.py                 # SSE event formatting
│   │   └── buffer.py              # Buffer & Replace strategy
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── schema.py              # Unified tool definitions
│   │   ├── registry.py            # Tool registry
│   │   ├── adapters.py            # Gemini + Claude adapters
│   │   ├── loop.py                # Tool execution loop
│   │   └── handlers/              # Individual tool handlers
│   │       ├── web_search.py
│   │       ├── web_fetch.py
│   │       └── code_exec.py
│   ├── workers/
│   │   ├── __init__.py
│   │   ├── main.py                # Worker entry point
│   │   ├── credit_worker.py       # Credit update consumer
│   │   └── analytics_worker.py    # Usage analytics processor
│   └── observability/
│       ├── __init__.py
│       ├── metrics.py             # Prometheus metrics
│       ├── logging.py             # Structured logging setup
│       └── tracing.py             # OpenTelemetry setup (v2)
├── migrations/
│   ├── alembic.ini
│   ├── env.py
│   ├── init.sql
│   └── versions/
│       └── 001_initial.py
├── observability/
│   ├── prometheus.yml
│   ├── alerts/
│   │   └── rules.yml
│   └── grafana/
│       ├── dashboards/
│       │   └── overview.json
│       └── datasources/
│           └── prometheus.yml
├── tests/
│   ├── conftest.py
│   ├── test_router.py
│   ├── test_credits.py
│   ├── test_streaming.py
│   └── test_tools.py
├── docker-compose.yml
├── Dockerfile
├── Caddyfile
├── pyproject.toml
├── .env.example
└── README.md
```

### B. Environment Variables

```bash
# .env.example

# Database
DATABASE_URL=postgresql+asyncpg://clawdbot:secret@localhost:5432/clawdbot

# Redis
REDIS_URL=redis://localhost:6379/0

# AI Model APIs
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxx
GOOGLE_AI_API_KEY=AIzaxxxxxxxxxx

# Auth
JWT_SECRET=your-256-bit-secret-here
JWT_ALGORITHM=ES256
JWT_EXPIRY_MINUTES=60

# Stripe
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxx
STRIPE_STARTER_PRICE_ID=price_xxxxxxxxxx
STRIPE_PRO_PRICE_ID=price_xxxxxxxxxx
STRIPE_BUSINESS_PRICE_ID=price_xxxxxxxxxx

# Application
ENVIRONMENT=development
LOG_LEVEL=debug
CORS_ORIGINS=["http://localhost:3000","https://app.clawdbot.com"]

# Routing
ROUTER_COMPLEXITY_THRESHOLD=0.4
ROUTER_BUFFER_TOKENS=50
ROUTER_CONFIDENCE_THRESHOLD=0.5

# Observability
PROMETHEUS_ENABLED=true
OTEL_ENABLED=false
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### C. Quick-Start Commands

```bash
# Clone and setup
git clone https://github.com/clawdbot/saas.git
cd saas
cp .env.example .env
# Edit .env with your API keys

# Start everything
docker compose up -d

# Run migrations
docker compose exec api alembic upgrade head

# Check health
curl http://localhost:8000/health
curl http://localhost:8000/ready

# View logs
docker compose logs -f api

# Run tests
docker compose exec api pytest tests/ -v

# Access Grafana
open http://localhost:3000  # admin/admin

# Access Prometheus
open http://localhost:9090
```

### D. Cost Model Spreadsheet

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           UNIT ECONOMICS PER USER/MONTH                             │
├──────────────────────────┬──────────────┬──────────────┬──────────────────────────────┤
│                          │ Starter $19  │ Pro $49      │ Business $99               │
├──────────────────────────┼──────────────┼──────────────┼──────────────────────────────┤
│ Opus credits             │ 200          │ 600          │ 1,500                      │
│ Est. Flash requests      │ 500          │ 1,500        │ 4,000                      │
│ Est. total requests      │ 700          │ 2,100        │ 5,500                      │
│ Opus % of total          │ 28.6%        │ 28.6%        │ 27.3%                      │
├──────────────────────────┼──────────────┼──────────────┼──────────────────────────────┤
│ Flash cost               │              │              │                            │
│  Avg tokens: 1K in/500out│              │              │                            │
│  Per req: $0.0003        │ $0.15        │ $0.45        │ $1.20                      │
├──────────────────────────┼──────────────┼──────────────┼──────────────────────────────┤
│ Opus cost                │              │              │                            │
│  Avg tokens: 1.5K in/1K │              │              │                            │
│  Per req: $0.0975        │ $19.50       │ $58.50       │ $146.25                    │
├──────────────────────────┼──────────────┼──────────────┼──────────────────────────────┤
│ Total model cost         │ $19.65       │ $58.95       │ $147.45                    │
│ Revenue                  │ $19.00       │ $49.00       │ $99.00                     │
│ Gross margin             │ -$0.65 ❌    │ -$9.95 ❌    │ -$48.45 ❌                 │
├──────────────────────────┴──────────────┴──────────────┴──────────────────────────────┤
│                                                                                      │
│  ⚠️  IF EVERY USER MAXES OUT OPUS CREDITS, WE LOSE MONEY.                           │
│                                                                                      │
│  REALITY CHECK: Most users won't max out. Assume 40-60% utilization:                │
│                                                                                      │
├──────────────────────────┬──────────────┬──────────────┬──────────────────────────────┤
│ At 50% Opus utilization  │ Starter $19  │ Pro $49      │ Business $99               │
├──────────────────────────┼──────────────┼──────────────┼──────────────────────────────┤
│ Opus calls used          │ 100          │ 300          │ 750                        │
│ Flash calls              │ 500          │ 1,500        │ 4,000                      │
│ Flash cost               │ $0.15        │ $0.45        │ $1.20                      │
│ Opus cost                │ $9.75        │ $29.25       │ $73.13                     │
│ Total cost               │ $9.90        │ $29.70       │ $74.33                     │
│ Revenue                  │ $19.00       │ $49.00       │ $99.00                     │
│ Gross margin             │ $9.10 (48%)  │ $19.30 (39%) │ $24.67 (25%)              │
├──────────────────────────┴──────────────┴──────────────┴──────────────────────────────┤
│                                                                                      │
│  LEVERS TO IMPROVE MARGINS:                                                          │
│  1. Lower Opus token usage via prompt optimization / caching                         │
│  2. Improve router to avoid unnecessary Opus calls                                   │
│  3. Use Anthropic's batch API (50% discount) for non-streaming                       │
│  4. Negotiate volume pricing as we scale                                             │
│  5. Add higher-priced Enterprise tier with better margins                            │
│  6. Cache common Opus responses (semantic dedup)                                     │
│  7. Prompt compression to reduce input tokens                                        │
│                                                                                      │
│  TARGET: 50%+ gross margin at scale with 40-50% Opus utilization                    │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

*Document authored for the Clawdbot-as-a-Service project. This is a living document — update as architecture decisions evolve.*