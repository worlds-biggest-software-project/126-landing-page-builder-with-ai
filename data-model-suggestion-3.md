# Data Model Suggestion 3: Event-Sourced / Audit-First (CQRS)

> Project: Landing Page Builder with AI · Created: 2026-05-19

## Philosophy

This model treats every change to a page — every section added, element edited, style modified, variant created, experiment started — as an immutable event appended to an event store. The current state of any page is derived by replaying its event stream. Separate read-optimized materialized views (the "Q" in CQRS) serve the page builder UI and the published page renderer, while the event store serves as the authoritative source of truth.

Event sourcing is the pattern behind systems that require complete audit trails: financial ledgers, medical record systems, collaborative editors (Google Docs), and version-controlled platforms (Git). For a landing page builder, the benefits are compelling: every edit is traceable to a user and timestamp, temporal queries ("what did this page look like last Tuesday?") are natural, undo/redo is free, and AI-driven analytics can mine the full history of page changes to discover what editing patterns correlate with conversion improvements.

This approach is best when the platform needs full audit trail compliance (useful for enterprise and regulated verticals like insurance and finance), wants to offer powerful undo/redo and time-travel features, or plans to use AI to analyze editing patterns and their impact on conversions.

**Best for:** Enterprise-grade deployments where audit compliance, temporal queries, and AI-driven editing pattern analysis are priorities, and the team is comfortable with eventual consistency and higher architectural complexity.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every change — who changed what, when, and why
- Pro: Time-travel queries are trivial ("show me this page as of 2026-03-15T14:30Z")
- Pro: Undo/redo is free — just replay events up to the desired point
- Pro: AI can mine event streams to find editing patterns that correlate with conversion lift
- Pro: Write scalability — appending events is fast and conflict-free
- Con: Higher complexity — requires event store, projectors/processors, and materialized views
- Con: Eventual consistency between event store and read models (typically milliseconds, but requires design consideration)
- Con: Read model must be rebuilt if projectors change — can be slow for large event stores
- Con: More storage — events accumulate indefinitely; snapshots needed for performance
- Con: Debugging requires understanding event replay, not just "look at the current row"

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WCAG 2.2 / ISO 40500 | `AccessibilityAuditCompleted` events capture audit results; read model materializes current compliance status |
| Schema.org | `StructuredDataUpdated` events track structured data changes; injected into page from read model |
| Core Web Vitals | `PerformanceMeasured` events record LCP/CLS/INP; read model aggregates trends over time |
| GDPR / CCPA | Event store provides complete data processing audit trail; `ConsentGranted`/`ConsentRevoked` events |
| IAB TCF v2.2 | TCF strings stored in consent events |
| ISO 27001 / 27701 | Immutable event log directly satisfies information security audit requirements |
| JSON Schema 2020-12 | Event payloads validated against registered schemas before append |
| Standard Webhooks | Events projected to webhook delivery queue for outbound notifications |
| OpenAPI 3.1 | Read model APIs documented with OpenAPI; event types documented with AsyncAPI 3.0 |
| AsyncAPI 3.0 | Event types and their payloads documented as AsyncAPI channels for consumers |

---

## Event Store

