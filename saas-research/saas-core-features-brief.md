# Moltbot SaaS — Core Features & Recommendation Brief

**Date:** January 29, 2026  
**Author:** Clawdbot Research Agent  
**For:** Chris — Moltbot SaaS Product Planning  
**Status:** DRAFT v1.0

---

## Executive Summary

Moltbot (formerly Clawdbot) has exploded to 29,900+ GitHub stars in weeks, an 8,900+ member Discord, TechCrunch/Mashable/MacStories coverage, and a viral wave across Reddit, X, and YouTube. The community has coalesced around 10 primary use cases. This brief analyzes each for demand signal, technical requirements, setup complexity, and revenue potential—then recommends the default feature set, onboarding flow, skills configuration, and pricing tiers for a managed SaaS offering.

**The core thesis:** Most users want the magic but not the setup. A managed SaaS that eliminates VPS provisioning, OAuth wrestling, and config-file editing can convert the ~90% of interested users who bounce off the self-hosted complexity into paying customers at $19–99/mo.

**Key finding:** The three highest-signal, lowest-friction use cases—**Email Management**, **Research & Content**, and **Calendar & Scheduling**—should form the core onboarding experience. They demonstrate value in under 5 minutes and require only OAuth grants the user already understands.

---

## Part 1: Use Case Deep Dive

### 1. Email Management

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥🔥 — Highest signal. Every article, demo, and showcase leads with inbox clearing. The clawd.bot homepage headlines it. Hormold's JMAP tweet went viral. Gmail Pub/Sub has dedicated docs. r/ClaudeCode and r/LocalLLM threads consistently mention "email summaries" as the first thing people set up. Multiple Medium articles list it as Use Case #1. |
| **Required Skills/Tools** | `gog` CLI (Google OAuth for Gmail), Gmail Pub/Sub webhook integration, `hooks` system for real-time email triggers, `web_fetch` for link processing, `cron` for periodic inbox sweeps. Skills: `gmail-pubsub` preset, email drafting via gog send. |
| **Complexity** | **Medium** — Requires Google OAuth setup (Cloud Console project, OAuth consent screen, credentials). Gmail Pub/Sub needs a GCP Pub/Sub topic + push subscription. For a SaaS, this can be pre-configured with a managed OAuth app — reducing to **Easy** (user just clicks "Connect Gmail"). |
| **Revenue Potential** | **Must-have / High** — Email overload is universal. Users report processing 1,000+ emails in a single session. This is the #1 "aha moment" that converts trial to paid. Competitive with SaneBox ($7/mo), Superhuman ($30/mo). Users will absolutely pay $19/mo for an AI that clears, categorizes, and responds to their inbox. |

**SaaS Recommendation:** ✅ Enabled by default. Core onboarding step #1. Pre-build the OAuth flow so users click one button to connect Gmail/Outlook.

---

### 2. Calendar & Scheduling

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥 — High. Consistently paired with email in demos. "What's on my calendar tomorrow?" is a canonical first-message example. DataCamp tutorial highlights it. The heartbeat system with morning briefings pulling calendar data is a primary showcase. Traffic-aware reminders are a viral demo clip. |
| **Required Skills/Tools** | `gog` CLI (Google Calendar API), `cron` for scheduled reminders, `heartbeat` for proactive briefings, `web_search` for traffic/weather context. Optional: Apple Calendar via macOS `osascript`, Outlook via Microsoft Graph API. |
| **Complexity** | **Medium** — Same Google OAuth as email (share the consent screen). Apple Calendar needs a paired Mac node. Microsoft Graph needs separate OAuth. For SaaS with managed Google OAuth: **Easy**. |
| **Revenue Potential** | **Must-have / High** — Calendar + email together form the core "daily briefing" that makes the product sticky. Reclaim.ai charges $8–18/mo for AI scheduling alone. This is table-stakes for any personal assistant. |

**SaaS Recommendation:** ✅ Enabled by default. Shares OAuth with Gmail. Part of the morning briefing that runs on day 1.

---

