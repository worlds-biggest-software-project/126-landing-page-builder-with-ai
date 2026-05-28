# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Landing Page Builder with AI · Created: 2026-05-19

## Philosophy

This model uses relational tables for identity, relationships, and queryable metadata, but stores variable-structure content — page sections, element trees, style definitions, personalization rules, and form schemas — as JSONB documents inside those relational rows. Instead of decomposing a page into dozens of joined tables (Page -> Section -> Element -> Style), a page's entire visual structure lives in a single `content JSONB` column on the page row.

This is the pattern used by Builder.io (pages stored as JSON blocks arrays with nested components), Sanity (structured content as JSON documents with portable text), and Payload CMS (collection schemas producing JSON documents backed by PostgreSQL JSONB). PostgreSQL 17's GIN indexes on JSONB columns provide efficient querying into document structures without sacrificing relational guarantees for the data that needs them (users, workspaces, experiments, leads).

The core insight is that a landing page's visual structure is inherently tree-shaped and polymorphic — a hero section contains different fields than a testimonial section, and element types vary widely. Forcing this into normalized tables creates a proliferation of joins and type-specific columns. JSONB lets the schema flex naturally while relational tables handle the entities that need referential integrity and cross-entity queries.

**Best for:** Teams that want rapid iteration on page structure, support for jurisdiction-specific or vertical-specific custom fields without migrations, and a developer experience closer to working with JSON APIs throughout the stack.

**Trade-offs:**
- Pro: Far fewer tables (~20 vs. 42); simpler migrations
- Pro: Page rendering requires a single row fetch, not a multi-JOIN query
- Pro: New element types or section types require zero schema changes
- Pro: JSONB content maps directly to the frontend component tree and API responses
- Pro: PostgreSQL JSONB operators and GIN indexes enable efficient querying into document content
- Con: Referential integrity within JSONB documents must be enforced in application code, not the database
- Con: Partial updates to deeply nested JSONB structures can be awkward (`jsonb_set` chains)
- Con: JSONB columns are harder to query ad-hoc compared to flat relational columns
- Con: Schema validation must be handled by application-layer JSON Schema, not DDL constraints

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema 2020-12 | Each element type has a registered JSON Schema in `element_type_schemas`; page content is validated against these schemas before save |
| WCAG 2.2 / ISO 40500 | Accessibility audit results stored in `accessibility_audits` table; issues array in JSONB |
| Schema.org | Structured data blocks stored as entries in the page content JSONB with `"type": "structured_data"` |
| Core Web Vitals | `performance_scores` relational table for queryable LCP/CLS/INP metrics |
| GDPR / CCPA | `consent_records` relational table with granular consent tracking; consent config in workspace JSONB settings |
| IAB TCF v2.2 | TCF string stored in `consent_records.tcf_string` |
| OAuth 2.1 / OIDC | Standard relational auth tables (users, sessions, oauth_connections) |
| Open Graph Protocol | OG metadata stored in `pages.seo` JSONB field |
| Standard Webhooks | `webhook_endpoints` and `webhook_deliveries` relational tables with Standard Webhooks signing |
| OpenAPI 3.1 | API responses return the JSONB content directly; page content structure documented as OpenAPI component schemas |

---

## Workspace & Authentication

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    -- JSONB for flexible workspace-level settings:
    -- {
    --   "custom_domain": "pages.acme.com",
    --   "default_consent_mode": "banner",
    --   "google_consent_mode_v2": true,
    --   "tcf_enabled": false,
    --   "branding": {"primary_color": "#2563eb", "logo_url": "..."},
    --   "ai_providers": {"copy": "anthropic", "image": "openai"},
    --   "data_residency": "eu"
    -- }
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
    invited_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at TIMESTAMPTZ,
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

