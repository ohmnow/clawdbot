# Deep Dive: Prompt Caching Strategy for Claude Opus 4.5

> **Context:** Hybrid SaaS architecture — Gemini 2.0 Flash handles 70-85% of requests (tool calls, simple actions), Claude Opus 4.5 handles 15-30% (complex reasoning). This document focuses on minimizing Opus costs through Anthropic's prompt caching.

---

## Table of Contents

1. [How Anthropic Prompt Caching Works](#1-how-anthropic-prompt-caching-works)
2. [What to Cache in Our Architecture](#2-what-to-cache-in-our-architecture)
3. [Cache Hierarchy Design](#3-cache-hierarchy-design)
4. [Concrete Cost Savings Math](#4-concrete-cost-savings-math)
5. [Implementation Patterns](#5-implementation-patterns)
6. [Cache Warming Strategies](#6-cache-warming-strategies)
7. [Gotchas and Limitations](#7-gotchas-and-limitations)
8. [Decision Framework](#8-decision-framework)

---

## 1. How Anthropic Prompt Caching Works

### Core Mechanism

Anthropic's prompt caching stores **prefixes** of your prompt on Anthropic's servers. When a subsequent request shares the same prefix, the cached portion is read from cache instead of being reprocessed — saving both latency and cost.

Key concept: **caching is prefix-based**. The cache matches from the beginning of the prompt forward. If byte `N` differs from the cached version, everything from byte `N` onward is a cache miss. This is fundamentally different from hash-based caching — **order matters absolutely**.

### Pricing (Claude Opus 4.5)

| Token Type | Cost per 1M Tokens | Relative to Base |
|---|---|---|
| **Base input** (no caching) | $5.00 | 1.0x |
| **Cache write** (first request) | $6.25 | 1.25x (25% MORE expensive) |
| **Cache read** (subsequent) | $0.50 | 0.10x (90% savings) |
| **Output** (always same) | $25.00 | — |

### TTL (Time-to-Live)

- **5 minutes** from last access (not from creation)
- Each cache **read** resets the TTL — the cache stays alive as long as it's being used
- After 5 minutes of no access, the cache is evicted
- No way to manually invalidate or extend TTL
- No way to query cache state (you only know from the response's `cache_creation_input_tokens` vs `cache_read_input_tokens` fields)

### Cache Breakpoints

You control caching by inserting `cache_control` markers in your prompt. These tell Anthropic: "everything up to this point should be cached as a unit."

```
┌──────────────────────────────┐
│ System prompt (1500 tokens)  │
│ cache_control: ephemeral ◄───┤ Breakpoint 1
├──────────────────────────────┤
│ Tool definitions (500 tokens)│
│ cache_control: ephemeral ◄───┤ Breakpoint 2
├──────────────────────────────┤
│ User context (500 tokens)    │
│ cache_control: ephemeral ◄───┤ Breakpoint 3
├──────────────────────────────┤
│ Conversation + query (500+)  │ ← Never cached (changes each request)
└──────────────────────────────┘
```

**Maximum breakpoints:** You can set up to **4** `cache_control` breakpoints per request.

### Minimum Token Requirements

- **Minimum cacheable prefix:** 1,024 tokens (for Claude Opus / Sonnet models)
  - Haiku has a lower minimum of 2,048 tokens (counterintuitively higher)
- The prefix up to a `cache_control` breakpoint must be **at least 1,024 tokens** to be eligible for caching
- If the prefix is below 1,024 tokens, the `cache_control` marker is silently ignored — no error, no caching, and you're charged standard input pricing

### How Cache Matching Works

1. Request arrives at Anthropic
2. Anthropic checks if the byte sequence from the start of the prompt up to any `cache_control` breakpoint matches an existing cache
3. Matching is **exact, byte-for-byte** — a single character change anywhere in the prefix invalidates the cache
4. The longest matching cached prefix is used
5. Tokens after the cached prefix are processed normally (charged at base input rate)

### What Can Be Cached

Cacheable content types:
- **System messages** (`system` field in the API)
- **Tool definitions** (`tools` field in the API)
- **Messages** (entries in the `messages` array — both `user` and `assistant` turns)

The cache operates across these in order: `system` → `tools` → `messages`. The prefix must be contiguous from the start.

---

## 2. What to Cache in Our Architecture

### Content Analysis

| Content | Size (est.) | Change Frequency | Shared Across Users | Cache Priority |
|---|---|---|---|---|
| System prompt | ~1,500 tokens | Rarely (deploy-time) | Yes (all users) | **Highest** |
| Tool definitions | ~500 tokens | Rarely (deploy-time) | Yes (all users) | **High** |
| User profile/memory | ~500 tokens | Per-session or daily | No (per-user) | **Medium** |
| Conversation history | Varies | Every turn | No (per-session) | **Low** (but valuable for multi-turn) |
| Current query | ~100-500 tokens | Every request | No | **Never cache** |

### System Prompt (~1,500 tokens)

This is our highest-value caching target. It's identical across all users and changes only on deployment.

```
You are an AI assistant for [Product Name]. You help users with...

## Capabilities
- Data analysis and visualization
- Code generation and debugging
- Document drafting and editing
...

## Behavioral Guidelines
- Always cite sources when making factual claims
- Ask clarifying questions when the request is ambiguous
- Format responses using markdown
...

## Response Format
...
```

**Why it's ideal:** Shared across every Opus request. At 1,500 tokens, it meets the 1,024-token minimum on its own. Every Opus request benefits.

### Tool Definitions (~500 tokens)

```json
{
  "name": "search_knowledge_base",
  "description": "Search the company knowledge base for relevant articles",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {"type": "string"},
      "filters": {...}
    }
  }
}
```

**Caching strategy:** Bundle with the system prompt. Combined (1,500 + 500 = 2,000 tokens), they form a stable prefix that's identical for every user.

### User Profile / Memory (~500 tokens)

```
## User Profile
- Name: Christopher
- Role: Senior Engineer
- Preferences: Concise responses, code-first, prefers Python
- Past topics: Kubernetes deployment, CI/CD pipelines

## Active Project Context
- Working on: API gateway migration
- Tech stack: FastAPI, PostgreSQL, Redis
- Current sprint goals: ...
```

**Caching strategy:** This is per-user, so it creates per-user cache entries. Cache breakpoint after user context means the system prompt + tools prefix can be shared across ALL users, while the user-specific portion creates a separate (but reusable within a session) cache.

### Conversation History (variable)

Multi-turn conversations benefit from caching because each new turn only adds to the end — the prefix (all previous turns) stays identical.

```
Turn 1: [system + tools + user_ctx + msg1]          → all cache write
Turn 2: [system + tools + user_ctx + msg1 + msg2]   → prefix cache read, msg2 is new
Turn 3: [system + tools + user_ctx + msg1..3 + msg4] → prefix cache read, msg4 is new
```

**This is where caching shines in multi-turn conversations** — you only pay cache-read prices for the growing prefix.

---

## 3. Cache Hierarchy Design

### The Four-Layer Cache Architecture

```
Layer 0 (Global Static)     — System prompt          — 1,500 tokens — Changes: per deploy
  ↓ cache_control breakpoint
Layer 1 (Global Semi-Static) — Tool definitions       — 500 tokens   — Changes: per deploy
  ↓ cache_control breakpoint  
Layer 2 (Per-User)           — User profile + memory  — 500 tokens   — Changes: per session
  ↓ cache_control breakpoint
Layer 3 (Per-Session)        — Conversation history   — 0-4000+ tokens — Changes: per turn
  ↓ (no breakpoint — this is the "tail")
Layer 4 (Ephemeral)          — Current query          — 100-500 tokens — Always new
```

### Why This Order?

The prefix-based nature of caching means the most stable content must come first:

```
MOST STABLE ──────────────────────────────────── LEAST STABLE
System Prompt → Tools → User Context → History → Query

If you put user context BEFORE tools, you'd need a separate
cache for every user even for the tools portion. By putting
shared content first, all users benefit from the same cache.
```

### Cache Sharing Matrix

| Request Type | L0 (System) | L1 (Tools) | L2 (User) | L3 (History) |
|---|---|---|---|---|
| New user, first message | WRITE | WRITE | WRITE | — |
| Same user, second message | READ ✓ | READ ✓ | READ ✓ | — |
| Same user, third message | READ ✓ | READ ✓ | READ ✓ | READ (turns 1-2) ✓ |
| Different user, first message | READ ✓ | READ ✓ | WRITE | — |
| After 5min idle, any user | WRITE | WRITE | WRITE | — |

### Practical Breakpoint Strategy

Given the 4-breakpoint maximum and 1,024-token minimum:

**Option A: Three Breakpoints (Recommended)**

```
Breakpoint 1: After system prompt (1,500 tokens) ← Meets 1,024 minimum ✓
Breakpoint 2: After system + tools (2,000 tokens cumulative) ← Meets minimum ✓
Breakpoint 3: After system + tools + user context (2,500 tokens cumulative) ← Meets minimum ✓
No breakpoint on conversation history (it's the variable tail)
```

**Option B: Two Breakpoints (Simpler)**

```
Breakpoint 1: After system + tools combined (2,000 tokens) ← Shared across all users
Breakpoint 2: After user context (2,500 tokens cumulative) ← Per-user cache
```

This is simpler and often sufficient. The system prompt and tools almost always change together (on deploy), so caching them as one unit is fine.

**Option C: Multi-turn Optimization (Four Breakpoints)**

For long conversations, add a breakpoint on the conversation history:

```
Breakpoint 1: After system prompt
Breakpoint 2: After tools
Breakpoint 3: After user context
Breakpoint 4: After conversation history (before the latest user message)
```

This is most aggressive and maximizes cache hits in multi-turn scenarios.

---

## 4. Concrete Cost Savings Math

### Baseline Assumptions

| Component | Tokens |
|---|---|
| System prompt | 1,500 |
| Tool definitions | 500 |
| User context | 500 |
| Current query | 500 |
| **Total input per request** | **3,000** |
| Average output | 800 |

### Scenario 1: No Caching (Baseline)

```
Input:  3,000 tokens × $5.00/M  = $0.0150
Output:   800 tokens × $25.00/M = $0.0200
─────────────────────────────────────────
Total per request:                $0.0350
```

### Scenario 2: First Request (Cache Write — Cold Start)

Caching the system prompt + tools + user context (2,500 tokens):

```
Cache write:  2,500 tokens × $6.25/M  = $0.015625  (25% more than uncached!)
Uncached input: 500 tokens × $5.00/M  = $0.002500  (the query)
Output:         800 tokens × $25.00/M = $0.020000
──────────────────────────────────────────────────
Total per request:                      $0.038125   (8.9% MORE than no caching)
```

**The first request is always more expensive with caching.** This is the cost of warming the cache.

### Scenario 3: Subsequent Requests (Cache Hit)

```
Cache read:   2,500 tokens × $0.50/M  = $0.001250  (90% savings!)
Uncached input: 500 tokens × $5.00/M  = $0.002500  (the query)
Output:         800 tokens × $25.00/M = $0.020000
──────────────────────────────────────────────────
Total per request:                      $0.023750
```

### Break-Even Analysis

```
Cache write cost:     $0.015625 (for 2,500 tokens)
Uncached cost:        $0.012500 (for same 2,500 tokens at base rate)
Cache write premium:  $0.003125 (the extra cost of the first request)

Per-request savings (cache hit): $0.012500 - $0.001250 = $0.011250

Break-even: $0.003125 / $0.011250 = 0.28 requests
```

**You break even after just 1.28 requests.** The second cache-hit request has already saved you more than the write premium cost. Caching is profitable starting from the second request.

### Monthly Cost Projection

Assumptions:
- 10,000 Opus requests/day (15-30% of total traffic)
- Average 3 requests per user session
- 70% cache hit rate (accounting for TTL expiry, new users, etc.)

**Without caching:**
```
10,000 requests × $0.0350 × 30 days = $10,500/month
```

**With caching:**
```
Cache misses (30%):  3,000 × $0.038125 = $114.38/day
Cache hits (70%):    7,000 × $0.023750 = $166.25/day
Daily total:                             $280.63/day
Monthly total:       $280.63 × 30       = $8,418.75/month

Monthly savings:     $10,500 - $8,418.75 = $2,081.25/month (19.8% reduction)
```

### Multi-Turn Conversation Savings

In a 5-turn conversation (each turn adds ~300 tokens to history):

| Turn | Cached Tokens | Uncached Tokens | Input Cost | vs. No Cache |
|---|---|---|---|---|
| 1 | 0 (write 2,500) | 500 | $0.01813 | +$0.00313 |
| 2 | 2,500 (read) | 800 | $0.00525 | −$0.00875 |
| 3 | 3,100 (read) | 800 | $0.00555 | −$0.01395 |
| 4 | 3,700 (read) | 800 | $0.00585 | −$0.01915 |
| 5 | 4,300 (read) | 800 | $0.00615 | −$0.02435 |
| **Total** | | | **$0.04093** | **−$0.06307** |

Without caching, the same 5-turn conversation would cost $0.10400 in input alone.
**With caching: $0.04093 — a 60.6% reduction in input costs.**

> **Key insight:** Multi-turn conversations are where caching delivers the biggest ROI. The longer the conversation, the more you save.

---

## 5. Implementation Patterns

### Pattern 1: Basic Caching with System Prompt + Tools

```python
import anthropic

client = anthropic.Anthropic()

def call_opus_with_caching(
    user_message: str,
    conversation_history: list[dict] | None = None,
) -> str:
    """Basic Opus call with prompt caching on system prompt and tools."""
    
    response = client.messages.create(
        model="claude-opus-4-5-20250229",
        max_tokens=4096,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,  # ~1,500 tokens
                "cache_control": {"type": "ephemeral"},  # ← Breakpoint 1
            }
        ],
        tools=[
            {
                "name": "search_knowledge_base",
                "description": "Search the company knowledge base",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "Search query"},
                        "limit": {"type": "integer", "description": "Max results"},
                    },
                    "required": ["query"],
                },
                "cache_control": {"type": "ephemeral"},  # ← Breakpoint 2 (on LAST tool)
            }
        ],
        messages=[
            *(conversation_history or []),
            {"role": "user", "content": user_message},
        ],
    )
    
    # Log cache performance
    usage = response.usage
    print(f"Cache write: {usage.cache_creation_input_tokens} tokens")
    print(f"Cache read:  {usage.cache_read_input_tokens} tokens")
    print(f"Uncached:    {usage.input_tokens} tokens")
    
    return response.content[0].text
```

### Pattern 2: Full Four-Layer Caching

```python
import anthropic
from dataclasses import dataclass

client = anthropic.Anthropic()

@dataclass
class UserContext:
    name: str
    role: str
    preferences: str
    memory: str
    
    def to_text(self) -> str:
        return f"""## User Profile
- Name: {self.name}
- Role: {self.role}
- Preferences: {self.preferences}

## Memory
{self.memory}"""


# Static content — defined once, shared across all users
SYSTEM_PROMPT = """You are an AI assistant for Acme Corp..."""  # ~1,500 tokens

TOOLS = [
    {
        "name": "search_knowledge_base",
        "description": "Search the company knowledge base for relevant articles and documentation.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "The search query"},
                "category": {
                    "type": "string",
                    "enum": ["engineering", "product", "support", "general"],
                },
                "limit": {"type": "integer", "description": "Maximum results to return"},
            },
            "required": ["query"],
        },
    },
    {
        "name": "run_sql_query",
        "description": "Execute a read-only SQL query against the analytics database.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "SQL query to execute"},
                "database": {"type": "string", "enum": ["analytics", "reporting"]},
            },
            "required": ["query"],
        },
    },
    # ... more tools ...
]

# Mark the LAST tool with cache_control
TOOLS[-1]["cache_control"] = {"type": "ephemeral"}


def call_opus_cached(
    user_message: str,
    user_context: UserContext,
    conversation_history: list[dict] | None = None,
) -> anthropic.types.Message:
    """
    Full four-layer cached Opus call.
    
    Cache layers:
      1. System prompt (cache_control on system message)
      2. Tool definitions (cache_control on last tool)
      3. User context (cache_control on first user message containing context)
      4. Conversation history grows naturally as a cached prefix
    """
    
    messages = []
    
    # Layer 3: User context as the first message
    # This creates a per-user cache that builds on the shared system+tools cache
    if user_context:
        messages.append({
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": f"<user_context>\n{user_context.to_text()}\n</user_context>",
                    "cache_control": {"type": "ephemeral"},  # ← Breakpoint 3
                }
            ],
        })
        messages.append({
            "role": "assistant",
            "content": "Understood. I have your context loaded.",
        })
    
    # Layer 4: Conversation history
    if conversation_history:
        # Add cache_control to the last message in history for multi-turn caching
        history = _deep_copy_messages(conversation_history)
        _add_cache_control_to_last_message(history)  # ← Breakpoint 4
        messages.extend(history)
    
    # Current query (never cached — always new)
    messages.append({
        "role": "user",
        "content": user_message,
    })
    
    response = client.messages.create(
        model="claude-opus-4-5-20250229",
        max_tokens=4096,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},  # ← Breakpoint 1
            }
        ],
        tools=TOOLS,  # Breakpoint 2 already on last tool
        messages=messages,
    )
    
    _log_cache_metrics(response.usage)
    return response


def _add_cache_control_to_last_message(messages: list[dict]):
    """Add cache_control to the last message in the history."""
    if not messages:
        return
    last = messages[-1]
    content = last.get("content")
    if isinstance(content, str):
        last["content"] = [
            {
                "type": "text",
                "text": content,
                "cache_control": {"type": "ephemeral"},
            }
        ]
    elif isinstance(content, list):
        # Add cache_control to the last content block
        last_block = content[-1]
        if isinstance(last_block, dict):
            last_block["cache_control"] = {"type": "ephemeral"}


def _deep_copy_messages(messages: list[dict]) -> list[dict]:
    """Deep copy messages to avoid mutating the original."""
    import copy
    return copy.deepcopy(messages)


def _log_cache_metrics(usage):
    """Log cache performance metrics."""
    total_input = (
        usage.input_tokens
        + getattr(usage, "cache_creation_input_tokens", 0)
        + getattr(usage, "cache_read_input_tokens", 0)
    )
    cache_hit_rate = (
        getattr(usage, "cache_read_input_tokens", 0) / total_input * 100
        if total_input > 0 else 0
    )
    
    print(f"📊 Cache metrics:")
    print(f"   Cache write:    {getattr(usage, 'cache_creation_input_tokens', 0):,} tokens")
    print(f"   Cache read:     {getattr(usage, 'cache_read_input_tokens', 0):,} tokens")
    print(f"   Uncached input: {usage.input_tokens:,} tokens")
    print(f"   Output:         {usage.output_tokens:,} tokens")
    print(f"   Cache hit rate: {cache_hit_rate:.1f}%")
```

### Pattern 3: Session Manager with Automatic Cache Management

```python
import time
import anthropic
from dataclasses import dataclass, field

client = anthropic.Anthropic()

CACHE_TTL_SECONDS = 300  # 5 minutes


@dataclass
class CachedSession:
    """Manages a user's session with cache-aware request handling."""
    
    user_id: str
    user_context: UserContext
    conversation_history: list[dict] = field(default_factory=list)
    last_request_time: float = 0.0
    total_cache_read_tokens: int = 0
    total_cache_write_tokens: int = 0
    total_uncached_tokens: int = 0
    request_count: int = 0
    
    @property
    def cache_is_likely_warm(self) -> bool:
        """Estimate whether the cache is still alive based on TTL."""
        if self.last_request_time == 0:
            return False
        elapsed = time.time() - self.last_request_time
        return elapsed < CACHE_TTL_SECONDS
    
    @property
    def estimated_savings(self) -> float:
        """Estimate dollar savings from caching in this session."""
        # What we would have paid without caching
        uncached_cost = (
            (self.total_cache_read_tokens + self.total_cache_write_tokens + self.total_uncached_tokens)
            * 5.00 / 1_000_000
        )
        # What we actually paid
        actual_cost = (
            self.total_cache_write_tokens * 6.25 / 1_000_000
            + self.total_cache_read_tokens * 0.50 / 1_000_000
            + self.total_uncached_tokens * 5.00 / 1_000_000
        )
        return uncached_cost - actual_cost

    def send_message(self, user_message: str) -> str:
        """Send a message with full cache management."""
        
        response = call_opus_cached(
            user_message=user_message,
            user_context=self.user_context,
            conversation_history=self.conversation_history,
        )
        
        # Update session state
        self.last_request_time = time.time()
        self.request_count += 1
        
        # Track cache metrics
        usage = response.usage
        self.total_cache_read_tokens += getattr(usage, "cache_read_input_tokens", 0)
        self.total_cache_write_tokens += getattr(usage, "cache_creation_input_tokens", 0)
        self.total_uncached_tokens += usage.input_tokens
        
        # Append to conversation history
        self.conversation_history.append({
            "role": "user",
            "content": user_message,
        })
        
        assistant_text = response.content[0].text
        self.conversation_history.append({
            "role": "assistant",
            "content": assistant_text,
        })
        
        return assistant_text
    
    def print_session_stats(self):
        """Print session-level caching statistics."""
        print(f"\n📈 Session stats for user {self.user_id}:")
        print(f"   Requests:         {self.request_count}")
        print(f"   Cache warm:       {'Yes' if self.cache_is_likely_warm else 'No'}")
        print(f"   Total cache read: {self.total_cache_read_tokens:,} tokens")
        print(f"   Total cache write:{self.total_cache_write_tokens:,} tokens")
        print(f"   Total uncached:   {self.total_uncached_tokens:,} tokens")
        print(f"   Est. savings:     ${self.estimated_savings:.4f}")


# Usage example
async def handle_user_request(user_id: str, message: str):
    """Example request handler showing session-based caching."""
    
    # Get or create session (from your session store)
    session = get_or_create_session(user_id)
    
    # If cache is cold, we might want to warm it (see Section 6)
    if not session.cache_is_likely_warm and should_prewarm(user_id):
        await warm_cache(session)
    
    # Send message — caching happens automatically
    response = session.send_message(message)
    
    return response
```

### Pattern 4: Monitoring and Observability

```python
import logging
from collections import defaultdict
from datetime import datetime, timedelta

logger = logging.getLogger("opus_cache")

class CacheMetricsCollector:
    """Collect and report cache performance metrics."""
    
    def __init__(self):
        self.metrics = defaultdict(lambda: {
            "requests": 0,
            "cache_read_tokens": 0,
            "cache_write_tokens": 0,
            "uncached_tokens": 0,
            "output_tokens": 0,
        })
    
    def record(self, usage, bucket: str = "default"):
        """Record metrics from a response's usage object."""
        m = self.metrics[bucket]
        m["requests"] += 1
        m["cache_read_tokens"] += getattr(usage, "cache_read_input_tokens", 0)
        m["cache_write_tokens"] += getattr(usage, "cache_creation_input_tokens", 0)
        m["uncached_tokens"] += usage.input_tokens
        m["output_tokens"] += usage.output_tokens
    
    def report(self, bucket: str = "default") -> dict:
        """Generate a cost report for a bucket."""
        m = self.metrics[bucket]
        
        actual_input_cost = (
            m["cache_write_tokens"] * 6.25 / 1_000_000
            + m["cache_read_tokens"] * 0.50 / 1_000_000
            + m["uncached_tokens"] * 5.00 / 1_000_000
        )
        
        baseline_input_cost = (
            (m["cache_write_tokens"] + m["cache_read_tokens"] + m["uncached_tokens"])
            * 5.00 / 1_000_000
        )
        
        output_cost = m["output_tokens"] * 25.00 / 1_000_000
        
        total_input = m["cache_write_tokens"] + m["cache_read_tokens"] + m["uncached_tokens"]
        cache_hit_rate = m["cache_read_tokens"] / total_input if total_input > 0 else 0
        
        return {
            "requests": m["requests"],
            "cache_hit_rate": f"{cache_hit_rate:.1%}",
            "actual_input_cost": f"${actual_input_cost:.4f}",
            "baseline_input_cost": f"${baseline_input_cost:.4f}",
            "input_savings": f"${baseline_input_cost - actual_input_cost:.4f}",
            "input_savings_pct": f"{(1 - actual_input_cost / baseline_input_cost) * 100:.1f}%" if baseline_input_cost > 0 else "N/A",
            "output_cost": f"${output_cost:.4f}",
            "total_cost": f"${actual_input_cost + output_cost:.4f}",
        }


# Global collector
metrics = CacheMetricsCollector()

# After each Opus call:
# metrics.record(response.usage, bucket="production")
# 
# Periodic reporting:
# report = metrics.report("production")
# logger.info(f"Cache report: {report}")
```

---

## 6. Cache Warming Strategies

### Why Warm the Cache?

The first request after a cache miss costs 25% more than an uncached request ($6.25/M vs $5.00/M). If you know a user is about to need Opus, you can absorb this cost proactively so their actual request benefits from cache-read pricing.

### Strategy 1: Warm on Session Start

When a user logs in or starts a session, fire a minimal Opus request to warm the cache:

```python
async def warm_cache_on_login(user_context: UserContext):
    """
    Fire a minimal request to warm the cache layers.
    
    This costs ~$0.038 (cache write on 2,500 tokens + minimal output)
    but saves ~$0.011 on every subsequent request in the session.
    Break-even: 4 Opus requests in the session.
    """
    response = client.messages.create(
        model="claude-opus-4-5-20250229",
        max_tokens=1,  # Minimize output cost ($25/M is expensive)
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},
            }
        ],
        tools=TOOLS,  # Last tool has cache_control
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"<user_context>\n{user_context.to_text()}\n</user_context>",
                        "cache_control": {"type": "ephemeral"},
                    }
                ],
            },
            {
                "role": "assistant",
                "content": "Ready.",
            },
            {
                "role": "user",
                "content": "Acknowledge.",
            },
        ],
    )
    
    # Cost: ~2,500 tokens × $6.25/M = $0.015625 (write)
    #        + ~10 tokens × $5.00/M = $0.00005 (query)  
    #        + ~1 token × $25.00/M = $0.000025 (output)
    # Total: ~$0.016 per warm-up
    
    logger.info(
        f"Cache warmed: write={response.usage.cache_creation_input_tokens}, "
        f"read={response.usage.cache_read_input_tokens}"
    )
```

### Strategy 2: Keep-Alive Pings (Use Sparingly)

If a user has an active session but goes quiet, you can send keep-alive requests to prevent cache eviction:

```python
import asyncio

async def cache_keepalive_loop(
    user_context: UserContext,
    session_active: asyncio.Event,
    interval_seconds: int = 240,  # Ping every 4 minutes (TTL is 5)
):
    """
    Keep the cache warm with periodic pings.
    
    WARNING: Each ping costs ~$0.016. Only use for high-value users
    where the expected savings outweigh the keepalive costs.
    
    Cost analysis:
    - 1 ping per 4 minutes = 15 pings/hour = $0.24/hour
    - Only worth it if user makes >22 Opus requests/hour
      (each saving $0.011 from cache reads)
    """
    while session_active.is_set():
        await asyncio.sleep(interval_seconds)
        if not session_active.is_set():
            break
        
        try:
            await warm_cache_on_login(user_context)
            logger.debug("Cache keepalive ping sent")
        except Exception as e:
            logger.warning(f"Cache keepalive failed: {e}")
```

> **⚠️ Verdict:** Keep-alive pings are rarely cost-effective. The math only works for extremely active users. In most cases, just accept the occasional cache miss.

### Strategy 3: Predictive Warming

Warm the cache when you predict Opus will be needed based on the user's action:

```python
async def maybe_warm_cache(user_id: str, user_action: str):
    """
    Predictively warm the Opus cache based on user behavior.
    
    Trigger warming when the user takes actions that historically
    lead to Opus requests (complex queries, analysis tasks, etc.)
    """
    WARM_TRIGGERS = {
        "open_analysis_page",
        "start_report_builder",
        "click_advanced_mode",
        "upload_document",
        "start_code_review",
    }
    
    if user_action in WARM_TRIGGERS:
        session = get_session(user_id)
        if session and not session.cache_is_likely_warm:
            # Fire and forget — don't block the user's action
            asyncio.create_task(warm_cache_on_login(session.user_context))
            logger.info(f"Predictive cache warm for {user_id} (trigger: {user_action})")
```

### Strategy 4: Shared Cache Warming via Cron

Since the system prompt + tools cache is shared across all users, you can keep it warm globally:

```python
async def global_cache_warmer():
    """
    Keep the shared system prompt + tools cache warm.
    
    Run every 4 minutes. Cost: ~$0.01 per ping (2,000 tokens × $6.25/M
    on first write, then $0.001 on reads).
    
    Monthly cost: ~$0.001 × 15/hour × 24 × 30 = ~$10.80/month
    Savings: Eliminates cold-start write penalty for system+tools
    on every single Opus request.
    """
    response = client.messages.create(
        model="claude-opus-4-5-20250229",
        max_tokens=1,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},
            }
        ],
        tools=TOOLS,  # Last tool has cache_control
        messages=[
            {"role": "user", "content": "ping"},
        ],
    )
    
    # After the first call, subsequent pings only cost cache-read prices
    # ~2,000 tokens × $0.50/M = $0.001 per ping
```

> **✅ Recommended:** The global cache warmer is almost always worth it if you have steady Opus traffic. At $10.80/month, it pays for itself if it saves even a handful of cache-write penalties.

---

## 7. Gotchas and Limitations

### 7.1 Cache Write is MORE Expensive Than No Cache

This is the #1 thing people miss:

```
No caching:    $5.00/M tokens
Cache write:   $6.25/M tokens  ← 25% MORE expensive
Cache read:    $0.50/M tokens  ← 90% LESS expensive
```

**Implication:** If a user sends exactly one Opus request and never returns, caching costs you more. You need at least 1.28 requests with the same prefix to break even.

**Mitigation:**
- Don't cache for one-off requests (if you can predict them)
- Focus caching on multi-turn conversations and repeat users
- The global system prompt cache amortizes across all users, so it's almost always profitable

### 7.2 Minimum 1,024 Tokens for Caching

The prefix up to a `cache_control` breakpoint must be at least **1,024 tokens**. Below this threshold, the breakpoint is **silently ignored** — no error, no warning.

```python
# ❌ BAD: This system prompt might be under 1,024 tokens
system=[{
    "type": "text",
    "text": "You are a helpful assistant.",  # ~6 tokens
    "cache_control": {"type": "ephemeral"},  # Silently ignored!
}]

# ✅ GOOD: Ensure the system prompt is padded to >1,024 tokens
# or combine system prompt + tools to exceed the threshold
```

**How to check:** Use `anthropic.count_tokens()` or estimate at ~0.75 tokens per word (English). 1,024 tokens ≈ 750-800 words.

**Strategy:** If your system prompt is short, combine it with tool definitions before the first breakpoint.

### 7.3 Exact Byte Matching

Cache matching is **byte-for-byte exact**. Any change invalidates the cache:

```python
# ❌ These create DIFFERENT caches:
"You are a helpful assistant."
"You are a helpful assistant. "  # Trailing space
"You are a helpful assistant.\n"  # Trailing newline

# ❌ Dynamic content in the cached prefix invalidates on every request:
f"You are a helpful assistant. Today's date is {date.today()}."
# This creates a new cache entry EVERY DAY

# ✅ Keep the cached prefix completely static:
"You are a helpful assistant."
# Put the date in the uncached portion (user message or after the last breakpoint)
```

### 7.4 Ordering Requirements

The prefix must follow this order:

```
system → tools → messages
```

You cannot cache `tools` without also including `system` in the prefix. You cannot have a cached message without the system + tools prefix being identical.

**Within messages**, order matters absolutely:

```python
# ❌ Reordering messages invalidates the cache:
# Request 1: [msg_A, msg_B, msg_C]
# Request 2: [msg_B, msg_A, msg_C]  ← Cache miss from byte 0

# ✅ Messages must be in the same order:
# Request 1: [msg_A, msg_B, msg_C]
# Request 2: [msg_A, msg_B, msg_C, msg_D]  ← Cache hit on A+B+C
```

### 7.5 The 4-Breakpoint Limit

You can only have **4** `cache_control` breakpoints per request. Plan your cache hierarchy carefully.

```python
# ❌ TOO MANY BREAKPOINTS (5):
system=[{"text": "...", "cache_control": {"type": "ephemeral"}}]  # 1
tools=[..., {"cache_control": {"type": "ephemeral"}}]             # 2
messages=[
    {"content": [{"text": "ctx", "cache_control": ...}]},         # 3
    {"content": [{"text": "history1", "cache_control": ...}]},    # 4
    {"content": [{"text": "history2", "cache_control": ...}]},    # 5 ← Ignored!
]

# ✅ PLAN YOUR 4 BREAKPOINTS:
# 1: System prompt
# 2: Last tool definition
# 3: User context
# 4: End of conversation history (before current query)
```

### 7.6 TTL Cannot Be Controlled

- 5-minute TTL, period. No way to set longer or shorter
- No API to check if a cache is warm
- No way to manually evict a cache
- You can only infer cache state from response `usage` fields

### 7.7 Streaming and Caching

Prompt caching works identically with streaming responses. The cache is populated/read during the prompt processing phase, not during output generation. No special handling needed for streaming.

### 7.8 Cache Scope

Caches are scoped to:
- Your **API key** (organization)
- The **model** (claude-opus-4-5 and claude-sonnet-4 have separate caches)
- The **exact byte sequence** of the prefix

Caches are **not** scoped to:
- Individual users (this is why the system prompt cache is shared!)
- API endpoints or regions (unclear, assume same region)

### 7.9 Tool Result Caching Caveat

Tool results in the conversation history can be cached as part of the message prefix, but be careful:

```python
# Tool results often contain dynamic data that changes:
messages = [
    {"role": "user", "content": "What's the current price?"},
    {"role": "assistant", "content": [{"type": "tool_use", ...}]},
    {"role": "user", "content": [
        {"type": "tool_result", "content": "Price: $142.50"}  # Dynamic!
    ]},
]
# If the price changes, the entire cache after this point is invalidated
```

### 7.10 Costs Don't Affect Rate Limits

Cached tokens still count toward your **rate limits** (tokens per minute). Caching saves money but doesn't increase throughput.

---

## 8. Decision Framework

### Should You Enable Caching?

```
                    ┌─────────────────────────┐
                    │ Is the cacheable prefix  │
                    │ ≥ 1,024 tokens?          │
                    └──────────┬──────────────┘
                               │
                    No ─────── ┤ ──────── Yes
                    │                      │
              ┌─────┴──────┐    ┌─────────┴──────────┐
              │ Can you pad │    │ Expected cache hits │
              │ to 1,024?   │    │ per cache write?    │
              └──────┬──────┘    └─────────┬──────────┘
                     │                     │
              No ── ┤ ── Yes       <2 ─── ┤ ──── ≥2
              │          │          │              │
         ┌────┴────┐  Pad it   ┌───┴───┐    ┌────┴─────┐
         │ Don't   │  & cache  │Maybe  │    │ Cache it │
         │ cache   │           │not    │    │ ✅       │
         └─────────┘           │worth  │    └──────────┘
                               │it     │
                               └───────┘
```

### Quick Reference: Cache vs. No Cache

| Scenario | Recommendation |
|---|---|
| Multi-turn conversation (3+ turns) | **Always cache** — massive savings |
| Repeat user, multiple sessions/day | **Cache** — system+tools stay warm |
| One-off API call, no follow-up | **Don't cache** — write penalty not recovered |
| Short system prompt (<1,024 tokens) | **Pad or combine** with tools to reach minimum |
| Dynamic system prompt (changes per request) | **Don't cache** the dynamic parts — restructure |
| High-volume SaaS (>100 Opus req/day) | **Cache + global warmer** — amortized savings |

### Our Recommended Configuration

For our SaaS architecture:

1. **Always cache** system prompt + tool definitions (global, shared)
2. **Cache** user context for users with 2+ expected Opus requests per session
3. **Cache** conversation history for multi-turn interactions
4. **Run global cache warmer** every 4 minutes (~$10/month)
5. **Use predictive warming** for power users entering complex-task flows
6. **Monitor** cache hit rates — target >60% overall
7. **Don't bother** with keep-alive pings for individual users (rarely cost-effective)

### Expected Overall Impact

With this strategy applied to our 10,000 daily Opus requests:

```
Without caching:  ~$10,500/month (input + output)
With caching:     ~$8,400/month
Savings:          ~$2,100/month (20% reduction)
Cache warmer:     ~$11/month
Net savings:      ~$2,089/month
```

As conversation length and user retention increase, savings approach **30-40%** on input costs.

---

*Last updated: 2025-07-13*
*Author: AI Architecture Team*
