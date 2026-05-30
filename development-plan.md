# AI Voice Agent Builder — Phased Development Plan

> Project: 349-ai-voice-agent-builder · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md`. The product is an **open-source, AI-native, end-to-end voice agent platform** — the gap the research identifies as unfilled (LiveKit provides the framework but no platform; Rasa lacks telephony). The platform handles inbound/outbound calls over SIP/WebRTC, runs a pluggable STT→LLM→TTS pipeline with semantic turn detection, ships HIPAA/GDPR/TCPA compliance as first-class features, and exposes both a drag-and-drop flow builder and a REST API.

The data model follows **data-model-suggestion-1 (Entity-Centric Normalized Relational, 16 tables)** — chosen because the product's core positioning is multi-tenant, compliance-first (HIPAA/GDPR/TCPA/PCI DSS), with independent lifecycles for provider configs, calls, campaigns, and consent records. The JSONB-collapsed alternative (suggestion 2) trades away the referential integrity and independent provider-config lifecycle that the compliance and "swap any provider" differentiators require.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The realtime voice domain's reference framework (LiveKit Agents) and every major STT/LLM/TTS SDK ship Python-first. Semantic turn detection models (transformer) and PII redaction (Presidio) are Python-native. |
| Realtime agent runtime | **LiveKit Agents (Apache 2.0)** | The only production-grade OSS framework with built-in WebRTC/SIP, semantic turn detection, multi-agent handoff, and MCP. Building on it (not reimplementing media transport) is the fastest path to sub-300 ms latency and avoids reinventing RTP/jitter buffering. |
| API framework | **FastAPI** | Async-native (required for streaming + webhooks), auto-generates the OpenAPI 3.1 spec the research mandates, Pydantic v2 validation for config/webhook schemas. |
| Control-plane DB | **PostgreSQL 16** | Suggestion-1 uses UUIDs, JSONB, `TEXT[]`, partitioned tables (`calls`, `audit_log`), `INET`, and GIN indexes — all Postgres features. Production multi-tenant compliance needs ACID + row-level access control. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Async ORM matching FastAPI; Alembic for the versioned migrations a 16-table schema needs. |
| Task queue | **Celery + Redis** | Outbound campaigns, post-call processing (transcription finalisation, PII redaction, sentiment, gap detection, cost rollup), and retry logic are async workloads. Redis doubles as the rate-limit and dedup store. |
| Realtime transport | **WebRTC + SIP** (via LiveKit) | WebRTC (W3C) for browser; SIP (RFC 3261) trunking via Twilio/Plivo/Vonage/BYOC for PSTN. SRTP/DTLS-SRTP (RFC 3711) for HIPAA/GDPR media encryption. |
| Audio streaming | **WebSocket (RFC 6455)** | Standard for STT/TTS streaming and live-call monitoring; FastAPI WebSocket endpoints for the dashboard's live transcript. |
| STT providers | Deepgram (default), AssemblyAI, Whisper, Azure, Google | Pluggable via a `STTProvider` adapter interface. Deepgram Nova-3 default for latency. |
| LLM providers | Anthropic (default), OpenAI, Google, Groq, Ollama, custom | Pluggable via `LLMProvider`. Streaming token output mandatory for low latency. |
| TTS providers | Cartesia (default), ElevenLabs, Deepgram Aura, OpenAI, Azure | Pluggable via `TTSProvider`. Cartesia Sonic-3 (40–90 ms first-audio) default; SSML 1.1 (W3C) support. |
| PII redaction | **Microsoft Presidio** + LLM fallback | OSS, supports custom recognizers for PCI PAN and PHI; satisfies HIPAA/PCI/GDPR redaction-as-standard requirement. |
| Tool calling | **MCP (JSON-RPC 2.0)** via `mcp` SDK | Research mandates MCP as default tool protocol; agents consume any MCP server (CRM, calendar, KB) without bespoke integrations. |
| Frontend | **Next.js 15 (React, TypeScript) + React Flow** | Dashboard + drag-and-drop flow builder. React Flow is the standard node-graph library for the conversation-flow canvas. shadcn/ui + Tailwind for components. |
| Auth | **OAuth 2.0 / OIDC** (Authlib) + API keys | API keys (hashed, prefixed) for developer/programmatic access; OIDC SSO (Okta/Azure AD) for enterprise per standards.md. |
| Secrets | **HashiCorp Vault** (ref-based) | `provider_configs.credential_ref` stores `vault://...` references, never raw keys — required for SOC 2 / HIPAA. |
| Containerisation | **Docker + docker-compose**, Helm chart | Self-hosted Kubernetes-compatible deployment per README; compose for local dev. |
| Testing | **pytest + pytest-asyncio**, Vitest + Playwright (frontend) | Standard. `respx`/`pytest-httpx` for mocked provider APIs. |
| Code quality | **ruff** (lint+format), **mypy** (strict), **pre-commit** | Standard modern Python toolchain. ESLint + Prettier for frontend. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, lockfile-based. |

### Project Structure

