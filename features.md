# AI Voice Agent Builder — Feature & Functionality Survey

> Candidate #349 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Vapi | API-first SaaS | Commercial / SaaS | https://vapi.ai |
| Retell AI | End-to-end SaaS | Commercial / SaaS | https://www.retellai.com |
| Bland AI | Infrastructure SaaS | Commercial / SaaS | https://www.bland.ai |
| Synthflow | No-code SaaS | Commercial / SaaS | https://synthflow.ai |
| Voiceflow | Visual builder SaaS | Commercial / SaaS (free tier) | https://www.voiceflow.com |
| Cognigy | Enterprise platform | Commercial / enterprise | https://www.cognigy.com |
| Sierra AI | Enterprise SaaS | Commercial / enterprise | https://sierra.ai |
| Lindy | General-purpose AI agent | Commercial / SaaS (free tier) | https://www.lindy.ai |
| LiveKit Agents | Open-source framework | Apache 2.0 | https://github.com/livekit/agents |
| Rasa | Open-source conversational AI | MIT / commercial (Rasa Pro) | https://rasa.com |
| ElevenLabs Conversational AI | TTS-native agent platform | Commercial / SaaS | https://elevenlabs.io/docs/conversational-ai/overview |
| OpenAI Realtime API | Speech-to-speech API | Commercial / API | https://developers.openai.com/api/docs/guides/realtime |

---

## Feature Analysis by Solution

### Vapi

**Core features**
- Modular orchestration layer over STT, LLM, and TTS — each component is swappable (OpenAI, Anthropic, Claude, Deepgram, ElevenLabs, etc.)
- REST API with programmatic control over every configuration parameter
- Inbound and outbound call support via Twilio, SIP trunking, and WebRTC
- Squads — orchestrate multiple specialised assistants with context-preserving transfers between them
- Visual conversation flow builder for multi-step branching logic (launched 2025/26)
- Advanced analytics dashboard: real-time insights, sales/conversion tracking, custom dashboards
- Automated evaluations: run agents against expected behaviours before deployment
- Webhook system for routing call events and transcripts to external systems
- Handles 1 million+ concurrent calls

**Differentiating features**
- Developer-first: every feature is API-accessible, giving the richest programmatic surface of any platform
- Squads architecture enables true multi-agent orchestration within a single call
- Broadest provider choice: mix-and-match any STT, TTS, and LLM provider combination

**UX patterns**
- Dashboard and CLI for configuration; code-first onboarding
- Steep initial learning curve; targets engineering teams
- Latency optimisations built in (700–900 ms with fast providers)

**Integration points**
- REST API, webhooks, programmatic SDKs
- STT: Deepgram, AssemblyAI, Whisper, Gladia
- TTS: ElevenLabs, Azure, Play.ht, OpenAI TTS, Cartesia
- LLM: OpenAI, Anthropic, Google Gemini, Groq, custom endpoints
- Telephony: Twilio, Vonage, Plivo, SIP trunking

**Known gaps**
- Complex pricing: platform fee plus separate STT, TTS, LLM, and telephony costs — hard to predict total bill
- No native no-code experience; non-technical users are blocked without engineering support
- Compliance features (HIPAA BAA, PCI DSS) require enterprise tier

**Licence / IP notes**
- Proprietary SaaS; no open-source components. Orchestration logic is closed-source.

---

### Retell AI

**Core features**
- Proprietary Voice AI Orchestration with ~600 ms latency for natural-feeling conversations
- Conversation Flow Agents (structured nodes/transitions with drag-and-drop builder) and Prompt-based Agents (flexible, LLM-native)
- 31+ languages with native-quality speech
- Built-in testing and QA: simulate conversations, A/B test flows, monitor live calls
- Outbound campaign management: batch calling, scheduling, and results dashboard
- CRM integration (HubSpot, Salesforce, GoHighLevel, etc.) with live action-triggering during calls
- Voicemail detection and handling
- SOC 2 Type II, HIPAA, and GDPR certified

