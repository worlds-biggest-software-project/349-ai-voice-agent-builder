# AI Voice Agent Builder

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for building voice agents that handle calls, IVR, and customer service with low latency, transparent pricing, and compliance baked in.

The AI Voice Agent Builder is a developer- and operator-friendly platform for designing, deploying, and operating conversational voice agents over real telephony. It targets contact centre teams, SMB operators, and developers who need to automate inbound and outbound calls without the billing complexity, vendor lock-in, or enterprise procurement cycles of incumbent platforms.

---

## Why AI Voice Agent Builder?

- **Billing complexity is the dominant complaint.** Incumbents like Vapi charge a platform fee on top of separately metered STT, TTS, LLM, and telephony — making total cost hard to predict. No major platform offers a single transparent all-in per-minute price.
- **Compliance is gated behind enterprise tiers.** HIPAA BAAs, PCI DSS handling, and PII redaction are typically add-ons (Synthflow charges a 30% premium for HIPAA) rather than first-class features, blocking regulated-industry adoption.
- **Latency is still pipeline-bound.** Most platforms assemble STT → LLM → TTS from commodity components and accept 1–2 second round-trip latency; a purpose-built architecture targeting sub-300 ms would feel meaningfully more human.
- **No open-source end-to-end platform exists.** LiveKit Agents (Apache 2.0) provides the framework but lacks number provisioning, analytics, and a compliance dashboard. Rasa lacks native telephony. There is no open-source project covering the full stack.
- **Cold transfers lose context.** Only about 40% of platforms implement warm transfer well — the most common customer experience failure point in production AI voice deployments.

---

## Key Features

### Telephony and Voice Pipeline

- Inbound and outbound call handling via SIP trunking and WebRTC with a bring-your-own-carrier model
- Configurable, pluggable STT, LLM, and TTS provider pipeline (no vendor lock-in)
- Semantic turn detection with interruption handling, not VAD-only
- Voicemail detection and graceful handling
- Call recording, transcript storage, and basic analytics dashboard

### Conversation Design and Developer Experience

- Drag-and-drop conversation flow builder paired with a REST API for developers
- Webhook-based integration for CRM and automation tools
- Built-in adversarial caller simulation for pre-deployment stress testing
- Knowledge Base with automatic gap detection that surfaces unanswered questions from real calls
- Multi-agent orchestration within a single call (Squads-style handoffs with context preservation)

### Compliance and Trust

- HIPAA and GDPR compliance baked in, not an add-on
- PII redaction in transcripts as standard
- Consent management and encrypted recording storage as first-class features

### Operations and Scale

- Outbound campaign management with batch scheduling, voicemail detection, and retry logic
- Warm transfer to human agents with full transcript and sentiment summary briefing
- Native multilingual support with mid-call language detection and switching

---

## AI-Native Advantage

The platform is designed around AI-native capabilities that incumbents have been slow to ship: low-latency voice-native architecture targeting sub-300 ms response times, semantic turn detection using transformer models rather than voice activity detection, and seamless mid-call language switching without pre-configuring separate agents. Compliance-grade automated PII redaction, knowledge base auto-expansion from gap-detection data, and warm transfer with full sentiment-aware context handoff make AI a structural product advantage rather than a feature bolt-on.

---

## Tech Stack & Deployment

The reference architecture is pluggable across STT (Deepgram, AssemblyAI, Whisper), TTS (ElevenLabs, Cartesia, Azure, OpenAI), and LLM providers (OpenAI, Anthropic, Google, Groq, custom endpoints), with telephony via standard SIP trunking and WebRTC. It interoperates with SIP, WebRTC, and MRCP-era enterprise IVR infrastructure, and integrates with CPaaS providers such as Twilio, Plivo, and Vonage. Deployment targets both self-hosted (Kubernetes-compatible) and managed cloud, with bring-your-own-carrier support.

---

## Market Context

The global conversational AI market was valued at approximately USD 13.2 billion in 2024 and is projected to exceed USD 49 billion by 2030, with enterprise contact centres representing the largest spending pool. Incumbent pricing ranges from roughly USD 0.07/min (Retell AI) to USD 0.15–0.30/min all-in (Vapi), with no-code tiers like Synthflow at USD 1,400/mo for 6,000 minutes. Primary buyers are contact centre operations managers, SMB owners replacing receptionists, outbound sales and lead qualification teams, healthcare and dental practice operators, and fintech and insurance firms automating outbound payment reminders.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
