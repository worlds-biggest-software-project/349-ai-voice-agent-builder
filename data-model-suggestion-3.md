# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: AI Voice Agent Builder · Created: 2026-05-25

## Philosophy

This model treats every state change as an immutable event in a single append-only `event_store` table, with materialised read models projected from the event stream. The event store is the sole source of truth; the read model tables are disposable projections that can be rebuilt by replaying events.

For an AI voice agent platform, event sourcing captures the real-time nature of calls: every turn in a conversation (caller speaks → STT transcribes → LLM reasons → TTS responds) is an event. The call itself is a stream of events with a natural beginning and end. This enables forensic replay of any call's complete journey — critical for HIPAA compliance (access auditing), TCPA dispute resolution (proving consent was verified), and QA (understanding why the agent gave a specific response).

Configuration changes are events too: when an operator swaps the TTS provider from ElevenLabs to Cartesia, that's a `tts_provider_changed` event. Replaying configuration events against call quality metrics answers "did latency improve after the provider swap?" — a question that's hard to answer with mutable state.

Read models are optimised for the platform's primary access patterns: agent context loading for call handling, call analytics dashboards, campaign progress tracking, and compliance reporting.

**Best for:** Regulated environments (HIPAA, TCPA, PCI DSS) where immutable call records and compliance audit trails are non-negotiable, platforms that need to replay calls for QA and training, and organisations that want to measure the impact of configuration changes (provider swaps, prompt updates) on call quality and cost.

**Trade-offs:**
- (+) Immutable call records are the data model itself — every turn, tool call, and handoff is an event
- (+) Full call replay: from ring to hangup, including STT/LLM/TTS timing at every turn
- (+) Configuration change impact analysis: correlate provider swaps with quality metrics
- (+) TCPA consent verification provable from event trail
- (+) Read models are disposable and rebuildable
- (-) 8 tables (3 infrastructure + 5 read models)
- (-) High event volume: each call generates 10-50+ events depending on duration
- (-) CQRS adds complexity for real-time call handling (eventual consistency)
- (-) PII in call events requires careful access control on the event store

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SIP (RFC 3261) | `call_initiated` events with SIP call ID and transport metadata |
| WebRTC | Transport type tracked in call events |
| SRTP (RFC 3711) | Encryption status recorded in call connection events |
| SSML 1.1 | TTS configuration events include SSML enablement |
| MCP v2.1 | `tool_invoked` events record MCP tool calls with inputs/outputs |
| CloudEvents 1.0.2 | Event envelope: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` |
| OAuth 2.0 / OIDC | Auth events in org stream |
| HIPAA | Event store IS the access audit trail; PII events with redaction metadata |
| PCI DSS | `pci_data_detected` events trigger redaction; provable via event replay |
| GDPR | Consent events; data retention via partition management |
| TCPA | `consent_verified` events provable for dispute resolution; calling hours tracked |
| SOC 2 | Every configuration and access change is an event |

---

## Infrastructure Tables

### event_store

Single append-only table for all domain events. Partitioned by time.

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN ('call','agent','campaign',
                                                          'knowledge','compliance',
                                                          'config','org','billing')),
    stream_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   INTEGER NOT NULL,
    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ce_source       TEXT NOT NULL DEFAULT '/voice-agent-builder',
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'system'
                    CHECK (actor_type IN ('user','system','api_key','ai','caller')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_actor ON event_store(actor_id, created_at DESC) WHERE actor_id IS NOT NULL;
```

**Stream types and key events:**

| Stream | Key Events |
|--------|------------|
| `call` | `call_initiated`, `call_connected`, `caller_spoke`, `stt_transcribed`, `llm_responded`, `tts_synthesised`, `agent_spoke`, `tool_invoked`, `pii_detected`, `pii_redacted`, `pci_data_detected`, `sentiment_shifted`, `language_switched`, `handoff_initiated`, `handoff_completed`, `human_transfer`, `voicemail_detected`, `call_ended`, `recording_stored` |
| `agent` | `agent_created`, `agent_updated`, `prompt_changed`, `flow_updated`, `voice_changed`, `provider_assigned`, `squad_configured`, `agent_activated`, `agent_deactivated` |
| `campaign` | `campaign_created`, `campaign_scheduled`, `campaign_started`, `call_queued`, `call_attempted`, `call_completed`, `campaign_paused`, `campaign_resumed`, `campaign_completed` |
| `knowledge` | `kb_created`, `entry_added`, `entry_updated`, `entry_removed`, `gap_detected`, `gap_resolved`, `auto_entry_generated` |
| `compliance` | `consent_given`, `consent_revoked`, `baa_signed`, `hipaa_enabled`, `tcpa_check_passed`, `tcpa_check_failed`, `dnc_list_updated`, `data_erasure_requested`, `data_erasure_completed` |
| `config` | `provider_added`, `provider_removed`, `provider_updated`, `phone_number_provisioned`, `phone_number_released`, `stt_provider_changed`, `llm_provider_changed`, `tts_provider_changed` |
| `org` | `org_created`, `user_invited`, `user_role_changed`, `api_key_created`, `api_key_revoked`, `settings_updated`, `plan_changed` |
| `billing` | `cost_recorded`, `budget_threshold_reached`, `budget_exceeded`, `daily_rollup_completed` |