**Differentiating features**
- Transparent usage-based pricing ($0.07/min all-in) — lowest total cost of comparable end-to-end platforms
- Native A/B testing of call flows within the platform
- Sentiment tracking (CSAT, latency, outcomes) in a unified dashboard

**UX patterns**
- Targets both developers and non-technical operations managers
- Drag-and-drop Conversation Flow Builder for structured IVR-style flows; Prompt Agent for open-ended use cases
- Hours-to-go-live for common use cases; onboarding is faster than Vapi for non-developers

**Integration points**
- REST API, webhooks, Zapier, Make, and native CRM integrations
- Twilio, SIP, BYOC (Bring Your Own Carrier)
- Salesforce, HubSpot, GoHighLevel, Calendly, Google Calendar

**Known gaps**
- Less customisable than Vapi for unusual pipeline configurations
- No native multi-agent (Squads-style) orchestration within a single call
- Fewer STT/TTS provider choices — more opinionated stack

**Licence / IP notes**
- Proprietary SaaS. Compliance certifications (SOC 2 Type II, HIPAA) are on all tiers.

---

### Bland AI

**Core features**
- Infrastructure and platform for high-volume AI call centres
- Conversational Pathways: map out support flows with custom branching logic mirroring business rules
- Up to 20,000 calls per hour; claimed capacity of 1 million concurrent calls
- Knowledge Base with automatic gap detection: identifies unanswered questions during real calls
- Real-time SMS triggering during calls (send links, confirmations, next steps mid-conversation)
- SIP and Twilio routing with real-time call tracking
- Call recordings with automated transcription and analytics
- Integrations with HubSpot, Slack, CRM webhooks
- SOC 2, HIPAA, and GDPR compliant