```
ai-voice-agent-builder/
├── pyproject.toml
├── uv.lock
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.agent
├── alembic.ini
├── deploy/
│   └── helm/                         # Kubernetes Helm chart
├── migrations/                       # Alembic migration versions
├── src/
│   └── voiceagent/
│       ├── __init__.py
│       ├── config.py                 # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py               # SQLAlchemy async engine/session
│       │   ├── models/               # ORM models, one file per entity group
│       │   │   ├── identity.py       # organisations, users, api_keys
│       │   │   ├── agents.py         # voice_agents, conversation_flows, provider_configs, squads
│       │   │   ├── telephony.py      # phone_numbers
│       │   │   ├── calls.py          # calls (partitioned), call_segments
│       │   │   ├── knowledge.py      # knowledge_bases, knowledge_entries
│       │   │   ├── campaigns.py      # campaigns
│       │   │   ├── compliance.py     # consent_records
│       │   │   └── ops.py            # cost_records, audit_log (partitioned)
│       │   └── partitioning.py       # monthly partition creation helpers
│       ├── api/
│       │   ├── app.py                # FastAPI app factory, OpenAPI 3.1 metadata
│       │   ├── deps.py               # auth, db session, org-scoping deps
│       │   ├── routers/              # agents, providers, calls, phone_numbers,
│       │   │                         #   squads, campaigns, knowledge, consent,
│       │   │                         #   webhooks, analytics, simulation
│       │   └── schemas/              # Pydantic request/response models
│       ├── auth/
│       │   ├── apikeys.py            # generate/hash/verify keys, scope checks
│       │   └── oidc.py               # OIDC SSO flow
│       ├── providers/
│       │   ├── base.py               # STTProvider, LLMProvider, TTSProvider ABCs
│       │   ├── stt/                  # deepgram.py, assemblyai.py, whisper.py, ...
│       │   ├── llm/                  # anthropic.py, openai.py, groq.py, ...
│       │   ├── tts/                  # cartesia.py, elevenlabs.py, ...
│       │   └── registry.py           # provider_name -> adapter class lookup
│       ├── pipeline/
│       │   ├── agent_session.py      # LiveKit AgentSession assembly from DB config
│       │   ├── turn_detection.py     # semantic turn detector wrapper
│       │   ├── flow_engine.py        # conversation_flows graph interpreter
│       │   ├── language.py           # mid-call language detection + switch
│       │   ├── mcp_tools.py          # MCP server discovery + tool binding
│       │   └── squad.py              # multi-agent handoff + warm transfer
│       ├── telephony/
│       │   ├── sip.py                # SIP trunk config, inbound routing, BYOC
│       │   ├── webrtc.py             # WebRTC room tokens
│       │   ├── carriers/             # twilio.py, plivo.py, vonage.py adapters
│       │   └── voicemail.py          # voicemail detection
│       ├── compliance/
│       │   ├── redaction.py          # Presidio PII/PCI/PHI redaction
│       │   ├── consent.py            # TCPA/GDPR consent verification
│       │   └── dnc.py                # do-not-call + calling-hours enforcement
│       ├── analytics/
│       │   ├── sentiment.py          # per-segment + per-call sentiment
│       │   ├── gap_detection.py      # KB gap detection from transcripts
│       │   ├── cost.py               # per-call cost computation + rollups
│       │   └── eval.py               # LLM-as-judge transcript scoring
│       ├── campaigns/
│       │   ├── scheduler.py          # batch scheduling, calling-hours, concurrency
│       │   └── runner.py             # outbound call dispatch + retry
│       ├── simulation/
│       │   └── adversarial.py        # synthetic caller simulation
│       ├── webhooks/
│       │   ├── dispatcher.py         # signed outbound webhook delivery
│       │   └── events.py             # event type catalogue
│       └── worker/
│           ├── celery_app.py
│           └── tasks/                # post_call.py, campaign.py, redaction.py
├── frontend/                         # Next.js dashboard + flow builder
│   ├── app/
│   ├── components/flow-builder/      # React Flow canvas
│   └── lib/api-client/               # generated from OpenAPI spec
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                     # sample configs, transcripts, SIP payloads
```

The structure groups by concern (providers, pipeline, telephony, compliance, analytics) so each phase adds files without restructuring.

---

## Phase 1: Foundation — Data Model, Config, Auth

### Purpose
Establish the multi-tenant control plane: the database schema, configuration system, and authentication. Every later phase reads org-scoped data and authenticates requests, so this must come first. After this phase, the platform can create organisations, users, and API keys, and persist agent/provider configurations — but cannot yet make calls.

### Tasks

#### 1.1 — Configuration & Settings

**What**: Centralised, env-driven settings with validation.

**Design**:
Pydantic `BaseSettings` loaded from environment / `.env`.

```python
class Settings(BaseSettings):
    database_url: str                       # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    vault_addr: str | None = None
    vault_token: str | None = None
    livekit_url: str                        # wss://...
    livekit_api_key: str
    livekit_api_secret: str
    default_stt_provider: str = "deepgram"
    default_llm_provider: str = "anthropic"
    default_tts_provider: str = "cartesia"
    data_retention_days: int = 90           # GDPR default
    environment: Literal["dev","staging","prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="VOICEAGENT_", env_file=".env")
```

**Testing**:
- Unit: all required env vars set → `Settings` loads with correct types.
- Unit: missing `database_url` → `ValidationError` naming the field.
- Unit: invalid `environment` value → `ValidationError`.

#### 1.2 — Database Schema & Migrations

**What**: Implement all 16 tables from data-model-suggestion-1 as SQLAlchemy models with an initial Alembic migration.

**Design**:
Implement the full DDL from `data-model-suggestion-1.md`: `organisations`, `users`, `api_keys` (identity); `voice_agents`, `conversation_flows`, `provider_configs`, `squads` (agents); `phone_numbers`; `calls` (RANGE-partitioned on `created_at`), `call_segments`; `knowledge_bases`, `knowledge_entries`; `campaigns`; `consent_records`; `cost_records`, `audit_log` (partitioned). Preserve all CHECK constraints, FKs, UNIQUE constraints, and indexes (including GIN on `knowledge_entries.tags`). Implement a `create_monthly_partition(table, year, month)` helper for `calls` and `audit_log`.

