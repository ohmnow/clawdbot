# Routing Architecture: Flash vs Opus
## Deciding What Goes Where

---

## Overview

The routing layer sits between the user's request and model inference. Its job:
**Send the cheapest model that can handle the task well. Escalate only when needed.**

Every unnecessary Opus call costs 115x more than it should.

---

## Architecture Diagram

```
User Request
     │
     ▼
┌─────────────┐
│   Router     │  ◄── Lightweight classifier (runs on every request)
│  (Layer 1)   │      Cost: ~$0.00002/request (negligible)
└──────┬──────┘
       │
       ├── SIMPLE (70-85%) ──────► Gemini 2.0 Flash
       │                            │
       │                            ▼
       │                     ┌──────────────┐
       │                     │  Confidence   │
       │                     │  Check        │◄── Did Flash handle it well?
       │                     │  (Layer 2)    │
       │                     └──────┬───────┘
       │                            │
       │                     NO ────┼──── YES → Return response
       │                            │
       │                            ▼
       │                     Escalate to Opus
       │
       └── COMPLEX (15-30%) ─────► Claude Opus 4.5
                                    │
                                    ▼
                              Return response
```

---

## Layer 1: The Router (Pre-Classification)

### Option A: Rule-Based Router (Cheapest, Fastest)

No LLM call needed. Pattern match on the request.

**Route to Flash:**
- Tool calls with clear parameters ("set a timer for 5 minutes")
- Lookups and fetches ("what's the weather", "check my calendar")
- Simple CRUD operations ("create a note", "add to list")
- Status checks ("what's on my schedule")
- Formatting and transformation ("summarize this", "translate this")
- Acknowledgments and simple follow-ups
- Any request matching a known tool signature directly
- Messages under ~50 tokens with clear intent
- Requests that are continuations of an already-routed-to-Flash conversation

**Route to Opus:**
- Multi-step planning ("help me plan a project")
- Creative/original content ("write a proposal", "draft an email to...")
- Analysis and reasoning ("compare these options", "what should I do about...")
- Ambiguous requests that need interpretation
- Requests involving sensitive external actions (sending emails, posting publicly)
- Debugging and complex problem-solving
- Requests that reference nuanced context or require memory synthesis
- Anything the user explicitly flags as important

**Implementation:**
```python
OPUS_KEYWORDS = [
    "analyze", "compare", "plan", "strategy", "think about",
    "write", "draft", "compose", "create", "design",
    "explain why", "help me decide", "what should I",
    "review", "evaluate", "assess", "recommend",
    "debug", "troubleshoot", "figure out"
]

FLASH_PATTERNS = [
    r"^(set|create|add|remove|delete|toggle|turn|check|get|show|list|open|close|start|stop)\b",
    r"^what('s| is) (the |my )(weather|time|schedule|status)",
    r"^remind me",
    r"^(yes|no|ok|sure|thanks|got it)",
]

def route_request(message, conversation_history):
    # Check explicit user override first
    if "use opus" in message.lower() or "/opus" in message:
        return "opus"
    if "use flash" in message.lower() or "/flash" in message:
        return "flash"

    # Rule-based classification
    msg_lower = message.lower()

    # Short, simple messages → Flash
    if len(message.split()) < 10 and not any(kw in msg_lower for kw in OPUS_KEYWORDS):
        return "flash"

    # Known tool invocations → Flash
    if any(re.match(p, msg_lower) for p in FLASH_PATTERNS):
        return "flash"

    # Opus keywords detected → Opus
    if any(kw in msg_lower for kw in OPUS_KEYWORDS):
        return "opus"

    # Default: Flash (cheaper to be wrong here than to over-route to Opus)
    return "flash"
```

**Pros:** Zero latency, zero cost, deterministic
**Cons:** Misclassifies edge cases, needs ongoing tuning

---

### Option B: Embedding-Based Router (Better Accuracy, Still Cheap)

Use a small embedding model to classify intent against pre-built clusters.

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')  # 22M params, runs on CPU

# Pre-computed centroids for each category
FLASH_EXAMPLES = [
    "set a reminder for 3pm",
    "what's the weather in SF",
    "turn off the lights",
    "check my calendar",
    "add milk to my shopping list",
    # ... 50+ examples
]

OPUS_EXAMPLES = [
    "help me think through this business decision",
    "write a professional email to my client about the delay",
    "analyze these three options and recommend one",
    "review this contract and flag any concerns",
    # ... 50+ examples
]

flash_centroid = model.encode(FLASH_EXAMPLES).mean(axis=0)
opus_centroid = model.encode(OPUS_EXAMPLES).mean(axis=0)

def route_request(message):
    embedding = model.encode(message)
    flash_sim = np.dot(embedding, flash_centroid)
    opus_sim = np.dot(embedding, opus_centroid)

    if opus_sim > flash_sim + THRESHOLD:
        return "opus"
    return "flash"  # Default to cheaper option
