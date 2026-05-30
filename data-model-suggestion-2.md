# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: AI Voice Agent Builder · Created: 2026-05-25

## Philosophy

This model collapses the 16-table normalized schema into 6 core tables by embedding related data as JSONB documents within their parent entities. Organisations embed their users, API keys, provider configurations, phone numbers, and compliance settings. Voice agents embed their conversation flow graph, squad/handoff configuration, and knowledge base. The principle: loading everything needed to handle a call should require at most two rows — the organisation (for auth and compliance) and the voice agent (for the complete pipeline config, flow, and knowledge).

Calls remain as a standalone partitioned table because they arrive at high volume and carry the per-turn transcript, cost breakdown, and compliance metadata. Campaigns remain standalone because they have independent lifecycle management (scheduling, progress tracking, TCPA enforcement).

**Best for:** Early-stage voice agent platforms prioritising development speed, single-query agent context loading, and rapid iteration on the provider pipeline and conversation flow formats. Ideal for single-org or small-team deployments where agent configuration and call analytics are the primary data concerns.

**Trade-offs:**
- (+) 6 tables — minimal migration surface as the product evolves
- (+) Single-row fetch loads complete agent context (flow + providers + squad + knowledge)
- (+) Organisation with full config in one row
- (+) Adding new provider types or flow node types is a JSONB schema change, not a migration
- (-) Cross-agent provider analysis requires JSONB extraction
- (-) No foreign-key enforcement between agent provider references and org provider configs
- (-) Knowledge base entries embedded in agent — no independent KB entity lifecycle
- (-) Per-turn transcript segments embedded in calls — large JSONB for long calls

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SIP (RFC 3261) | `organisations.phone_numbers_json[].sip_trunk_config` for telephony |
| WebRTC | `calls.transport` tracks channel type |
| SSML 1.1 | `voice_agents.tts_json.ssml_enabled` |
| MCP v2.1 | `voice_agents.mcp_servers_json` for tool calling |
| OAuth 2.0 / OIDC | API keys in `organisations.api_keys_json[]` |
| HIPAA | `organisations.compliance_json.hipaa`; `calls.compliance_json.hipaa_compliant` |
| PCI DSS | `calls.transcript_json[].pci_redacted` |
| GDPR | `calls.compliance_json.consent_verified`; `organisations.compliance_json.gdpr` |
| TCPA | `campaigns.tcpa_json` with consent verification and calling hours |
| SOC 2 | `audit_log` captures all changes |
| OpenAPI 3.1 | REST API published as OpenAPI |

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
    monthly_budget_cents BIGINT,

    users_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"ops@example.com","full_name":"Jane Ops",
    --   "role":"operator","is_active":true}]

    api_keys_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"prod-api","key_prefix":"voice_",
    --   "key_hash":"sha256...","scopes":["calls","agents","campaigns"],
    --   "rate_limit_rpm":500,"is_active":true}]

    providers_json  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","slug":"deepgram-nova","type":"stt",
    --   "provider":"deepgram","model":"nova-3","credential_ref":"vault://deepgram-prod",
    --   "config":{"language":"en","punctuate":true,"interim_results":true}},
    --  {"id":"uuid","slug":"claude-sonnet","type":"llm",
    --   "provider":"anthropic","model":"claude-sonnet-4-6",
    --   "credential_ref":"vault://anthropic-prod",
    --   "config":{"temperature":0.7,"max_tokens":500}},
    --  {"id":"uuid","slug":"elevenlabs-rachel","type":"tts",
    --   "provider":"elevenlabs","model":"eleven_turbo_v2.5",
    --   "credential_ref":"vault://elevenlabs-prod",
    --   "config":{"voice_id":"rachel","stability":0.5}}]

    phone_numbers_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","number":"+14155551234","country_code":"US",
    --   "type":"local","carrier":"twilio","agent_slug":"receptionist",
    --   "sip_trunk_config":null,"is_active":true}]

    compliance_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"hipaa":{"enabled":true,"baa_signed":true,"baa_date":"2026-01-15"},
    --   "gdpr":{"enabled":true,"data_retention_days":90},
    --   "tcpa":{"enabled":true,"dnc_list_source":"internal"},
    --   "pci_dss":{"enabled":false}}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