**Testing**:
- Integration (real Postgres via testcontainers): `alembic upgrade head` → all 16 tables, indexes, and constraints exist.
- Integration: insert `voice_agent` referencing non-existent `org_id` → FK violation.
- Integration: insert `provider_configs` with `provider_type='xyz'` → CHECK violation.
- Integration: `create_monthly_partition('calls', 2026, 6)` then insert a row dated 2026-06-15 → lands in the June partition.
- Integration: `alembic downgrade base` then `upgrade head` round-trips cleanly.

#### 1.3 — API Key Auth & Org Scoping

**What**: API-key generation, hashing, verification, scope enforcement, and an org-scoping FastAPI dependency.

**Design**:
Keys formatted `voice_<prefix>_<secret>`; store `key_prefix` and `key_hash = sha256(secret)`. Scopes: `agents`, `calls`, `campaigns`, `analytics`, `admin`.

```python
def generate_api_key(org_id: UUID, scopes: list[str]) -> tuple[str, ApiKey]: ...
async def verify_api_key(raw_key: str, db) -> ApiKey | None: ...     # constant-time compare
async def get_current_org(authorization: str = Header()) -> Organisation: ...  # FastAPI dep
def require_scope(scope: str): ...                                  # dep factory -> 403 if missing
```

All resource queries filter by `org_id` from the authenticated key. `last_used_at` updated async.

**Testing**:
- Unit: `generate_api_key` → returned raw key verifies against its stored hash; raw secret not persisted.
- Integration (mocked db): valid `Bearer` key → `get_current_org` returns the org.
- Integration: revoked/expired key → 401.
- Integration: key lacking `campaigns` scope hitting a campaign route → 403.
- Integration: org A's key requesting org B's agent → 404 (not 403, no existence leak).

#### 1.4 — Audit Log & OIDC SSO

**What**: Write-through audit logging for all mutations and OIDC login for the dashboard.

**Design**:
`record_audit(db, org_id, actor_id, actor_type, action, entity_type, entity_id, changes, ip)` called from a FastAPI middleware/decorator on every non-GET request. OIDC via Authlib: `/auth/login` → IdP redirect; `/auth/callback` validates the ID token, upserts a `user`, issues a session cookie. SOC 2 requires complete audit coverage.

**Testing**:
- Integration: `POST /agents` → an `audit_log` row with `action='agent.create'`, correct `entity_id`, and a diff in `changes_json`.
- Unit (mocked IdP): valid ID token → user upserted, session issued.
- Unit (mocked IdP): expired/invalid token signature → 401, no user created.

---

## Phase 2: Provider Abstraction Layer

### Purpose
Build the pluggable STT/LLM/TTS adapter layer — the architectural core of the "no vendor lock-in" differentiator. This phase produces no telephony yet, but after it the platform can stream audio→text, text→tokens, and text→audio through any configured provider, validated in isolation. This must precede the pipeline (Phase 3) which composes these adapters.

### Tasks

#### 2.1 — Provider Interfaces

**What**: Abstract base classes defining the streaming contract for each provider type.

**Design**:

```python
class STTProvider(ABC):
    @abstractmethod
    async def stream(self, audio: AsyncIterator[bytes], language: str
                     ) -> AsyncIterator[STTEvent]: ...   # partial/final transcripts

@dataclass
class STTEvent:
    text: str; is_final: bool; confidence: float
    language: str; start_ms: int; end_ms: int

class LLMProvider(ABC):
    @abstractmethod
    async def generate(self, messages: list[Message], tools: list[ToolDef],
                       config: dict) -> AsyncIterator[LLMDelta]: ...  # streamed tokens + tool calls

class TTSProvider(ABC):
    @abstractmethod
    async def synthesize(self, text: str, voice_id: str, ssml: bool,
                         config: dict) -> AsyncIterator[bytes]: ...    # streamed audio frames
    @abstractmethod
    def first_audio_latency_target_ms(self) -> int: ...
```

A `registry.py` maps `provider_name` → adapter class; `build_provider(provider_config: ProviderConfig)` resolves a credential from Vault via `credential_ref` and instantiates the adapter.

**Testing**:
- Unit: `build_provider` for unknown `provider_name` → `UnknownProviderError`.
- Unit: `build_provider` resolves `vault://x` via a mocked Vault client; raw credential never logged.

#### 2.2 — STT Adapters

**What**: Deepgram (default), AssemblyAI, Whisper adapters implementing `STTProvider`.

**Design**:
Each opens a provider WebSocket, forwards audio frames, emits `STTEvent`s. Deepgram config: `{"model":"nova-3","punctuate":true,"interim_results":true}`. Whisper (local/OpenAI) batches in short windows since it lacks true streaming.

**Testing**:
- Integration (mocked WS via fixture audio): Deepgram adapter on `tests/fixtures/hello.wav` → final transcript "hello", `is_final=True`.
- Unit: malformed provider frame → logged + skipped, stream continues.
- Integration (real, opt-in `@pytest.mark.realapi`): live Deepgram returns a transcript for sample audio.

#### 2.3 — LLM Adapters

**What**: Anthropic (default), OpenAI, Groq adapters implementing `LLMProvider` with streaming + tool calls.

