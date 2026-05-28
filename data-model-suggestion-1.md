# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Landing Page Builder with AI · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational design where every concept — pages, sections, elements, variants, experiments, leads, analytics events — has its own dedicated table with well-defined foreign key relationships. The page structure is decomposed into a strict hierarchy: Page -> Section -> Element, with each level independently versioned and referenceable.

Normalized relational schemas are the backbone of platforms like Webflow (collections/items/fields hierarchy), HubSpot CMS (pages/modules/rows/columns), and traditional CMS systems like WordPress (posts/postmeta/terms). This approach maximizes referential integrity and makes complex cross-entity queries (e.g., "find all pages using a specific template section that have conversion rates above 5%") straightforward with standard JOINs.

This approach is best when the team values data integrity, needs complex reporting across entities, and wants the schema to serve as living documentation of the domain. It requires more upfront modeling effort and more migrations as the domain evolves, but eliminates ambiguity about where data lives.

**Best for:** Teams building a production-grade platform with complex reporting needs, regulatory compliance requirements, and a well-understood domain model.

**Trade-offs:**
- Pro: Maximum referential integrity; no orphaned data
- Pro: Complex cross-entity queries are natural SQL JOINs
- Pro: Schema serves as documentation; new developers understand the domain by reading DDL
- Pro: Standard ORM tooling works naturally
- Con: High table count (~45+) increases migration complexity
- Con: Adding new element types or section variants requires schema changes
- Con: Deep JOIN chains for full page rendering (page -> sections -> elements -> styles)
- Con: Rigid structure makes rapid prototyping slower

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WCAG 2.2 / ISO 40500 | `accessibility_audits` table stores per-page audit results with WCAG rule references; `accessibility_issues` tracks individual violations |
| Schema.org | `page_structured_data` table stores structured data blocks (FAQ, HowTo, Product) with schema type references |
| Core Web Vitals | `performance_scores` table records LCP, CLS, INP per page/variant with threshold compliance flags |
| GDPR / CCPA | `consent_configurations` table per workspace; `lead_consents` tracks individual consent records with legal basis |
| IAB TCF v2.2 | `consent_configurations.tcf_string` stores the TC string; consent mode flags in `page_settings` |
| OAuth 2.1 / OIDC | `users` and `oauth_connections` tables for authentication; `api_keys` for programmatic access |
| OpenAPI 3.1 | API endpoints follow resource-per-table convention; each table maps to a REST resource |
| JSON Schema 2020-12 | `element_types.schema` column stores the JSON Schema validation definition for each element's properties |
| Open Graph Protocol | `page_seo.og_title`, `og_description`, `og_image` columns for social sharing metadata |
| Standard Webhooks | `webhook_endpoints` and `webhook_deliveries` tables following the Standard Webhooks payload signing spec |

---

## Workspace & User Management

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, business, enterprise
    custom_domain VARCHAR(255),
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    password_hash TEXT,  -- NULL if SSO-only
    email_verified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'editor',  -- owner, admin, editor, viewer
    invited_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at TIMESTAMPTZ,
    PRIMARY KEY (workspace_id, user_id)
);

