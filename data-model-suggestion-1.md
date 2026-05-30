# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI Voice Agent Builder · Created: 2026-05-25

## Philosophy

This model gives every concept its own table: organisations, users, API keys, voice agent configurations, conversation flows, provider pipeline configs, phone numbers, multi-agent squads, calls with per-turn transcript segments, knowledge bases, outbound campaigns, and consent records. Foreign keys enforce referential integrity across the full lifecycle — from configuring a voice agent with its STT/LLM/TTS pipeline, through deploying it on a phone number, to recording and transcribing calls, managing campaigns, and tracking TCPA/GDPR compliance.

The call pipeline is modelled as a chain: a `call` connects to a `voice_agent` via a `phone_number`, produces `call_segments` (per-turn transcript entries with speaker identification and PII redaction status), and may belong to a `campaign`. Each entity has its own table because it has an independent lifecycle and distinct query patterns — calls are high-volume and partitioned, while agent configurations change infrequently.

Provider configurations are relational: `provider_configs` stores STT, LLM, and TTS provider settings independently, enabling mix-and-match pipeline assembly without vendor lock-in. Squads define multi-agent orchestration with handoff rules and context preservation.

**Best for:** Production voice agent platforms serving multiple customers where HIPAA, GDPR, TCPA, and PCI DSS compliance are primary requirements. Ideal when the provider pipeline, campaign management, and per-turn transcript analysis are core features with independent lifecycles.

**Trade-offs:**
- (+) Full referential integrity across agents, calls, transcripts, and campaigns
- (+) Per-turn transcript segments independently queryable for sentiment analysis and QA
- (+) Provider configs independently manageable — swap STT without touching agent config
- (+) Consent records as first-class entities for TCPA/GDPR compliance
- (+) Knowledge base entries with gap detection as standalone entities
- (-) 16 tables — more migrations as the product evolves
- (-) Loading a complete agent context (config + flow + providers + squad) requires multiple joins
- (-) High-volume call segments generate many rows
- (-) Campaign management requires joining campaigns, calls, and consent records

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SIP (RFC 3261) | `phone_numbers.sip_trunk_id` and `calls.sip_call_id` for telephony routing |
| WebRTC | `calls.transport` tracks WebRTC vs SIP channel |
| SRTP (RFC 3711) | `calls.encryption` tracks media encryption status for HIPAA |
| SSML 1.1 (W3C) | `voice_agents.ssml_enabled` for TTS markup support |
| MCP v2.1 | `voice_agents.mcp_servers_json` stores MCP tool configurations for agent tool calling |
| OAuth 2.0 / OIDC | `api_keys` for platform auth; OIDC for SSO |
| HIPAA | `calls.hipaa_compliant`; `call_segments.pii_redacted`; `organisations.baa_signed` |
| PCI DSS | `call_segments.pci_redacted` for payment card data removal |
| GDPR | `consent_records` tracks consent; `calls.data_retention_expires_at` for erasure |
| TCPA | `consent_records` with consent type and calling-hours enforcement; `campaigns.tcpa_compliant` |
| SOC 2 Type II | `audit_log` captures all access and changes |
| OpenAPI 3.1 | REST API documented as OpenAPI; `api_keys.scopes` map to security schemes |
| WebSocket (RFC 6455) | Real-time audio streaming; `calls.is_streaming` |

---

## Core Tables