**Design**:
Translate the internal `Message`/`ToolDef` shape to each SDK. Emit `LLMDelta(text_delta | tool_call)`. Use prompt caching (Anthropic) for the static system prompt to cut latency/cost.

**Testing**:
- Integration (mocked SDK): streamed deltas reassemble to the expected completion.
- Integration (mocked SDK): a tool-call response yields an `LLMDelta` with parsed tool name + JSON args.
- Unit: provider rate-limit (429) → retried with backoff, then surfaced as `ProviderUnavailable`.

#### 2.4 — TTS Adapters

**What**: Cartesia (default), ElevenLabs, Deepgram Aura adapters implementing `TTSProvider` with SSML.

**Design**:
Stream audio frames; expose `first_audio_latency_target_ms` (Cartesia ~60, ElevenLabs ~75, Aura ~90) so the pipeline can pick the lowest-latency available. Validate SSML against SSML 1.1 before send; strip unsupported tags per provider.

**Testing**:
- Integration (mocked WS): "Hello world" → non-empty audio frame stream; first frame within mocked target.
- Unit: SSML with `<prosody rate="fast">` → passed through for ElevenLabs, stripped for a provider that rejects it.
- Unit: invalid SSML XML → `SSMLValidationError` before any provider call.

---

## Phase 3: Realtime Voice Pipeline (WebRTC) — Core Value

### Purpose
Assemble the STT→LLM→TTS pipeline into a working voice agent over WebRTC, with semantic turn detection and interruption handling. This is the heart of the product. After this phase a developer can talk to an agent in the browser and get a low-latency, interruptible spoken response, with the full turn-by-turn transcript persisted. Telephony (Phase 4) reuses this exact pipeline.

### Tasks

#### 3.1 — Agent Session Assembly

**What**: Build a LiveKit `AgentSession` from a `voice_agent` DB row + its provider configs.

**Design**:
`build_session(agent_id, db) -> AgentSession` loads the agent context (the suggestion-1 "Load agent context for call handling" join: agent + active flow + STT/LLM/TTS provider configs), instantiates adapters via the registry, and wires them into LiveKit with the agent's `system_prompt`, `first_message`, `voice_id`, `end_call_phrases`, and `max_call_duration_seconds`.

**Testing**:
- Integration (mocked providers + LiveKit): `build_session` for a seeded agent → session with the correct three adapters and system prompt.
- Integration: agent missing an active flow → falls back to prompt-only mode (no error).

#### 3.2 — Semantic Turn Detection & Interruption

**What**: Transformer-based turn detection replacing VAD-only, with barge-in handling.

**Design**:
Wrap LiveKit's semantic turn detector. On detected user-turn-end → trigger LLM. On user speech during agent TTS playback (barge-in) → cancel TTS stream, flush the partial agent utterance to the transcript marked interrupted, restart listening. `interruption_handling` ∈ `{semantic, vad, none}` from the agent config selects the strategy.

**Testing**:
- Integration (scripted audio): two-utterance exchange → exactly two caller turns detected, one agent response each.
- Integration: caller speaks 300 ms into agent playback → TTS canceled within one frame budget; transcript shows agent segment as interrupted.
- Unit: `interruption_handling='none'` → barge-in ignored, agent finishes.

#### 3.3 — Transcript Capture & Call Lifecycle

**What**: Persist the call and per-turn segments through the call state machine.

**Design**:
States (from `calls.status`): `ringing → connected → in_progress → (transferring) → completed | failed | voicemail | no_answer | busy`. On each finalised turn, append a `call_segments` row (`speaker`, `text`, `start_ms`, `end_ms`, `confidence`, `language`, `intent_detected`). On call end, set `ended_at`, `duration_seconds`, `end_reason`. Raw `text` stored now; redaction in Phase 6. Segment writes are buffered and flushed to avoid per-turn DB round-trips on the hot path.

**Testing**:
- Integration: a 3-turn WebRTC session → 1 `calls` row reaching `completed` + 3 ordered `call_segments`.
- Unit: state machine rejects illegal transition (`completed → in_progress`) with `InvalidCallState`.
- Integration: session exceeding `max_call_duration_seconds` → ends with `end_reason='max_duration'`.

#### 3.4 — WebRTC Entry Point & Live Transcript WS

**What**: REST endpoint to start a browser session + WebSocket for live transcript.

**Design**:
`POST /agents/{id}/web-call` → `{ "room": str, "token": str, "livekit_url": str }` (JWT scoped to the room). `WS /calls/{call_id}/live` streams `{type:"segment", ...}` events as turns finalise, for the dashboard.

**Testing**:
- Integration: `POST /agents/{id}/web-call` → valid LiveKit JWT decoding to the expected room + identity.
- Integration (mocked pipeline): WS client receives a `segment` event per finalised turn.
- Integration: web-call for another org's agent → 404.

---

## Phase 4: Telephony — SIP, Carriers, Inbound/Outbound

### Purpose
Connect the Phase 3 pipeline to the PSTN so agents handle real phone calls. Adds SIP trunking (RFC 3261), carrier adapters (Twilio/Plivo/Vonage/BYOC), number provisioning, inbound routing, programmatic outbound dialling, and voicemail detection. After this phase the platform handles real inbound and outbound calls — the table-stakes capability of every competitor.

### Tasks

#### 4.1 — SIP Trunk Integration & Carrier Adapters

**What**: Bridge SIP trunks into LiveKit and abstract carrier operations.

**Design**:

