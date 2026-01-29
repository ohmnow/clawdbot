# Deep Dive: Router Classifier Prompt

> Dual-model SaaS — Gemini 2.0 Flash (simple, 70-85%) · Claude Opus 4.5 (complex, 15-30%)
> Hybrid routing: rule-based patterns → LLM classifier → model dispatch

---

## 1. Classifier System Prompt

This prompt runs on a lightweight model (Flash or Haiku) to classify ambiguous requests that survive the rule-based layer.

```
You are a request complexity classifier for an AI assistant platform.

Classify the user's message as SIMPLE or COMPLEX.

SIMPLE — Direct actions, lookups, tool calls, short factual answers, confirmations,
format conversions, CRUD operations, single-step tasks. No nuance or judgment needed.

COMPLEX — Multi-step reasoning, creative writing, analysis, planning, persuasion,
ambiguous/sensitive topics, domain expertise, emotional intelligence, debate,
synthesis across multiple sources, or anything requiring careful judgment.

RULES:
- Default to SIMPLE when uncertain (cost optimization).
- If the conversation is already on the COMPLEX model, bias toward COMPLEX
  (thread continuity) unless the message is a trivial follow-up like "yes", "ok", "thanks".
- Multi-turn context matters: a simple question mid-analysis stays COMPLEX.
- Tool calls with no reasoning = SIMPLE. Tool calls requiring judgment = COMPLEX.

OUTPUT FORMAT (JSON only, no explanation):
{"classification":"SIMPLE"|"COMPLEX","confidence":0.0-1.0,"reason":"<8 words max>"}
```

---

## 2. Classification Taxonomy

### SIMPLE (→ Gemini Flash)

| Example | Why |
|---------|-----|
| "What time is it in Tokyo?" | Single lookup |
| "Set a reminder for 3pm" | Direct tool call |
| "Convert 50 USD to EUR" | Deterministic |
| "Turn off the lights" | Device command |
| "Show my calendar for today" | Data retrieval |
| "Delete that last message" | CRUD operation |
| "Translate 'hello' to Spanish" | Direct translation |
| "What's 15% of 230?" | Arithmetic |
| "Summarize this in one sentence" | Constrained output |
| "Add milk to my shopping list" | Append operation |
| "What's the weather in Berlin?" | API lookup |
| "Rename this file to report.pdf" | File operation |
| "Yes, send it" | Confirmation |
| "Read me the last email" | Data retrieval |
| "Play some jazz music" | Simple command |
| "How tall is the Eiffel Tower?" | Single fact |

### COMPLEX (→ Claude Opus)

| Example | Why |
|---------|-----|
| "Write a persuasive cover letter for a PM role" | Creative + persuasion |
| "Compare AWS vs GCP for our startup" | Multi-factor analysis |
| "Debug why this React component re-renders" | Code reasoning |
| "Help me plan a 2-week Japan itinerary" | Multi-step planning |
| "Explain quantum entanglement to a 10-year-old" | Pedagogical adaptation |
| "Review this contract for red flags" | Domain expertise |
| "Why am I feeling stuck in my career?" | Emotional intelligence |
| "Refactor this function for testability" | Architecture judgment |
| "Write a blog post about sustainable energy" | Long-form creative |
| "Analyze this dataset and find trends" | Multi-step analysis |
| "Help me negotiate a higher salary" | Strategy + nuance |
| "What are the ethical implications of AI art?" | Nuanced reasoning |
| "Design a database schema for a social app" | System design |
| "Critique my business plan" | Evaluation + judgment |
| "Explain the pros/cons of this architecture" | Trade-off analysis |
| "Help me resolve a conflict with my coworker" | Interpersonal reasoning |

### Edge Cases