CREATE INDEX idx_workspace_members_user ON workspace_members(user_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

## Pages — The Core Document Model

```sql
CREATE TABLE pages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    template_id UUID REFERENCES templates(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    published_at TIMESTAMPTZ,

    -- The entire page visual structure as a JSONB document.
    -- This is the source of truth for the page builder UI.
    -- Structure:
    -- {
    --   "sections": [
    --     {
    --       "id": "sec_abc123",
    --       "type": "hero",
    --       "name": "Hero Section",
    --       "visible": true,
    --       "styles": {
    --         "backgroundColor": "#ffffff",
    --         "paddingTop": 60,
    --         "paddingBottom": 60,
    --         "maxWidth": 1200
    --       },
    --       "elements": [
    --         {
    --           "id": "el_def456",
    --           "type": "heading",
    --           "tag": "h1",
    --           "content": "Transform Your Marketing",
    --           "styles": {
    --             "desktop": {"fontSize": "48px", "fontWeight": "700", "color": "#111827"},
    --             "tablet": {"fontSize": "36px"},
    --             "mobile": {"fontSize": "28px"}
    --           }
    --         },
    --         {
    --           "id": "el_ghi789",
    --           "type": "button",
    --           "content": "Get Started Free",
    --           "href": "#signup",
    --           "styles": {
    --             "desktop": {"backgroundColor": "#2563eb", "color": "#ffffff", "padding": "16px 32px"}
    --           }
    --         },
    --         {
    --           "id": "el_form01",
    --           "type": "form",
    --           "config": {
    --             "submitText": "Get My Free Guide",
    --             "successMessage": "Thanks! Check your email.",
    --             "redirectUrl": null,
    --             "fields": [
    --               {"name": "email", "type": "email", "label": "Email", "required": true, "placeholder": "you@company.com"},
    --               {"name": "company", "type": "text", "label": "Company", "required": false}
    --             ]
    --           }
    --         }
    --       ]
    --     },
    --     {
    --       "id": "sec_jkl012",
    --       "type": "structured_data",
    --       "schemaType": "FAQPage",
    --       "data": { ... Schema.org JSON-LD ... }
    --     }
    --   ]
    -- }
    content JSONB NOT NULL DEFAULT '{"sections": []}',

    -- SEO metadata as a flat JSONB object (queried less often than content).
    -- {
    --   "title": "Get Started | Acme Corp",
    --   "metaDescription": "Sign up for...",
    --   "ogTitle": "Get Started",
    --   "ogDescription": "...",
    --   "ogImage": "https://...",
    --   "canonicalUrl": null,
    --   "noindex": false,
    --   "nofollow": false
    -- }
    seo JSONB NOT NULL DEFAULT '{}',

    -- Page-level settings
    -- {
    --   "customDomain": null,
    --   "faviconUrl": null,
    --   "customCss": "",
    --   "customJs": "",
    --   "customHeadHtml": "",
    --   "consentMode": "banner"
    -- }
    settings JSONB NOT NULL DEFAULT '{}',

    -- Personalization rules stored alongside the page content.
    -- {
    --   "rules": [
    --     {
    --       "id": "pr_abc",
    --       "name": "Google Ads Traffic",
    --       "priority": 10,
    --       "active": true,
    --       "conditions": [
    --         {"type": "utm_source", "operator": "equals", "value": "google"}
    --       ],
    --       "replacements": [
    --         {"elementId": "el_def456", "field": "content", "value": "Welcome from Google!"}
    --       ]
    --     }
    --   ],
    --   "dtr": [
    --     {
    --       "parameter": "keyword",
    --       "elementId": "el_def456",
    --       "defaultText": "Transform Your Marketing"
    --     }
    --   ]
    -- }
    personalization JSONB NOT NULL DEFAULT '{"rules": [], "dtr": []}',

    created_by UUID NOT NULL REFERENCES users(id),
    updated_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE TABLE page_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    content JSONB NOT NULL,  -- snapshot of page content at this version
    seo JSONB,
    personalization JSONB,
    snapshot_html TEXT,  -- pre-rendered HTML for fast serving
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, version_number)
);

-- GIN index enables queries like: find all pages containing a section of type 'hero'
-- SELECT * FROM pages WHERE content @> '{"sections": [{"type": "hero"}]}';
CREATE INDEX idx_pages_workspace ON pages(workspace_id);
CREATE INDEX idx_pages_status ON pages(status);
CREATE INDEX idx_pages_content_gin ON pages USING GIN (content jsonb_path_ops);
CREATE INDEX idx_pages_seo_gin ON pages USING GIN (seo jsonb_path_ops);
CREATE INDEX idx_page_versions_page ON page_versions(page_id);
```

## Templates

```sql
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
    usage_count INTEGER NOT NULL DEFAULT 0,
    avg_conversion_rate NUMERIC(5,4),

    -- Same JSONB content structure as pages.content
    content JSONB NOT NULL DEFAULT '{"sections": []}',
    seo JSONB NOT NULL DEFAULT '{}',

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- JSON Schema definitions for element types — used for application-layer validation.
CREATE TABLE element_type_schemas (
    type_name VARCHAR(100) PRIMARY KEY,  -- heading, paragraph, button, image, form, video, etc.
    display_name VARCHAR(255) NOT NULL,
    icon VARCHAR(50),
    -- JSON Schema 2020-12 definition for this element type's properties
    schema JSONB NOT NULL,
    default_styles JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_industry ON templates(industry);
CREATE INDEX idx_templates_category ON templates(category);
CREATE INDEX idx_templates_content_gin ON templates USING GIN (content jsonb_path_ops);
```

## A/B Testing & Experiments

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    experiment_type VARCHAR(50) NOT NULL DEFAULT 'ab',
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    traffic_percentage INTEGER NOT NULL DEFAULT 100,
    confidence_threshold NUMERIC(4,3) NOT NULL DEFAULT 0.950,
    primary_metric VARCHAR(100) NOT NULL DEFAULT 'conversion_rate',
    winner_variant_id UUID,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    is_control BOOLEAN NOT NULL DEFAULT false,
    traffic_weight INTEGER NOT NULL DEFAULT 50,

    -- Variant stores a FULL copy of the modified content, NOT just overrides.
    -- This avoids complex merge logic at render time and makes each variant self-contained.
    -- The control variant's content is NULL (uses the page's content directly).
    content JSONB,  -- NULL for control; full page content for variants
    personalization JSONB,  -- variant-specific personalization overrides

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES variants(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    visitors INTEGER NOT NULL DEFAULT 0,
    conversions INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(7,6),
    confidence NUMERIC(4,3),
    is_significant BOOLEAN NOT NULL DEFAULT false,
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, variant_id, date)
);