```python
class CarrierAdapter(ABC):
    async def provision_number(self, country: str, type: str) -> ProvisionedNumber: ...
    async def configure_inbound(self, number: str, sip_uri: str) -> None: ...
    async def place_outbound(self, from_: str, to: str, sip_uri: str) -> str: ...  # sip_call_id
```

Implement `TwilioAdapter`, `PlivoAdapter`, `VonageAdapter`, and `SipTrunkAdapter` (BYOC, raw SIP credentials). Inbound calls land on a LiveKit SIP endpoint; media encrypted with SRTP/DTLS-SRTP (`calls.encryption`).

**Testing**:
- Integration (mocked Twilio API): `provision_number('US','local')` → `phone_numbers` row with `carrier='twilio'`.
- Integration (mocked): BYOC trunk config validated and stored in `sip_trunk_config_json`.
- Unit: outbound to an unconfigured carrier → `CarrierNotConfigured`.

#### 4.2 — Inbound Call Routing

**What**: Route incoming PSTN calls to the agent assigned to the dialled number.

**Design**:
SIP INVITE → look up `phone_numbers` by dialled `number` → resolve `agent_id` → create `calls` row (`direction='inbound'`, `transport='sip'`, `sip_call_id`) → start the Phase 3 pipeline in the SIP-bridged room.

**Testing**:
- Integration (simulated SIP INVITE fixture): call to a provisioned number → `calls` row created, pipeline started.
- Integration: INVITE to an unmapped number → 404/SIP 404, no `calls` row.

#### 4.3 — Outbound Dialling & Voicemail Detection

**What**: API-initiated outbound calls with voicemail detection and graceful handling.

**Design**:
`POST /calls/outbound` `{agent_id, to_number, from_number_id, metadata?}` → consent gate (Phase 6 hook, no-op until then) → carrier `place_outbound` → on answer start pipeline. Voicemail detection: classify the answering audio (beep/greeting cadence) in the first ~3 s; if voicemail → optionally play `voicemail_message`, set `status='voicemail'`, `end_reason='voicemail'`.

**Testing**:
- Integration (mocked carrier): outbound → `calls` row `direction='outbound'` reaching `connected`.
- Integration (voicemail fixture audio): detector → `voicemail_detected=true`, `voicemail_message` played, status `voicemail`.
- Integration: no-answer → `status='no_answer'`, `end_reason='no_answer'`.

---

## Phase 5: Conversation Flow Builder, MCP Tools & REST/Webhooks

### Purpose
Give both developers (REST + webhooks + MCP) and non-technical operators (drag-and-drop flow builder) full control of agent behaviour. After this phase agents can follow structured flows, call external tools via MCP (CRM/calendar/KB), and emit events to external systems — the developer-experience and integration surface that differentiates the platform from framework-only OSS. Parallelisable with Phase 4 once Phase 3 is done.

### Tasks

#### 5.1 — Conversation Flow Engine

**What**: Interpret the `conversation_flows.graph_json` node graph at runtime.

**Design**:
Node types: `greeting`, `prompt`, `collect` (gather a slot), `tool_call`, `condition`, `handoff`, `end`. Transitions: `{condition, target}` where condition ∈ `intent:<x>`, `slot_filled:<x>`, `success`, `failure`, `default`. The engine walks nodes, injecting node prompts into the LLM context and routing on detected intent/slot. `flow_type` ∈ `{prompt, structured, hybrid}` — `prompt` ignores the graph; `structured` follows it strictly; `hybrid` lets the LLM deviate within guardrails. Flows are versioned (`UNIQUE(agent_id, version)`).

**Testing**:
- Unit: `structured` flow, caller intent `schedule` → transitions greeting → schedule node.
- Unit: no matching condition → `default` transition taken.
- Unit: cyclic graph with no `end` reachable within `max_handoffs`/node budget → `FlowDepthExceeded`.
- Integration: full flow run with a mocked LLM emitting scripted intents → expected node path + persisted segments.

#### 5.2 — MCP Tool Integration

**What**: Let agents discover and call tools from configured MCP servers.

**Design**:
`voice_agents.mcp_servers_json` lists `{name, url, tools[]}`. At session start, connect to each MCP server (JSON-RPC 2.0), discover tool schemas, expose them to the LLM as `ToolDef`s. On an LLM tool call → invoke via MCP, return the result, record `call_segments.tool_call_json`.

**Testing**:
- Integration (mock MCP server): discovery → declared tools surfaced to the LLM.
- Integration: LLM emits `book_appointment` call → MCP invoked, result stored in `tool_call_json`.
- Unit: MCP server unreachable at session start → agent runs without those tools, warning logged (no crash).

#### 5.3 — REST API Surface & OpenAPI 3.1

**What**: Full CRUD REST API for all configuration entities, publishing an OpenAPI 3.1 spec.

**Design**:
Routers for agents, provider_configs, conversation_flows, phone_numbers, squads, knowledge bases, campaigns, consent, calls (read/list/transcript). Standard REST semantics; cursor pagination on list endpoints; all scoped by org. FastAPI auto-emits OpenAPI 3.1 at `/openapi.json`; the frontend API client is generated from it.

**Testing**:
- Integration: `POST /agents` then `GET /agents/{id}` → created agent returned; audit row written.
- Integration: `GET /calls?cursor=...&limit=50` → page of 50 + next cursor.
- Unit: `/openapi.json` validates against the OpenAPI 3.1 meta-schema; security scheme maps to API-key scopes.
- Integration: validation error (missing `system_prompt`) → 422 naming the field.

#### 5.4 — Outbound Webhooks

**What**: Signed webhook delivery for call lifecycle and analytics events.