CREATE INDEX idx_orgs_keys ON organisations USING GIN (api_keys_json);
CREATE INDEX idx_orgs_numbers ON organisations USING GIN (phone_numbers_json);
```

### voice_agents

Agent configurations with embedded conversation flow, squad config, and knowledge base.

```sql
CREATE TABLE voice_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,

    system_prompt   TEXT NOT NULL,
    first_message   TEXT,
    language        TEXT NOT NULL DEFAULT 'en',
    languages_supported TEXT[] NOT NULL DEFAULT '{en}',

    stt_provider_slug TEXT,
    llm_provider_slug TEXT,
    tts_provider_slug TEXT,

    voice_json      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"voice_id":"rachel","ssml_enabled":true,
    --   "interruption_handling":"semantic","end_call_phrases":["goodbye","thank you bye"],
    --   "max_call_duration_seconds":1800,"voicemail_detection":true,
    --   "voicemail_message":"Please leave a message after the beep."}

    flow_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"type":"hybrid","version":3,
    --   "nodes":[
    --     {"id":"start","type":"greeting","message":"Hi, how can I help?",
    --       "transitions":[{"condition":"intent:schedule","target":"schedule"},
    --         {"condition":"default","target":"fallback"}]},
    --     {"id":"schedule","type":"tool_call","tool":"mcp:calendar:book",
    --       "transitions":[{"condition":"success","target":"confirm"}]}
    --   ]}

    squad_json      JSONB,
    -- Example: {"name":"Customer Service Squad",
    --   "members":[
    --     {"agent_slug":"receptionist","role":"entry_point",
    --       "handoff_rules":[{"condition":"intent:billing","target":"billing_agent",
    --         "transfer_type":"warm","context":["customer_name","sentiment"]}]},
    --     {"agent_slug":"billing_agent","role":"specialist"}
    --   ],
    --   "context_preservation":["transcript","sentiment","customer_info"],
    --   "max_handoffs":3}

    knowledge_json  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"name":"Product FAQ","entry_count":150,"gap_count":12,
    --   "entries":[
    --     {"id":"uuid","question":"What are your hours?","answer":"We're open 9-5 M-F.",
    --       "source":"manual","tags":["hours","general"],"retrieval_count":45},
    --     {"id":"uuid","question":"How do I reset my password?",
    --       "answer":"Go to settings > security > reset.",
    --       "source":"gap_detection","gap_query":"password reset help",
    --       "retrieval_count":12}
    --   ]}

    mcp_servers_json JSONB NOT NULL DEFAULT '[]',

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_agents_org ON voice_agents(org_id);
```

### calls

Call records with embedded transcript, cost breakdown, and compliance metadata. Partitioned by time.

```sql
CREATE TABLE calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    agent_slug      TEXT NOT NULL,
    campaign_id     UUID,

    direction       TEXT NOT NULL CHECK (direction IN ('inbound','outbound')),
    transport       TEXT NOT NULL DEFAULT 'sip'
                    CHECK (transport IN ('sip','webrtc','mrcp')),
    caller_number   TEXT,
    callee_number   TEXT,
    sip_call_id     TEXT,

    status          TEXT NOT NULL DEFAULT 'ringing'
                    CHECK (status IN ('ringing','connected','in_progress','transferring',
                                       'completed','failed','voicemail','no_answer','busy')),

    started_at      TIMESTAMPTZ,
    connected_at    TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,

    recording_url   TEXT,

    transcript_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"order":1,"speaker":"agent","text":"Hi, how can I help you today?",
    --     "start_ms":0,"duration_ms":2100,"confidence":0.98,"language":"en"},
    --   {"order":2,"speaker":"caller","text":"I need to schedule an appointment.",
    --     "text_redacted":"I need to schedule an appointment.",
    --     "start_ms":2500,"duration_ms":1800,"confidence":0.95,
    --     "sentiment":0.6,"intent":"schedule","pii_redacted":false},
    --   {"order":3,"speaker":"agent","text":"I'd be happy to help with that.",
    --     "start_ms":4500,"duration_ms":1500,
    --     "tool_call":{"tool":"mcp:calendar:book","input":{"date":"2026-05-26"},"output":{"confirmed":true}}}
    -- ]

    analytics_json  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"sentiment_score":0.72,"sentiment_label":"positive",
    --   "languages_used":["en"],"language_switches":0,
    --   "voicemail_detected":false,"handoffs":0,"human_transfer":false,
    --   "intents_detected":["schedule","confirm"],
    --   "knowledge_gaps":["after-hours emergency number"]}

    cost_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"stt_cents":2,"llm_cents":3,"tts_cents":4,"telephony_cents":5,
    --   "total_cents":14,"tokens_used":850}

    compliance_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"hipaa_compliant":true,"pii_detected":true,
    --   "pii_categories":["phone_number","name"],"pci_redacted":false,
    --   "consent_verified":true,"recording_encrypted":true}

    end_reason      TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_calls_org ON calls(org_id, created_at DESC);
CREATE INDEX idx_calls_agent ON calls(agent_slug, created_at DESC);
CREATE INDEX idx_calls_campaign ON calls(campaign_id, created_at DESC)
    WHERE campaign_id IS NOT NULL;
CREATE INDEX idx_calls_caller ON calls(caller_number, created_at DESC);
CREATE INDEX idx_calls_status ON calls(status) WHERE status NOT IN ('completed','failed');
```

### campaigns

Outbound batch calling with TCPA enforcement.

```sql
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    agent_slug      TEXT NOT NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,

    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','scheduled','running','paused','completed','cancelled')),

    targets_json    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"phone":"+14155551234","name":"John Doe","consent_verified":true,
    --   "custom_data":{"account_id":"A-123","amount_due_cents":5000}},
    --  {"phone":"+14155555678","name":"Jane Smith","consent_verified":true}]

    schedule_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"start_date":"2026-05-26","end_date":"2026-05-30",
    --   "calling_hours":{"start":"09:00","end":"17:00","timezone":"America/New_York"},
    --   "max_concurrent":50,"retry_attempts":2,"retry_delay_hours":4}

    tcpa_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"compliant":true,"dnc_checked":true,
    --   "consent_type":"tcpa_express_written","consent_verified_at":"2026-05-20"}

    progress_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"target_count":500,"completed":320,"connected":280,
    --   "voicemail":25,"failed":15,"pending":180,
    --   "conversion_rate":0.42,"avg_duration_seconds":180}

    total_cost_cents BIGINT NOT NULL DEFAULT 0,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_campaigns_org ON campaigns(org_id, created_at DESC);