CREATE INDEX idx_experiments_page ON experiments(page_id);
CREATE INDEX idx_experiments_status ON experiments(status);
CREATE INDEX idx_variants_experiment ON variants(experiment_id);
CREATE INDEX idx_variants_content_gin ON variants USING GIN (content jsonb_path_ops);
CREATE INDEX idx_experiment_results_experiment ON experiment_results(experiment_id);
```

## AI Generation

```sql
CREATE TABLE ai_generation_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    page_id UUID REFERENCES pages(id) ON DELETE SET NULL,
    job_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    prompt TEXT NOT NULL,
    model_provider VARCHAR(50) NOT NULL,
    model_name VARCHAR(100) NOT NULL,
    input_tokens INTEGER,
    output_tokens INTEGER,

    -- Generated content as JSONB — could be a full page content tree,
    -- a set of copy variants, or layout suggestions.
    -- {
    --   "type": "page_content",
    --   "content": { ... page content JSONB ... },
    --   "copyVariants": [
    --     {"elementId": "el_def456", "angle": "benefit_driven", "text": "...", "confidence": 0.87},
    --     {"elementId": "el_def456", "angle": "fomo", "text": "...", "confidence": 0.72}
    --   ],
    --   "predictions": {
    --     "estimatedConversionRate": 0.043,
    --     "confidenceInterval": [0.031, 0.056],
    --     "riskLevel": "low",
    --     "recommendations": ["Add social proof section", "Reduce form to 2 fields"]
    --   }
    -- }
    result JSONB,
    error_message TEXT,

    created_by UUID NOT NULL REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_jobs_workspace ON ai_generation_jobs(workspace_id);
CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs(status);
CREATE INDEX idx_ai_jobs_page ON ai_generation_jobs(page_id);
```

## Lead Capture

```sql
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    email VARCHAR(320),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    utm_source VARCHAR(255),
    utm_medium VARCHAR(255),
    utm_campaign VARCHAR(255),
    utm_term VARCHAR(255),

    -- All form field values stored as JSONB — no need for a separate field_values table.
    -- {
    --   "email": "jane@acme.com",
    --   "company": "Acme Corp",
    --   "phone": "+1-555-0123",
    --   "message": "Interested in enterprise plan"
    -- }
    form_data JSONB NOT NULL DEFAULT '{}',

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE consent_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
    consent_type VARCHAR(100) NOT NULL,
    legal_basis VARCHAR(50) NOT NULL,
    granted BOOLEAN NOT NULL,
    tcf_string TEXT,
    ip_address INET,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leads_workspace ON leads(workspace_id);
CREATE INDEX idx_leads_page ON leads(page_id);
CREATE INDEX idx_leads_email ON leads(email);
CREATE INDEX idx_leads_created ON leads(created_at);
CREATE INDEX idx_leads_form_data_gin ON leads USING GIN (form_data jsonb_path_ops);
CREATE INDEX idx_consent_records_lead ON consent_records(lead_id);
```

## Analytics & Performance

```sql
CREATE TABLE page_views (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    visitor_id VARCHAR(64) NOT NULL,
    session_id VARCHAR(64) NOT NULL,
    device_type VARCHAR(20),
    country_code CHAR(2),  -- ISO 3166-1 alpha-2
    referrer TEXT,
    utm_source VARCHAR(255),
    viewed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversion_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    visitor_id VARCHAR(64) NOT NULL,
    session_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_value NUMERIC(12,4),

    -- Flexible event metadata as JSONB
    -- {
    --   "formId": "el_form01",
    --   "buttonText": "Get Started",
    --   "scrollPercentage": 75,
    --   "timeOnPageMs": 45000
    -- }
    metadata JSONB,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE performance_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    lcp_ms INTEGER,
    cls_score NUMERIC(5,4),
    inp_ms INTEGER,
    ttfb_ms INTEGER,
    fcp_ms INTEGER,
    page_size_bytes INTEGER,
    passes_cwv BOOLEAN GENERATED ALWAYS AS (
        lcp_ms <= 2500 AND cls_score <= 0.1 AND inp_ms <= 200
    ) STORED,
    measured_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE accessibility_audits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    tool VARCHAR(50) NOT NULL DEFAULT 'axe-core',
    wcag_level VARCHAR(10) NOT NULL DEFAULT 'AA',
    total_violations INTEGER NOT NULL DEFAULT 0,
    total_passes INTEGER NOT NULL DEFAULT 0,
    score NUMERIC(5,2),

    -- Issues stored as JSONB array — no separate issues table needed.
    -- [
    --   {
    --     "ruleId": "color-contrast",
    --     "impact": "serious",
    --     "wcagCriteria": ["1.4.3"],
    --     "elementSelector": ".hero h1",
    --     "description": "Element has insufficient color contrast",
    --     "autoFixable": true,
    --     "fixedAt": null
    --   }
    -- ]
    issues JSONB NOT NULL DEFAULT '[]',

    audited_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_page_views_page ON page_views(page_id);
CREATE INDEX idx_page_views_viewed ON page_views(viewed_at);
CREATE INDEX idx_conversion_events_page ON conversion_events(page_id);
CREATE INDEX idx_conversion_events_occurred ON conversion_events(occurred_at);
CREATE INDEX idx_performance_page ON performance_scores(page_id);
CREATE INDEX idx_accessibility_page ON accessibility_audits(page_id);
```

## Integrations & Webhooks

```sql
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

CREATE INDEX idx_integrations_workspace ON integrations(workspace_id);
CREATE INDEX idx_webhook_endpoints_workspace ON webhook_endpoints(workspace_id);
CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries(endpoint_id);
CREATE INDEX idx_assets_workspace ON assets(workspace_id);
```

---

## Example Queries

### Fetch a complete page for rendering (single query)

```sql
SELECT p.id, p.name, p.slug, p.content, p.seo, p.settings, p.personalization
FROM pages p
WHERE p.workspace_id = $1 AND p.slug = $2 AND p.status = 'published';
```

Compare this to the normalized model, which requires JOINing pages -> sections -> elements -> element_styles (4+ tables).

### Find all pages containing a FAQ section

```sql
SELECT id, name FROM pages
WHERE content @> '{"sections": [{"type": "faq"}]}'
  AND workspace_id = $1;
```

### Find all pages where a specific element has a particular heading

```sql
SELECT id, name FROM pages
WHERE content @? '$.sections[*].elements[*] ? (@.type == "heading" && @.content like_regex "(?i)free trial")'
  AND workspace_id = $1;
```

### Query lead form data without a separate field_values table

```sql
SELECT id, email, form_data->>'company' AS company, created_at
FROM leads
WHERE workspace_id = $1
  AND form_data @> '{"company": "Acme Corp"}'
ORDER BY created_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Auth | 4 | workspaces, users, workspace_members, api_keys |
| Pages & Content | 2 | pages (with JSONB content/seo/personalization), page_versions |
| Templates | 2 | templates, element_type_schemas |
| A/B Testing | 3 | experiments, variants (with JSONB content), experiment_results |
| AI Generation | 1 | ai_generation_jobs (result as JSONB) |
| Lead Capture | 2 | leads (form_data as JSONB), consent_records |
| Analytics | 4 | page_views, conversion_events, performance_scores, accessibility_audits (issues as JSONB) |
| Integrations | 4 | integrations, webhook_endpoints, webhook_deliveries, assets |
| **Total** | **22** | ~50% fewer tables than the normalized model |

---

## Key Design Decisions

1. **Page content is a single JSONB column** rather than decomposed into sections/elements/styles tables. This mirrors how Builder.io and Sanity store page structures — as JSON document trees — and eliminates the 4-table JOIN chain needed to render a page.

2. **Variants store full content copies, not diffs.** While this uses more storage, it makes rendering trivial (just swap the content blob) and avoids complex merge logic. Content deduplication can be handled by PostgreSQL TOAST compression.

3. **Form schemas live inside the page content JSONB** as form-type elements with a `config.fields` array. This means form structure changes are versioned with the page, and the rendering pipeline sees forms as just another element type.

4. **Lead form values stored as a flat JSONB object** (`form_data`) instead of a separate field_values junction table. This simplifies writes (one INSERT) and reads (no JOINs), and GIN indexes enable efficient querying by any form field.

5. **Element type schemas registered in a dedicated table** provide application-layer validation. When a page is saved, each element in the content JSONB is validated against its registered JSON Schema 2020-12 definition.

6. **Accessibility issues stored as a JSONB array** inside the audit row rather than a separate table. Issues are always read and written as a batch with their parent audit, so the join is unnecessary.

7. **AI generation results stored as a single JSONB column** that can contain page content trees, copy variant arrays, or prediction objects depending on the job type. The polymorphic nature of AI outputs maps naturally to JSONB.

8. **Analytics tables remain fully relational** because they are queried by time range, aggregated by dimension, and partitioned by date. JSONB provides no advantage for time-series analytics data.

9. **Personalization rules stored alongside page content** in a dedicated JSONB column. This ensures personalization is versioned with the page and can reference element IDs from the content tree without cross-table foreign keys.

10. **GIN indexes on all JSONB columns** with `jsonb_path_ops` enable efficient containment queries (`@>`) and path queries (`@?`), making JSONB searchable for common patterns like "find all pages with a hero section" or "find all leads from a specific company."