```

**Pros:** Handles novel phrasings, more nuanced
**Cons:** ~5-10ms latency (negligible), needs example curation

---

### Option C: LLM-as-Router (Most Accurate, Small Cost)

Use Flash itself (or a tiny model) to classify before routing.

```
System: You are a request classifier. Respond with exactly one word: SIMPLE or COMPLEX.

SIMPLE = can be handled by tool calls, lookups, simple actions, or short responses.
COMPLEX = requires reasoning, planning, creative writing, analysis, or nuanced judgment.

User request: {message}
Classification:
```

- Cost per classification: ~$0.00003 (200 tokens on Flash)
- Adds ~200ms latency per request
- Most accurate option

**Pros:** Highest accuracy, adapts to any phrasing
**Cons:** Adds latency and (tiny) cost to every request, still imperfect

---

### Recommended: Hybrid (A + C)

```
1. Rule-based check first (Option A)
   - If HIGH CONFIDENCE match → route immediately (no extra cost)
   - Handles 60-70% of requests instantly

2. If UNCERTAIN → use Flash micro-classification (Option C)
   - 200-token classifier call
   - Handles the remaining 30-40%
   - Cost: ~$0.00003 per uncertain request
```

**Total router cost: <$0.00001 average per request** (effectively free)

---

## Layer 2: Confidence-Based Escalation

Even after routing to Flash, monitor the response for signs it's struggling.

### Escalation Triggers

```python
def should_escalate_to_opus(flash_response, original_request):
    triggers = []

    # 1. Flash explicitly says it can't handle it
    if any(phrase in flash_response.lower() for phrase in [
        "i'm not sure", "i don't know", "this is complex",
        "you might want to", "i'd recommend asking",
        "beyond my ability"
    ]):
        triggers.append("uncertainty_detected")

    # 2. Response is suspiciously short for a complex question
    if len(original_request.split()) > 30 and len(flash_response.split()) < 20:
        triggers.append("thin_response")

    # 3. Tool call failed and Flash can't recover
    if "tool_error" in flash_response and "retry_failed" in flash_response:
        triggers.append("tool_failure")

    # 4. User feedback (thumbs down, "try again", "that's wrong")
    # This is caught in the next turn, not same-turn

    return len(triggers) > 0, triggers
```

### Escalation Flow

```
Flash generates response
     │
     ▼
Confidence check passes? ──YES──► Deliver to user
     │
     NO
     │
     ▼
Escalate to Opus with context:
  - Original request
  - Flash's attempt (as reference)
  - "The initial model was uncertain. Please provide a thorough response."
     │
     ▼
Deliver Opus response to user
(User sees one seamless response, doesn't know about the retry)
```

**Important:** The escalation is invisible to the user. They just get a slightly slower but better response.

---

## Layer 3: User-Driven Override

Let users control routing when they want to:

- **`/think`** or **`/opus`** — forces Opus for the next message
- **`/quick`** or **`/flash`** — forces Flash
- **Toggle in UI** — "Deep thinking mode" switch
- **Per-conversation setting** — "This is an important conversation, use the smart model"

This gives power users control and reduces frustration when the router gets it wrong.

---

## Routing Decision Matrix

| Signal | Route | Confidence |
|--------|-------|------------|
| Known tool pattern (set timer, check weather) | Flash | High |
| Short acknowledgment (<10 words, no question) | Flash | High |
| "Write/draft/compose" + content type | Opus | High |
| "Analyze/compare/evaluate" | Opus | High |
| Multi-sentence complex question | Opus | Medium |
| Follow-up in Flash conversation | Flash | Medium |
| Ambiguous mid-length request | Classify | Low |
| User explicitly requests model | Override | Absolute |
| Flash response triggers escalation | Opus | High |

---

## Monitoring & Tuning

Track these metrics weekly:

| Metric | Target | Why |
|--------|--------|-----|
| **Opus routing rate** | <20% | Cost control |
| **Escalation rate** (Flash → Opus) | <5% | Router accuracy |
| **User override rate** | <3% | Router satisfaction |
| **User thumbs-down after Flash** | <5% | Quality check |
| **Avg response time (Flash)** | <1s | UX |
| **Avg response time (Opus)** | <5s | UX |

If escalation rate is high → your router is under-routing to Opus (quality problem).
If Opus rate is high → your router is over-routing (cost problem).

The sweet spot: **15-20% Opus, <3% escalation, <2% user override.**

---

## Cost Impact Summary

| Routing Accuracy | Opus % | Cost/User/Mo (Medium) | Annual (1K users) |
|-----------------|--------|----------------------|-------------------|
| No routing (all Opus) | 100% | $41.25 | $495,000 |
| Naive 70/30 | 30% | $12.63 | $151,560 |
| Good routing 85/15 | 15% | $6.50 | $78,000 |
| Great routing 90/10 | 10% | $4.45 | $53,400 |
| Perfect routing 95/5 | 5% | $2.38 | $28,560 |

**The difference between naive and great routing: $73K/year at 1,000 users.**

---

*Architecture: v1.0 — January 2026*