CREATE TABLE oauth_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,  -- google, github, microsoft
    provider_user_id VARCHAR(255) NOT NULL,
    access_token_encrypted TEXT,
    refresh_token_encrypted TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider, provider_user_id)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 of the key
    key_prefix VARCHAR(8) NOT NULL,  -- first 8 chars for identification
    scopes TEXT[] NOT NULL DEFAULT '{}',  -- pages:read, pages:write, leads:read, etc.
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workspace_members_user ON workspace_members(user_id);
CREATE INDEX idx_api_keys_workspace ON api_keys(workspace_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

## Templates & Design System

```sql
CREATE TABLE template_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id UUID REFERENCES template_categories(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    thumbnail_url TEXT,
    preview_url TEXT,
    industry VARCHAR(100),  -- insurance, real_estate, ecommerce, saas, etc.
    conversion_goal VARCHAR(100),  -- lead_gen, signup, purchase, download
    is_published BOOLEAN NOT NULL DEFAULT false,
    usage_count INTEGER NOT NULL DEFAULT 0,
    avg_conversion_rate NUMERIC(5,4),  -- 0.0000 to 9.9999
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE template_sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES templates(id) ON DELETE CASCADE,
    section_type VARCHAR(100) NOT NULL,  -- hero, features, testimonials, cta, faq, pricing, footer
    name VARCHAR(255) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE template_elements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id UUID NOT NULL REFERENCES template_sections(id) ON DELETE CASCADE,
    element_type VARCHAR(100) NOT NULL,  -- heading, paragraph, button, image, form, video, icon
    tag_name VARCHAR(20),  -- h1, h2, p, button, img, div, etc.
    default_content TEXT,
    default_styles JSONB NOT NULL DEFAULT '{}',
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON templates(category_id);
CREATE INDEX idx_templates_industry ON templates(industry);
CREATE INDEX idx_template_sections_template ON template_sections(template_id);
CREATE INDEX idx_template_elements_section ON template_elements(section_id);
```

## Pages & Content

```sql
CREATE TABLE pages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    template_id UUID REFERENCES templates(id) ON DELETE SET NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',  -- draft, published, archived, scheduled
    published_at TIMESTAMPTZ,
    scheduled_publish_at TIMESTAMPTZ,
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
    snapshot_html TEXT,  -- rendered HTML snapshot for fast serving
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, version_number)
);

CREATE TABLE page_seo (
    page_id UUID PRIMARY KEY REFERENCES pages(id) ON DELETE CASCADE,
    title VARCHAR(70),
    meta_description VARCHAR(160),
    og_title VARCHAR(95),
    og_description VARCHAR(200),
    og_image_url TEXT,
    canonical_url TEXT,
    noindex BOOLEAN NOT NULL DEFAULT false,
    nofollow BOOLEAN NOT NULL DEFAULT false,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE page_settings (
    page_id UUID PRIMARY KEY REFERENCES pages(id) ON DELETE CASCADE,
    custom_domain VARCHAR(255),
    favicon_url TEXT,
    custom_css TEXT,
    custom_js TEXT,
    custom_head_html TEXT,
    consent_mode VARCHAR(50) NOT NULL DEFAULT 'banner',  -- banner, implicit, none
    google_consent_mode_v2 BOOLEAN NOT NULL DEFAULT false,
    tcf_enabled BOOLEAN NOT NULL DEFAULT false,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    section_type VARCHAR(100) NOT NULL,  -- hero, features, testimonials, cta, faq, pricing, footer, custom
    name VARCHAR(255),
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_visible BOOLEAN NOT NULL DEFAULT true,
    background_color VARCHAR(7),
    background_image_url TEXT,
    padding_top INTEGER DEFAULT 40,
    padding_bottom INTEGER DEFAULT 40,
    max_width INTEGER DEFAULT 1200,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE elements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id UUID NOT NULL REFERENCES sections(id) ON DELETE CASCADE,
    parent_element_id UUID REFERENCES elements(id) ON DELETE CASCADE,  -- for nested elements (e.g., columns)
    element_type VARCHAR(100) NOT NULL,  -- heading, paragraph, button, image, form, video, icon, container, column
    tag_name VARCHAR(20),
    content TEXT,
    href TEXT,
    src TEXT,  -- image/video source URL
    alt_text TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_visible BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE element_styles (
    element_id UUID PRIMARY KEY REFERENCES elements(id) ON DELETE CASCADE,
    desktop_styles JSONB NOT NULL DEFAULT '{}',
    tablet_styles JSONB NOT NULL DEFAULT '{}',
    mobile_styles JSONB NOT NULL DEFAULT '{}',
    hover_styles JSONB NOT NULL DEFAULT '{}',
    css_classes TEXT[],
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE page_structured_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    schema_type VARCHAR(100) NOT NULL,  -- FAQPage, HowTo, Product, Organization, BreadcrumbList
    data JSONB NOT NULL,  -- Schema.org JSON-LD content
    auto_generated BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_workspace ON pages(workspace_id);
CREATE INDEX idx_pages_status ON pages(status);
CREATE INDEX idx_page_versions_page ON page_versions(page_id);
CREATE INDEX idx_sections_page ON sections(page_id);
CREATE INDEX idx_sections_sort ON sections(page_id, sort_order);
CREATE INDEX idx_elements_section ON elements(section_id);
CREATE INDEX idx_elements_parent ON elements(parent_element_id);
CREATE INDEX idx_elements_sort ON elements(section_id, sort_order);
CREATE INDEX idx_page_structured_data_page ON page_structured_data(page_id);
```

## Dynamic Text Replacement & Personalization

```sql
CREATE TABLE personalization_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    priority INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE personalization_conditions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES personalization_rules(id) ON DELETE CASCADE,
    condition_type VARCHAR(100) NOT NULL,  -- utm_source, utm_medium, utm_campaign, referrer, device, location, keyword
    operator VARCHAR(20) NOT NULL,  -- equals, contains, starts_with, regex, in
    value TEXT NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE personalization_replacements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES personalization_rules(id) ON DELETE CASCADE,
    element_id UUID NOT NULL REFERENCES elements(id) ON DELETE CASCADE,
    field VARCHAR(50) NOT NULL,  -- content, src, href, alt_text
    replacement_value TEXT NOT NULL
);

CREATE TABLE dtr_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    parameter_name VARCHAR(100) NOT NULL,  -- URL parameter to match (e.g., keyword, utm_term)
    element_id UUID NOT NULL REFERENCES elements(id) ON DELETE CASCADE,
    default_text TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_personalization_rules_page ON personalization_rules(page_id);
CREATE INDEX idx_personalization_conditions_rule ON personalization_conditions(rule_id);
CREATE INDEX idx_dtr_rules_page ON dtr_rules(page_id);
```

## A/B Testing & Experiments

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    experiment_type VARCHAR(50) NOT NULL DEFAULT 'ab',  -- ab, multivariate, bandit
    status VARCHAR(50) NOT NULL DEFAULT 'draft',  -- draft, running, paused, completed, archived
    traffic_percentage INTEGER NOT NULL DEFAULT 100,  -- % of total page traffic entering experiment
    confidence_threshold NUMERIC(4,3) NOT NULL DEFAULT 0.950,  -- statistical significance threshold
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
    name VARCHAR(255) NOT NULL,  -- 'Control', 'Variant A', 'Variant B', etc.
    is_control BOOLEAN NOT NULL DEFAULT false,
    traffic_weight INTEGER NOT NULL DEFAULT 50,  -- relative weight for traffic allocation
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE variant_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES variants(id) ON DELETE CASCADE,
    element_id UUID NOT NULL REFERENCES elements(id) ON DELETE CASCADE,
    field VARCHAR(50) NOT NULL,  -- content, src, href, styles, is_visible
    override_value TEXT NOT NULL,
    UNIQUE (variant_id, element_id, field)
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
CREATE INDEX idx_variant_overrides_variant ON variant_overrides(variant_id);
CREATE INDEX idx_experiment_results_experiment ON experiment_results(experiment_id);
CREATE INDEX idx_experiment_results_date ON experiment_results(date);
```

## AI Generation

```sql
CREATE TABLE ai_generation_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    page_id UUID REFERENCES pages(id) ON DELETE SET NULL,
    job_type VARCHAR(100) NOT NULL,  -- prompt_to_page, copy_generation, image_generation, layout_suggestion
    status VARCHAR(50) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    prompt TEXT NOT NULL,
    model_provider VARCHAR(50) NOT NULL,  -- openai, anthropic, internal
    model_name VARCHAR(100) NOT NULL,  -- gpt-4o, claude-sonnet-4, etc.
    input_tokens INTEGER,
    output_tokens INTEGER,
    result_data JSONB,  -- generated content/structure
    error_message TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_copy_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES ai_generation_jobs(id) ON DELETE CASCADE,
    element_id UUID REFERENCES elements(id) ON DELETE SET NULL,
    copy_angle VARCHAR(100),  -- benefit_driven, fomo, authority, social_proof, curiosity
    generated_text TEXT NOT NULL,
    confidence_score NUMERIC(4,3),
    selected BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE predictive_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id) ON DELETE SET NULL,
    predicted_conversion_rate NUMERIC(7,6),
    confidence_interval_low NUMERIC(7,6),
    confidence_interval_high NUMERIC(7,6),
    risk_level VARCHAR(20) NOT NULL,  -- low, medium, high
    recommendations JSONB,  -- array of improvement suggestions
    model_version VARCHAR(50) NOT NULL,
    scored_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_jobs_workspace ON ai_generation_jobs(workspace_id);