**Example events:**

```json
// call_initiated
{"stream_type":"call","stream_id":"call-uuid","event_type":"call_initiated",
 "data":{"org_id":"org-uuid","agent_slug":"receptionist","direction":"inbound",
   "transport":"sip","caller_number":"+14155551234","callee_number":"+14155559876",
   "sip_call_id":"abc123","encryption":"srtp","phone_number_id":"pn-uuid"}}

// stt_transcribed
{"stream_type":"call","stream_id":"call-uuid","event_type":"stt_transcribed",
 "data":{"segment_order":2,"speaker":"caller",
   "text":"I need to schedule an appointment for my daughter.",
   "start_ms":2500,"duration_ms":2100,"confidence":0.96,"language":"en",
   "stt_provider":"deepgram","stt_model":"nova-3","stt_latency_ms":85}}

// llm_responded
{"stream_type":"call","stream_id":"call-uuid","event_type":"llm_responded",
 "data":{"segment_order":3,"text":"I'd be happy to help schedule that appointment.",
   "llm_provider":"anthropic","llm_model":"claude-sonnet-4-6",
   "tokens_input":420,"tokens_output":35,"llm_latency_ms":180,
   "intent_detected":"schedule","sentiment":0.8}}

// tool_invoked
{"stream_type":"call","stream_id":"call-uuid","event_type":"tool_invoked",
 "data":{"tool":"mcp:calendar:book_appointment",
   "input":{"date":"2026-05-26","time":"14:00","patient":"daughter"},
   "output":{"confirmed":true,"confirmation_id":"APT-789"},
   "tool_latency_ms":250}}

// pii_detected
{"stream_type":"call","stream_id":"call-uuid","event_type":"pii_detected",
 "data":{"segment_order":5,"categories":["phone_number","name"],
   "original_text":"My number is 555-0123, name is Sarah.",
   "redacted_text":"My number is [REDACTED], name is [REDACTED].",
   "pci_data":false}}

// handoff_completed
{"stream_type":"call","stream_id":"call-uuid","event_type":"handoff_completed",
 "data":{"from_agent":"receptionist","to_agent":"billing_specialist",
   "transfer_type":"warm","context_transferred":["customer_name","sentiment","transcript_summary"],
   "handoff_latency_ms":150}}

// consent_given
{"stream_type":"compliance","stream_id":"org-uuid:+14155551234",
 "event_type":"consent_given",
 "data":{"phone_number":"+14155551234","consent_type":"tcpa_express_written",
   "consent_method":"web_form","consent_text":"I agree to receive automated calls...",
   "ip_address":"203.0.113.42"}}

// tts_provider_changed
{"stream_type":"config","stream_id":"org-uuid","event_type":"tts_provider_changed",
 "data":{"agent_slug":"receptionist","old_provider":"elevenlabs","old_model":"eleven_turbo_v2.5",
   "new_provider":"cartesia","new_model":"sonic-3",
   "reason":"Reducing TTS latency"}}
```

### stream_snapshots

Periodic snapshots for fast stream reconstruction.

```sql
CREATE TABLE stream_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    event_count     INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON stream_snapshots(stream_type, stream_id, snapshot_version DESC);
```

### projection_checkpoints

Tracks the last event processed by each read model projection.

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model Tables

### rm_agents

Materialised view of agent configurations with provider pipeline and flow state.

```sql
CREATE TABLE rm_agents (
    agent_id        UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    system_prompt   TEXT NOT NULL,
    first_message   TEXT,
    language        TEXT NOT NULL,
    languages_supported TEXT[] NOT NULL DEFAULT '{}',

    stt_json        JSONB NOT NULL DEFAULT '{}',
    llm_json        JSONB NOT NULL DEFAULT '{}',
    tts_json        JSONB NOT NULL DEFAULT '{}',

    voice_json      JSONB NOT NULL DEFAULT '{}',
    flow_json       JSONB NOT NULL DEFAULT '{}',
    squad_json      JSONB,
    knowledge_json  JSONB NOT NULL DEFAULT '{}',
    mcp_servers_json JSONB NOT NULL DEFAULT '[]',

    phone_numbers   TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,

    compliance_json JSONB NOT NULL DEFAULT '{}',

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_agents_org ON rm_agents(org_id);
CREATE INDEX idx_rm_agents_slug ON rm_agents(org_id, slug);
```

