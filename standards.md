# Standards & API Reference

> Project: AI Voice Agent Builder · Generated: 2026-05-04

## Industry Standards & Specifications

### Telephony & Real-Time Audio Standards

**SIP — Session Initiation Protocol (RFC 3261)**
- URL: https://datatracker.ietf.org/doc/html/rfc3261
- The foundational IETF standard for initiating, maintaining, modifying, and terminating multimedia communication sessions over IP. Any voice agent platform that connects to the PSTN (public switched telephone network) — inbound phone numbers, outbound dialling — must implement SIP or integrate with a SIP-compliant carrier. All major CPaaS providers (Twilio, Vonage, Plivo) expose SIP trunking based on this standard.

**SIP Telephony Device Requirements (RFC 4504)**
- URL: https://www.rfc-editor.org/rfc/rfc4504
- IETF RFC specifying requirements and configuration guidelines for SIP telephony devices. Relevant for implementing SIP trunk connections, SIP proxy interaction, and call routing within an AI voice agent platform.

**SIP Event Package for Voice Quality Reporting (RFC 6035)**
- URL: https://tools.ietf.org/html/rfc6035
- Defines how SIP endpoints can report real-time voice quality metrics (latency, jitter, packet loss). Relevant for building monitoring dashboards that surface call quality issues to platform operators.

**WebRTC — Web Real-Time Communication (W3C/IETF)**
- URL: https://www.w3.org/TR/webrtc/
- Browser-native open standard for real-time audio, video, and data communication. WebRTC handles NAT traversal, adaptive jitter buffering, echo cancellation, and noise suppression natively, and encrypts all media by default using DTLS-SRTP. For AI voice agents accessed via browser or web-app interfaces (rather than traditional telephony), WebRTC is the de-facto transport standard. Bridges between PSTN/SIP and WebRTC are required when both channels are needed.

**SRTP — Secure Real-time Transport Protocol (RFC 3711)**
- URL: https://datatracker.ietf.org/doc/html/rfc3711
- IETF standard for encrypting RTP media streams. WebRTC mandates SRTP/DTLS; SIP-based deployments should also implement SRTP to meet HIPAA and GDPR requirements for encrypting voice data in transit.

**MRCP — Media Resource Control Protocol (RFC 4463 / RFC 6787)**
- URL: https://datatracker.ietf.org/doc/html/rfc4463 (v1) / https://datatracker.ietf.org/doc/html/rfc6787 (v2)
- IETF protocol allowing client applications to control speech resources (ASR — automatic speech recognition, and TTS — text-to-speech synthesis) hosted on media servers. MRCP v2 uses SIP for session establishment. Primarily relevant for interoperating with legacy enterprise telephony infrastructure (Avaya, Genesys) that still uses MRCP-based speech servers rather than modern cloud STT/TTS APIs.

---

### Voice & Speech Markup Standards

**SSML — Speech Synthesis Markup Language 1.1 (W3C)**
- URL: https://www.w3.org/TR/speech-synthesis11/
- W3C recommendation providing an XML-based markup language for controlling aspects of speech synthesis output: pronunciation, volume, pitch, rate, pauses, and emphasis. All major TTS APIs (ElevenLabs, Google TTS, AWS Polly, Azure TTS, Deepgram) accept SSML input. Implementing SSML support in a voice agent builder allows precise control over how agents sound without custom model fine-tuning.

**Web Speech API (W3C Community Group)**
- URL: https://dvcs.w3.org/hg/speech-api/raw-file/tip/webspeechapi
- Browser-native JavaScript API for speech recognition (SpeechRecognition) and speech synthesis (SpeechSynthesis). Relevant for browser-based voice agent interfaces and testing tools; not applicable to server-side telephony pipelines.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard machine-readable API contract format. All major voice agent platforms (Vapi, Retell, Bland, ElevenLabs, Deepgram) publish OpenAPI 3.x specs for their REST APIs. An open-source voice agent builder should publish an OpenAPI 3.1 spec to enable SDK auto-generation, API explorer tooling, and third-party integrations.

**JSON Schema (IETF Draft)**
- URL: https://json-schema.org/specification
- Standard for describing and validating JSON data structures. Used for defining webhook payload schemas, agent configuration schemas, and call event data models within voice agent platforms.