CREATE INDEX idx_ai_jobs_page ON ai_generation_jobs(page_id);
CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs(status);
CREATE INDEX idx_ai_copy_variants_job ON ai_copy_variants(job_id);
CREATE INDEX idx_predictive_scores_page ON predictive_scores(page_id);
```

## Lead Capture & Forms

```sql
CREATE TABLE forms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    section_id UUID NOT NULL REFERENCES sections(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    submit_button_text VARCHAR(100) NOT NULL DEFAULT 'Submit',
    success_message TEXT,
    redirect_url TEXT,
    is_multi_step BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE form_fields (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id UUID NOT NULL REFERENCES forms(id) ON DELETE CASCADE,
    field_type VARCHAR(50) NOT NULL,  -- text, email, phone, select, checkbox, radio, textarea, hidden, date
    label VARCHAR(255) NOT NULL,
    placeholder VARCHAR(255),
    name VARCHAR(100) NOT NULL,  -- field name for submission data
    is_required BOOLEAN NOT NULL DEFAULT false,
    validation_pattern VARCHAR(255),  -- regex for validation
    options TEXT[],  -- for select/radio/checkbox
    step_number INTEGER NOT NULL DEFAULT 1,  -- for multi-step forms
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    form_id UUID NOT NULL REFERENCES forms(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    email VARCHAR(320),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    utm_source VARCHAR(255),
    utm_medium VARCHAR(255),
    utm_campaign VARCHAR(255),
    utm_term VARCHAR(255),
    utm_content VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lead_field_values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
    form_field_id UUID NOT NULL REFERENCES form_fields(id) ON DELETE CASCADE,
    value TEXT NOT NULL
);

CREATE TABLE lead_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
    consent_type VARCHAR(100) NOT NULL,  -- marketing, analytics, third_party, tcf
    legal_basis VARCHAR(50) NOT NULL,  -- consent, legitimate_interest, contract
    granted BOOLEAN NOT NULL,
    tcf_string TEXT,  -- IAB TCF v2.2 TC string if applicable
    ip_address INET,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forms_page ON forms(page_id);
CREATE INDEX idx_form_fields_form ON form_fields(form_id);
CREATE INDEX idx_leads_workspace ON leads(workspace_id);
CREATE INDEX idx_leads_page ON leads(page_id);
CREATE INDEX idx_leads_email ON leads(email);
CREATE INDEX idx_leads_created ON leads(created_at);
CREATE INDEX idx_lead_field_values_lead ON lead_field_values(lead_id);
CREATE INDEX idx_lead_consents_lead ON lead_consents(lead_id);
```

## Analytics & Performance

```sql
CREATE TABLE page_views (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    visitor_id VARCHAR(64) NOT NULL,  -- anonymous hashed visitor identifier
    session_id VARCHAR(64) NOT NULL,
    device_type VARCHAR(20),  -- desktop, tablet, mobile
    browser VARCHAR(50),
    os VARCHAR(50),
    country_code CHAR(2),  -- ISO 3166-1 alpha-2
    region VARCHAR(100),
    referrer TEXT,
    utm_source VARCHAR(255),
    utm_medium VARCHAR(255),
    utm_campaign VARCHAR(255),
    viewed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversion_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    visitor_id VARCHAR(64) NOT NULL,
    session_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(100) NOT NULL,  -- form_submit, button_click, scroll_depth, time_on_page
    event_value NUMERIC(12,4),  -- e.g., revenue amount, scroll percentage
    metadata JSONB,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE heatmap_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    visitor_id VARCHAR(64) NOT NULL,
    interaction_type VARCHAR(20) NOT NULL,  -- click, move, scroll
    x_position INTEGER NOT NULL,
    y_position INTEGER NOT NULL,
    viewport_width INTEGER NOT NULL,
    viewport_height INTEGER NOT NULL,
    element_selector TEXT,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE performance_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES variants(id),
    lcp_ms INTEGER,  -- Largest Contentful Paint in ms (target: <2500)
    cls_score NUMERIC(5,4),  -- Cumulative Layout Shift (target: <0.1)
    inp_ms INTEGER,  -- Interaction to Next Paint in ms (target: <200)
    ttfb_ms INTEGER,  -- Time to First Byte
    fcp_ms INTEGER,  -- First Contentful Paint
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
    wcag_level VARCHAR(10) NOT NULL DEFAULT 'AA',  -- A, AA, AAA
    total_violations INTEGER NOT NULL DEFAULT 0,
    total_passes INTEGER NOT NULL DEFAULT 0,
    score NUMERIC(5,2),  -- 0-100 accessibility score
    audited_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE accessibility_issues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id UUID NOT NULL REFERENCES accessibility_audits(id) ON DELETE CASCADE,
    rule_id VARCHAR(100) NOT NULL,  -- axe-core rule ID (e.g., color-contrast, image-alt)
    impact VARCHAR(20) NOT NULL,  -- critical, serious, moderate, minor
    wcag_criteria TEXT[] NOT NULL,  -- e.g., {'1.4.3', '1.4.6'}
    element_selector TEXT NOT NULL,
    description TEXT NOT NULL,
    auto_fixable BOOLEAN NOT NULL DEFAULT false,
    fixed_at TIMESTAMPTZ
);

-- Partitioned by month for high-volume tables
-- In production, page_views and heatmap_data should be range-partitioned on their timestamp column.

CREATE INDEX idx_page_views_page ON page_views(page_id);
CREATE INDEX idx_page_views_visitor ON page_views(visitor_id);
CREATE INDEX idx_page_views_viewed ON page_views(viewed_at);
CREATE INDEX idx_conversion_events_page ON conversion_events(page_id);
CREATE INDEX idx_conversion_events_type ON conversion_events(event_type);
CREATE INDEX idx_conversion_events_occurred ON conversion_events(occurred_at);
CREATE INDEX idx_heatmap_page ON heatmap_data(page_id);
CREATE INDEX idx_heatmap_recorded ON heatmap_data(recorded_at);
CREATE INDEX idx_performance_page ON performance_scores(page_id);
CREATE INDEX idx_accessibility_audits_page ON accessibility_audits(page_id);
CREATE INDEX idx_accessibility_issues_audit ON accessibility_issues(audit_id);
```

## Integrations & Webhooks

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider VARCHAR(100) NOT NULL,  -- salesforce, hubspot, mailchimp, klaviyo, zapier, google_ads
    status VARCHAR(50) NOT NULL DEFAULT 'connected',  -- connected, disconnected, error
    credentials_encrypted JSONB,
    config JSONB NOT NULL DEFAULT '{}',  -- provider-specific configuration
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    secret_hash VARCHAR(64) NOT NULL,  -- HMAC signing secret hash (Standard Webhooks spec)
    events TEXT[] NOT NULL,  -- lead.created, page.published, experiment.completed, etc.
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    response_status INTEGER,
    response_body TEXT,
    attempts INTEGER NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_workspace ON integrations(workspace_id);
CREATE INDEX idx_webhook_endpoints_workspace ON webhook_endpoints(workspace_id);
CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries(endpoint_id);
CREATE INDEX idx_webhook_deliveries_retry ON webhook_deliveries(next_retry_at) WHERE next_retry_at IS NOT NULL;
```

## Assets & Media

```sql
CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    file_size_bytes INTEGER NOT NULL,
    width INTEGER,  -- for images/videos
    height INTEGER,  -- for images/videos
    storage_url TEXT NOT NULL,
    cdn_url TEXT,
    alt_text TEXT,
    ai_generated BOOLEAN NOT NULL DEFAULT false,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_workspace ON assets(workspace_id);
CREATE INDEX idx_assets_mime ON assets(mime_type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Users | 5 | workspaces, users, workspace_members, oauth_connections, api_keys |
| Templates & Design | 4 | template_categories, templates, template_sections, template_elements |
| Pages & Content | 8 | pages, page_versions, page_seo, page_settings, sections, elements, element_styles, page_structured_data |
| Personalization | 4 | personalization_rules, personalization_conditions, personalization_replacements, dtr_rules |
| A/B Testing | 4 | experiments, variants, variant_overrides, experiment_results |
| AI Generation | 3 | ai_generation_jobs, ai_copy_variants, predictive_scores |
| Lead Capture | 5 | forms, form_fields, leads, lead_field_values, lead_consents |
| Analytics | 5 | page_views, conversion_events, heatmap_data, performance_scores, accessibility_audits + accessibility_issues |
| Integrations | 3 | integrations, webhook_endpoints, webhook_deliveries |
| Assets | 1 | assets |
| **Total** | **42** | |

---

## Key Design Decisions

1. **Strict page hierarchy (Page -> Section -> Element)** mirrors the visual builder's DOM structure. Each section is a standalone row; elements within sections support nesting via `parent_element_id` for containers and columns.

2. **Element styles separated into their own table** (`element_styles`) with responsive breakpoints as separate JSONB columns. This avoids bloating the elements table while keeping style data queryable.

3. **A/B test variants use override records** rather than duplicating entire page structures. `variant_overrides` stores only the fields that differ from the control, keeping storage efficient and making it easy to see what changed.

4. **Leads and lead field values normalized** into separate tables so that arbitrary form fields can be captured without schema changes. The `leads` table stores common fields (email, UTM parameters) for fast querying; custom fields go to `lead_field_values`.

5. **Analytics tables designed for partitioning** — `page_views`, `conversion_events`, and `heatmap_data` will generate high volumes and should be range-partitioned by their timestamp column in production.

6. **Consent tracking separated from leads** (`lead_consents`) to support GDPR's requirement for granular, auditable consent records with legal basis documentation.

7. **Performance scores use a generated column** (`passes_cwv`) that automatically computes Core Web Vitals compliance, enabling simple `WHERE passes_cwv = true` filtering.

8. **AI generation jobs are first-class entities** with full token tracking and provider metadata, enabling cost analysis and model comparison across providers (OpenAI vs. Anthropic).

9. **Webhook deliveries follow the Standard Webhooks spec** with HMAC signing, retry tracking, and event type filtering, ensuring reliable integration with downstream systems.

10. **Templates and pages share the same section/element model** — templates define default structure; pages reference templates but own their own sections and elements, allowing full customization after creation.