### rm_call_analytics

Materialised view for call quality, cost, and sentiment analysis.

```sql
CREATE TABLE rm_call_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    agent_slug      TEXT,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),

    direction       TEXT,

    call_count      INTEGER NOT NULL DEFAULT 0,
    connected_count INTEGER NOT NULL DEFAULT 0,
    voicemail_count INTEGER NOT NULL DEFAULT 0,
    failed_count    INTEGER NOT NULL DEFAULT 0,
    human_transfer_count INTEGER NOT NULL DEFAULT 0,

    avg_duration_seconds INTEGER,
    avg_sentiment   NUMERIC(5,4),
    avg_latency_stt_ms INTEGER,
    avg_latency_llm_ms INTEGER,
    avg_latency_tts_ms INTEGER,
    avg_latency_total_ms INTEGER,

    stt_cost_cents  BIGINT NOT NULL DEFAULT 0,
    llm_cost_cents  BIGINT NOT NULL DEFAULT 0,
    tts_cost_cents  BIGINT NOT NULL DEFAULT 0,
    telephony_cost_cents BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,

    pii_detected_count INTEGER NOT NULL DEFAULT 0,
    knowledge_gap_count INTEGER NOT NULL DEFAULT 0,
    top_intents     TEXT[] NOT NULL DEFAULT '{}',
    top_knowledge_gaps TEXT[] NOT NULL DEFAULT '{}',

    stt_provider_at_period TEXT,
    llm_provider_at_period TEXT,
    tts_provider_at_period TEXT,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, agent_slug, period, period_type, direction)
);

CREATE INDEX idx_rm_analytics_org ON rm_call_analytics(org_id, period_type, period DESC);
```

### rm_campaign_dashboard

Materialised view for campaign progress and TCPA compliance tracking.

```sql
CREATE TABLE rm_campaign_dashboard (
    campaign_id     UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    agent_slug      TEXT NOT NULL,
    name            TEXT NOT NULL,
    status          TEXT NOT NULL,

    target_count    INTEGER NOT NULL DEFAULT 0,
    completed_count INTEGER NOT NULL DEFAULT 0,
    connected_count INTEGER NOT NULL DEFAULT 0,
    voicemail_count INTEGER NOT NULL DEFAULT 0,
    failed_count    INTEGER NOT NULL DEFAULT 0,
    pending_count   INTEGER NOT NULL DEFAULT 0,

    conversion_rate NUMERIC(5,4),
    avg_duration_seconds INTEGER,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,

    schedule_json   JSONB NOT NULL DEFAULT '{}',
    tcpa_verified   BOOLEAN NOT NULL DEFAULT true,
    consent_check_failures INTEGER NOT NULL DEFAULT 0,

    started_at      TIMESTAMPTZ,
    last_call_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_campaign_org ON rm_campaign_dashboard(org_id, status);
```

### rm_compliance_dashboard

Materialised view for HIPAA, TCPA, GDPR, and PCI DSS compliance tracking.

```sql
CREATE TABLE rm_compliance_dashboard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','monthly')),

    hipaa_json      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"baa_signed":true,"calls_with_phi":45,"pii_redactions":120,
    --   "access_log_entries":350}

    tcpa_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"outbound_calls":500,"consent_verified":495,
    --   "consent_failures":5,"dnc_blocks":12,"calling_hours_violations":0}

    gdpr_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"data_subject_requests":3,"erasures_completed":2,
    --   "consent_recordings":150,"data_retention_purges":45}

    pci_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"pci_detections":8,"pci_redactions":8,"unredacted":0}

    total_calls     INTEGER NOT NULL DEFAULT 0,
    calls_with_pii  INTEGER NOT NULL DEFAULT 0,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, period, period_type)
);

CREATE INDEX idx_rm_compliance_org ON rm_compliance_dashboard(org_id, period_type, period DESC);
```

### rm_cost_dashboard

Materialised view for cost monitoring with provider-level breakdown.

```sql
CREATE TABLE rm_cost_dashboard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
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

    cost_per_minute_cents NUMERIC(8,2),
    budget_cents    BIGINT,
    budget_utilisation NUMERIC(5,4),

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, agent_slug, period, period_type, direction)
);

CREATE INDEX idx_rm_cost_org ON rm_cost_dashboard(org_id, period_type, period DESC);
```

---

## Example Queries

### Replay a call's complete lifecycle

```sql
SELECT event_type, data, ce_time
FROM event_store
WHERE stream_type = 'call' AND stream_id = $1
ORDER BY event_version;
```

### Correlate provider swap with latency improvement