**WebSocket Protocol (RFC 6455)**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- IETF standard for full-duplex communication channels over a single TCP connection. Essential for streaming audio in real time between browser/telephony clients and the AI processing pipeline (STT streaming, TTS streaming). All major voice agent platforms use WebSockets for the real-time audio stream.

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Open standard (Anthropic-originated, now multi-vendor) defining how AI agents discover, access, and interact with external tools, databases, and APIs via a JSON-RPC client-server architecture. As of 2026, MCP is natively supported by OpenAI, Google, Microsoft, AWS, LiveKit Agents, and all major AI frameworks. A voice agent builder that implements MCP allows agents to call any MCP-compliant tool server (CRM, calendar, knowledge base, etc.) without custom integration code.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- IETF standard for delegated authorisation. Used by voice agent platforms for granting third-party integrations (CRM, calendar, ticketing) access to call data and agent controls without sharing credentials. Required for any platform offering OAuth-based integration with HubSpot, Salesforce, Google Calendar, etc.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on OAuth 2.0; enables SSO (single sign-on) for enterprise customers. Contact centre buyers expect OIDC-compatible SSO with their identity providers (Okta, Azure AD).

**HIPAA — Health Insurance Portability and Accountability Act**
- URL: https://www.hhs.gov/hipaa/
- US federal regulation requiring physical, technical, and administrative safeguards for Protected Health Information (PHI). Voice recordings containing medical details are PHI. AI voice agent platforms used in healthcare must sign a Business Associate Agreement (BAA) with customers, encrypt data at rest and in transit, log access, and enforce minimum-necessary access controls. Violations carry penalties exceeding $1 million per incident.

**PCI DSS — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Industry security standard governing systems that process, store, or transmit payment card data. Voice agents that accept spoken payment card numbers over the phone bring those recordings into PCI DSS scope. Compliance requires PII/PAN redaction from transcripts, controlled recording storage, and annual QSA (Qualified Security Assessor) audits for higher merchant tiers.

**GDPR — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- EU regulation classifying voice recordings as personal data (and as biometric data when used to identify individuals). Voice agent platforms serving EU customers must implement: explicit consent collection before recording, real-time opt-out mechanisms, data subject access and erasure rights, data processing agreements (DPAs) with vendors, and breach notification within 72 hours. Fines up to 4% of global annual revenue.

**TCPA — Telephone Consumer Protection Act (US)**
- URL: https://www.fcc.gov/document/fcc-confirms-tcpa-applies-ai-technologies-generate-human-voices
- US federal law governing automated telephone calls and text messages. The FCC confirmed in February 2024 that AI-generated voices qualify as "artificial or pre-recorded voices" under TCPA, requiring prior express written consent before contacting mobile phones. Non-compliance carries $500–$1,500 per call in penalties with no cap — a 10,000-call campaign could expose $15 million in liability. Any outbound calling platform must implement consent management, do-not-call list enforcement, and calling-hours controls.

**SOC 2 Type II**
- URL: https://www.aicpa.org/interestareas/frc/assuranceadvisoryservices/aicpasoc2report
- AICPA auditing standard that verifies a service organisation's controls for security, availability, processing integrity, confidentiality, and privacy over a 12-month period. SOC 2 Type II certification is a baseline procurement requirement for enterprise buyers evaluating voice agent platforms. All leading platforms (Retell, Bland, Synthflow, Cognigy) hold SOC 2 Type II.

---

### MCP Server Specifications

**Model Context Protocol Specification**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- GitHub: https://github.com/modelcontextprotocol/specification
- MCP defines a JSON-RPC 2.0 based protocol enabling AI agents to discover and invoke tools exposed by MCP servers. Relevant to an AI voice agent builder as it provides a standardised way for voice agents to call tools (CRM lookups, calendar booking, knowledge base queries, ticketing) without one-off custom integrations. Native MCP support is now available in LiveKit Agents (Python 1.5.x), the OpenAI Agents SDK, Google Gemini, and Anthropic Claude, making it the emerging standard for agent tool calling.

---

## Similar Products — Developer Documentation & APIs

### Vapi
- **Description:** Developer-first API platform for building, testing, and deploying conversational voice AI agents. Orchestrates pluggable STT, LLM, and TTS providers with built-in telephony, analytics, and multi-agent Squads.
- **API Documentation:** https://docs.vapi.ai/
- **API Reference:** https://docs.vapi.ai/api-reference/assistants/create
- **SDKs/Libraries:** Python, TypeScript/JavaScript, and community SDKs documented at https://docs.vapi.ai/
- **Developer Guide:** https://docs.vapi.ai/quickstart/introduction
- **Standards:** REST/JSON, WebSocket for audio streaming, OpenAPI 3.x, Webhooks
- **Authentication:** Bearer token (API key)

