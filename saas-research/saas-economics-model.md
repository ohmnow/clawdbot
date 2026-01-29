# SaaS Per-User Economics Model
## Clawdbot-as-a-Service: Dual-Model Architecture

---

## 1. Model Pricing (Current)

| Model | Input $/1M | Output $/1M | Role |
|-------|-----------|------------|------|
| **Gemini 2.0 Flash** | $0.10 | $0.40 | Workhorse (70%+ of requests) |
| **Claude Opus 4.5** | $5.00 | $25.00 | Brain (complex reasoning, 30% of requests) |

**Opus is ~100x more expensive per token than Flash.** This single fact drives the entire economics.

---

## 2. Request Cost Assumptions

### Flash Request (tool calls, simple actions)
- Average: 800 input tokens + 400 output tokens
- **Cost: $0.00024/request**

### Opus Request (complex reasoning, nuanced decisions)
- Average: 1,500 input tokens + 800 output tokens
- **Cost: $0.0275/request**

> An Opus call costs **~115x** more than a Flash call.

---

## 3. Per-User Monthly Cost by Usage Tier

### At 70% Flash / 30% Opus split:

| User Type | Requests/mo | Flash Cost | Opus Cost | **Total Cost** |
|-----------|-------------|-----------|-----------|---------------|
| Light (20/day) | 600 | $0.10 | $4.95 | **$5.05** |
| Medium (50/day) | 1,500 | $0.25 | $12.38 | **$12.63** |
| Heavy (100/day) | 3,000 | $0.50 | $24.75 | **$25.25** |
| Power (170/day) | 5,000 | $0.84 | $41.25 | **$42.09** |

### At 85% Flash / 15% Opus split (better routing):

| User Type | Requests/mo | Flash Cost | Opus Cost | **Total Cost** | Savings vs 70/30 |
|-----------|-------------|-----------|-----------|---------------|-------------------|
| Light | 600 | $0.12 | $2.48 | **$2.60** | 49% |
| Medium | 1,500 | $0.31 | $6.19 | **$6.50** | 49% |
| Heavy | 3,000 | $0.61 | $12.38 | **$12.99** | 49% |
| Power | 5,000 | $1.02 | $20.63 | **$21.65** | 49% |

### At 90% Flash / 10% Opus split (aggressive routing):

| User Type | Requests/mo | Flash Cost | Opus Cost | **Total Cost** | Savings vs 70/30 |
|-----------|-------------|-----------|-----------|---------------|-------------------|
| Light | 600 | $0.13 | $1.65 | **$1.78** | 65% |
| Medium | 1,500 | $0.32 | $4.13 | **$4.45** | 65% |
| Heavy | 3,000 | $0.65 | $8.25 | **$8.90** | 65% |
| Power | 5,000 | $1.08 | $13.75 | **$14.83** | 65% |

**Key insight: Every 5% you shift from Opus to Flash saves ~16% on total cost.**

---

## 4. Cost Optimization Levers

### Prompt Caching (Opus)
Anthropic offers prompt caching:
- Cache write: $6.25/M tokens (one-time)
- Cache read: $0.50/M tokens (vs $5.00 standard input)
- **90% reduction on input tokens for cached system prompts**

If your system prompt is ~1,000 tokens and gets cached:
- Uncached Opus request: $0.0275 → Cached: $0.0230
- Saves ~16% per Opus call for repeated system prompts

### Batch API (Opus)
- 50% discount on both input and output tokens
- Requires async processing (not real-time)
- Good for: scheduled reports, batch analysis, non-urgent tasks
- Opus batch rate: $2.50 input / $12.50 output per M tokens

### "Unlimited" Flash Tier
At $0.00024/request, even a power user doing 5,000 requests/month costs $1.08 on Flash.
**You can safely offer "unlimited" Flash usage.** It's essentially free at any realistic user count.

---

## 5. SaaS Pricing Model Suggestions

### Option A: Credit-Based
| Plan | Monthly Price | Opus Credits | Flash | Overage |
|------|--------------|-------------|-------|---------|
| Starter | $19/mo | 200 Opus calls | Unlimited | $0.05/Opus call |
| Pro | $49/mo | 600 Opus calls | Unlimited | $0.04/Opus call |
| Business | $99/mo | 1,500 Opus calls | Unlimited | $0.03/Opus call |

**Margins at 85/15 routing:**
| Plan | Revenue | Avg User Cost | **Gross Margin** |
|------|---------|--------------|-----------------|
| Starter | $19 | ~$2.60 | **86%** |
| Pro | $49 | ~$6.50 | **87%** |
| Business | $99 | ~$13.00 | **87%** |

### Option B: Simple Tiers (User-Friendly)
| Plan | Monthly Price | Requests/mo | Model Mix |
|------|--------------|-------------|-----------|
| Basic | $15/mo | 1,000 total | Auto-routed |
| Standard | $39/mo | 3,000 total | Auto-routed |
| Unlimited | $79/mo | Unlimited Flash + 1,000 Opus | Auto-routed |

### Option C: Opus-as-Premium
- Base plan: $19/mo — unlimited Flash, zero Opus (fast + capable for most tasks)
- Premium: $49/mo — unlimited Flash + 500 Opus calls
- Enterprise: $99/mo — unlimited Flash + 2,000 Opus calls
- **Users explicitly trigger Opus** (button, command, or auto-detected)

---

## 6. Scaling Economics

### At 100 Users (Early Stage)
| Scenario | Monthly Revenue | Monthly Cost | **Profit** |
|----------|----------------|-------------|-----------|
| 100 × Starter ($19) | $1,900 | ~$260 | **$1,640** |
| 100 × Pro ($49) | $4,900 | ~$650 | **$4,250** |

### At 1,000 Users (Growth)
| Scenario | Monthly Revenue | Monthly Cost | **Profit** |
|----------|----------------|-------------|-----------|
| 1,000 × mixed avg $35 | $35,000 | ~$5,000 | **$30,000** |

### At 10,000 Users (Scale)
At this point you'd negotiate volume pricing with both Google and Anthropic.
Typical enterprise discounts: 20-40% off list prices.

| Scenario | Monthly Revenue | Monthly Cost | **Profit** |
|----------|----------------|-------------|-----------|
| 10,000 × mixed avg $35 | $350,000 | ~$35,000 | **$315,000** |

> At scale, your inference cost is ~10% of revenue. The rest is engineering, support, and infrastructure.

---

## 7. Prompt Caching Impact at Scale

If you implement aggressive prompt caching on Opus calls:
- System prompt (~1,000-2,000 tokens) cached across all users
- User context/memory (~500-1,000 tokens) cached per session
- **Reduces Opus input cost by 60-80%**
- Per-Opus-call cost drops from $0.0275 to ~$0.021
- At 1,000 users: saves ~$1,000/month

---

*Model: v1.0 — January 2026*
*Assumes current published API pricing. Prices trend downward over time.*