```sql
-- The append-only event store. This is the source of truth for the entire system.
-- Events are immutable once written. Current state is derived by replaying events.
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,       -- aggregate ID (page_id, workspace_id, experiment_id, etc.)
    stream_type VARCHAR(50) NOT NULL,  -- page, workspace, experiment, lead
    sequence_number BIGINT NOT NULL,   -- monotonically increasing within a stream
    event_type VARCHAR(100) NOT NULL,  -- PageCreated, SectionAdded, ElementUpdated, ExperimentStarted, etc.
    event_version INTEGER NOT NULL DEFAULT 1,  -- schema version of this event type

    -- The event payload. Structure varies by event_type.
    -- See Event Catalog below for examples.
    data JSONB NOT NULL,

    -- Metadata about the event (who, where, why)
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "userId": "usr_abc",
    --   "userName": "Jane Smith",
    --   "ipAddress": "203.0.113.42",
    --   "userAgent": "Mozilla/5.0...",
    --   "correlationId": "req_xyz",  -- ties together events from one user action
    --   "causationId": "evt_prev",   -- the event that caused this event
    --   "source": "editor",          -- editor, api, ai_generation, system
    --   "aiJobId": "job_123"         -- if generated by AI
    -- }

    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (stream_id, sequence_number)
);

-- Snapshots for performance — store materialized state every N events
-- to avoid replaying long event streams from the beginning.
CREATE TABLE snapshots (
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,  -- event sequence at which this snapshot was taken
    state JSONB NOT NULL,             -- full materialized state at this point
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);

-- Optimistic concurrency: the UNIQUE constraint on (stream_id, sequence_number)
-- ensures that concurrent writers to the same stream will conflict, preventing
-- lost updates without locking.

CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_stream_type ON events(stream_type, stream_id);
CREATE INDEX idx_events_metadata_user ON events USING GIN ((metadata->'userId'));
CREATE INDEX idx_snapshots_stream ON snapshots(stream_id);
```

## Event Catalog

Below are the key event types. Each event's `data` field follows a specific structure.

```
┌─────────────────────────────────────────────────────────────────┐
│ Page Lifecycle Events                                           │
├─────────────────────────────────────────────────────────────────┤
│ PageCreated        │ Initial page creation                      │
│ PagePublished      │ Page goes live                             │
│ PageUnpublished    │ Page taken offline                         │
│ PageArchived       │ Page archived                              │
│ PageDeleted        │ Soft delete                                │
├─────────────────────────────────────────────────────────────────┤
│ Content Events                                                  │
├─────────────────────────────────────────────────────────────────┤
│ SectionAdded       │ New section added to page                  │
│ SectionRemoved     │ Section removed from page                  │
│ SectionReordered   │ Section moved to new position              │
│ SectionUpdated     │ Section properties changed                 │
│ ElementAdded       │ New element added to a section             │
│ ElementRemoved     │ Element removed                            │
│ ElementUpdated     │ Element content/properties changed         │
│ ElementStyleChanged│ Element styles modified                    │
│ ElementReordered   │ Element moved within/between sections      │
│ FormConfigured     │ Form fields/settings updated               │
│ SeoUpdated         │ SEO metadata changed                       │
│ SettingsUpdated    │ Page settings changed                      │
│ StructuredDataSet  │ Schema.org data added/updated              │
├─────────────────────────────────────────────────────────────────┤
│ Personalization Events                                          │
├─────────────────────────────────────────────────────────────────┤
│ PersonalizationRuleAdded    │ New rule created                  │
│ PersonalizationRuleUpdated  │ Rule conditions/replacements changed│
│ PersonalizationRuleRemoved  │ Rule deleted                      │
│ DtrRuleAdded               │ Dynamic text replacement added     │
│ DtrRuleRemoved             │ DTR rule removed                   │
├─────────────────────────────────────────────────────────────────┤
│ Experiment Events                                               │
├─────────────────────────────────────────────────────────────────┤
│ ExperimentCreated  │ New A/B or multivariate test               │
│ ExperimentStarted  │ Test begins receiving traffic              │
│ ExperimentPaused   │ Test paused                                │
│ ExperimentCompleted│ Test concluded; winner determined           │
│ VariantAdded       │ New variant added to experiment            │
│ VariantUpdated     │ Variant content changed                    │
│ TrafficAllocated   │ Traffic weights changed                    │
│ ResultsRecorded    │ Daily results snapshot                     │
├─────────────────────────────────────────────────────────────────┤
│ AI Events                                                       │
├─────────────────────────────────────────────────────────────────┤
│ AiGenerationRequested  │ User initiated AI generation           │
│ AiGenerationCompleted  │ AI returned results                    │
│ AiGenerationFailed     │ AI generation errored                  │
│ AiSuggestionAccepted   │ User accepted AI-generated content     │
│ AiSuggestionRejected   │ User rejected AI suggestion            │
│ PredictiveScoreGenerated│ Conversion prediction calculated      │
├─────────────────────────────────────────────────────────────────┤
│ Lead & Analytics Events                                         │
├─────────────────────────────────────────────────────────────────┤
│ LeadCaptured       │ Form submission received                   │
│ ConsentGranted     │ User gave consent                          │
│ ConsentRevoked     │ User revoked consent                       │
│ PageViewed         │ Visitor viewed page                        │
│ ConversionOccurred │ Conversion event recorded                  │
│ PerformanceMeasured│ Core Web Vitals measured                   │
│ AccessibilityAuditCompleted │ WCAG audit results recorded       │
└─────────────────────────────────────────────────────────────────┘
```