**Design**:
Events: `call.started`, `call.ended`, `call.transcript.ready`, `call.transferred`, `campaign.completed`, `knowledge.gap.detected`. Payload signed with HMAC-SHA256 (`X-VoiceAgent-Signature`). Delivered via Celery with exponential-backoff retry; failures logged. JSON Schema published per event type.

**Testing**:
- Integration: completing a call → `call.ended` POSTed to the configured URL with a valid signature.
- Unit: signature computed over the raw body verifies with the shared secret.
- Integration (mocked endpoint 500): delivery retried per backoff, then marked failed.

---

## Phase 6: Compliance & Trust — Redaction, Consent, Encryption

### Purpose
Make HIPAA, GDPR, PCI DSS, and TCPA first-class rather than enterprise add-ons — the platform's strongest differentiator. Adds PII/PCI/PHI redaction of transcripts, consent management and enforcement (gating Phase 4 outbound), encrypted recording storage, and data-retention/erasure. After this phase the platform is safe for regulated-industry deployment. Requires Phases 3–4 (calls + transcripts exist).

### Tasks

#### 6.1 — PII / PCI / PHI Redaction

**What**: Redact sensitive data from transcripts as standard, populating the redaction fields.

**Design**:
Post-call Celery task runs Presidio over each `call_segments.text` → writes `text_redacted`, sets `pii_redacted`, `pii_categories` (`phone_number`, `name`, `email`, `ssn`, `address`...), and `pci_redacted` for PAN patterns (Luhn-validated). When `organisations.hipaa_enabled`, redaction is mandatory and the unredacted `text` is access-controlled (admin scope only). Sets `calls.pii_detected`.

**Testing**:
- Unit: "my card is 4111 1111 1111 1111" → `text_redacted` masks the PAN; `pci_redacted=true`.
- Unit: "call me at 415-555-1234" → `phone_number` in `pii_categories`; masked.
- Integration: HIPAA-enabled org, viewer-scope reads transcript → only `text_redacted`; admin → unredacted.
- Unit: non-PAN 16-digit number failing Luhn → not flagged PCI.

#### 6.2 — Consent Management & Enforcement (TCPA/GDPR)

**What**: Record consent and enforce it before outbound calls and recording.

**Design**:
`consent_records` per `(phone_number, consent_type)`. `verify_consent(org_id, phone, type)` checks `consent_given AND revoked_at IS NULL`. The Phase 4 outbound path calls `verify_consent(..., 'tcpa_express_written')` and refuses (`status='failed'`, `end_reason='error'`, reason logged) if absent — the FCC AI-voice TCPA ruling makes this mandatory. GDPR: recording requires `gdpr_recording` consent; revocation honoured immediately.

**Testing**:
- Integration: outbound to a number without TCPA consent → call refused, no carrier dial, audit row.
- Integration: with valid consent → call proceeds.
- Unit: revoked consent (`revoked_at` set) → `verify_consent` false.
- Integration: recording without `gdpr_recording` consent for a GDPR org → recording disabled, call still proceeds.

#### 6.3 — DNC List & Calling-Hours Enforcement

**What**: Block calls to do-not-call numbers and outside permitted hours.

**Design**:
`is_callable(org_id, phone, now, schedule)` checks the org DNC list and that `now` (in the campaign timezone) falls within `calling_hours`. Enforced on every outbound dial and at campaign dispatch.

**Testing**:
- Unit: number on DNC → `is_callable` false.
- Unit: 18:30 local with hours 09:00–17:00 → false.
- Unit: 14:00 local, not on DNC → true.

#### 6.4 — Encrypted Recording Storage & Retention

**What**: Encrypt recordings at rest and enforce GDPR retention/erasure.

**Design**:
Recordings written to object storage (S3-compatible) with SSE; `calls.recording_encrypted=true`, `recording_url` references the encrypted object. A daily Celery job deletes recordings + redacts/erases transcripts past `organisations` retention (`data_retention_days`, GDPR default 90). `DELETE /calls/{id}/data` (admin) performs immediate erasure (GDPR right-to-erasure), writing an audit row.

**Testing**:
- Integration (mocked S3): recording stored with SSE header; `recording_encrypted=true`.
- Integration: retention job deletes a recording older than the window; `recording_url` nulled, audit row written.
- Integration: `DELETE /calls/{id}/data` → segments erased, recording deleted, audit row.

---

## Phase 7: Multi-Agent Squads & Warm Transfer

### Purpose
Add in-call multi-agent orchestration (Vapi Squads pattern) and warm transfer to humans with full context — research flags cold transfers as the top CX failure point and warm transfer as well-implemented by only ~40% of platforms. After this phase a single call can hand off between specialist agents and escalate to a human with transcript + sentiment briefing. Requires Phase 3 (pipeline) and Phase 6 (sentiment hooks).

### Tasks

#### 7.1 — Squad Orchestration & Agent Handoff

**What**: Execute `squads.members_json` handoff rules within one call, preserving context.

**Design**:
Entry-point member handles the call; on a member's `handoff_rules` condition (`intent:billing`), swap the active agent in the same room, carrying `context_preservation` fields (transcript, sentiment, collected slots). Enforce `max_handoffs`. Each handoff writes a `call_segments` row with `is_handoff=true` and the new `agent_id`; increment `calls.handoffs`.

**Testing**:
- Integration (mocked LLM scripted intents): caller asks billing → handoff to billing agent; `is_handoff` segment recorded; `handoffs=1`.
- Unit: exceeding `max_handoffs` → no further handoff; routed to fallback/end.
- Integration: context fields visible in the receiving agent's prompt context.