### 3. Flight & Travel

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥 — High. "Checks you in for flights" is on the clawd.bot homepage and in nearly every article. Reddit setup guides mention flight check-in as a headline feature. YouTube demos show booking flights via browser automation. Padel court booking skill is a community showcase. Vienna transport skill exists. |
| **Required Skills/Tools** | `browser` tool (Playwright-based), `cron` for check-in timing, `web_search` for flight monitoring, email parsing (Gmail webhook) to detect booking confirmations. Skills: airline-specific check-in flows, price monitoring scripts. |
| **Complexity** | **Hard** — Browser automation is fragile (airlines change UIs, use CAPTCHAs, require login state). Check-in timing must be precise (exactly 24h before departure). Needs persistent browser sessions. For SaaS: **Medium-Hard** — manageable with headless browser infrastructure but requires maintenance. |
| **Revenue Potential** | **Nice-to-have / High perceived value** — Users love the demo, but actual frequency is low (most people fly 2–10x/year). However, the "wow factor" is enormous for marketing. Flighty app charges $4/mo just for flight tracking. This is a premium/differentiator feature. |

**SaaS Recommendation:** 🔶 Opt-in / Premium. Available via browser automation add-on. Great for marketing but too fragile for default onboarding. Enable flight *tracking* (via email parsing) by default; enable flight *check-in* (via browser) in Pro tier.

---

### 4. Coding & Development

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥🔥 — Extremely high among developers (who are the current user base). 15+ coding skills on ClawdHub. Codex orchestration, Claude Code integration, PR review → Telegram is a community showcase. "Autonomous Claude Code loops from my phone" is a homepage testimonial. Adam's 14-agent Dream Team orchestration is legendary. Kev's Dream Team write-up has spawned an entire sub-community. |
| **Required Skills/Tools** | `exec` tool, `process` management, `coding-agent` skill, `github` skill (gh CLI), `codex-orchestration`, `codex-monitor`, Docker sandboxing, `sessions_spawn` for parallel agents. Needs: git, Node.js, language runtimes in the sandbox. |
| **Complexity** | **Medium-Hard** — Requires sandbox/Docker setup for safe code execution, git credentials, possibly SSH keys. For SaaS: provide pre-built dev containers with common runtimes. |
| **Revenue Potential** | **Must-have for dev users / Very High** — Developers are the highest-LTV users and most willing to pay. Cursor charges $20/mo, Codex is $200/mo for Pro. This competes directly. The "manage PRs from your phone" pitch is extremely compelling. However, this is expensive (heavy model usage, compute for sandboxes). |