### Example Event Payloads

```json
-- PageCreated
{
  "event_type": "PageCreated",
  "data": {
    "workspaceId": "ws_abc",
    "name": "Summer Promo Landing Page",
    "slug": "summer-promo",
    "templateId": "tpl_xyz",
    "initialContent": {
      "sections": [...]
    }
  }
}

-- ElementUpdated
{
  "event_type": "ElementUpdated",
  "data": {
    "sectionId": "sec_abc",
    "elementId": "el_def",
    "changes": {
      "content": {"from": "Old Headline", "to": "New Headline"},
      "styles.desktop.fontSize": {"from": "36px", "to": "48px"}
    }
  }
}

-- AiSuggestionAccepted
{
  "event_type": "AiSuggestionAccepted",
  "data": {
    "jobId": "job_123",
    "elementId": "el_def",
    "field": "content",
    "originalValue": "Sign Up Now",
    "acceptedValue": "Start Your Free Trial Today",
    "copyAngle": "benefit_driven",
    "confidenceScore": 0.87
  }
}

-- LeadCaptured
{
  "event_type": "LeadCaptured",
  "data": {
    "formId": "el_form01",
    "variantId": "var_abc",
    "email": "jane@acme.com",
    "formData": {"email": "jane@acme.com", "company": "Acme Corp"},
    "visitorId": "v_hash123",
    "sessionId": "s_hash456",
    "utmSource": "google",
    "utmCampaign": "summer-2026",
    "referrer": "https://google.com/search?q=...",
    "ipAddress": "203.0.113.42"
  }
}
```

---

## Read Models (Materialized Views)

These tables are projections maintained by event processors. They are NOT the source of truth — they can be rebuilt from the event store at any time.

### Page Read Model (for the editor and renderer)

```sql
-- Materialized current state of each page, updated by event processors.
CREATE TABLE rm_pages (
    id UUID PRIMARY KEY,  -- same as the stream_id for this page
    workspace_id UUID NOT NULL,
    template_id UUID,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    published_at TIMESTAMPTZ,
    content JSONB NOT NULL DEFAULT '{"sections": []}',
    seo JSONB NOT NULL DEFAULT '{}',
    settings JSONB NOT NULL DEFAULT '{}',
    personalization JSONB NOT NULL DEFAULT '{"rules": [], "dtr": []}',
    last_event_sequence BIGINT NOT NULL,  -- tracks which event this projection is up to
    created_by UUID NOT NULL,
    updated_by UUID,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_rm_pages_workspace ON rm_pages(workspace_id);
CREATE INDEX idx_rm_pages_status ON rm_pages(status);
CREATE INDEX idx_rm_pages_content_gin ON rm_pages USING GIN (content jsonb_path_ops);
```

### Experiment Read Model