| Request | Classification | Reasoning |
|---------|---------------|-----------|
| "Send an email to Bob" | SIMPLE | Direct action, no composition |
| "Draft a professional email to Bob about the delay" | COMPLEX | Requires tone + judgment |
| "Summarize this in one sentence" | SIMPLE | Constrained output |
| "Summarize the key strategic takeaways" | COMPLEX | Requires judgment on what's key |
| "Translate this to French" | SIMPLE | Mechanical translation |
| "Translate this preserving the poetic tone" | COMPLEX | Creative adaptation |
| "What's the capital of France?" | SIMPLE | Single fact |
| "What caused the French Revolution?" | COMPLEX | Historical analysis |

---

## 3. Few-Shot Examples

Appended to the system prompt for boundary calibration:

```
EXAMPLES:
User: "Send the report to Sarah"       → {"classification":"SIMPLE","confidence":0.95,"reason":"direct send action"}
User: "Draft a diplomatic response"     → {"classification":"COMPLEX","confidence":0.92,"reason":"requires tone and judgment"}
User: "What's 2+2?"                     → {"classification":"SIMPLE","confidence":0.99,"reason":"arithmetic"}
User: "Why is my code throwing KeyError?"→ {"classification":"COMPLEX","confidence":0.85,"reason":"debugging requires reasoning"}
User: "Yes, go ahead"                   → {"classification":"SIMPLE","confidence":0.99,"reason":"confirmation"}
User: "Make that sound more professional"→ {"classification":"COMPLEX","confidence":0.88,"reason":"subjective refinement"}
User: "List all files in current dir"   → {"classification":"SIMPLE","confidence":0.97,"reason":"direct command"}
User: "Help me structure this essay"    → {"classification":"COMPLEX","confidence":0.90,"reason":"creative restructuring"}
User: "Set my status to away"           → {"classification":"SIMPLE","confidence":0.98,"reason":"single action"}
User: "What should I prioritize?"       → {"classification":"COMPLEX","confidence":0.87,"reason":"planning requires context"}
```

---

## 4. Confidence Scoring & Routing

```python
from dataclasses import dataclass
from enum import Enum

class Model(Enum):
    FLASH = "gemini-2.0-flash"
    OPUS = "claude-opus-4-5"

@dataclass
class Classification:
    label: str        # "SIMPLE" or "COMPLEX"
    confidence: float # 0.0 - 1.0
    reason: str

# Thresholds
COMPLEX_THRESHOLD = 0.75   # Must be this confident to route to Opus
SIMPLE_THRESHOLD = 0.70    # Must be this confident to route to Flash
# Between thresholds → default to Flash (cost optimization)

def route(cls: Classification, thread_model: Model | None = None) -> Model:
    """Route based on classification + thread context."""
    # High-confidence routing
    if cls.label == "COMPLEX" and cls.confidence >= COMPLEX_THRESHOLD:
        return Model.OPUS
    if cls.label == "SIMPLE" and cls.confidence >= SIMPLE_THRESHOLD:
        # Thread stickiness: if already on Opus, stay unless very confident
        if thread_model == Model.OPUS and cls.confidence < 0.90:
            return Model.OPUS
        return Model.FLASH

    # Low confidence → use thread context or default
    if thread_model:
        return thread_model
    return Model.FLASH  # Default: optimize cost
```

---

## 5. Context-Aware Classification

```python
def build_classifier_input(
    message: str,
    history: list[dict],   # [{"role": "user"|"assistant", "content": str, "model": str}]
    thread_model: Model | None
) -> str:
    """Build context-enriched input for the classifier."""
    parts = []

    # Thread state signal
    if thread_model == Model.OPUS:
        parts.append(f"[CONTEXT: Thread is on {thread_model.value}. "
                      "Bias COMPLEX unless message is trivial.]")

    # Last 3 exchanges for context (trimmed)
    if history:
        recent = history[-6:]  # last 3 pairs
        convo = "\n".join(f"{m['role']}: {m['content'][:100]}" for m in recent)
        parts.append(f"RECENT CONVERSATION:\n{convo}")

    parts.append(f"CLASSIFY THIS MESSAGE:\n{message}")
    return "\n\n".join(parts)

# Fast-path overrides (skip classifier entirely)
TRIVIAL_CONFIRMATIONS = {"yes", "no", "ok", "sure", "thanks", "thank you",
                          "got it", "perfect", "great", "done", "cancel", "stop"}

def needs_classifier(message: str, thread_model: Model | None) -> tuple[bool, Model | None]:
    """Check if we can skip the classifier entirely."""
    normalized = message.strip().lower().rstrip("!.")

    # Trivial messages always go to Flash
    if normalized in TRIVIAL_CONFIRMATIONS:
        return False, Model.FLASH

    # If message is under 5 words and thread is on Opus, keep on Opus
    if thread_model == Model.OPUS and len(message.split()) <= 5:
        return False, Model.OPUS

    return True, None  # Needs classification
```