**SaaS Recommendation:** 🔶 Opt-in / Pro tier. Not in default onboarding (non-devs won't use it), but prominently featured for developer signups. Requires sandbox compute allocation.

---

### 5. Health & Fitness Tracking

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥 — Moderate-High. Oura ring integration is a community showcase on docs.molt.bot. WHOOP/Apple Health mentioned in TechLoy, Medium articles, and multiple setup guides. "Morning briefing with health data" is a common configuration. YouTube video "I Built My Second Brain with Claude Code + Obsidian" touches on this. The community member @AS built a dedicated Oura health assistant. |
| **Required Skills/Tools** | Health device APIs (Oura API, WHOOP API, Apple Health via macOS Shortcuts), `cron`/`heartbeat` for morning briefings, `web_fetch` for API calls. Skills: `oura-ring` skill exists on ClawdHub. Apple Health requires a paired Mac node. |
| **Complexity** | **Medium-Hard** — Each health platform has its own OAuth/API key flow. Apple Health requires a Mac node (not available in cloud SaaS). Oura and WHOOP have developer APIs but require user authorization. For SaaS: **Medium** — pre-build OAuth for Oura/WHOOP; Apple Health requires node pairing. |
| **Revenue Potential** | **Nice-to-have / Medium** — Health-conscious users are a passionate niche but not the majority. However, they tend to be high-income, tech-forward (exactly the Moltbot demographic). Good for retention — daily health briefings create habit loops. |

**SaaS Recommendation:** 🔶 Opt-in. Available in skill marketplace. Include health data in morning briefings for users who connect a device. Not in core onboarding.

---

### 6. Smart Home Automation

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥 — Moderate. Home Assistant skill exists on ClawdHub. Winix air purifier control was a viral showcase. Philips Hue is mentioned across articles. The r/roonlabs Clawdbot + Home Assistant + Roon post showed real enthusiasm. Bambu 3D printer control skill exists. However, smart home users are a subset and typically already have Home Assistant/HomeKit. |
| **Required Skills/Tools** | `homeassistant` skill, `nodes` tool for device control, MQTT/HTTP integrations, `cron` for schedules. Philips Hue: direct HTTP API. Home Assistant: REST API with long-lived access token. Requires either local network access (node) or Home Assistant Cloud. |
| **Complexity** | **Hard** — Requires either a local node (Mac/Raspberry Pi on same network as devices) or Home Assistant Cloud. Device-specific setup varies wildly. Not feasible in a pure cloud SaaS without node pairing. For SaaS: **Hard** — always requires node. |
| **Revenue Potential** | **Nice-to-have / Low-Medium** — Smart home users already pay for Home Assistant Cloud ($6.50/mo) or similar. Moltbot adds an AI layer on top, but the value-add over existing voice assistants (Alexa, Google Home) is marginal for most users. Power users love it though. |

**SaaS Recommendation:** 🔶 Opt-in / Premium. Requires node pairing. Available in Pro/Business tiers. Not part of onboarding.

---

### 7. Research & Content

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥🔥 — Very high. "Deep research" is consistently mentioned across all community channels. Multiple search/research skills on ClawdHub (Exa, Kagi, Parallel.ai, literature-review). Background research tasks are a core Moltbot value proposition. Content creation pipelines (blog posts, newsletters, social media) are frequently discussed. The accounting intake showcase (collecting PDFs from email, prepping for tax) went viral. |
| **Required Skills/Tools** | `web_search`, `web_fetch`, `browser` for deep research, `read`/`write` for document creation, `exec` for format conversion. Skills: `exa`, `kagi-search`, `parallel`, `literature-review`. Content pipelines: `cron` for scheduled research, email integration for delivery. |
| **Complexity** | **Easy-Medium** — Web search and fetch work out of the box. Browser-based research needs the browser tool enabled. Document creation is native. For SaaS: **Easy** — these are built-in tools. |
| **Revenue Potential** | **Must-have / Very High** — This is what makes Moltbot more than a chatbot. "Run this research in the background and send me results in 2 hours" is a killer use case. Competes with Perplexity Pro ($20/mo) and deep research tools. High model usage but high perceived value. |

**SaaS Recommendation:** ✅ Enabled by default. Core capability. Web search + fetch available on all tiers. Deep research (browser-based, multi-step) on Pro+.

---

### 8. Document Processing

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥 — Moderate. PDF extraction, format conversion, and file organization come up regularly but aren't the headline use case. The accounting intake showcase is popular. Receipt processing and warranty tracking mentioned in dev.to guide. The Moltbot creator's background (PSPDFKit/Nutrient, a PDF SDK company) adds credibility here. |
| **Required Skills/Tools** | `read`/`write`/`exec` for file operations, PDF tools (poppler, ghostscript, pandoc), image analysis for scanned docs, `web_fetch` for downloading files. Skills: PDF-specific processing skills on ClawdHub. |
| **Complexity** | **Easy-Medium** — Basic file operations work natively. PDF processing needs CLI tools installed (poppler-utils, pandoc). OCR needs additional setup. For SaaS: **Easy** — pre-install common document tools in the container. |
| **Revenue Potential** | **Nice-to-have / Medium** — Useful but not a primary driver. More of a "and it can also..." feature. Combined with email (auto-process PDF attachments) it becomes more compelling. |

**SaaS Recommendation:** ✅ Enabled by default (basic file operations). PDF tools pre-installed. Part of the email pipeline.

---

### 9. Insurance & Administrative Tasks

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥 — Moderate but viral. The insurance claim reinvestigation story is on the clawd.bot homepage and has been cited in nearly every article about Moltbot. It's the "accidental genius" anecdote that proves the system's capability. Administrative automation (reimbursements, disputes, form-filling) is consistently mentioned. |
| **Required Skills/Tools** | `browser` for form-filling and portal navigation, `email` integration for correspondence, `read`/`write` for document preparation, `web_search` for policy research. Lobster workflows with approval checkpoints for sensitive actions. |
| **Complexity** | **Hard** — These are high-stakes, one-off tasks requiring careful human-in-the-loop oversight. Browser automation on insurance portals is fragile. Email correspondence on behalf of users raises liability questions. For SaaS: **Hard** — needs approval workflows, audit logging, and possibly legal review. |
| **Revenue Potential** | **Nice-to-have but high storytelling value / Medium** — Users won't subscribe *for* this, but it's a powerful retention/word-of-mouth story. "My AI assistant got my insurance claim reopened" is shareable. One successful dispute could save hundreds/thousands of dollars, justifying the subscription. |

**SaaS Recommendation:** 🔶 Available but not promoted. Enable browser + email as building blocks. Add prominent approval checkpoints. Use the insurance story in marketing, not in onboarding.

---

### 10. Personal Knowledge Management

| Dimension | Assessment |
|---|---|
| **Demand Signal** | 🔥🔥🔥🔥 — High. Obsidian integration is mentioned on the clawd.bot homepage (testimonial from @svenkataram). The memory system natively stores in Obsidian-compatible Markdown format. YouTube video "I Built My Second Brain with Claude Code + Obsidian" exists. 44 Notes & PKM skills on ClawdHub (obsidian-daily, second-brain, Notion integration, Bear Notes). The system's built-in memory (MEMORY.md, daily notes) *is* a PKM system. |
| **Required Skills/Tools** | `read`/`write` for Markdown files, `memorySearch` with embeddings (Gemini), Obsidian vault access (local files or synced), `cron` for daily note creation. Skills: `obsidian-daily`, `second-brain`, `notion-integration`. Notion API requires OAuth. |
| **Complexity** | **Easy-Medium** — Moltbot's memory system is built-in. Obsidian vault is just a folder of Markdown files (easy to access). Notion requires OAuth. For SaaS: **Easy** for built-in memory, **Medium** for external PKM tools. |
| **Revenue Potential** | **Must-have / High** — PKM users are passionate, organized, and willing to pay (Notion: $10/mo, Obsidian Sync: $4/mo, Readwise: $8/mo). The persistent memory *is* the differentiator vs. ChatGPT/Claude. This is what makes Moltbot feel like JARVIS. |

**SaaS Recommendation:** ✅ Enabled by default. The built-in memory system should be a core selling point. Obsidian sync and Notion integration available as opt-in connections.

---

## Part 2: Recommended Default Feature Set

### Tier Classification

| Category | Use Cases | Rationale |
|---|---|---|
| **Enabled by Default** (Core Onboarding) | Email Management, Calendar & Scheduling, Research & Content, Document Processing, Personal Knowledge Management | Low friction, immediate value, universal appeal. These work with just a Google OAuth grant + built-in tools. |
| **Available but Opt-in** (Skill Marketplace) | Health & Fitness Tracking, Insurance & Admin Tasks | Require device-specific OAuth or involve sensitive actions. Available to add after onboarding. |
| **Premium Tier** | Flight & Travel, Coding & Development, Smart Home Automation | Require browser automation, sandbox compute, or node pairing. Higher cost to serve, higher value delivered. |

### The "Day 1 Stack" — What Works Immediately

```
┌─────────────────────────────────────────────────┐
│              MOLTBOT SaaS — DAY 1               │
├─────────────────────────────────────────────────┤
│  ✅ Chat via Telegram/WhatsApp/Slack/Discord     │
│  ✅ Web search & research                        │
│  ✅ File reading/writing/editing                  │
│  ✅ Memory & daily notes                          │
│  ✅ Cron jobs & heartbeat briefings               │
│  ✅ Document processing (PDF, Markdown, etc.)     │
├─────────────────────────────────────────────────┤
│  🔗 After OAuth: Gmail + Calendar                │
│  🔗 After OAuth: Notion (optional)               │
│  🔗 After pairing: Mac node (optional)           │
├─────────────────────────────────────────────────┤
│  💎 Pro: Browser automation, Coding sandboxes    │
│  💎 Business: Multi-agent, Smart home, Nodes     │
└─────────────────────────────────────────────────┘
```

---

## Part 3: Onboarding Flow Design

### First Signup Experience

#### Step 0: Landing Page → Sign Up
- **Headline:** "Your AI assistant that messages *you* first."
- **Subhead:** "Moltbot manages your email, calendar, research, and memory — from your favorite chat app."
- **CTA:** "Start Free Trial (14 days)" → email/Google sign-up
- **Social proof:** GitHub stars counter, testimonial quotes, TechCrunch logo

#### Step 1: Choose Your Chat Channel (30 seconds)
```
┌─────────────────────────────────────────┐
│  How should Moltbot reach you?          │
│                                         │
│  📱 Telegram     (recommended)          │
│  💬 WhatsApp                            │
│  💼 Slack                               │
│  🎮 Discord                             │
│  🌐 Web Chat     (works now, no setup)  │
│                                         │
│  [Continue with Telegram →]             │
└─────────────────────────────────────────┘
```

**Why Telegram first:** Easiest bot setup (just tap a link), richest features (inline buttons, reactions, voice notes), no phone number sharing required. WhatsApp requires business API. Slack/Discord need workspace admin.

**Fallback:** Web Chat works instantly with zero setup. Always available as a "try it now" option.

#### Step 2: Connect Google Account (60 seconds)
```
┌─────────────────────────────────────────┐
│  Connect your Google account to unlock: │
│                                         │
│  📧 Email management & inbox clearing   │
│  📅 Calendar & scheduling               │
│  📁 Drive access (optional)             │
│                                         │
│  [Connect with Google →]                │
│  [Skip for now]                         │
└─────────────────────────────────────────┘
```

**Implementation:** Managed OAuth app (Moltbot SaaS is the OAuth client). User clicks one button → standard Google consent screen → tokens stored server-side. No GCP Console needed.

#### Step 3: Personalize (45 seconds)
```
┌─────────────────────────────────────────┐
│  Tell Moltbot about you:                │
│                                         │
│  Name: [Christopher____________]        │
│  Timezone: [America/Los_Angeles ▼]      │
│  Wake-up time: [7:00 AM_________]       │
│                                         │
│  Morning briefing: ☑ On (recommended)   │
│  Briefing includes:                     │
│    ☑ Calendar overview                  │
│    ☑ Email highlights                   │
│    ☑ Weather                            │
│    ☐ Health data (connect later)        │
│    ☐ News summary                       │
│                                         │
│  [Start Using Moltbot →]               │
└─────────────────────────────────────────┘
```

#### Step 4: First Value Moment (< 5 minutes)

**Immediate automated actions after setup:**

1. **Moltbot sends first message** (proactive, not reactive):
   > "Hey Christopher! 👋 I'm Moltbot, your AI assistant. I'm now connected to your email and calendar. Here's what I found:
   > 
   > 📧 **Inbox:** 847 unread emails. Want me to clear the noise? I can unsubscribe from 23 obvious newsletters, archive 312 promotions, and flag 15 emails that need your reply.
   > 
   > 📅 **Today:** You have 3 meetings. Your first is in 2 hours (Team standup at 10am).
   > 
   > What should I tackle first?"

2. **User says "clear my inbox"** → Moltbot processes emails in batches, reporting progress:
   > "Working on it... 🧹
   > ✅ Unsubscribed from 23 mailing lists
   > ✅ Archived 312 promotional emails  
   > ✅ Categorized 187 notifications
   > 📌 15 emails flagged for your review — here are the top 3..."

3. **First morning briefing arrives** the next day at their wake-up time.

**This is the critical retention moment.** If the user's inbox is cleaner within 5 minutes and they get a useful briefing the next morning, they're hooked.

### Value Demonstration Timeline

| Timeframe | What Happens | Retention Signal |
|---|---|---|
| **0–5 min** | First message, inbox scan, email summary | "This is real" |
| **5–30 min** | Inbox clearing, calendar overview, first research task | "This saves me time" |
| **Day 1 morning** | First automated briefing (calendar + email + weather) | "I want this every day" |
| **Day 2–3** | User asks for research, document processing, or scheduling | "This replaces 3 tools" |
| **Week 1** | Cron jobs running, memory accumulating, habits forming | "I can't go back" |
| **Week 2** | Suggests connecting more tools, shows usage stats | Trial → paid conversion |

---

## Part 4: Default Skills & Configuration

### Pre-Installed Skills

| Skill | Source | Purpose | Tier |
|---|---|---|---|
| `gmail-pubsub` | Bundled | Real-time email processing | Starter |
| `github` | ClawdHub | Git operations, PR management | Pro |
| `coding-agent` | ClawdHub | Codex/Claude Code orchestration | Pro |
| `obsidian-daily` | ClawdHub | Daily note creation/management | Starter |
| `second-brain` | ClawdHub | PKM with embeddings | Starter |
| `exa` | ClawdHub | Neural web search | Pro |
| `kagi-search` | ClawdHub | High-quality web search | Pro |
| `homeassistant` | ClawdHub | Smart home control | Business |
| `slack` | Bundled | Slack channel control | Starter |
| `discord` | Bundled | Discord integration | Starter |
| `frontend-design` | ClawdHub | UI/UX generation | Pro |
| `deploy-agent` | ClawdHub | Deployment automation | Business |

### Channel Configuration Priority

| Channel | Priority | Rationale | Setup Effort |
|---|---|---|---|
| **Web Chat** | Default (always on) | Zero friction, works immediately | None |
| **Telegram** | Primary recommended | Best bot API, richest features, easy setup | Click a link |
| **WhatsApp** | Secondary | Most popular messenger globally, but needs Business API | Phone verification |
| **Slack** | Workspace option | Best for work contexts | App install in workspace |
| **Discord** | Community option | Popular with devs/gamers | Bot token setup |
| **Signal** | Privacy option | For privacy-focused users | Phone number |
| **iMessage** | Apple ecosystem | Requires Mac node (BlueBubbles) | Node pairing + setup |
| **Email** | Passive | Receive briefings via email (no chat) | Auto-configured |

### Default Tools (Enabled by Tier)

| Tool | Starter | Pro | Business | Notes |
|---|---|---|---|---|
| `web_search` | ✅ | ✅ | ✅ | Brave Search API, managed |
| `web_fetch` | ✅ | ✅ | ✅ | URL content extraction |
| `read` | ✅ | ✅ | ✅ | File reading |
| `write` | ✅ | ✅ | ✅ | File creation/editing |
| `edit` | ✅ | ✅ | ✅ | Precise file edits |
| `exec` | ❌ | ✅ (sandboxed) | ✅ (elevated) | Shell commands |
| `process` | ❌ | ✅ | ✅ | Background process management |
| `browser` | ❌ | ✅ (limited) | ✅ (full) | Playwright browser control |
| `canvas` | ❌ | ✅ | ✅ | UI rendering |
| `cron` | ✅ (5 jobs) | ✅ (25 jobs) | ✅ (unlimited) | Scheduled tasks |
| `nodes` | ❌ | ❌ | ✅ | Device pairing |
| `tts` | ✅ (Edge) | ✅ (OpenAI) | ✅ (OpenAI HD) | Voice synthesis |
| `image` | ✅ (5/day) | ✅ (50/day) | ✅ (unlimited) | Vision analysis |
| `message` | ✅ | ✅ | ✅ | Send messages across channels |

### Default Model Routing Configuration

```jsonc
{
  "models": {
    "routing": {
      // Starter tier: cost-optimized
      "starter": {
        "primary": "google/gemini-3-flash",        // ~$0.10/M input — fast, cheap
        "complex": "anthropic/claude-sonnet-4-5",   // ~$3/M input — when Flash fails
        "reasoning": "anthropic/claude-sonnet-4-5",  // thinking mode for hard tasks
        "vision": "google/gemini-3-flash",
        "embedding": "google/gemini-embedding-001",
        "transcription": "openai/gpt-4o-mini-transcribe"
      },
      // Pro tier: quality-balanced
      "pro": {
        "primary": "anthropic/claude-sonnet-4-5",   // Best balance of quality/cost
        "complex": "anthropic/claude-opus-4-5",     // Heavy lifting
        "reasoning": "anthropic/claude-opus-4-5",
        "vision": "anthropic/claude-sonnet-4-5",
        "embedding": "google/gemini-embedding-001",
        "transcription": "openai/gpt-4o-mini-transcribe",
        "coding": "anthropic/claude-opus-4-5"       // Coding tasks get Opus
      },
      // Business tier: maximum capability
      "business": {
        "primary": "anthropic/claude-opus-4-5",
        "complex": "anthropic/claude-opus-4-5",
        "reasoning": "anthropic/claude-opus-4-5",   // thinking=high
        "vision": "anthropic/claude-opus-4-5",
        "embedding": "google/gemini-embedding-001",
        "transcription": "openai/gpt-4o-mini-transcribe",
        "coding": "anthropic/claude-opus-4-5",
        "orchestrator": "anthropic/claude-opus-4-5" // Multi-agent delegation
      }
    },
    // Smart routing rules (all tiers)
    "autoRoute": {
      "simpleQueries": "flash",        // "what time is it?" → Flash
      "emailTriage": "flash",          // Sorting/categorizing → Flash
      "emailDraft": "primary",         // Writing responses → primary model
      "research": "complex",           // Deep research → complex model
      "coding": "coding",             // Code gen → coding model
      "briefing": "primary",          // Morning briefings → primary
      "heartbeat": "flash"            // Periodic check-ins → Flash (cheap)
    }
  }
}
```

**Cost optimization strategy:**
- Heartbeats and simple triage use Gemini Flash (~$0.10/M tokens) — this is where most token volume goes
- Email categorization uses Flash (high volume, low complexity)
- Drafting, briefings, and conversations use Sonnet (balanced)
- Research, coding, and complex reasoning escalate to Opus (expensive but justified)
- **Expected cost per user:** Starter ~$3–8/mo in API costs, Pro ~$15–30/mo, Business ~$40–80/mo

### Heartbeat & Proactive Configuration

```jsonc
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "activeHours": { "start": "08:00", "end": "22:00" },
        "model": "google/gemini-3-flash",  // Cheap for check-ins
        "target": "last",
        "ackMaxChars": 300
      },
      "briefing": {
        "schedule": "0 7 * * *",  // 7 AM daily (user's timezone)
        "model": "anthropic/claude-sonnet-4-5",
        "sections": ["calendar", "email_highlights", "weather", "tasks"],
        "deliver": true
      }
    }
  }
}
```

### Security Defaults for SaaS

```jsonc
{
  "sandbox": {
    "mode": "always",           // All code execution in Docker
    "perSession": true,         // Isolated per session
    "docker": {
      "image": "moltbot-saas:bookworm",
      "readOnlyRoot": true,
      "network": "restricted",  // Allowlist for APIs only
      "user": "1000:1000",
      "memoryLimit": "512m",    // Starter
      "cpuShares": 256
    }
  },
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowedCommands": [
        "gog", "gh", "pandoc", "pdftotext",
        "curl", "jq", "node", "python3"
      ]
    },
    "elevated": { "enabled": false },  // No elevated access in SaaS
    "browser": {
      "maxConcurrent": 1,       // Starter: 0, Pro: 1, Business: 3
      "maxPageLoadMs": 30000,
      "blockDomains": ["*.bank.*", "*.gov.*"]  // Safety
    }
  }
}
```

---

## Part 5: Pricing Tier Mapping

### Tier Overview

| Feature | Starter ($19/mo) | Pro ($49/mo) | Business ($99/mo) |
|---|---|---|---|
| **Chat Channels** | 2 channels | 4 channels | Unlimited |
| **Model Quality** | Flash + Sonnet | Sonnet + Opus | Opus primary |
| **Monthly Token Budget** | ~5M tokens | ~25M tokens | ~100M tokens |
| **Cron Jobs** | 5 | 25 | Unlimited |
| **Heartbeat** | Every 2h | Every 30m | Every 15m |
| **Browser Automation** | ❌ | ✅ (20 sessions/mo) | ✅ (unlimited) |
| **Code Execution** | ❌ | ✅ (sandboxed) | ✅ (elevated) |
| **Node Pairing** | ❌ | ❌ | ✅ (3 devices) |
| **Memory Search** | Basic keyword | Embedding-based | Embedding + cross-reference |
| **Voice/TTS** | Edge TTS (free) | OpenAI TTS | OpenAI HD TTS |
| **Skills** | 10 installed | 50 installed | Unlimited |
| **Sessions** | 1 concurrent | 3 concurrent | 10 concurrent |
| **Support** | Community | Email + Discord priority | Dedicated onboarding |

### Use Case → Tier Mapping

| Use Case | Starter ($19) | Pro ($49) | Business ($99) |
|---|---|---|---|
| **1. Email Management** | ✅ Full (Gmail, Outlook) | ✅ Full + advanced rules | ✅ Full + multi-account |
| **2. Calendar & Scheduling** | ✅ Full (Google Cal) | ✅ Full + multi-calendar | ✅ Full + team scheduling |
| **3. Flight & Travel** | 📊 Tracking only (email parse) | ✅ Full (browser check-in) | ✅ Full + booking |
| **4. Coding & Dev** | ❌ | ✅ Sandboxed (1 agent) | ✅ Multi-agent + Codex |
| **5. Health & Fitness** | 📊 Manual entry only | ✅ Oura/WHOOP API | ✅ Full + Apple Health (node) |
| **6. Smart Home** | ❌ | ❌ | ✅ Full (node required) |
| **7. Research & Content** | ✅ Basic (web search) | ✅ Deep (browser + Exa) | ✅ Full pipeline + scheduling |
| **8. Document Processing** | ✅ Basic (read/write) | ✅ Full (PDF, OCR, convert) | ✅ Full + batch processing |
| **9. Insurance & Admin** | ❌ | ✅ With approval workflow | ✅ Full automation |
| **10. PKM** | ✅ Built-in memory | ✅ + Obsidian/Notion sync | ✅ + cross-vault search |

### Revenue Projections & Unit Economics

| Metric | Starter | Pro | Business |
|---|---|---|---|
| **Price** | $19/mo | $49/mo | $99/mo |
| **Estimated API cost** | $3–8/mo | $15–30/mo | $40–80/mo |
| **Compute cost** | $2/mo (shared) | $5/mo (sandbox) | $15/mo (dedicated) |
| **Gross margin** | 45–70% | 30–55% | 5–40% |
| **Target mix** | 60% of users | 30% of users | 10% of users |
| **Break-even users** | ~500 | ~300 | ~100 |

**Note on margins:** The Business tier has thin margins at heavy usage. Consider:
- Usage-based overflow pricing above token budget ($5/M additional tokens)
- Annual billing discount (20% off → better cash flow, locks in users)
- Token budget as soft limit with alerts, not hard cutoff

### Competitive Positioning

| Competitor | Price | What They Do | Moltbot Advantage |
|---|---|---|---|
| ChatGPT Plus | $20/mo | Chat only, no automation | Moltbot acts, doesn't just answer |
| Claude Pro | $20/mo | Chat only, forgets between sessions | Persistent memory, proactive |
| Superhuman | $30/mo | Email only | Moltbot does email + everything else |
| Reclaim.ai | $8–18/mo | Calendar only | Moltbot does calendar + everything else |
| Cursor | $20/mo | Coding only | Moltbot does coding + personal assistant |
| Perplexity Pro | $20/mo | Research only | Moltbot does research + acts on findings |
| **Moltbot Starter** | **$19/mo** | **Email + Calendar + Research + Memory + Chat** | **Replaces 3–4 single-purpose tools** |
| **Moltbot Pro** | **$49/mo** | **All above + Coding + Browser + Deep Research** | **Replaces 5–6 tools** |

**Pricing message:** "Replace your AI assistant, email tool, calendar app, research service, and knowledge manager — all for $19/mo."

---

## Appendix A: Risk Factors & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **API cost spikes** (user triggers expensive loop) | Margin destruction | Per-user daily token budgets, auto-downgrade to Flash when budget exhausted, circuit breakers on runaway agents |
| **Gmail OAuth scope review** (Google may restrict) | Blocks core feature | Apply for Google verification early, prepare for sensitive scope review, support JMAP as fallback |
| **Browser automation fragility** (sites break, CAPTCHAs) | User frustration | Keep browser features in Pro+ tier, set expectations, provide graceful fallbacks |
| **Security incidents** (prompt injection via email) | Reputation damage | Sandboxed execution, input sanitization, approval workflows for sensitive actions, audit logging |
| **Anthropic API pricing changes** | Margin squeeze | Multi-model routing (can shift to Gemini/GPT), negotiate volume pricing, reserve capacity |
| **Self-hosted competition** (it's open source) | Cannibalizes SaaS | SaaS value is convenience + managed infra + pre-built integrations. Power users will self-host; that's fine. Target the 90% who won't. |

## Appendix B: Launch Sequence

| Phase | Timeline | Milestone |
|---|---|---|
| **Alpha** | Weeks 1–4 | 50 hand-picked users, Web Chat + Telegram only, Starter tier features |
| **Beta** | Weeks 5–8 | 500 users, WhatsApp + Slack added, Pro tier features, billing active |
| **Public Launch** | Weeks 9–12 | Open signups, all channels, all tiers, ClawdHub marketplace |
| **Scale** | Months 4–6 | Team/business features, node management UI, enterprise pilot |

## Appendix C: Key Community Quotes (Demand Signals)

> "Added JMAP search to my Clawdbot. Along with 20 other things. From my phone." — @jokull

> "Apparently @clawdbot checks in during heartbeats!? A kinda awesome surprise!" — @HixVAC

> "Using Clawdbot is the first time since the beginning of LLM that I genuinely feel like I'm talking to J.A.R.V.I.S. from Iron Man." — r/LocalLLM user

> "Current level of open-source apps capabilities: does everything, connects to everything, remembers everything. It's all collapsing into one unique personal OS." — @jakubkrcmar

> "I woke up to a $120 API bill. Here is the fix..." — r/LocalLLM (validates need for managed cost control)

> "It's a fricking wrapper with a pipe to Whatsapp and Cron jobs." — r/LocalLLM skeptic (this is exactly the value prop — simple infra, massive capability)

---

*End of brief. This document should be revisited monthly as the community evolves and usage data from alpha/beta users becomes available.*