### Retell AI
- **Description:** End-to-end voice automation platform for inbound and outbound call automation at production scale. Offers both drag-and-drop Conversation Flow Agents and flexible Prompt-based Agents; ~600 ms latency; SOC 2 / HIPAA / GDPR certified.
- **API Documentation:** https://docs.retellai.com/
- **Webhook Reference:** https://docs.retellai.com/features/webhook-overview
- **SDKs/Libraries:** Official Python and Node.js SDKs (documented in developer portal)
- **Developer Guide:** https://docs.retellai.com/
- **Standards:** REST/JSON, WebSocket audio streaming, OpenAPI, Webhooks
- **Authentication:** API key (Bearer token)

### Bland AI
- **Description:** High-throughput AI call centre infrastructure for outbound and inbound calling at scale. Supports Conversational Pathways (visual branching logic), real-time SMS, and up to 20,000 calls/hour.
- **API Documentation:** https://www.bland.ai/docs (official docs)
- **SDKs/Libraries:** REST API with webhook-based event handling; community Python/JS wrappers
- **Developer Guide:** https://www.bland.ai/docs
- **Standards:** REST/JSON, WebSocket, SIP, Webhooks
- **Authentication:** API key

### ElevenLabs Conversational AI
- **Description:** TTS-native conversational AI platform connecting STT, LLM, and ElevenLabs' industry-leading TTS into a single session API. Handles turn detection, session management, tool calling, and multi-agent tracking natively. Official SDKs for Python, TypeScript, Flutter, Swift, and Kotlin.
- **API Documentation:** https://elevenlabs.io/docs/conversational-ai/overview
- **API Reference:** https://elevenlabs.io/docs/api-reference/introduction
- **SDKs/Libraries:** Python SDK, TypeScript/JavaScript SDK, Flutter, Swift (Kotlin) — https://elevenlabs.io/developers
- **Developer Guide:** https://elevenlabs.io/docs/eleven-agents/overview
- **Standards:** REST/JSON, WebSocket streaming, OpenAPI
- **Authentication:** API key (xi-api-key header)

### OpenAI Realtime API
- **Description:** Low-latency speech-to-speech API supporting native audio input and output with multimodal capabilities (audio, images, text). Unified billing across the OpenAI ecosystem. gpt-realtime model supports SIP-based phone calling, MCP tool calling, and remote MCP servers.
- **API Documentation:** https://developers.openai.com/api/docs/guides/realtime
- **Voice Agents Guide:** https://developers.openai.com/api/docs/guides/voice-agents
- **SDKs/Libraries:** Python (openai-agents-python), TypeScript (openai-agents-js) — https://openai.github.io/openai-agents-js/guides/voice-agents/
- **Developer Guide:** https://openai.github.io/openai-agents-js/guides/voice-agents/quickstart/
- **Standards:** WebSocket streaming, JSON-RPC for tool calls, MCP, REST/JSON
- **Authentication:** Bearer token (OPENAI_API_KEY)

### LiveKit Agents Framework
- **Description:** Open-source (Apache 2.0) framework for building real-time voice, video, and multimodal AI agents. Production-ready with built-in load balancing, Kubernetes compatibility, semantic turn detection, MCP tool calling, and SIP telephony. Python and Node.js SDKs. Highly extensible with any STT/LLM/TTS provider.
- **API Documentation:** https://docs.livekit.io/agents/
- **GitHub:** https://github.com/livekit/agents
- **SDKs/Libraries:** Python SDK, Node.js SDK — documented at https://docs.livekit.io/agents/
- **Developer Guide:** https://docs.livekit.io/agents/quickstarts/voice-agent/
- **Standards:** WebRTC, WebSocket, SIP, MCP, REST/JSON, Apache 2.0 licence
- **Authentication:** LiveKit API key + secret; JWT tokens for room access

### Deepgram Voice Agent API
- **Description:** Unified STT + LLM orchestration + TTS API for real-time voice agents in a single streaming endpoint. Priced at $4.50/hour. Supports bring-your-own TTS providers (ElevenLabs, Cartesia, OpenAI TTS, AWS Polly) and integrates with any LLM endpoint.
- **API Documentation:** https://developers.deepgram.com/docs/configure-voice-agent
- **TTS Model Reference:** https://developers.deepgram.com/docs/voice-agent-tts-models
- **SDKs/Libraries:** Python, Node.js, Go SDKs — https://developers.deepgram.com/
- **Developer Guide:** https://deepgram.com/learn/voice-agent-api-generally-available
- **Standards:** WebSocket streaming, REST/JSON, OpenAPI
- **Authentication:** API key (Authorization header)