**Differentiating features**
- Highest raw throughput of any platform — engineered for outbound campaign scale
- Real-time SMS companion messaging during voice calls
- Knowledge Base gap detection as a first-class feature (surfaces what agents can't answer)
- Edge delivery network to reduce latency globally

**UX patterns**
- Developer-oriented API; also offers Conversational Pathways as a more visual alternative
- Pricing model charges for failed calls, which surprises some users
- Positioned as "infrastructure" — expects users to bring their own business logic

**Integration points**
- REST API, webhooks
- SIP trunking, Twilio
- HubSpot, Slack, custom CRM via webhooks
- Voicemail detection and retry logic

**Known gaps**
- Billing for failed/unanswered calls is a common complaint
- Less polished no-code experience than Synthflow or Retell
- Fewer pre-built templates vs. Synthflow

**Licence / IP notes**
- Proprietary SaaS; no open-source components.

---

### Synthflow

**Core features**
- No-code drag-and-drop builder: configure, prompt, add actions, deploy, and test in real time without code
- Conditional logic ("if user says X, do Y") for dynamic conversation branching
- Inbound and outbound call support with voicemail detection and AI IVR routing
- Appointment booking integrations (Google Calendar, Calendly, Cal.com, Acuity Scheduling)
- 200+ native integrations across CRM, scheduling, automation, and e-commerce tools
- Sub-100 ms in-house telephony latency; carrier-grade reliability without replacing existing SIP/PBX
- 20+ pre-built templates for service industry use cases (HVAC, dental, law firm, real estate, e-commerce)
- SOC 2, HIPAA, PCI DSS, and GDPR certified (HIPAA BAA is a 30% premium add-on)
- White-label option for agencies

**Differentiating features**
- Best no-code experience in the market — designed for non-technical business owners and agencies
- Agency white-label tier with reseller economics built in
- Most comprehensive pre-built template library for service businesses
- Broadest native integration count (200+) with no webhook required for the common service-business stack

**UX patterns**
- Onboarding targets business owners, not developers; no code required end-to-end
- Deploy within a day using templates; deployment timeline under 3 weeks for custom builds
- Progressive disclosure: simple defaults with advanced conditional logic accessible when needed

**Integration points**
- 200+ native integrations: HubSpot, Salesforce, Pipedrive, GoHighLevel, Zoho, Google Calendar, Calendly, Slack, Notion, Airtable, Make, Zapier, and more
- SIP trunking and existing PBX compatibility

**Known gaps**
- Limited customisation depth for complex or non-standard call flows vs. Vapi/Retell
- More expensive at higher volumes than usage-based competitors
- HIPAA BAA is a paid add-on (30% premium) rather than included

**Licence / IP notes**
- Proprietary SaaS. White-label terms exist for agency tier. No open-source.

---

### Voiceflow

**Core features**
- Visual canvas drag-and-drop builder with real-time collaboration (comments, role-based permissions, shared workspaces)
- Design once, deploy to both chat and voice/IVR channels from the same flow
- Built-in LLMs, NLU, and vector-based Knowledge Base for custom data/answers
- Plug-in any LLM, API, or backend system (reduces vendor lock-in)
- Reusable modular components; pro-code features with custom JavaScript
- Built-in analytics: transcripts, custom evaluation criteria, user path visualisation, step-by-step debugger
- Multi-LLM routing within a single agent
- Real-time collaboration for design/product/engineering teams
- Integrations: Salesforce, Shopify, Zendesk, Snowflake, CRM/database webhooks

**Differentiating features**
- Best conversation designer UX of any platform — non-technical stakeholders can fully participate
- Omnichannel single-flow deployment: same logic for chat and voice, reducing duplication
- Prototype-to-production pathway with real stakeholder collaboration tooling
- Reusable component library encourages organisational knowledge sharing

**UX patterns**
- Targets product teams at mid-market and enterprise; assumes mixed technical/non-technical teams
- Visual-first design environment; code is optional but available
- Prototyping is fast; voice capability requires connecting telephony providers separately

**Integration points**
- Integrates with major CRMs, databases, messaging platforms, and telephony via API/webhook
- Works with ElevenLabs for custom voice models
- Telephony via Twilio, Vonage (not native — requires external carrier)

**Known gaps**
- Voice/telephony is secondary to its chat origins — fewer telephony-native features vs. Vapi/Retell
- Does not include built-in telephony; must bring own carrier (Twilio, Vonage)
- Less suitable for outbound campaign management

**Licence / IP notes**
- Proprietary SaaS; free tier available. No open-source components.

---

### Cognigy

**Core features**
- Agentic AI platform for enterprise contact centres: inbound and outbound voice and chat
- Deepgram Flux speech recognition: context-aware turn detection, 200–600 ms latency reduction vs. pipeline approaches
- 100+ languages; 24/7 operation
- Automation discovery: analyses engagement data to identify where automation opportunities exist
- Voice gateway with plug-and-play integration for Avaya, Amazon Connect, Genesys, and other CCaaS platforms
- Omnichannel (voice + chat) with unified agent context
- Advanced agentic AI: multi-step reasoning, configurable loop limits, fallback behaviours
- Enterprise security and compliance (SOC 2, HIPAA, GDPR, custom deployments)
- AI Agent Control: developers can set maximum reasoning loops and guard against runaway agents

**Differentiating features**
- Deepest CCaaS (contact centre as a service) integrations of any platform — native to Avaya, Amazon Connect, Genesys
- Automation discovery provides a data-driven way to identify which call types to automate next
- Handles tens of thousands of concurrent voice calls at enterprise scale

**UX patterns**
- IT and operations-team oriented; implementation-partner deployment model typical
- Slow to deploy vs. SaaS-native competitors — enterprise procurement and integration cycle
- Advanced configuration controls for agentic behaviour (loop limits, fallbacks) preferred by compliance-sensitive teams

**Integration points**
- Avaya, Amazon Connect, Genesys (native integrations)
- Deepgram, ElevenLabs, AWS, Azure TTS/STT
- REST API, webhooks, enterprise SSO
- SAP, Salesforce, ServiceNow, and major enterprise CRMs

**Known gaps**
- Enterprise-only pricing; no self-serve or SMB tier
- Slow time-to-deploy (weeks to months) compared to API-first competitors
- Higher total cost of ownership than SaaS alternatives

**Licence / IP notes**
- Proprietary enterprise software. No open-source.

---

### Sierra AI

**Core features**
- Goal-oriented AI agent design: optimises for call resolution rather than scripted flow completion
- Proprietary voice stack: natural pacing, interruption-friendly turn-taking, empathetic tone, low latency
- Multi-model architecture (OpenAI, Anthropic, Meta) for reliability and hallucination reduction with automatic fallbacks
- Strong brand-safety controls: shape tone, vocabulary, and context handling to reflect brand identity
- Policy enforcement and compliance governance: trace and audit how decisions are made
- Emotionally intelligent interaction handling: detects frustration, urgency, and heightened customer emotions

**Differentiating features**
- Only platform explicitly designed around brand safety as a first-class product principle
- Goal-oriented resolution design vs. turn-by-turn scripted flows
- Multi-model redundancy for enterprise uptime guarantees
- Premium emotionally intelligent voice model tuned for sensitive interactions (e.g., ADT security, healthcare)

**UX patterns**
- Enterprise-only with no self-serve; implementation is a white-glove engagement
- Targets large consumer-facing brands where brand voice consistency is mission-critical
- $10B valuation (2026) positions it as the premium enterprise option

**Integration points**
- Custom enterprise integrations — specifics not publicly documented
- Multi-model LLM routing (OpenAI, Anthropic, Meta)

**Known gaps**
- No self-serve access; inaccessible to SMBs and startups
- Pricing and features not publicly disclosed
- No developer API/SDK documentation available

**Licence / IP notes**
- Proprietary enterprise SaaS. Closed source. No public API.

---

### Lindy

**Core features**
- Gaia voice calling system: AI agents can make and receive phone calls autonomously
- 2,500+ app integrations via Pipedream partnership; hundreds of native integrations (Gmail, Slack, Notion, HubSpot)
- Multi-agent delegation: voice agent can simultaneously handle phone calls, trigger email follow-ups, update CRM, and coordinate with other AI agents from a single workflow
- Drag-and-drop workflow builder for cross-channel automation
- Agents can run calls, handle email, resolve tickets, process documents, manage calendars, and operate inside a virtual machine
- Inbound and outbound call support for customer support, sales outreach, appointment scheduling, and lead qualification
- Free tier; Pro from $49/mo; calls billed at $0.19/min with GPT-4o

**Differentiating features**
- Best cross-channel multi-agent coordination: voice is one input in a broader agentic workflow, not a standalone silo
- Deepest out-of-the-box app integration count of any voice platform (2,500+ via Pipedream)
- Enables agentic workflows that span voice, email, document processing, and CRM in a single flow

**UX patterns**
- No-code workflow builder appeals to operations and business users
- Voice is one channel among many; not telephony-native like Vapi or Retell
- Lower barrier to entry than Vapi; broader general-agent positioning vs. voice-specialist platforms

**Integration points**
- 2,500+ apps via Pipedream; native integrations with Gmail, Slack, Notion, HubSpot, Salesforce
- Telephony via Gaia; call billing is separate ($0.19/min)

**Known gaps**
- Higher per-minute cost ($0.19) than pure-voice competitors (Retell at $0.07)
- Fewer telephony-native features (IVR design, voicemail detection, outbound campaign management)
- Not designed for contact-centre scale or high-volume calling

**Licence / IP notes**
- Proprietary SaaS. Free tier available. No open-source components.

---

### LiveKit Agents

**Core features**
- Open-source framework (Apache 2.0) for building real-time voice, video, and multimodal AI agents
- Python and Node.js SDKs; agents run as full real-time participants in LiveKit rooms
- Semantic turn detection: transformer model determines when a user has finished speaking (reduces false interruptions)
- Native MCP (Model Context Protocol) support for tool calling
- Multi-agent handoff: complex workflows decomposed into simpler specialised agents
- Built-in test framework with judges to verify agent behaviour
- Telephony integration: inbound and outbound SIP calls via LiveKit's telephony stack
- Production-ready: built-in orchestration, load balancing, Kubernetes compatibility
- Extensive LLM, STT, and TTS provider integrations
- Agent Builder: browser-based no-code prototype/deploy tool (complements the code SDK)

**Differentiating features**
- Only fully open-source framework with production-grade telephony, load balancing, and Kubernetes support out of the box
- Semantic turn detection using a custom transformer model — more accurate than VAD-only approaches
- Native MCP support enables agents to consume any MCP server as tools without custom integration work
- Supports voice, video, and text modalities — broadest multimodal scope of any framework

**UX patterns**
- Developer-first; Python and Node.js SDK with clear documentation
- Agent Builder provides a no-code prototype path without requiring full SDK knowledge
- Open-source community; active GitHub (15,000+ stars as of 2026)

**Integration points**
- STT: Deepgram, AssemblyAI, Whisper, Silero, Azure, Google
- TTS: ElevenLabs, Cartesia, OpenAI, Azure, Google, Deepgram
- LLM: OpenAI, Anthropic, Google, Groq, Ollama, custom endpoints
- Telephony: LiveKit SIP, compatible with standard SIP trunks

**Known gaps**
- Requires self-hosting or LiveKit Cloud — more operational overhead than fully managed SaaS
- No built-in telephony number provisioning (requires a separate SIP trunk/carrier)
- Analytics, monitoring, and evaluation tooling is less mature than Vapi/Retell

**Licence / IP notes**
- Apache 2.0 open source. LiveKit Cloud is commercial. No patent concerns identified.

---

### Rasa

**Core features**
- Open-source ML framework for text- and voice-based conversational AI
- CALM engine (Conversational AI with Language Models): combines LLM fluency with deterministic business logic
- NLU component for intent/entity recognition in custom voice flows
- STT and TTS integration support for voice agent development
- Built-in copilot: AI assistant that helps generate code, debug flows, and expand agents
- Rasa Pro enterprise tier with additional governance and deployment tooling
- Compatible with Alexa, WhatsApp, and other channels

**Differentiating features**
- CALM engine: unique hybrid of LLM flexibility and rule-based reliability — reduces hallucinations in production
- Longest-running open-source conversational AI framework with large community
- Full self-hosted deployment with no per-minute fees

**UX patterns**
- Developer and ML-engineer focused; requires Python knowledge
- Training-based NLU pipeline; less suited to zero-shot LLM-only use cases without CALM
- Higher engineering overhead vs. API-first SaaS platforms

**Integration points**
- Connects to WhatsApp, Slack, Alexa, Facebook Messenger, and custom channels
- STT/TTS via third-party integrations (Deepgram, Google, AWS)
- REST API and webhook support

**Known gaps**
- Native telephony support is limited; requires third-party SIP/IVR integration
- Steeper learning curve; slower to deploy than managed SaaS
- Enterprise features (Rasa Pro) move it into commercial licensing territory

**Licence / IP notes**
- Rasa Open Source: MIT licence. Rasa Pro: commercial. No patent concerns identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Inbound and outbound call handling via SIP/PSTN and WebRTC
- Low-latency speech pipeline (STT + LLM + TTS) with interruption handling
- Voicemail detection and graceful handling
- Call recording, transcription, and storage
- Webhook/event-based integration with external systems
- Basic analytics (call duration, completion rate, transcripts)
- At least one of: drag-and-drop builder, REST API, or SDK
- HIPAA and GDPR compliance certification (essential for regulated sectors)

### Differentiating Features
- Squads / multi-agent orchestration within a single call (Vapi)
- Semantic turn detection using transformer models rather than voice activity detection alone (LiveKit)
- Goal-oriented resolution design rather than scripted flow completion (Sierra)
- Automation discovery: data-driven identification of which call types to automate next (Cognigy)
- Real-time SMS companion messaging during voice calls (Bland)
- Knowledge Base gap detection — surfaces unanswered questions post-call (Bland)
- Agency white-label and reseller economics (Synthflow)
- Cross-channel multi-agent workflow (voice + email + CRM in one pipeline) (Lindy)
- MCP native tool calling (LiveKit)
- Omnichannel single-flow design across chat and voice (Voiceflow)

### Underserved Areas / Opportunities
- **Adversarial caller simulation**: most platforms lack built-in simulation of frustrated, confused, or off-script callers for pre-deployment stress testing; dedicated third-party tools (Hamming, Coval, Cekura) are emerging but not integrated into any major platform
- **Warm transfer with full context handoff**: only 40% of platforms implement warm transfer well; cold transfers with lost context are the most common customer experience failure point
- **Multilingual mid-call language switching**: no platform supports seamless detection and switching between languages mid-conversation without pre-configuring separate agents
- **PII redaction and consent management as default**: compliance features are add-ons or enterprise-tier; no platform ships automated PII redaction in transcripts and consent logging as standard
- **Predictable all-in pricing**: the dominant complaint is billing complexity; no platform offers a single transparent all-in per-minute price covering STT + LLM + TTS + telephony
- **Emotional intelligence metrics**: tone and frustration detection exists in some platforms but is rarely surfaced as an actionable dashboard metric with escalation triggers
- **Open-source end-to-end platform**: LiveKit provides the framework but no full platform (number provisioning, analytics, compliance dashboard); Rasa has no native telephony — no single open-source project covers the full stack

### AI-Augmentation Candidates
- **Turn detection**: replacing simple VAD (voice activity detection) with transformer-based semantic turn detection dramatically reduces false interruptions — a well-studied AI improvement
- **Automated evaluation**: LLM-as-judge scoring of conversation transcripts against rubrics can replace expensive human QA reviewers
- **Adaptive latency routing**: AI-based selection of STT/TTS provider based on accent, language, and network conditions in real time
- **Caller sentiment and intent prediction**: predicting escalation risk and caller frustration before they verbalise it, enabling proactive tone shifts or preemptive human escalation
- **Knowledge base auto-expansion**: automatically generating new FAQ entries from gap-detection data (calls where the agent couldn't answer), closing the loop from production to training
- **Dynamic script personalisation**: using caller history, CRM data, and real-time sentiment to adapt the conversation tone and content rather than following a static prompt

---

## Legal & IP Summary

No patent concerns were identified for the open-source tools (LiveKit Agents under Apache 2.0; Rasa Open Source under MIT). All commercial platforms (Vapi, Retell, Bland, Synthflow, Voiceflow, Cognigy, Sierra, Lindy, ElevenLabs) operate under proprietary SaaS terms; replicating their specific UI designs, proprietary orchestration logic, or branded templates would risk IP issues and should be avoided. The CALM engine (Rasa) and Squads orchestration (Vapi) are distinctive architectural patterns but are not patented to the best of available information. An open-source AI-native voice agent builder can safely adopt the general architectural patterns (STT → LLM → TTS pipeline, drag-and-drop flow builders, webhook-based integrations) without licensing concerns, provided it does not directly copy closed-source code or bespoke UI.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Inbound and outbound call handling via SIP trunking + WebRTC with a bring-your-own-carrier model
- Configurable STT, LLM, and TTS provider pipeline (pluggable, not vendor-locked)
- Semantic turn detection with interruption handling (not VAD-only)
- Drag-and-drop conversation flow builder + REST API for developers
- Call recording, transcript storage, and basic analytics dashboard
- Webhook-based integration for CRM and automation tools
- HIPAA and GDPR compliance baked in (not an add-on); PII redaction in transcripts as standard

**Should-have (v1.1)**
- Multi-agent orchestration within a single call (Squads-style handoffs with context preservation)
- Warm transfer to human agents with full transcript + sentiment summary briefing
- Built-in adversarial caller simulation for pre-deployment testing
- Knowledge Base with automatic gap detection (surfaces unanswered questions from real calls)
- Outbound campaign management (batch scheduling, voicemail detection, retry logic)
- Native multilingual support with mid-call language detection and switching

**Nice-to-have (backlog)**
- White-label and agency reseller tier
- Native MCP server support for tool calling
- Emotional intelligence dashboard: real-time sentiment scoring with configurable escalation triggers
- Automation discovery: analyse call data to surface new automation candidates
- Per-call cost estimator and transparent all-in pricing calculator
- Pre-built industry templates (healthcare scheduling, law firm intake, real estate qualification)