CREATE INDEX idx_campaigns_status ON campaigns(status) WHERE status IN ('scheduled','running');
```

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

### Load complete agent context for call handling

```sql
SELECT va.system_prompt, va.first_message, va.voice_json, va.flow_json,
       va.squad_json, va.knowledge_json, va.mcp_servers_json,
       va.stt_provider_slug, va.llm_provider_slug, va.tts_provider_slug,
       o.providers_json, o.compliance_json
FROM voice_agents va
JOIN organisations o ON o.id = va.org_id
WHERE va.org_id = $1 AND va.slug = $2;
```

### Call transcript with analytics

```sql
SELECT c.direction, c.caller_number, c.duration_seconds,
       c.transcript_json, c.analytics_json, c.cost_json, c.compliance_json
FROM calls c
WHERE c.id = $1;
```

### Campaign progress

```sql
SELECT c.name, c.status, c.progress_json, c.tcpa_json,
       c.total_cost_cents, c.schedule_json
FROM campaigns c
WHERE c.org_id = $1 AND c.status IN ('running','scheduled')
ORDER BY c.created_at DESC;
```

### Cost breakdown by provider component

```sql
SELECT cr.agent_slug, cr.direction,
       SUM(cr.stt_cost_cents) AS stt, SUM(cr.llm_cost_cents) AS llm,
       SUM(cr.tts_cost_cents) AS tts, SUM(cr.telephony_cost_cents) AS telephony,
       SUM(cr.total_cost_cents) AS total,
       SUM(cr.call_count) AS calls
FROM cost_records cr
WHERE cr.org_id = $1 AND cr.period_type = 'daily'
  AND cr.period >= date_trunc('month', now())
GROUP BY cr.agent_slug, cr.direction
ORDER BY total DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation (with users, keys, providers, numbers, compliance) | 1 | organisations |
| Voice Agents (with flow, squad, knowledge, MCP) | 1 | voice_agents |
| Calls (with transcript, analytics, costs, compliance) | 1 | calls (partitioned) |
| Campaigns (with targets, schedule, TCPA, progress) | 1 | campaigns |
| Cost Records | 1 | cost_records |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Agent as complete context document** — `voice_agents` embeds `flow_json` (conversation graph), `squad_json` (multi-agent orchestration), `knowledge_json` (FAQ entries), and `mcp_servers_json` (tool configs). Loading one row provides everything needed to handle a call. Provider slugs reference entries in the org's `providers_json`.

2. **Transcript embedded in calls** — `calls.transcript_json` stores the full per-turn transcript as a JSONB array. Each segment includes speaker, text, redacted text, timing, sentiment, intent, and tool calls. This keeps the complete call context in one row for debugging and QA. The trade-off: very long calls produce large JSONB documents.

3. **Provider configs embedded in org** — `organisations.providers_json` stores all configured STT, LLM, and TTS providers with credentials (vault references). Agents reference providers by slug. This enables single-query context loading: one org row plus one agent row provides the complete pipeline configuration.

4. **Compliance settings embedded in org** — `organisations.compliance_json` stores HIPAA, GDPR, TCPA, and PCI DSS settings at the org level. When HIPAA is enabled, all calls automatically inherit compliance requirements. This avoids a separate compliance configuration table.

5. **Campaign targets as JSONB array** — `campaigns.targets_json` embeds the target phone list with per-target consent verification and custom data. The trade-off: very large campaigns (10,000+ targets) will produce large JSONB documents; at that scale, a separate target table would be better.

6. **Call analytics embedded** — `calls.analytics_json` stores derived metrics (sentiment, language switches, intents detected, knowledge gaps) directly on the call row. This enables single-query call review without computing analytics at read time.

7. **Cost breakdown embedded in calls** — `calls.cost_json` breaks down cost by STT, LLM, TTS, and telephony components. Combined with the relational `cost_records` for aggregation, this provides both per-call transparency and period-level billing.

8. **Knowledge base with gap tracking** — `voice_agents.knowledge_json.entries[]` includes entries with `source: "gap_detection"` and the original `gap_query`. This supports the automatic knowledge base expansion pattern without a separate gap tracking table.

9. **Phone numbers embedded in org** — `organisations.phone_numbers_json` stores provisioned numbers with their assigned agent slug. Number lookup for incoming calls uses the GIN index for fast routing.

10. **TCPA enforcement on campaigns** — `campaigns.tcpa_json` tracks consent type, verification status, and DNC list checks. Combined with the calling hours in `schedule_json`, this enforces TCPA compliance at the campaign level.