### Twilio Programmable Voice API
- **Description:** The dominant CPaaS (Communications Platform as a Service) provider for telephony infrastructure underlying most voice agent platforms. Provides SIP trunking, phone number provisioning, call recording, PSTN connectivity, and Media Streams (real-time audio streaming to WebSocket endpoints). ConversationRelay API bridges Twilio telephony and AI voice agents.
- **API Documentation:** https://www.twilio.com/docs/voice/api
- **ConversationRelay (AI Bridge):** https://www.twilio.com/docs/voice
- **SDKs/Libraries:** C#/.NET, Java, Node.js, PHP, Python, Ruby server SDKs; iOS, Android, React Native, JavaScript client SDKs — https://www.twilio.com/docs/voice/sdks
- **Developer Guide:** https://www.twilio.com/docs/voice/quickstart/server
- **Standards:** REST/JSON, TwiML (Twilio Markup Language), WebSocket Media Streams, SIP, OpenAPI
- **Authentication:** Account SID + Auth Token; API keys for OAuth flows

### Voiceflow API
- **Description:** Visual conversation design platform with an API for programmatically managing agents, deploying to voice/chat channels, and accessing transcripts and analytics. Positions as the collaboration layer for product + engineering teams building conversational AI.
- **API Documentation:** https://www.voiceflow.com/api (developer portal)
- **Developer Guide:** https://www.voiceflow.com/blog/ai-ivr
- **SDKs/Libraries:** REST API + JavaScript SDK; integrations via webhooks and Zapier
- **Standards:** REST/JSON, Webhooks, OpenAPI
- **Authentication:** API key

### Rasa Open Source
- **Description:** MIT-licensed ML framework for conversational AI. The CALM engine combines LLM reasoning with deterministic business logic. Integrates with third-party STT/TTS for voice deployment. Community edition is free to self-host; Rasa Pro adds enterprise governance and support.
- **API Documentation:** https://legacy-docs-oss.rasa.com/docs/rasa/ (community) / https://rasa.com/docs/ (Rasa Pro)
- **GitHub:** https://github.com/RasaHQ/rasa
- **SDKs/Libraries:** Python; REST API for integration with external systems
- **Developer Guide:** https://www.projectpro.io/article/conversational-ai-with-rasa/1143
- **Standards:** REST/JSON, Webhooks, MIT licence (open source)
- **Authentication:** Self-hosted; token-based for API access in production

---

## Notes

**Latency threshold**: The emerging industry consensus is that sub-500 ms end-to-end response latency is required for natural-feeling conversation; sub-300 ms is considered excellent. Only three TTS providers consistently deliver first-audio under 150 ms: Cartesia Sonic-3 (40–90 ms), ElevenLabs Flash v2.5 (~75 ms), and Deepgram Aura (~90 ms). Architecture decisions about STT, LLM, and TTS provider selection have a direct and measurable impact on user experience.

**Realtime vs. pipeline architecture**: Two competing architectures are emerging. The traditional pipeline (STT → LLM → TTS) converts audio to text, reasons in text, and synthesises audio — losing parallelingual cues (tone, hesitation, emotion). Speech-to-speech (multimodal) models (e.g., OpenAI gpt-realtime) process raw audio end-to-end and preserve these signals at the cost of model opacity and less control over intermediate steps. The optimal architecture depends on use case: pipeline is more debuggable and cheaper; speech-to-speech is more natural at the cost of interpretability.

**MCP adoption trajectory**: MCP has reached critical mass in 2026 (97 million monthly SDK downloads, supported by all frontier AI vendors). New voice agent platforms should treat MCP as the default tool-calling protocol rather than implementing proprietary function-calling schemas.

**TCPA compliance gap**: The FCC's February 2024 ruling that AI-generated voices are "artificial voices" under TCPA is not widely reflected in current platform compliance documentation. This represents a material legal risk for any outbound calling feature — consent management, do-not-call enforcement, and calling-hours controls must be built in by default, not left to the end operator.

**Open-source landscape**: As of 2026, LiveKit Agents (Apache 2.0) is the most complete open-source framework for production voice agents, covering real-time audio, telephony, multi-agent handoff, MCP, and semantic turn detection. Rasa Open Source (MIT) covers conversational AI logic but lacks native telephony. No single open-source project provides a complete end-to-end platform (number provisioning + agent builder + analytics + compliance dashboard) — this is the primary open-source opportunity in the space.
