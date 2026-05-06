# Standards & API Reference

> Project: Landing Page Builder with AI · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 40500:2012 (WCAG 2.0)** — https://www.iso.org/standard/58625.html — Formal ISO adoption of W3C WCAG 2.0; the baseline accessibility standard cited in many regulatory frameworks. Landing pages must comply to meet legal accessibility requirements in the EU, US (ADA), and other jurisdictions.
- **ISO/IEC 27001:2022** — https://www.iso.org/standard/27001 — Information security management. Relevant for landing page builders handling lead data, especially when serving regulated verticals (insurance, finance, healthcare).
- **ISO/IEC 27701:2019** — https://www.iso.org/standard/71670.html — Privacy information management extension to ISO 27001; relevant for handling personal data captured through lead forms under GDPR/CCPA.
- **ISO/IEC 25010:2023** — https://www.iso.org/standard/78176.html — Software quality model covering performance, usability, and reliability characteristics relevant to evaluating landing page builder quality.
- **ISO 9241-210:2019** — https://www.iso.org/standard/77520.html — Human-centred design for interactive systems; informs UX design patterns and usability evaluation for the page builder UI itself.

### W3C & IETF Standards

- **WCAG 2.2** — https://www.w3.org/TR/WCAG22/ — Web Content Accessibility Guidelines (current version, October 2023). Landing pages must meet Level AA for legal compliance in most jurisdictions.
- **HTML Living Standard (WHATWG)** — https://html.spec.whatwg.org/ — The canonical HTML specification governing all generated landing page markup.
- **CSS Snapshot 2023** — https://www.w3.org/TR/CSS/ — Current state of CSS specifications relevant to page rendering, layout, and responsive design.
- **ARIA 1.2** — https://www.w3.org/TR/wai-aria-1.2/ — Accessible Rich Internet Applications spec for adding semantic meaning to interactive page elements.
- **RFC 9110 (HTTP Semantics)** — https://www.rfc-editor.org/rfc/rfc9110 — Updated HTTP semantics; foundation for page delivery and API interactions.
- **RFC 6265bis (Cookies)** — https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/ — Updated cookie spec covering SameSite, Partitioned cookies, and modern privacy controls relevant for analytics and consent.
- **RFC 7519 (JSON Web Tokens)** — https://www.rfc-editor.org/rfc/rfc7519 — JWT for authenticating API access to the builder's data plane.
- **RFC 8259 (JSON)** — https://www.rfc-editor.org/rfc/rfc8259 — Canonical JSON spec used in all REST API payloads.
- **W3C Web Vitals (Core Web Vitals metrics)** — https://web.dev/vitals/ — LCP, CLS, INP performance signals; landing pages must pass these to maintain Google ad Quality Scores and organic rankings.
- **Schema.org** — https://schema.org/ — Structured data vocabulary; landing pages should auto-inject FAQ, HowTo, Product, and Organization schema for SEO and rich SERP results.
- **AMP HTML Specification** — https://amp.dev/documentation/guides-and-tutorials/learn/spec/amphtml/ — Accelerated Mobile Pages spec, relevant if targeting near-instant mobile load times. Note: Google has de-prioritized AMP, so weigh long-term relevance.
- **Open Graph Protocol** — https://ogp.me/ — Metadata for rich social sharing previews, essential for landing pages distributed via social channels.

### Data Model & API Specifications

- **OpenAPI Specification 3.1.0** — https://spec.openapis.org/oas/v3.1.0 — Recommended for documenting the builder's REST API; aligns with JSON Schema 2020-12.
- **JSON Schema 2020-12** — https://json-schema.org/specification — For validating page definitions, form schemas, and API payloads.
- **GraphQL October 2021 Spec** — https://spec.graphql.org/October2021/ — Optional API style for flexible querying of page, variant, and analytics data.
- **AsyncAPI 3.0** — https://www.asyncapi.com/docs/reference/specification/v3.0.0 — For documenting webhook and event-driven integrations (e.g., form submissions, conversion events).
- **JSON:API 1.1** — https://jsonapi.org/format/ — Convention for REST API responses if a standardized resource representation is desired.
- **Webhooks (Standard Webhooks spec)** — https://www.standardwebhooks.com/ — Emerging cross-vendor webhook spec covering signing, retries, and payload conventions for outbound events.
- **Model Context Protocol (MCP)** — https://modelcontextprotocol.io/ — Anthropic-led open protocol for connecting LLMs to tools and data. A landing page builder could expose an MCP server enabling AI agents (Claude, others) to read page state, request edits, run tests, and pull analytics. Highly relevant for AI-native architecture.