#### 7.2 — Warm Transfer to Human

**What**: Transfer to a human agent with a transcript + sentiment summary briefing.

**Design**:
`handoff` node / rule with `transfer_type='warm'` → place a call/connect to the human's number, generate an LLM summary (transcript + `sentiment_label` + intents + collected data) delivered as a whisper/screen-pop payload, then bridge caller + human. `calls.human_transfer=true`, `transfer_context_json` stores the briefing; `status` passes through `transferring`.

**Testing**:
- Integration (mocked carrier + LLM): warm transfer → human dialled, `transfer_context_json` populated with summary + sentiment, `human_transfer=true`.
- Unit: summary generation includes sentiment label and last N turns.
- Integration: human declines/no-answer → caller returned to agent, status reverts to `in_progress`.

---

## Phase 8: Multilingual, Knowledge Base & Gap Detection

### Purpose
Add mid-call language switching (no platform does this seamlessly) and a knowledge base with automatic gap detection (Bland-style) that closes the loop from production to training. After this phase agents answer from a KB, switch languages mid-call, and auto-surface unanswered questions as draft KB entries. Requires Phases 3 and 6.

### Tasks

#### 8.1 — Mid-Call Language Detection & Switching

**What**: Detect the caller's language per turn and switch STT/TTS/LLM locale without a preconfigured separate agent.

**Design**:
Per finalised caller segment, detect language; if it differs from the active locale and is in `voice_agents.languages_supported`, switch STT model, TTS voice locale, and LLM response language for subsequent turns. Record `call_segments.language`, append to `calls.languages_used`, set `calls.language_detected`.

**Testing**:
- Integration (bilingual fixture audio EN→ES): agent responds in Spanish after the switch; `languages_used=['en','es']`.
- Unit: detected language not in `languages_supported` → stay on current locale.
- Unit: brief foreign phrase below confidence threshold → no spurious switch (hysteresis).

#### 8.2 — Knowledge Base Retrieval

**What**: Vector retrieval over `knowledge_entries` injected into agent responses.

**Design**:
Embed `knowledge_entries.question` (pgvector); at query time retrieve top-k relevant entries for the caller's question and inject into LLM context; increment `retrieval_count`. Entries via manual, import, or auto-generated sources.

**Testing**:
- Integration: KB with "What are your hours?" → caller asking hours retrieves that entry; `retrieval_count` incremented.
- Unit: empty KB → retrieval returns nothing, agent falls back to prompt.

#### 8.3 — Gap Detection & Auto-Expansion

**What**: Detect questions the agent couldn't answer and create draft KB entries.

**Design**:
Post-call task: LLM-as-judge scans the transcript for low-confidence/"I don't know" turns → creates `knowledge_entries` with `source='gap_detection'`, the `gap_query`, and a proposed answer (inactive until reviewed); increments `knowledge_bases.gap_count`; emits `knowledge.gap.detected` webhook.

**Testing**:
- Integration: transcript with an unanswered question → a `gap_detection` entry created (inactive); `gap_count` incremented; webhook fired.
- Unit: fully-answered transcript → no gap entries.

---

## Phase 9: Outbound Campaign Management

### Purpose
Add batch outbound calling with scheduling, concurrency control, voicemail handling, retry logic, and TCPA-enforced calling — the operations-team capability needed for sales/reminders use cases. Requires Phases 4 (outbound) and 6 (consent/DNC/hours).

### Tasks

#### 9.1 — Campaign Scheduler & Runner

**What**: Dispatch a campaign's targets respecting hours, concurrency, and consent.

**Design**:
`campaigns.schedule_json` defines `start/end_date`, `calling_hours`+timezone, `max_concurrent`, `retry_attempts`, `retry_delay_hours`. A Celery beat scheduler dispatches due `scheduled`/`running` campaigns: for each target, check `is_callable` + `verify_consent`, dial up to `max_concurrent` at once, create org-scoped `calls` linked via `campaign_id`. States: `draft → scheduled → running → (paused) → completed | cancelled`.

**Testing**:
- Integration (mocked carrier): 5 targets, `max_concurrent=2` → never more than 2 concurrent dials.
- Integration: target outside calling hours → skipped, retried in-window.
- Integration: pause mid-run → in-flight calls finish, no new dials.

#### 9.2 — Retry Logic & Progress Tracking

**What**: Retry no-answer/voicemail/failed targets and maintain live progress counters.

**Design**:
On `no_answer`/`busy`/`voicemail`/`failed`, schedule a retry after `retry_delay_hours` up to `retry_attempts`. Update `campaigns` counters (`completed_count`, `connected_count`, `voicemail_count`, `failed_count`, `conversion_rate`, `avg_duration_seconds`, `total_cost_cents`). On finish → `completed` + `campaign.completed` webhook.

**Testing**:
- Integration: no-answer target retried once with `retry_attempts=1`, not twice.
- Integration: counters match call outcomes after a run.
- Unit: `conversion_rate` computed correctly from connected vs target.

---

## Phase 10: Analytics, Simulation, Cost Transparency & Frontend

### Purpose
Surface the data the platform captures: analytics dashboards, adversarial pre-deployment simulation, transparent all-in per-minute cost, and the operator/developer web UI including the drag-and-drop flow builder. After this phase the platform is feature-complete against the MVP + v1.1 scope. Requires most prior phases; frontend work parallelises throughout.

### Tasks

#### 10.1 — Analytics & Cost Computation