---

## 6. Rule-Based Layer

```python
import re

# Pattern sets
SIMPLE_PATTERNS = [
    # Direct commands
    r"\b(set|create|delete|remove|add|send|show|list|open|close|play|stop|pause)\b.*",
    # Lookups
    r"\b(what time|weather|temperature|price of|how many|how much|how tall|how far)\b",
    # Conversions
    r"\b(convert|translate)\b.{0,30}$",  # short = simple
    # Math
    r"^\d[\d\s\+\-\*\/\.\%\(\)]+$",
    # Yes/no/confirmations
    r"^(yes|no|ok|sure|cancel|confirm|done|thanks|stop|go ahead)[\.!\s]*$",
    # Tool-like commands
    r"^(remind me|set alarm|set timer|toggle|switch|turn on|turn off)\b",
]

COMPLEX_PATTERNS = [
    # Creative/analytical keywords
    r"\b(write|draft|compose|create).{10,}(essay|letter|blog|article|story|report|proposal)",
    # Reasoning requests
    r"\b(explain why|analyze|compare|evaluate|critique|review|assess|debate)\b",
    # Planning
    r"\b(plan|design|architect|strategy|roadmap|outline)\b.{15,}",
    # Emotional/interpersonal
    r"\b(feel|struggling|advice|help me (deal|cope|understand|decide|figure))\b",
    # Code reasoning (not just "run this")
    r"\b(debug|refactor|optimize|why.{0,10}(error|bug|fail|crash|wrong))\b",
    # Multi-step signals
    r"\b(step by step|pros and cons|trade.?offs|implications|considerations)\b",
]

def rule_based_classify(message: str) -> Classification | None:
    """Return classification if rules match, None if ambiguous."""
    msg = message.strip().lower()

    for pattern in SIMPLE_PATTERNS:
        if re.search(pattern, msg, re.IGNORECASE):
            return Classification("SIMPLE", 0.95, "rule:pattern-match")

    for pattern in COMPLEX_PATTERNS:
        if re.search(pattern, msg, re.IGNORECASE):
            return Classification("COMPLEX", 0.90, "rule:pattern-match")

    return None  # Ambiguous → send to LLM classifier
```

### Full Pipeline

```python
async def classify_and_route(
    message: str,
    history: list[dict],
    thread_model: Model | None = None
) -> Model:
    # Layer 1: Trivial skip
    needs_cls, fast_model = needs_classifier(message, thread_model)
    if not needs_cls:
        return fast_model

    # Layer 2: Rule-based
    rule_result = rule_based_classify(message)
    if rule_result:
        return route(rule_result, thread_model)

    # Layer 3: LLM classifier
    prompt = build_classifier_input(message, history, thread_model)
    cls = await call_classifier_llm(prompt)  # ~50ms with Flash/Haiku
    return route(cls, thread_model)
```

---

## 7. Testing Framework