```sql
SELECT e.data->>'new_provider' AS provider,
       e.data->>'new_model' AS model,
       e.ce_time AS changed_at,
       ca.avg_latency_tts_ms, ca.avg_latency_total_ms, ca.period
FROM event_store e
JOIN rm_call_analytics ca ON ca.org_id = $1
    AND ca.agent_slug = (e.data->>'agent_slug')
    AND ca.period >= (e.ce_time::date)
WHERE e.event_type = 'tts_provider_changed'
  AND e.stream_id = $1::uuid
ORDER BY e.ce_time, ca.period;
```

### TCPA consent verification for dispute resolution

```sql
SELECT event_type, data->>'consent_type' AS consent_type,
       data->>'consent_method' AS method,
       data->>'consent_text' AS consent_text,
       data->>'ip_address' AS ip,
       ce_time
FROM event_store
WHERE stream_type = 'compliance'
  AND stream_id = $1
  AND event_type IN ('consent_given','consent_revoked')
ORDER BY event_version;
```

### Knowledge base gap detection over time

```sql
SELECT data->>'gap_query' AS unanswered_question,
       COUNT(*) AS occurrences,
       MIN(ce_time) AS first_seen,
       MAX(ce_time) AS last_seen
FROM event_store
WHERE stream_type = 'knowledge' AND event_type = 'gap_detected'
  AND stream_id = $1
GROUP BY data->>'gap_query'
ORDER BY occurrences DESC
LIMIT 20;
```

### Per-turn latency breakdown for a specific call

```sql
SELECT event_type,
       (data->>'segment_order')::int AS turn,
       data->>'stt_latency_ms' AS stt_ms,
       data->>'llm_latency_ms' AS llm_ms,
       data->>'tts_latency_ms' AS tts_ms,
       ce_time
FROM event_store
WHERE stream_type = 'call' AND stream_id = $1
  AND event_type IN ('stt_transcribed','llm_responded','tts_synthesised')
ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Agent Config | 1 | rm_agents (with providers, flow, squad, knowledge) |
| Call Analytics | 1 | rm_call_analytics (with latency, cost, sentiment, knowledge gaps) |
| Campaign Dashboard | 1 | rm_campaign_dashboard (with TCPA compliance) |
| Compliance Dashboard | 1 | rm_compliance_dashboard (HIPAA, TCPA, GDPR, PCI DSS) |
| Cost Dashboard | 1 | rm_cost_dashboard (with provider-level breakdown) |
| **Total** | **8** | 3 infrastructure + 5 read models |

---

## Key Design Decisions

1. **Call as event stream** — Each call generates 10-50+ events covering the complete lifecycle: `call_initiated` → per-turn events (`caller_spoke`, `stt_transcribed`, `llm_responded`, `tts_synthesised`, `agent_spoke`) → `call_ended`. This enables forensic replay of every turn, including per-component latency (STT, LLM, TTS), which is critical for the sub-300ms latency target.

2. **Per-turn latency tracking** — `stt_transcribed`, `llm_responded`, and `tts_synthesised` events each record component-level latency. Replaying these events answers "which component is the bottleneck on slow calls?" and enables data-driven provider selection.

3. **Configuration changes as events** — `tts_provider_changed`, `llm_provider_changed`, and `prompt_changed` events enable before/after analysis. `rm_call_analytics` tracks `stt_provider_at_period`, `llm_provider_at_period`, and `tts_provider_at_period` to correlate configuration with quality metrics.

4. **TCPA consent as events** — `consent_given` and `consent_revoked` events with method, text, and IP address provide a provable consent trail for TCPA dispute resolution. The event store answers "when did this phone number give consent, via what method, and has it been revoked?"

5. **PII detection and redaction as events** — `pii_detected` and `pii_redacted` events record what was detected, the redaction applied, and whether PCI data was involved. This satisfies HIPAA access auditing and PCI DSS evidence requirements.

6. **Knowledge base gaps as events** — `gap_detected` events record unanswered questions from real calls. Aggregating these events surfaces the most common knowledge gaps for auto-expansion, without a separate gap tracking table.

7. **Multi-agent handoffs as events** — `handoff_initiated` and `handoff_completed` events record which agents were involved, the transfer type (warm/cold), what context was preserved, and handoff latency. This enables QA analysis of the warm transfer experience.

8. **Compliance dashboard from event aggregation** — `rm_compliance_dashboard` projects HIPAA, TCPA, GDPR, and PCI DSS metrics from compliance and call events. This enables real-time compliance monitoring rather than periodic report generation.

9. **Campaign as event stream** — Campaign events (`call_queued`, `call_attempted`, `call_completed`) track individual call progress within a campaign. `rm_campaign_dashboard` projects the aggregate progress, conversion rate, and TCPA compliance status.

10. **Cost-per-minute calculation** — `rm_cost_dashboard` includes `cost_per_minute_cents` calculated from total cost and duration. Combined with the provider-level breakdown (STT, LLM, TTS, telephony), this addresses the billing transparency gap identified as the dominant user complaint.