**What**: Per-call cost breakdown, period rollups, and dashboard metrics.

**Design**:
Post-call task computes `calls.stt/llm/tts/telephony_cost_cents` and `total_cost_cents` from provider usage + `tokens_used`, plus `sentiment_score`/`sentiment_label`. A daily job aggregates into `cost_records` (per agent/direction/period). Analytics endpoints: call volume, completion/resolution rate, avg duration, sentiment distribution, all-in cost/min — directly addressing the "billing transparency" gap.

**Testing**:
- Unit: cost = sum of components; `total_cost_cents` matches.
- Integration: daily rollup aggregates a day's calls into one `cost_records` row per agent/direction.
- Integration: `GET /analytics/cost?period=month` → per-component breakdown + all-in cost/min.

#### 10.2 — Adversarial Caller Simulation

**What**: Simulate frustrated/confused/off-script callers against an agent pre-deployment.

**Design**:
`POST /agents/{id}/simulate` `{persona, scenario, turns}` → an LLM-driven synthetic caller runs against the agent through the real pipeline (no telephony), producing a transcript + an LLM-as-judge score (goal completion, compliance adherence, hallucination, handled-interruptions). Personas: `frustrated`, `confused`, `adversarial`, `happy_path`.

**Testing**:
- Integration (mocked LLMs): `adversarial` persona → completed simulation with a populated scorecard.
- Unit: judge rubric scores a known-good transcript higher than a known-bad one.

#### 10.3 — Dashboard & Drag-and-Drop Flow Builder

**What**: Next.js web UI for configuration, monitoring, and visual flow design.

**Design**:
Pages: org/auth (OIDC login), agents CRUD, provider config, phone numbers, live call monitor (WS live transcript), call history + transcript viewer (redacted by default), campaigns, KB editor + gap review queue, analytics, compliance dashboard. Flow builder: React Flow canvas editing `conversation_flows.graph_json` (node palette = the Phase 5 node types), saving versioned flows via REST. API client generated from the OpenAPI spec.

**Testing**:
- E2E (Playwright, mocked API): create agent → configure providers → save → appears in list.
- E2E: drag two nodes, connect, save → persisted `graph_json` matches the canvas.
- E2E: live call monitor renders incoming `segment` WS events in order.
- Component (Vitest): transcript viewer shows redacted text for a viewer role.

#### 10.4 — Packaging & Deployment

**What**: Docker images, compose stack, and Helm chart for self-hosted Kubernetes.

**Design**:
`Dockerfile.api`, `Dockerfile.worker`, `Dockerfile.agent`; `docker-compose.yml` (api, worker, agent, postgres, redis, livekit, frontend) for local dev; a Helm chart for production with externalised Postgres/Redis/LiveKit and Vault integration. Healthchecks and readiness probes on each service.

**Testing**:
- Integration: `docker compose up` → API healthy at `/health`, migrations applied, a seeded agent answers a WebRTC test call.
- Integration: `helm template` renders valid manifests; `helm lint` passes.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, config, auth)        ─── required by everything
    │
Phase 2: Provider abstraction layer            ─── requires P1
    │
Phase 3: Realtime voice pipeline (WebRTC)      ─── requires P2   ★ core value
    ├── Phase 4: Telephony (SIP, carriers)      ─── requires P3
    └── Phase 5: Flow builder, MCP, REST/webhooks ─ requires P3   (parallel with P4)
         │
Phase 6: Compliance (redaction, consent)        ─── requires P3 + P4
    ├── Phase 7: Squads & warm transfer          ─── requires P3 + P6
    ├── Phase 8: Multilingual, KB, gap detection ─── requires P3 + P6  (parallel with P7, P9)
    └── Phase 9: Outbound campaigns              ─── requires P4 + P6  (parallel with P7, P8)
         │
Phase 10: Analytics, simulation, cost, frontend ─── requires most prior phases
         (frontend work can begin against the OpenAPI spec from P5 onward)
```

**Parallelism opportunities:**
- Phases 4 and 5 can be developed concurrently once Phase 3 is complete.
- Phases 7, 8, and 9 can be developed concurrently once Phase 6 is complete.
- Frontend (10.3) can begin as soon as the OpenAPI spec stabilises in Phase 5 and proceed in parallel.

**MVP cutline:** Phases 1–6 deliver the research's "Must-have (MVP)" scope (pluggable pipeline, semantic turn detection, inbound/outbound telephony, flow builder + REST, recording/transcripts/analytics, webhooks, HIPAA/GDPR + PII redaction). Phases 7–9 deliver "Should-have (v1.1)" (squads, warm transfer, simulation, KB gap detection, campaigns, multilingual). Phase 10's simulation, cost calculator, and white-label hooks reach into the "Nice-to-have" backlog.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and mocked-integration tests pass; real-API tests (`@pytest.mark.realapi`) pass or are explicitly skipped in CI.
3. `ruff check` and `ruff format --check` pass (and ESLint/Prettier for frontend changes).
4. `mypy --strict` passes for changed Python modules.
5. Docker images for affected services build successfully.
6. The phase's feature works end-to-end (demonstrated by an integration or E2E test).
7. New configuration options are documented in `.env.example` and the README config section.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec and validate against the meta-schema.
9. Database changes ship as an Alembic migration that round-trips (`upgrade head` → `downgrade` → `upgrade`).
10. All mutating endpoints write an `audit_log` row (SOC 2 coverage).
11. Any handling of recordings/transcripts respects the org's compliance settings (encryption, redaction, retention).
```