```python
import json
from pathlib import Path

# Test case format
TEST_CASES: list[dict] = [
    {"input": "What time is it?", "expected": "SIMPLE", "category": "lookup"},
    {"input": "Write a persuasive essay on climate change", "expected": "COMPLEX", "category": "creative"},
    {"input": "Send it", "expected": "SIMPLE", "category": "confirmation"},
    {"input": "Debug this segfault in my C code", "expected": "COMPLEX", "category": "reasoning"},
    # ... 200+ cases across categories
]

# Categories (15-25 cases each): lookup, command, confirmation, arithmetic,
# creative, analysis, planning, debugging, emotional, edge-case, translation

def evaluate(classify_fn, test_cases: list[dict]) -> dict:
    results = {"correct": 0, "wrong": 0, "errors": [], "by_category": {}}
    for tc in test_cases:
        cls = classify_fn(tc["input"])
        cat = tc["category"]
        results["by_category"].setdefault(cat, {"correct": 0, "total": 0})
        results["by_category"][cat]["total"] += 1
        if cls.label == tc["expected"]:
            results["correct"] += 1
            results["by_category"][cat]["correct"] += 1
        else:
            results["wrong"] += 1
            results["errors"].append({
                "input": tc["input"], "expected": tc["expected"],
                "got": cls.label, "confidence": cls.confidence, "category": cat,
            })
    total = results["correct"] + results["wrong"]
    results["accuracy"] = results["correct"] / total if total else 0
    return results

def print_report(r: dict):
    print(f"Overall: {r['accuracy']:.1%} ({r['correct']}/{r['correct']+r['wrong']})")
    for cat, d in sorted(r["by_category"].items()):
        print(f"  {cat:15s} {d['correct']/d['total']:.0%} ({d['correct']}/{d['total']})")
    for e in r["errors"][:10]:
        print(f"  [{e['expected']}→{e['got']} @{e['confidence']:.2f}] {e['input'][:60]}")
```

Target: **≥92% accuracy** overall, **≥85%** per category.

---

## 8. Tuning Methodology

### Feedback Loop

```python
import datetime, json
from pathlib import Path

LOG_PATH = Path("classifier_logs.jsonl")

def log_classification(msg: str, cls: Classification, model: Model, feedback: str | None = None):
    entry = {
        "ts": datetime.datetime.utcnow().isoformat(),
        "message": msg[:200], "label": cls.label,
        "confidence": cls.confidence, "routed_to": model.value, "feedback": feedback,
    }
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(entry) + "\n")

def analyze_misclassifications(log_path: Path = LOG_PATH) -> dict:
    entries = [json.loads(l) for l in log_path.read_text().splitlines() if l.strip()]
    wrong = [e for e in entries if e.get("feedback", "").startswith("wrong")]
    false_simple = [e["message"] for e in wrong if "should-be-complex" in e["feedback"]]
    false_complex = [e["message"] for e in wrong if "should-be-simple" in e["feedback"]]
    return {
        "total": len(entries), "errors": len(wrong),
        "error_rate": len(wrong) / len(entries) if entries else 0,
        "false_simple": false_simple[:20], "false_complex": false_complex[:20],
    }
```

### Tuning Cycle (Weekly)

1. **Collect** — Log all classifications + user feedback signals
2. **Analyze** — Run `analyze_misclassifications()`, group by pattern
3. **Fix rules** — Pattern repeats 3+ times → add to regex layer
4. **Fix prompt** — Add misclassified examples to few-shot section
5. **Test** — Re-run suite, verify no regression
6. **Adjust thresholds** — High false-simple → lower `COMPLEX_THRESHOLD`; cost spike → raise it

### Key Metrics

| Metric | Target | Action |
|--------|--------|--------|
| Overall accuracy | ≥92% | Add few-shot examples |
| False-simple rate | ≤3% | Lower complex threshold |
| False-complex rate | ≤10% | Add rule-based patterns |
| Classifier p95 latency | ≤80ms | Simplify prompt |
| Opus cost share | ≤35% | Tighten complex routing |

---

## Architecture Summary

```
User Message → Trivial Skip (30%) → Flash
            → Rule-Based  (25%)  → Flash or Opus
            → LLM Classify(45%) → Flash or Opus
Result: ~75% Flash, ~25% Opus — cost-optimized, quality-preserved.
```