### Security & Authentication Standards

- **OAuth 2.1 (draft)** — https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ — Consolidated OAuth spec; recommended over OAuth 2.0 for new implementations of third-party integrations and API access.
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Identity layer on OAuth 2.0; used for user sign-in and SSO.
- **OWASP ASVS 4.0** — https://owasp.org/www-project-application-security-verification-standard/ — Application Security Verification Standard; the practical security baseline for the builder's web app and API.
- **OWASP Top 10 (2021)** — https://owasp.org/www-project-top-ten/ — Critical web application risks; landing pages handle untrusted user input via forms and need protection against injection, XSS, and CSRF.
- **NIST SP 800-63B** — https://pages.nist.gov/800-63-3/sp800-63b.html — Digital identity guidelines covering authenticator strength and session management.
- **NIST AI Risk Management Framework (AI RMF 1.0)** — https://www.nist.gov/itl/ai-risk-management-framework — Relevant for governing the AI copy/page generation pipeline (bias, safety, transparency).
- **Content Security Policy Level 3** — https://www.w3.org/TR/CSP3/ — Required for protecting embedded landing pages from XSS and clickjacking.
- **Subresource Integrity (SRI)** — https://www.w3.org/TR/SRI/ — Verification of third-party scripts injected into landing pages.
- **GDPR (EU 2016/679)** — https://eur-lex.europa.eu/eli/reg/2016/679/oj — Governs collection and processing of EU resident data through lead forms; requires explicit consent and data residency controls.
- **CCPA / CPRA** — https://oag.ca.gov/privacy/ccpa — California privacy regulations governing personal information collected via landing pages.
- **IAB TCF v2.2** — https://iabeurope.eu/transparency-consent-framework/ — Transparency and Consent Framework for cookie consent management on lead-capture pages.
- **Google Consent Mode v2** — https://developers.google.com/tag-platform/security/guides/consent — Required signaling for GDPR-compliant analytics and ads measurement.

### MCP Server Specifications

- **MCP Specification** — https://spec.modelcontextprotocol.io/ — Core protocol describing the JSON-RPC message format, capabilities negotiation, and transport.
- **MCP Server SDK (TypeScript)** — https://github.com/modelcontextprotocol/typescript-sdk — Reference SDK for building an MCP server exposing the page builder's tools and resources to AI clients.
- **MCP Server SDK (Python)** — https://github.com/modelcontextprotocol/python-sdk — Python equivalent.
- An MCP server for this project would expose tools such as `create_page`, `update_section`, `run_ab_test`, `get_analytics`, and resources such as `page://{id}`, `template://{slug}`, `variant://{id}` enabling third-party AI agents to operate the builder.

## Similar Products — Developer Documentation & APIs

### Unbounce
- **Description:** Conversion intelligence platform with drag-and-drop landing page builder, Smart Traffic AI routing, and A/B testing.
- **API Documentation:** https://developer.unbounce.com/ (Unbounce Developer Portal)
- **SDKs/Libraries:** No official SDKs; community Node and Ruby wrappers exist.
- **Developer Guide:** https://developer.unbounce.com/api_reference/
- **Standards:** REST/JSON; pagination via cursor.
- **Authentication:** OAuth 2.0 (authorization code flow) and personal access tokens.