```sql
CREATE TABLE rm_experiments (
    id UUID PRIMARY KEY,
    page_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    experiment_type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    traffic_percentage INTEGER NOT NULL DEFAULT 100,
    confidence_threshold NUMERIC(4,3) NOT NULL DEFAULT 0.950,
    primary_metric VARCHAR(100) NOT NULL DEFAULT 'conversion_rate',
    winner_variant_id UUID,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_variants (
    id UUID PRIMARY KEY,
    experiment_id UUID NOT NULL REFERENCES rm_experiments(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    is_control BOOLEAN NOT NULL DEFAULT false,
    traffic_weight INTEGER NOT NULL DEFAULT 50,
    content JSONB,
    visitors INTEGER NOT NULL DEFAULT 0,
    conversions INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(7,6),
    confidence NUMERIC(4,3),
    is_significant BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_experiments_page ON rm_experiments(page_id);
CREATE INDEX idx_rm_experiments_status ON rm_experiments(status);
CREATE INDEX idx_rm_variants_experiment ON rm_variants(experiment_id);
```

### Lead Read Model

```sql
CREATE TABLE rm_leads (
    id UUID PRIMARY KEY,
    workspace_id UUID NOT NULL,
    page_id UUID NOT NULL,
    variant_id UUID,
    email VARCHAR(320),
    form_data JSONB NOT NULL DEFAULT '{}',
    utm_source VARCHAR(255),
    utm_medium VARCHAR(255),
    utm_campaign VARCHAR(255),
    consents JSONB NOT NULL DEFAULT '[]',  -- array of consent records
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_leads_workspace ON rm_leads(workspace_id);
CREATE INDEX idx_rm_leads_page ON rm_leads(page_id);
CREATE INDEX idx_rm_leads_email ON rm_leads(email);
CREATE INDEX idx_rm_leads_created ON rm_leads(created_at);
```

### Analytics Read Model

```sql
-- Daily aggregated analytics per page/variant — projected from PageViewed and ConversionOccurred events.
CREATE TABLE rm_daily_analytics (
    page_id UUID NOT NULL,
    variant_id UUID,
    date DATE NOT NULL,
    visitors INTEGER NOT NULL DEFAULT 0,
    unique_visitors INTEGER NOT NULL DEFAULT 0,
    conversions INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(7,6),
    avg_time_on_page_ms INTEGER,
    bounce_rate NUMERIC(5,4),
    device_breakdown JSONB NOT NULL DEFAULT '{}',  -- {"desktop": 450, "mobile": 320, "tablet": 30}
    country_breakdown JSONB NOT NULL DEFAULT '{}', -- {"US": 500, "GB": 150, ...}
    utm_breakdown JSONB NOT NULL DEFAULT '{}',     -- {"google": 300, "facebook": 200, ...}
    last_event_sequence BIGINT NOT NULL,
    PRIMARY KEY (page_id, COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'), date)
);

CREATE INDEX idx_rm_daily_analytics_date ON rm_daily_analytics(date);

-- Performance trend read model
CREATE TABLE rm_performance_trends (
    page_id UUID NOT NULL,
    variant_id UUID,
    date DATE NOT NULL,
    avg_lcp_ms INTEGER,
    avg_cls_score NUMERIC(5,4),
    avg_inp_ms INTEGER,
    passes_cwv_percentage NUMERIC(5,2),
    measurement_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (page_id, COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'), date)
);

-- Accessibility trend read model
CREATE TABLE rm_accessibility_trends (
    page_id UUID NOT NULL,
    variant_id UUID,
    date DATE NOT NULL,
    avg_score NUMERIC(5,2),
    total_violations INTEGER,
    critical_violations INTEGER,
    auto_fixed_count INTEGER,
    PRIMARY KEY (page_id, COALESCE(variant_id, '00000000-0000-0000-0000-000000000000'), date)
);
```

### AI Insights Read Model