### organisations

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'starter' CHECK (plan IN ('starter','growth','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',
    baa_signed      BOOLEAN NOT NULL DEFAULT false,
    hipaa_enabled   BOOLEAN NOT NULL DEFAULT false,
    monthly_budget_cents BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    full_name       TEXT,
    role            TEXT NOT NULL DEFAULT 'developer'
                    CHECK (role IN ('owner','admin','developer','operator','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE INDEX idx_users_org ON users(org_id);
```

### api_keys

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    key_prefix      TEXT NOT NULL,
    key_hash        TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{calls,agents}',
    rate_limit_rpm  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_keys_org ON api_keys(org_id);
CREATE INDEX idx_keys_prefix ON api_keys(key_prefix);
```

---

## Agent Configuration

### voice_agents

```sql
CREATE TABLE voice_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,

    system_prompt   TEXT NOT NULL,
    first_message   TEXT,
    voice_id        TEXT,
    language        TEXT NOT NULL DEFAULT 'en',
    languages_supported TEXT[] NOT NULL DEFAULT '{en}',

    stt_provider_id UUID REFERENCES provider_configs(id),
    llm_provider_id UUID REFERENCES provider_configs(id),
    tts_provider_id UUID REFERENCES provider_configs(id),

    ssml_enabled    BOOLEAN NOT NULL DEFAULT false,
    interruption_handling TEXT NOT NULL DEFAULT 'semantic'
                    CHECK (interruption_handling IN ('semantic','vad','none')),
    end_call_phrases TEXT[] NOT NULL DEFAULT '{}',

    mcp_servers_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name":"crm","url":"https://mcp.example.com/crm",
    --   "tools":["lookup_customer","create_ticket"]}]

    max_call_duration_seconds INTEGER NOT NULL DEFAULT 1800,
    voicemail_detection BOOLEAN NOT NULL DEFAULT true,
    voicemail_message TEXT,

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_agents_org ON voice_agents(org_id);
```

### conversation_flows

Drag-and-drop designed conversation flows. The graph is stored as JSONB.

```sql
CREATE TABLE conversation_flows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id        UUID NOT NULL REFERENCES voice_agents(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    flow_type       TEXT NOT NULL DEFAULT 'prompt'
                    CHECK (flow_type IN ('prompt','structured','hybrid')),

    graph_json      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"nodes":[
    --   {"id":"start","type":"greeting","message":"Hi, how can I help?",
    --     "transitions":[{"condition":"intent:schedule","target":"schedule_node"},
    --       {"condition":"intent:billing","target":"billing_node"},
    --       {"condition":"default","target":"fallback_node"}]},
    --   {"id":"schedule_node","type":"tool_call","tool":"mcp:calendar:book_appointment",
    --     "transitions":[{"condition":"success","target":"confirm_node"}]}
    -- ]}

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agent_id, version)
);

CREATE INDEX idx_flows_agent ON conversation_flows(agent_id);
```

### provider_configs

Pluggable STT, LLM, and TTS provider configurations.

```sql
CREATE TABLE provider_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('stt','llm','tts')),
    provider_name   TEXT NOT NULL,
    -- STT: deepgram, assemblyai, whisper, azure, google
    -- LLM: openai, anthropic, google, groq, ollama, custom
    -- TTS: elevenlabs, cartesia, azure, openai, deepgram, google
    model_id        TEXT NOT NULL,
    api_endpoint    TEXT,
    credential_ref  TEXT NOT NULL,
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- STT example: {"language":"en","model":"nova-3","punctuate":true,"interim_results":true}
    -- LLM example: {"temperature":0.7,"max_tokens":500,"streaming":true}
    -- TTS example: {"voice_id":"rachel","stability":0.5,"similarity_boost":0.75,"speed":1.0}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_providers_org ON provider_configs(org_id, provider_type);
```

### phone_numbers

Provisioned telephony numbers mapped to agents.

```sql
CREATE TABLE phone_numbers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID REFERENCES voice_agents(id),
    number          TEXT NOT NULL,
    country_code    TEXT NOT NULL,
    number_type     TEXT NOT NULL DEFAULT 'local'
                    CHECK (number_type IN ('local','toll_free','mobile','sip')),
    carrier         TEXT NOT NULL CHECK (carrier IN ('twilio','vonage','plivo','bandwidth','sip_trunk')),
    sip_trunk_config_json JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_numbers_org ON phone_numbers(org_id);
CREATE INDEX idx_numbers_number ON phone_numbers(number);
```

### squads

Multi-agent orchestration configurations with handoff rules.

```sql
CREATE TABLE squads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,

    members_json    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"agent_id":"uuid","role":"receptionist","is_entry_point":true,
    --   "handoff_rules":[
    --     {"condition":"intent:billing","target_agent_id":"uuid","transfer_type":"warm",
    --       "context_fields":["customer_name","account_number","sentiment"]},
    --     {"condition":"intent:technical","target_agent_id":"uuid","transfer_type":"warm"}
    --   ]},
    --  {"agent_id":"uuid","role":"billing_specialist","is_entry_point":false},
    --  {"agent_id":"uuid","role":"technical_support","is_entry_point":false}]

    context_preservation TEXT[] NOT NULL DEFAULT '{transcript,sentiment,customer_info}',
    max_handoffs    INTEGER NOT NULL DEFAULT 3,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_squads_org ON squads(org_id);
```

---

## Call Pipeline

### calls

Call records. Partitioned by time for high volume.

```sql
CREATE TABLE calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    agent_id        UUID NOT NULL,
    phone_number_id UUID,
    squad_id        UUID,
    campaign_id     UUID,

    direction       TEXT NOT NULL CHECK (direction IN ('inbound','outbound')),
    transport       TEXT NOT NULL DEFAULT 'sip'
                    CHECK (transport IN ('sip','webrtc','mrcp')),
    caller_number   TEXT,
    callee_number   TEXT,
    sip_call_id     TEXT,
    encryption      TEXT NOT NULL DEFAULT 'srtp'
                    CHECK (encryption IN ('srtp','dtls_srtp','none')),

    status          TEXT NOT NULL DEFAULT 'ringing'
                    CHECK (status IN ('ringing','connected','in_progress','transferring',
                                       'completed','failed','voicemail','no_answer','busy')),

    started_at      TIMESTAMPTZ,
    connected_at    TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,

    recording_url   TEXT,
    recording_encrypted BOOLEAN NOT NULL DEFAULT true,

    language_detected TEXT,
    languages_used  TEXT[] NOT NULL DEFAULT '{}',
    voicemail_detected BOOLEAN NOT NULL DEFAULT false,

    sentiment_score NUMERIC(5,4),
    sentiment_label TEXT CHECK (sentiment_label IS NULL OR sentiment_label IN (
                        'very_positive','positive','neutral','negative','very_negative')),

    handoffs        INTEGER NOT NULL DEFAULT 0,
    human_transfer  BOOLEAN NOT NULL DEFAULT false,
    transfer_context_json JSONB,

    tokens_used     INTEGER,
    stt_cost_cents  BIGINT NOT NULL DEFAULT 0,
    llm_cost_cents  BIGINT NOT NULL DEFAULT 0,
    tts_cost_cents  BIGINT NOT NULL DEFAULT 0,
    telephony_cost_cents BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,

    hipaa_compliant BOOLEAN NOT NULL DEFAULT false,
    pii_detected    BOOLEAN NOT NULL DEFAULT false,

    end_reason      TEXT CHECK (end_reason IS NULL OR end_reason IN (
                        'caller_hangup','agent_ended','max_duration','transfer','error',
                        'voicemail','no_answer')),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_calls_org ON calls(org_id, created_at DESC);
CREATE INDEX idx_calls_agent ON calls(agent_id, created_at DESC);
CREATE INDEX idx_calls_campaign ON calls(campaign_id, created_at DESC)
    WHERE campaign_id IS NOT NULL;
CREATE INDEX idx_calls_status ON calls(status) WHERE status NOT IN ('completed','failed');
CREATE INDEX idx_calls_caller ON calls(caller_number, created_at DESC);
```

### call_segments

Per-turn transcript segments with speaker identification, timestamps, and PII redaction.

```sql
CREATE TABLE call_segments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL,
    segment_order   INTEGER NOT NULL,
    speaker         TEXT NOT NULL CHECK (speaker IN ('caller','agent','system')),
    text            TEXT NOT NULL,
    text_redacted   TEXT,

    start_ms        INTEGER NOT NULL,
    end_ms          INTEGER NOT NULL,
    duration_ms     INTEGER NOT NULL,

    confidence      NUMERIC(5,4),
    language        TEXT,

    pii_redacted    BOOLEAN NOT NULL DEFAULT false,
    pii_categories  TEXT[] NOT NULL DEFAULT '{}',
    pci_redacted    BOOLEAN NOT NULL DEFAULT false,

    sentiment_score NUMERIC(5,4),
    intent_detected TEXT,
    tool_call_json  JSONB,
    -- Example: {"tool":"mcp:calendar:book_appointment",
    --   "input":{"date":"2026-05-26","time":"14:00"},"output":{"confirmed":true}}

    agent_id        UUID,
    is_handoff      BOOLEAN NOT NULL DEFAULT false,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_segments_call ON call_segments(call_id, segment_order);
CREATE INDEX idx_segments_intent ON call_segments(intent_detected)
    WHERE intent_detected IS NOT NULL;
```

---

## Knowledge Base

### knowledge_bases

```sql
CREATE TABLE knowledge_bases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID REFERENCES voice_agents(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    entry_count     INTEGER NOT NULL DEFAULT 0,
    gap_count       INTEGER NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_kb_org ON knowledge_bases(org_id);
```

### knowledge_entries

Individual FAQ/document entries with gap tracking.

```sql
CREATE TABLE knowledge_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    kb_id           UUID NOT NULL REFERENCES knowledge_bases(id) ON DELETE CASCADE,
    question        TEXT NOT NULL,
    answer          TEXT NOT NULL,
    source          TEXT NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual','auto_generated','gap_detection','import')),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    retrieval_count INTEGER NOT NULL DEFAULT 0,
    gap_query       TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entries_kb ON knowledge_entries(kb_id);
CREATE INDEX idx_entries_source ON knowledge_entries(source) WHERE source = 'gap_detection';
CREATE INDEX idx_entries_tags ON knowledge_entries USING GIN (tags);
```

---

## Campaigns

### campaigns

Outbound batch calling configurations.

```sql
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_id        UUID NOT NULL REFERENCES voice_agents(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,

    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','scheduled','running','paused','completed','cancelled')),

    target_count    INTEGER NOT NULL DEFAULT 0,
    completed_count INTEGER NOT NULL DEFAULT 0,
    connected_count INTEGER NOT NULL DEFAULT 0,
    voicemail_count INTEGER NOT NULL DEFAULT 0,
    failed_count    INTEGER NOT NULL DEFAULT 0,

    schedule_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"start_date":"2026-05-26","end_date":"2026-05-30",
    --   "calling_hours":{"start":"09:00","end":"17:00","timezone":"America/New_York"},
    --   "max_concurrent":50,"retry_attempts":2,"retry_delay_hours":4}

    tcpa_compliant  BOOLEAN NOT NULL DEFAULT true,
    dnc_list_checked BOOLEAN NOT NULL DEFAULT true,

    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    avg_duration_seconds INTEGER,
    conversion_rate NUMERIC(5,4),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_campaigns_org ON campaigns(org_id, created_at DESC);
CREATE INDEX idx_campaigns_status ON campaigns(status) WHERE status IN ('scheduled','running');
```

---

## Compliance

### consent_records

TCPA and GDPR consent tracking per caller.

```sql
CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    phone_number    TEXT NOT NULL,
    consent_type    TEXT NOT NULL CHECK (consent_type IN ('tcpa_express_written',
                                                           'tcpa_prior_express','gdpr_recording',
                                                           'gdpr_processing','hipaa_authorization')),
    consent_given   BOOLEAN NOT NULL,
    consent_method  TEXT NOT NULL CHECK (consent_method IN ('web_form','verbal','written','api')),
    consent_text    TEXT,
    ip_address      INET,
    given_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_org ON consent_records(org_id, phone_number);
CREATE INDEX idx_consent_phone ON consent_records(phone_number, consent_type);
```

---

## Operations

### cost_records

```sql
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_slug      TEXT,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','monthly')),
    direction       TEXT,
    call_count      INTEGER NOT NULL DEFAULT 0,
    total_duration_seconds BIGINT NOT NULL DEFAULT 0,
    stt_cost_cents  BIGINT NOT NULL DEFAULT 0,
    llm_cost_cents  BIGINT NOT NULL DEFAULT 0,
    tts_cost_cents  BIGINT NOT NULL DEFAULT 0,
    telephony_cost_cents BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    avg_sentiment   NUMERIC(5,4),
    avg_duration_seconds INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, agent_slug, period, period_type, direction)
);

CREATE INDEX idx_cost_org ON cost_records(org_id, period_type, period DESC);
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','system','api_key','ai')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Load agent context for call handling

```sql
SELECT va.*, cf.graph_json, cf.flow_type,
       stt.provider_name AS stt_provider, stt.model_id AS stt_model, stt.config_json AS stt_config,
       llm.provider_name AS llm_provider, llm.model_id AS llm_model, llm.config_json AS llm_config,
       tts.provider_name AS tts_provider, tts.model_id AS tts_model, tts.config_json AS tts_config
FROM voice_agents va
LEFT JOIN conversation_flows cf ON cf.agent_id = va.id AND cf.is_active
LEFT JOIN provider_configs stt ON stt.id = va.stt_provider_id
LEFT JOIN provider_configs llm ON llm.id = va.llm_provider_id
LEFT JOIN provider_configs tts ON tts.id = va.tts_provider_id
WHERE va.id = $1;
```

### Call transcript with PII redaction status

```sql
SELECT cs.segment_order, cs.speaker, cs.text_redacted AS text,
       cs.start_ms, cs.duration_ms, cs.sentiment_score,
       cs.intent_detected, cs.pii_redacted, cs.pii_categories,
       cs.tool_call_json, cs.agent_id, cs.is_handoff
FROM call_segments cs
WHERE cs.call_id = $1
ORDER BY cs.segment_order;
```

### Campaign progress dashboard

```sql
SELECT c.name, c.status, c.target_count, c.completed_count,
       c.connected_count, c.voicemail_count, c.failed_count,
       c.total_cost_cents, c.avg_duration_seconds, c.conversion_rate
FROM campaigns c
WHERE c.org_id = $1 AND c.status IN ('running','scheduled')
ORDER BY c.created_at DESC;
```

### TCPA consent check before outbound call

```sql
SELECT cr.consent_type, cr.consent_given, cr.consent_method,
       cr.given_at, cr.revoked_at
FROM consent_records cr
WHERE cr.org_id = $1 AND cr.phone_number = $2
  AND cr.consent_type = 'tcpa_express_written'
  AND cr.consent_given = true AND cr.revoked_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenant Identity | 3 | organisations, users, api_keys |
| Agent Configuration | 4 | voice_agents, conversation_flows, provider_configs, squads |
| Telephony | 1 | phone_numbers |
| Call Pipeline | 2 | calls (partitioned), call_segments |
| Knowledge Base | 2 | knowledge_bases, knowledge_entries |
| Campaigns | 1 | campaigns |
| Compliance | 1 | consent_records |
| Operations | 2 | cost_records, audit_log (partitioned) |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Provider configs as independent entities** — `provider_configs` stores STT, LLM, and TTS provider settings as separate rows. A voice agent references providers by FK, enabling mix-and-match pipeline assembly: swap Deepgram for AssemblyAI without touching the agent config. This models the pluggable pipeline architecture that differentiates open platforms from vendor-locked incumbents.

2. **Call segments with PII and PCI redaction** — `call_segments` stores per-turn transcript entries with `pii_redacted`, `pii_categories`, and `pci_redacted` flags. Redacted text is stored in `text_redacted` alongside the original (access-controlled) `text`. This supports HIPAA (PHI protection), PCI DSS (card number redaction), and GDPR (right to erasure) compliance.

3. **Squads for multi-agent orchestration** — `squads` defines multi-agent configurations with handoff rules stored as JSONB. Each member references a `voice_agent` with conditions for warm transfer and context preservation fields. This models the Vapi Squads pattern identified as a key differentiator.

4. **Consent records for TCPA compliance** — `consent_records` tracks per-phone-number consent with type (TCPA express written, GDPR recording, HIPAA authorization), method, and revocation. The FCC ruling that AI voices are "artificial voices" under TCPA makes this table a legal requirement for outbound calling.

5. **Knowledge entries with gap detection** — `knowledge_entries` with `source = 'gap_detection'` and `gap_query` captures questions the agent couldn't answer during real calls. This enables the automatic knowledge base expansion pattern identified as a key differentiator.

6. **Conversation flows as versioned graphs** — `conversation_flows` stores the drag-and-drop designed graph as JSONB with versioning. The `flow_type` distinguishes prompt-based (LLM-native), structured (IVR-style), and hybrid approaches.

7. **Cost breakdown per provider** — `calls` stores cost broken down by STT, LLM, TTS, and telephony components. This addresses the billing transparency gap identified as the dominant user complaint — operators can see exactly where their money goes.

8. **Sentiment tracking on calls and segments** — Both `calls.sentiment_score` and `call_segments.sentiment_score` track sentiment at the call level and per-turn level. This enables the warm transfer use case: escalate to a human agent when segment-level sentiment drops below a threshold.

9. **Campaigns with TCPA enforcement** — `campaigns` includes `tcpa_compliant` and `dnc_list_checked` flags, plus `schedule_json` with calling hours and timezone. The platform enforces TCPA calling-hours restrictions at the campaign level.

10. **HIPAA as a first-class org setting** — `organisations.baa_signed` and `organisations.hipaa_enabled` gate HIPAA compliance features at the org level. When enabled, all calls automatically use encrypted recording, PII redaction, and access-controlled transcripts.