### Instapage
- **Description:** Ad-to-page personalization platform with AdMap, AI copy assistance, and dynamic text replacement.
- **API Documentation:** https://help.instapage.com/hc/en-us (search for API; access typically gated to enterprise customers)
- **SDKs/Libraries:** No publicly documented SDKs.
- **Developer Guide:** Provided directly to enterprise customers; partner integrations via Zapier and native connectors.
- **Standards:** REST/JSON.
- **Authentication:** API key (enterprise) and OAuth 2.0 for marketplace integrations.

### Leadpages
- **Description:** Affordable drag-and-drop builder with built-in AI copy/image generation and Conversion Guidance.
- **API Documentation:** https://docs.leadpages.com/ and https://www.leadpages.com/integrations
- **SDKs/Libraries:** Webhook-based integration; no official SDKs.
- **Developer Guide:** https://help.leadpages.com/
- **Standards:** REST/JSON; webhook events for form submissions.
- **Authentication:** API key.

### HubSpot Landing Pages (HubSpot CMS API)
- **Description:** Landing pages embedded in HubSpot Marketing Hub with Breeze AI generation and OpenAI integration.
- **API Documentation:** https://developers.hubspot.com/docs/api/cms/pages
- **SDKs/Libraries:** Official SDKs in JavaScript (https://github.com/HubSpot/hubspot-api-nodejs), Python, PHP, Ruby.
- **Developer Guide:** https://developers.hubspot.com/docs/cms/overview
- **Standards:** REST/JSON; OpenAPI spec at https://api.hubspot.com/api-catalog-public/v1/apis. GraphQL available for some objects.
- **Authentication:** OAuth 2.0 (recommended) and private app access tokens.

### Webflow
- **Description:** Visual web design platform widely used for landing pages with CMS, hosting, and editor APIs.
- **API Documentation:** https://developers.webflow.com/
- **SDKs/Libraries:** Official JavaScript SDK https://github.com/webflow/js-webflow-api; community Python and Go libraries.
- **Developer Guide:** https://developers.webflow.com/data/docs/getting-started-apps
- **Standards:** REST/JSON v2 API following OpenAPI conventions; webhook events.
- **Authentication:** OAuth 2.0 for marketplace apps; site API tokens for direct integrations.

### Framer
- **Description:** Design-driven website and landing page builder with code components and CMS.
- **API Documentation:** https://www.framer.com/developers/ and https://www.framer.com/developers/plugins-introduction
- **SDKs/Libraries:** Plugin SDK in TypeScript; Framer Motion for React animations https://www.framer.com/motion/.
- **Developer Guide:** https://www.framer.com/developers/plugins-quickstart
- **Standards:** REST API for CMS; plugin SDK uses postMessage and JSON.
- **Authentication:** OAuth 2.0 for plugins; CMS API tokens.

### Wix (Velo / Wix Studio)
- **Description:** Large-scale website and landing page builder with AI site generation (Wix ADI / Wix AI).
- **API Documentation:** https://dev.wix.com/api/rest and https://dev.wix.com/docs
- **SDKs/Libraries:** JavaScript SDK https://dev.wix.com/docs/sdk; Velo backend APIs in JS/Node.
- **Developer Guide:** https://dev.wix.com/docs/build-apps
- **Standards:** REST/JSON with OpenAPI specs per service domain; webhooks.
- **Authentication:** OAuth 2.0 for apps; API keys for headless usage.

### Squarespace (Pages & Commerce APIs)
- **Description:** Website and landing page builder with growing AI assist features.
- **API Documentation:** https://developers.squarespace.com/
- **SDKs/Libraries:** No official multi-language SDK; community libraries on GitHub.
- **Developer Guide:** https://developers.squarespace.com/getting-started
- **Standards:** REST/JSON; webhooks.
- **Authentication:** OAuth 2.0.

### Sanity (Headless CMS commonly powering landing pages)
- **Description:** Structured content platform often used as the backing store for AI-generated landing page content.
- **API Documentation:** https://www.sanity.io/docs/http-api
- **SDKs/Libraries:** Official JavaScript, Python, PHP, Go SDKs https://www.sanity.io/docs/client-libraries
- **Developer Guide:** https://www.sanity.io/docs/getting-started
- **Standards:** REST/JSON plus GROQ query language; GraphQL optional; webhooks.
- **Authentication:** Bearer tokens (project-scoped) and OAuth 2.0 for the Sanity Manage API.

### Builder.io
- **Description:** Visual headless CMS and AI-driven page builder with Visual Copilot generating React/Vue/Svelte components.
- **API Documentation:** https://www.builder.io/c/docs/developers
- **SDKs/Libraries:** Official SDKs for React, Next.js, Vue, Nuxt, Svelte, Angular, Qwik, React Native, Gatsby https://github.com/BuilderIO/builder
- **Developer Guide:** https://www.builder.io/c/docs/quickstart
- **Standards:** REST/JSON Content API; GraphQL API; OpenAPI documented.
- **Authentication:** Public API keys for read; private keys for write; OAuth for plugins.

### OpenAI Platform (AI generation backbone)
- **Description:** LLM and image generation APIs commonly used to power landing page copy and creative generation.
- **API Documentation:** https://platform.openai.com/docs/api-reference
- **SDKs/Libraries:** Official Python, Node.js, Go, Java, .NET SDKs https://platform.openai.com/docs/libraries
- **Developer Guide:** https://platform.openai.com/docs/quickstart
- **Standards:** REST/JSON; SSE streaming; OpenAPI specs published.
- **Authentication:** API key (Bearer); project-scoped keys with OIDC available.

### Anthropic Claude API
- **Description:** Claude LLM API used for high-quality copy generation, structured page generation, and tool-using agents.
- **API Documentation:** https://docs.anthropic.com/en/api/
- **SDKs/Libraries:** Official Python, TypeScript, Java, Go SDKs https://docs.anthropic.com/en/api/client-sdks
- **Developer Guide:** https://docs.anthropic.com/en/docs/welcome
- **Standards:** REST/JSON; SSE streaming; supports MCP for tool use.
- **Authentication:** API key (Bearer).

### Google PageSpeed Insights / CrUX API (performance validation)
- **Description:** APIs for measuring Core Web Vitals on generated landing pages.
- **API Documentation:** https://developer.chrome.com/docs/crux/api and https://developers.google.com/speed/docs/insights/v5/get-started
- **SDKs/Libraries:** Lighthouse CI https://github.com/GoogleChrome/lighthouse-ci
- **Developer Guide:** https://developers.google.com/speed/docs/insights/v5/about
- **Standards:** REST/JSON.
- **Authentication:** API key.

### axe-core (accessibility validation)
- **Description:** Open-source accessibility engine for automated WCAG auditing of generated pages.
- **API Documentation:** https://github.com/dequelabs/axe-core/tree/develop/doc
- **SDKs/Libraries:** axe-core (JS), axe-playwright, axe-puppeteer, @axe-core/react.
- **Developer Guide:** https://www.deque.com/axe/core-documentation/
- **Standards:** WCAG 2.0/2.1/2.2 rule mapping.
- **Authentication:** N/A (library).

## Notes

- **MCP adoption** is moving quickly in 2025–2026; an MCP server would let third-party AI agents (Claude Desktop, Cursor, IDE assistants) operate the page builder directly. This is an underexplored differentiator among landing page tools and worth prioritizing early.
- **AMP's relevance is declining.** Google reduced AMP's special treatment in 2024; new builders should treat AMP as optional rather than core architecture and instead optimise standard HTML/CSS/JS for Core Web Vitals.
- **WCAG 2.2 is now the baseline** (Oct 2023); WCAG 3.0 is in active drafting and will introduce a scoring model rather than pass/fail. Builder architecture should anticipate scored accessibility outcomes.
- **Standard Webhooks** is gaining traction as a cross-vendor convention; adopting it early simplifies integrations with downstream CRMs and automation platforms.
- **Consent management standards** (IAB TCF, Google Consent Mode v2) are mandatory for any builder serving EU traffic and using analytics or ads pixels.
- **Privacy-preserving personalization** standards are still emerging; watch W3C's Private Advertising Technology Community Group (PATCG) and Google's Privacy Sandbox APIs for relevant primitives.