```sql
-- Projected from AI events to surface patterns in AI usage and acceptance rates.
CREATE TABLE rm_ai_insights (
    workspace_id UUID NOT NULL,
    period DATE NOT NULL,  -- first day of the period (month)
    total_generations INTEGER NOT NULL DEFAULT 0,
    accepted_count INTEGER NOT NULL DEFAULT 0,
    rejected_count INTEGER NOT NULL DEFAULT 0,
    acceptance_rate NUMERIC(5,4),
    total_input_tokens BIGINT NOT NULL DEFAULT 0,
    total_output_tokens BIGINT NOT NULL DEFAULT 0,
    top_copy_angles JSONB NOT NULL DEFAULT '{}',  -- {"benefit_driven": 42, "fomo": 28, ...}
    avg_confidence_score NUMERIC(4,3),
    PRIMARY KEY (workspace_id, period)
);
```

---

## Supporting Tables (Not Event-Sourced)

Some entities are reference data, not event-sourced aggregates.

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    password_hash TEXT,
    email_verified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'editor',
    PRIMARY KEY (workspace_id, user_id)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(64) NOT NULL UNIQUE,
    key_prefix VARCHAR(8) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category VARCHAR(100),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    thumbnail_url TEXT,
    industry VARCHAR(100),
    conversion_goal VARCHAR(100),
    is_published BOOLEAN NOT NULL DEFAULT false,
    content JSONB NOT NULL DEFAULT '{"sections": []}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    width INTEGER,
    height INTEGER,
    storage_url TEXT NOT NULL,
    cdn_url TEXT,
    alt_text TEXT,
    ai_generated BOOLEAN NOT NULL DEFAULT false,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret_hash VARCHAR(64) NOT NULL,
    events TEXT[] NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    response_status INTEGER,
    attempts INTEGER NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'connected',
    credentials_encrypted JSONB,
    config JSONB NOT NULL DEFAULT '{}',
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workspace_members_user ON workspace_members(user_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_templates_industry ON templates(industry);
CREATE INDEX idx_assets_workspace ON assets(workspace_id);
CREATE INDEX idx_webhook_endpoints_workspace ON webhook_endpoints(workspace_id);
CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries(endpoint_id);
CREATE INDEX idx_integrations_workspace ON integrations(workspace_id);
```

---

## Example Queries

### Replay page state at a specific point in time

```sql
-- Get the most recent snapshot before the target time, then replay events after it.
WITH latest_snapshot AS (
    SELECT state, sequence_number
    FROM snapshots
    WHERE stream_id = $1
      AND created_at <= $2  -- target timestamp
    ORDER BY sequence_number DESC
    LIMIT 1
),
subsequent_events AS (
    SELECT data, event_type, sequence_number
    FROM events
    WHERE stream_id = $1
      AND sequence_number > COALESCE((SELECT sequence_number FROM latest_snapshot), 0)
      AND occurred_at <= $2  -- target timestamp
    ORDER BY sequence_number ASC
)
SELECT * FROM latest_snapshot
UNION ALL
SELECT data AS state, sequence_number FROM subsequent_events;
-- Application code replays events on top of the snapshot to derive the page state at that time.
```

### Find all changes to a specific element (full audit trail)

```sql
SELECT e.event_type, e.data, e.metadata, e.occurred_at
FROM events e
WHERE e.stream_id = $1  -- page_id
  AND e.data->>'elementId' = $2  -- target element ID
ORDER BY e.sequence_number ASC;
```

### Discover which AI suggestions were accepted vs. rejected

```sql
SELECT
    e.data->>'copyAngle' AS copy_angle,
    COUNT(*) FILTER (WHERE e.event_type = 'AiSuggestionAccepted') AS accepted,
    COUNT(*) FILTER (WHERE e.event_type = 'AiSuggestionRejected') AS rejected,
    ROUND(
        COUNT(*) FILTER (WHERE e.event_type = 'AiSuggestionAccepted')::NUMERIC /
        NULLIF(COUNT(*), 0), 3
    ) AS acceptance_rate
FROM events e
WHERE e.stream_type = 'page'
  AND e.event_type IN ('AiSuggestionAccepted', 'AiSuggestionRejected')
  AND e.metadata->>'userId' IS NOT NULL
GROUP BY e.data->>'copyAngle'
ORDER BY accepted DESC;
```

### Correlate editing patterns with conversion outcomes

```sql
-- Find pages where sections were reordered and compare their conversion rates
-- to pages that were not reordered.
WITH reordered_pages AS (
    SELECT DISTINCT stream_id AS page_id
    FROM events
    WHERE event_type = 'SectionReordered'
      AND occurred_at >= now() - INTERVAL '90 days'
),
page_conversions AS (
    SELECT page_id, AVG(conversion_rate) AS avg_cr
    FROM rm_daily_analytics
    WHERE date >= now() - INTERVAL '90 days'
    GROUP BY page_id
)
SELECT
    CASE WHEN rp.page_id IS NOT NULL THEN 'Reordered' ELSE 'Not Reordered' END AS cohort,
    COUNT(DISTINCT pc.page_id) AS pages,
    ROUND(AVG(pc.avg_cr), 6) AS avg_conversion_rate
FROM page_conversions pc
LEFT JOIN reordered_pages rp ON pc.page_id = rp.page_id
GROUP BY (rp.page_id IS NOT NULL);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Read Models — Pages | 1 | rm_pages (JSONB content) |
| Read Models — Experiments | 2 | rm_experiments, rm_variants |
| Read Models — Leads | 1 | rm_leads |
| Read Models — Analytics | 3 | rm_daily_analytics, rm_performance_trends, rm_accessibility_trends |
| Read Models — AI | 1 | rm_ai_insights |
| Supporting — Identity | 4 | workspaces, users, workspace_members, api_keys |
| Supporting — Content | 2 | templates, assets |
| Supporting — Integrations | 3 | integrations, webhook_endpoints, webhook_deliveries |
| **Total** | **19** | Plus the event store which replaces many normalized tables |

---

## Key Design Decisions

1. **Single events table rather than separate event tables per aggregate.** A unified event store simplifies infrastructure, enables cross-aggregate queries (e.g., correlate editing patterns with conversion outcomes), and allows a single event processor framework.

2. **Optimistic concurrency via UNIQUE(stream_id, sequence_number).** Two concurrent editors appending to the same page stream will experience a conflict on the sequence number, which the application resolves by retrying with the next sequence number. This avoids pessimistic locking.

3. **Snapshots for performance.** For pages with hundreds of edits, replaying the full event stream on every read would be slow. Snapshots capture the materialized state every N events (e.g., every 50), so replay only needs to process events since the last snapshot.

4. **Read models use JSONB for page content** (same as Suggestion 2), combining the benefits of event sourcing for writes with the JSONB hybrid approach for reads. The read model is essentially the Hybrid JSONB model from Suggestion 2, but derived from events rather than being the source of truth.

5. **AI events are first-class citizens.** Every AI interaction — generation request, result, acceptance, rejection — is an event. This enables rich analytics on AI effectiveness: which copy angles get accepted most often, which AI-generated elements correlate with conversion improvements, and how AI usage patterns vary across verticals.

6. **Analytics events (PageViewed, ConversionOccurred) flow through the same event store** but are projected into aggregated daily read models for performance. Raw analytics events can be archived to cold storage after projection.

7. **Webhook deliveries are triggered by event processors.** When an event matching a webhook subscription occurs, the processor creates a webhook delivery record. This ensures webhooks are reliably delivered even if the webhook endpoint is temporarily down.

8. **Supporting tables (users, workspaces, templates, assets) are NOT event-sourced.** These are low-change reference data where the complexity of event sourcing is not justified. They use standard CRUD with relational integrity.

9. **Event metadata captures rich context** including the user, IP address, correlation ID (linking events from one user action), causation ID (the event that triggered this event), and source (editor, API, AI generation, system). This enables sophisticated audit queries.

10. **The event catalog is extensible** — new event types can be added without schema migrations. The `events` table structure never changes; only the event type vocabulary and payload schemas grow. This makes the system highly adaptable as new features are added.
