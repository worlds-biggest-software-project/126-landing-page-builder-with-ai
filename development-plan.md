# Landing Page Builder with AI — Phased Development Plan

> Project: 126-landing-page-builder-with-ai · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (full-stack) | The project is fundamentally a visual web editor with a REST/WebSocket API. TypeScript provides type safety across both the frontend drag-and-drop builder and the backend API, enabling shared type definitions for page content schemas. The LLM integration layer is straightforward in TypeScript via official Anthropic and OpenAI SDKs. |
| API framework | Next.js 15 (App Router) with tRPC | Next.js provides server-side rendering for the published landing pages (critical for Core Web Vitals and SEO), API routes for the builder backend, and React Server Components for the editor. tRPC provides end-to-end type safety between frontend and backend without code generation. |
| Database | PostgreSQL 17 with JSONB | The hybrid relational + JSONB model (Data Model Suggestion 2) fits naturally: relational tables for users, workspaces, experiments, leads, and analytics; JSONB for the polymorphic page content tree. PostgreSQL 17 GIN indexes enable efficient querying into JSONB content. Drizzle ORM for type-safe schema management and migrations. |
| Cache / queue | Redis 7 (via Upstash for serverless) | Caching published pages, rate limiting AI generation requests, and backing the async job queue for AI generation and webhook delivery. BullMQ as the job queue library on top of Redis. |
| Frontend framework | React 19 (via Next.js) | React is the natural choice for a visual drag-and-drop builder requiring complex state management, real-time preview, and component composition. |
| Drag-and-drop engine | dnd-kit | Accessible, performant drag-and-drop library purpose-built for React. Supports nested sortable lists (sections containing elements), collision detection strategies, and keyboard navigation for WCAG compliance. |
| State management | Zustand + Immer | Zustand provides lightweight, TypeScript-friendly stores. Immer enables immutable updates to the deeply nested JSONB page content tree without spread-operator gymnastics. Supports undo/redo via patches. |
| AI / LLM integration | Vercel AI SDK + Anthropic/OpenAI SDKs | Vercel AI SDK provides streaming, structured output, and tool-use patterns. Direct SDK usage for non-streaming batch operations (prompt-to-page generation). Pluggable provider architecture. |
| CSS / styling (builder UI) | Tailwind CSS 4 + shadcn/ui | Tailwind for utility-first styling of the builder application itself. shadcn/ui for accessible, composable UI components (dialogs, dropdowns, toolbars). |
| CSS (generated pages) | Inline styles + optimised CSS extraction | Generated landing pages use inline styles (stored in the JSONB content tree) extracted to a scoped `<style>` block at render time. No external CSS dependency ensures zero-FOUC and predictable Core Web Vitals. |
| Image storage | S3-compatible (AWS S3 / Cloudflare R2) | R2 preferred for zero-egress pricing on CDN-served assets. Presigned upload URLs for direct browser-to-storage uploads. |
| CDN | Cloudflare | Published landing pages served via Cloudflare CDN for global sub-100ms TTFB. Automatic image optimisation (WebP/AVIF) and Brotli compression. |
| Authentication | NextAuth.js v5 (Auth.js) | OAuth 2.1 (Google, GitHub, Microsoft) plus email/password with bcrypt. JWT sessions with OIDC-compliant claims. |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest for fast, TypeScript-native unit and integration tests with built-in mocking. Playwright for end-to-end browser testing of the drag-and-drop builder and published page rendering. |
| Code quality | Biome (lint + format) + TypeScript strict mode | Biome replaces ESLint + Prettier with a single fast tool. TypeScript strict mode with `noUncheckedIndexedAccess` for maximum type safety. |
| Package manager | pnpm | Strict dependency resolution, workspace support for monorepo structure, and disk efficiency via content-addressable storage. |
| Containerisation | Docker + Docker Compose | Multi-stage Dockerfile for production builds. Docker Compose for local development with PostgreSQL, Redis, and MinIO (S3-compatible) services. |
| Accessibility auditing | axe-core (runtime) + pa11y (CI) | axe-core integrated into the builder for real-time accessibility checking during page construction. pa11y for automated WCAG 2.2 AA auditing in CI pipelines. |
| Performance measurement | Lighthouse CI + web-vitals SDK | Lighthouse CI for pre-deployment Core Web Vitals validation. web-vitals SDK embedded in published pages for field data collection (LCP, CLS, INP). |
| Schema validation | Zod | Runtime validation of page content schemas, API payloads, form submissions, and configuration. Zod schemas generate TypeScript types and OpenAPI definitions. |

### Project Structure

```
landing-page-builder/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json                          # Turborepo for monorepo task orchestration
├── docker-compose.yml                  # PostgreSQL, Redis, MinIO for local dev
├── Dockerfile
├── .env.example
├── apps/
│   └── web/                            # Next.js application
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── src/
│       │   ├── app/                    # Next.js App Router
│       │   │   ├── (auth)/             # Login, signup, OAuth callback routes
│       │   │   ├── (dashboard)/        # Workspace dashboard, page list, settings
│       │   │   ├── (editor)/           # Visual page builder editor
│       │   │   ├── (published)/        # Published landing page renderer ([slug] route)
│       │   │   └── api/                # API routes (tRPC handler, webhooks, file upload)
│       │   ├── components/
│       │   │   ├── editor/             # Builder UI components (canvas, toolbar, sidebar, property panel)
│       │   │   ├── elements/           # Renderable element components (heading, button, image, form, etc.)
│       │   │   ├── dashboard/          # Dashboard UI components
│       │   │   └── ui/                 # shadcn/ui base components
│       │   ├── hooks/                  # React hooks (useEditor, useUndoRedo, useDragAndDrop)
│       │   ├── stores/                 # Zustand stores (editorStore, experimentStore, analyticsStore)
│       │   └── styles/                 # Global styles, Tailwind layers
│       └── public/                     # Static assets for the builder app
├── packages/
│   ├── db/                             # Drizzle schema, migrations, connection
│   │   ├── src/
│   │   │   ├── schema/                 # Table definitions (pages, users, workspaces, experiments, etc.)
│   │   │   ├── migrations/             # Drizzle migration files
│   │   │   └── index.ts                # DB client export
│   │   └── drizzle.config.ts
│   ├── api/                            # tRPC routers
│   │   └── src/
│   │       ├── routers/                # pages, experiments, leads, ai, integrations, analytics
│   │       ├── middleware/             # auth, rate-limiting, workspace-scoping
│   │       └── trpc.ts                 # tRPC init and context
│   ├── core/                           # Shared business logic (no framework dependency)
│   │   └── src/
│   │       ├── ai/                     # AI generation logic (prompt-to-page, copy generation, scoring)
│   │       ├── experiments/            # A/B test engine (traffic allocation, statistical significance)
│   │       ├── personalization/        # DTR engine, visitor-intent matching, rule evaluation
│   │       ├── rendering/              # Page content → HTML renderer, CSS extraction
│   │       ├── accessibility/          # axe-core integration, auto-remediation
│   │       ├── seo/                    # Schema.org injection, Open Graph generation
│   │       ├── analytics/              # Event processing, heatmap aggregation
│   │       └── validation/             # Zod schemas for page content, forms, API payloads
│   ├── types/                          # Shared TypeScript types and Zod schemas
│   │   └── src/
│   │       ├── page.ts                 # PageContent, Section, Element, ElementStyles types
│   │       ├── experiment.ts           # Experiment, Variant, ExperimentResult types
│   │       ├── ai.ts                   # AiGenerationJob, CopyVariant, PredictiveScore types
│   │       ├── lead.ts                 # Lead, FormField, ConsentRecord types
│   │       └── analytics.ts            # PageView, ConversionEvent, PerformanceScore types
│   └── config/                         # Shared configuration (env parsing, defaults)
│       └── src/
│           └── index.ts                # Zod-validated environment config
├── workers/                            # Background job processors
│   ├── ai-generation.ts                # BullMQ worker for AI page/copy generation
│   ├── webhook-delivery.ts             # BullMQ worker for outbound webhook delivery
│   ├── analytics-aggregation.ts        # Periodic analytics roll-up
│   └── accessibility-audit.ts          # Background WCAG audit runner
└── tests/
    ├── unit/                           # Vitest unit tests
    ├── integration/                    # Vitest integration tests (with test DB)
    ├── e2e/                            # Playwright browser tests
    └── fixtures/                       # Test data (sample pages, templates, form submissions)
```

---

## Phase 1: Project Foundation & Infrastructure

### Purpose

Establish the monorepo structure, database schema, authentication system, and Docker-based development environment. After this phase, a developer can clone the repo, run `docker compose up`, sign in, and see an empty workspace dashboard. This phase delivers no user-facing page-building functionality but provides the foundation every subsequent phase depends on.

### Tasks

#### 1.1 — Monorepo Scaffolding & Tooling

**What**: Initialise the pnpm workspace, Turborepo configuration, TypeScript project references, Biome config, and CI pipeline.

**Design**:

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": { "dependsOn": ["^build"] },
    "typecheck": { "dependsOn": ["^build"] },
    "test": { "dependsOn": ["^build"] },
    "test:e2e": { "dependsOn": ["build"] },
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false }
  }
}
```

```typescript
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: landing_builder
      POSTGRES_USER: builder
      POSTGRES_PASSWORD: builder_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Unit: pnpm install completes without peer dependency warnings`
- `Unit: turbo build succeeds across all packages with no circular dependencies`
- `Unit: biome lint --check passes on scaffolded code`
- `Unit: TypeScript typecheck passes in strict mode across all packages`
- `Integration: docker compose up starts PostgreSQL, Redis, and MinIO; all health checks pass`

#### 1.2 — Database Schema & Migrations

**What**: Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 2) using Drizzle ORM.

**Design**:

```typescript
// packages/db/src/schema/workspaces.ts
import { pgTable, uuid, varchar, jsonb, timestamp, text } from "drizzle-orm/pg-core";

export const workspaces = pgTable("workspaces", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  plan: varchar("plan", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/users.ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 320 }).notNull().unique(),
  name: varchar("name", { length: 255 }).notNull(),
  avatarUrl: text("avatar_url"),
  passwordHash: text("password_hash"),
  emailVerifiedAt: timestamp("email_verified_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/workspace-members.ts
export const workspaceMembers = pgTable("workspace_members", {
  workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  role: varchar("role", { length: 50 }).notNull().default("editor"),
  invitedAt: timestamp("invited_at", { withTimezone: true }).notNull().defaultNow(),
  acceptedAt: timestamp("accepted_at", { withTimezone: true }),
}, (table) => ({
  pk: primaryKey({ columns: [table.workspaceId, table.userId] }),
}));

// packages/db/src/schema/pages.ts
export const pages = pgTable("pages", {
  id: uuid("id").primaryKey().defaultRandom(),
  workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
  templateId: uuid("template_id").references(() => templates.id, { onDelete: "set null" }),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 255 }).notNull(),
  status: varchar("status", { length: 50 }).notNull().default("draft"),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  content: jsonb("content").notNull().default({ sections: [] }),
  seo: jsonb("seo").notNull().default({}),
  settings: jsonb("settings").notNull().default({}),
  personalization: jsonb("personalization").notNull().default({ rules: [], dtr: [] }),
  createdBy: uuid("created_by").notNull().references(() => users.id),
  updatedBy: uuid("updated_by").references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  workspaceSlugUnique: unique().on(table.workspaceId, table.slug),
  workspaceIdx: index("idx_pages_workspace").on(table.workspaceId),
  statusIdx: index("idx_pages_status").on(table.status),
  contentGin: index("idx_pages_content_gin").using("gin", table.content),
}));
```

Remaining tables (api_keys, templates, element_type_schemas, experiments, variants, experiment_results, ai_generation_jobs, leads, consent_records, page_views, conversion_events, performance_scores, accessibility_audits, integrations, webhook_endpoints, webhook_deliveries, assets, page_versions) follow the same pattern from Data Model Suggestion 2 (22 tables total).

```typescript
// packages/db/src/seed.ts
export async function seed(db: DrizzleClient) {
  // Insert default element type schemas
  await db.insert(elementTypeSchemas).values([
    { typeName: "heading", displayName: "Heading", icon: "heading", schema: headingSchema, defaultStyles: {} },
    { typeName: "paragraph", displayName: "Paragraph", icon: "text", schema: paragraphSchema, defaultStyles: {} },
    { typeName: "button", displayName: "Button", icon: "mouse-pointer", schema: buttonSchema, defaultStyles: {} },
    { typeName: "image", displayName: "Image", icon: "image", schema: imageSchema, defaultStyles: {} },
    { typeName: "form", displayName: "Form", icon: "file-text", schema: formSchema, defaultStyles: {} },
    { typeName: "video", displayName: "Video", icon: "video", schema: videoSchema, defaultStyles: {} },
  ]);
}
```

**Testing**:
- `Unit: Drizzle schema compiles without errors; all relations resolve`
- `Integration: drizzle-kit push applies schema to a fresh PostgreSQL database without errors`
- `Integration: drizzle-kit generate produces migration files from schema changes`
- `Integration: seed script inserts default element type schemas; SELECT count(*) returns 6`
- `Unit: all JSONB default values are valid JSON`

#### 1.3 — Authentication & Workspace Management

**What**: Implement user authentication (OAuth + email/password) via Auth.js v5, workspace CRUD, and workspace member invitations.

**Design**:

```typescript
// packages/api/src/routers/auth.ts
import { z } from "zod";
import { router, publicProcedure, protectedProcedure } from "../trpc";

export const authRouter = router({
  register: publicProcedure
    .input(z.object({
      email: z.string().email().max(320),
      name: z.string().min(1).max(255),
      password: z.string().min(8).max(128),
    }))
    .mutation(async ({ input, ctx }) => {
      // Hash password with bcrypt (cost=12), insert user, create default workspace
      // Returns: { userId: string, workspaceId: string }
    }),

  getSession: protectedProcedure
    .query(async ({ ctx }) => {
      // Returns current user + active workspace from session
      // Returns: { user: User, workspace: Workspace, role: WorkspaceRole }
    }),
});

// packages/api/src/routers/workspaces.ts
export const workspaceRouter = router({
  create: protectedProcedure
    .input(z.object({
      name: z.string().min(1).max(255),
      slug: z.string().min(3).max(100).regex(/^[a-z0-9-]+$/),
    }))
    .mutation(async ({ input, ctx }) => {
      // Insert workspace, add current user as owner
      // Returns: Workspace
    }),

  inviteMember: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      email: z.string().email(),
      role: z.enum(["admin", "editor", "viewer"]),
    }))
    .mutation(async ({ input, ctx }) => {
      // Validate caller is owner/admin, insert workspace_member with null acceptedAt
      // Send invitation email
      // Returns: { invitationId: string }
    }),

  listMembers: protectedProcedure
    .input(z.object({ workspaceId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns: WorkspaceMember[]
    }),

  updateSettings: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      settings: workspaceSettingsSchema,
    }))
    .mutation(async ({ input, ctx }) => {
      // JSONB merge update on workspace settings
      // Returns: Workspace
    }),
});
```

```typescript
// packages/types/src/workspace.ts
export const workspaceSettingsSchema = z.object({
  customDomain: z.string().optional(),
  defaultConsentMode: z.enum(["banner", "implicit", "none"]).default("banner"),
  googleConsentModeV2: z.boolean().default(false),
  tcfEnabled: z.boolean().default(false),
  branding: z.object({
    primaryColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).default("#2563eb"),
    logoUrl: z.string().url().optional(),
  }).default({}),
  aiProviders: z.object({
    copy: z.enum(["anthropic", "openai"]).default("anthropic"),
    image: z.enum(["openai", "stability"]).default("openai"),
  }).default({}),
  dataResidency: z.enum(["us", "eu"]).default("us"),
});

export type WorkspaceSettings = z.infer<typeof workspaceSettingsSchema>;

export type WorkspaceRole = "owner" | "admin" | "editor" | "viewer";
```

**Testing**:
- `Unit: register with valid email/name/password → user created, default workspace created, user is workspace owner`
- `Unit: register with duplicate email → 409 Conflict error`
- `Unit: register with password <8 chars → validation error`
- `Integration (mocked): Google OAuth callback with valid code → user created or linked, session established`
- `Unit: inviteMember as viewer → 403 Forbidden (insufficient role)`
- `Unit: inviteMember as owner → invitation created, member role stored`
- `Unit: workspace slug with invalid characters → validation error`
- `Unit: updateSettings merges with existing settings (does not overwrite unset fields)`
- `E2E: sign up flow → redirect to dashboard → workspace name visible in header`

#### 1.4 — Dashboard Shell & Navigation

**What**: Build the workspace dashboard layout with sidebar navigation, page listing (empty state), and workspace settings page.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
// Authenticated layout with:
// - Left sidebar: workspace name, navigation (Pages, Experiments, Leads, Analytics, Integrations, Settings)
// - Top bar: user avatar, workspace switcher, notification bell
// - Main content area (children)

// apps/web/src/app/(dashboard)/pages/page.tsx
// Page listing with:
// - Empty state: illustration + "Create your first landing page" CTA
// - List view: name, status badge (draft/published/archived), last updated, conversion rate
// - Actions: Create, Duplicate, Archive, Delete
// - Filtering: status, search by name
```

```typescript
// packages/api/src/routers/pages.ts
export const pageRouter = router({
  list: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      status: z.enum(["draft", "published", "archived"]).optional(),
      search: z.string().optional(),
      cursor: z.string().uuid().optional(),
      limit: z.number().min(1).max(100).default(20),
    }))
    .query(async ({ input, ctx }) => {
      // Cursor-based pagination, filtered by workspace, status, search
      // Returns: { pages: PageSummary[], nextCursor: string | null }
    }),

  create: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      name: z.string().min(1).max(255),
      slug: z.string().optional(), // auto-generated from name if omitted
      templateId: z.string().uuid().optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      // Create page with default content (or clone from template)
      // Returns: Page
    }),

  delete: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Soft delete (status → "archived") or hard delete
    }),
});
```

**Testing**:
- `E2E: authenticated user navigates to /pages → sees empty state with "Create" button`
- `E2E: user creates a page → redirected to editor → page appears in list on return`
- `Unit: page list API with cursor returns correct next page of results`
- `Unit: page list API filters by status correctly`
- `Unit: page creation auto-generates slug from name (slugify: "My Page" → "my-page")`
- `Unit: duplicate slug within workspace → appends numeric suffix ("my-page-2")`

---

## Phase 2: Visual Page Builder — Core Editor

### Purpose

Build the drag-and-drop page builder editor that allows users to construct landing pages from sections and elements. After this phase, users can open a blank page, add sections (hero, features, testimonials, CTA, FAQ), populate them with elements (headings, paragraphs, buttons, images), edit content inline, rearrange via drag-and-drop, and save the page. No AI, no publishing, no A/B testing yet — just the core editing experience.

### Tasks

#### 2.1 — Page Content Type System

**What**: Define the TypeScript types and Zod schemas for the JSONB page content tree (sections, elements, styles, forms).

**Design**:

```typescript
// packages/types/src/page.ts
import { z } from "zod";

export const responsiveStylesSchema = z.object({
  desktop: z.record(z.string()).default({}),
  tablet: z.record(z.string()).default({}),
  mobile: z.record(z.string()).default({}),
  hover: z.record(z.string()).default({}),
});
export type ResponsiveStyles = z.infer<typeof responsiveStylesSchema>;

export const formFieldSchema = z.object({
  name: z.string(),
  type: z.enum(["text", "email", "phone", "select", "checkbox", "radio", "textarea", "hidden", "date"]),
  label: z.string(),
  placeholder: z.string().optional(),
  required: z.boolean().default(false),
  validationPattern: z.string().optional(),
  options: z.array(z.string()).optional(),
  stepNumber: z.number().int().default(1),
});
export type FormField = z.infer<typeof formFieldSchema>;

export const formConfigSchema = z.object({
  submitText: z.string().default("Submit"),
  successMessage: z.string().optional(),
  redirectUrl: z.string().url().optional().nullable(),
  isMultiStep: z.boolean().default(false),
  fields: z.array(formFieldSchema),
});
export type FormConfig = z.infer<typeof formConfigSchema>;

export const elementSchema: z.ZodType<Element> = z.object({
  id: z.string(),
  type: z.enum(["heading", "paragraph", "button", "image", "form", "video", "icon", "container", "column", "divider", "spacer", "html"]),
  tag: z.string().optional(),           // h1, h2, p, etc.
  content: z.string().optional(),        // text content
  href: z.string().optional(),           // link URL
  src: z.string().optional(),            // image/video source
  alt: z.string().optional(),            // alt text (WCAG)
  styles: responsiveStylesSchema.default({ desktop: {}, tablet: {}, mobile: {}, hover: {} }),
  config: formConfigSchema.optional(),   // only for form elements
  children: z.lazy(() => z.array(elementSchema)).optional(),  // nested elements (container/column)
  visible: z.boolean().default(true),
});
export type Element = z.infer<typeof elementSchema>;

export const sectionSchema = z.object({
  id: z.string(),
  type: z.enum(["hero", "features", "testimonials", "cta", "faq", "pricing", "footer", "custom", "structured_data"]),
  name: z.string(),
  visible: z.boolean().default(true),
  styles: z.object({
    backgroundColor: z.string().optional(),
    backgroundImageUrl: z.string().optional(),
    paddingTop: z.number().default(40),
    paddingBottom: z.number().default(40),
    maxWidth: z.number().default(1200),
  }).default({}),
  elements: z.array(elementSchema).default([]),
  schemaType: z.string().optional(),     // for structured_data sections
  data: z.record(z.unknown()).optional(), // for structured_data sections
});
export type Section = z.infer<typeof sectionSchema>;

export const pageContentSchema = z.object({
  sections: z.array(sectionSchema).default([]),
});
export type PageContent = z.infer<typeof pageContentSchema>;

export const pageSeoSchema = z.object({
  title: z.string().max(70).optional(),
  metaDescription: z.string().max(160).optional(),
  ogTitle: z.string().max(95).optional(),
  ogDescription: z.string().max(200).optional(),
  ogImage: z.string().url().optional(),
  canonicalUrl: z.string().url().optional().nullable(),
  noindex: z.boolean().default(false),
  nofollow: z.boolean().default(false),
});
export type PageSeo = z.infer<typeof pageSeoSchema>;
```

**Testing**:
- `Unit: valid page content JSON → parses successfully with pageContentSchema`
- `Unit: page content with missing section id → Zod validation error`
- `Unit: element with unknown type → Zod validation error`
- `Unit: deeply nested container → children elements → validates recursively`
- `Unit: form element with config.fields → formConfigSchema validates field types`
- `Unit: responsiveStyles with arbitrary CSS properties → accepts string values`
- `Unit: round-trip serialisation: parse → serialise → parse yields identical output`

#### 2.2 — Editor Canvas & Section Management

**What**: Build the editor layout with a visual canvas that renders sections, a section toolbar for adding/removing/reordering sections, and section property editing.

**Design**:

```typescript
// apps/web/src/stores/editorStore.ts
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";
import type { PageContent, Section, Element } from "@landing-builder/types";

interface EditorState {
  pageId: string;
  content: PageContent;
  selectedSectionId: string | null;
  selectedElementId: string | null;
  isDirty: boolean;
  undoStack: PageContent[];
  redoStack: PageContent[];

  // Section operations
  addSection: (section: Section, index?: number) => void;
  removeSection: (sectionId: string) => void;
  moveSection: (sectionId: string, newIndex: number) => void;
  updateSectionStyles: (sectionId: string, styles: Partial<Section["styles"]>) => void;

  // Element operations
  addElement: (sectionId: string, element: Element, index?: number) => void;
  removeElement: (sectionId: string, elementId: string) => void;
  moveElement: (sourceSectionId: string, targetSectionId: string, elementId: string, newIndex: number) => void;
  updateElement: (sectionId: string, elementId: string, updates: Partial<Element>) => void;
  updateElementStyles: (sectionId: string, elementId: string, breakpoint: string, styles: Record<string, string>) => void;

  // Selection
  selectSection: (sectionId: string | null) => void;
  selectElement: (elementId: string | null) => void;

  // History
  undo: () => void;
  redo: () => void;

  // Persistence
  setContent: (content: PageContent) => void;
  markClean: () => void;
}

export const useEditorStore = create<EditorState>()(
  immer((set, get) => ({
    // Implementation uses Immer for immutable updates to nested JSONB structure
    // Every mutation pushes current state to undoStack and clears redoStack
    // ...
  }))
);
```

```typescript
// apps/web/src/components/editor/EditorCanvas.tsx
// Renders sections as vertical stack
// Each section has:
//   - Drag handle (left edge)
//   - Section type label + visibility toggle
//   - Element rendering area
//   - "Add element" floating toolbar (appears on hover at bottom)
//   - Section settings gear icon (opens property panel)

// apps/web/src/components/editor/SectionToolbar.tsx
// Floating toolbar with section type buttons:
//   Hero | Features | Testimonials | CTA | FAQ | Pricing | Footer | Custom
// Clicking inserts a section with default content for that type

// apps/web/src/components/editor/PropertyPanel.tsx
// Right sidebar that shows properties for the selected section or element
// Section properties: background color, background image, padding, max width
// Element properties: vary by element type (content, src, href, alt, styles per breakpoint)
```

**Testing**:
- `Unit: addSection inserts at correct index; content.sections length increases by 1`
- `Unit: removeSection removes correct section; other sections unaffected`
- `Unit: moveSection reorders sections correctly (index 0 → 2)`
- `Unit: updateSectionStyles merges partial styles without overwriting existing`
- `Unit: undo after addSection restores previous content`
- `Unit: redo after undo restores the added section`
- `Unit: undo stack is capped at 50 entries to prevent memory issues`
- `E2E: click "Hero" in section toolbar → hero section appears on canvas with default headline`
- `E2E: drag section from position 1 to position 3 → sections reorder visually`
- `E2E: select section → property panel shows section styles → change background color → canvas updates`

#### 2.3 — Element Editing & Inline Content

**What**: Implement element rendering, inline text editing (contentEditable for headings/paragraphs), element property editing in the right panel, and element drag-and-drop within and across sections.

**Design**:

```typescript
// apps/web/src/components/elements/ElementRenderer.tsx
// Renders the appropriate component based on element.type:
//   heading → <h1>...<h6> with contentEditable in editor mode
//   paragraph → <p> with contentEditable
//   button → <a> styled as button; click selects (not navigates) in editor
//   image → <img> with placeholder if no src; click to upload/select
//   form → FormRenderer with field list
//   video → video embed with placeholder
//   container → flex/grid wrapper rendering children recursively
//   column → flex child rendering children recursively

// apps/web/src/components/editor/InlineEditor.tsx
// contentEditable wrapper that:
//   - Syncs text changes to editorStore on blur
//   - Handles Enter key (create new paragraph or prevent default based on element type)
//   - Strips HTML paste to plain text
//   - Shows placeholder text when content is empty
//   - Applies styles from element.styles.desktop (or active breakpoint)

// apps/web/src/hooks/useDragAndDrop.ts
// Uses dnd-kit with:
//   - SortableContext for elements within a section
//   - DragOverlay for smooth visual feedback
//   - Collision detection: closestCorners for elements, closestCenter for sections
//   - onDragEnd → calls moveElement(sourceSection, targetSection, elementId, newIndex)
```

```typescript
// apps/web/src/components/editor/ElementToolbar.tsx
// Floating toolbar that appears when hovering the bottom of a section or between elements
// Shows element type buttons:
//   Heading | Text | Button | Image | Form | Video | Divider | Spacer | HTML
// Clicking inserts element with sensible defaults

// Default element factories
export const defaultElements: Record<string, () => Omit<Element, "id">> = {
  heading: () => ({
    type: "heading",
    tag: "h2",
    content: "Your Headline Here",
    styles: {
      desktop: { fontSize: "36px", fontWeight: "700", color: "#111827", textAlign: "center" },
      tablet: { fontSize: "28px" },
      mobile: { fontSize: "24px" },
      hover: {},
    },
  }),
  button: () => ({
    type: "button",
    content: "Get Started",
    href: "#",
    styles: {
      desktop: { backgroundColor: "#2563eb", color: "#ffffff", padding: "16px 32px", borderRadius: "8px", fontSize: "18px", fontWeight: "600", display: "inline-block", textAlign: "center" },
      tablet: {},
      mobile: { padding: "12px 24px", fontSize: "16px" },
      hover: { backgroundColor: "#1d4ed8" },
    },
  }),
  // ... paragraph, image, form, video, divider, spacer, html
};
```

**Testing**:
- `Unit: ElementRenderer renders heading element as <h2> with correct text content`
- `Unit: ElementRenderer renders button with href and styled correctly`
- `Unit: ElementRenderer renders image with alt text (WCAG requirement)`
- `Unit: InlineEditor syncs text change to store on blur`
- `Unit: InlineEditor strips HTML tags from pasted content`
- `E2E: click "Heading" button → heading appears in section → click heading → text becomes editable → type new text → click away → store updated`
- `E2E: drag element from section A to section B → element moves; section A loses it, section B gains it`
- `E2E: drag element within section → element reorders`
- `Unit: default element factory for "button" produces valid Element per elementSchema`
- `Unit: container element renders children recursively`

#### 2.4 — Responsive Preview & Breakpoint Switching

**What**: Implement device preview (desktop/tablet/mobile) in the editor and breakpoint-specific style editing.

**Design**:

```typescript
// apps/web/src/stores/editorStore.ts (extended)
interface EditorState {
  // ... existing fields
  activeBreakpoint: "desktop" | "tablet" | "mobile";
  setBreakpoint: (bp: "desktop" | "tablet" | "mobile") => void;
}

// Breakpoint widths used for canvas sizing
export const BREAKPOINTS = {
  desktop: 1280,
  tablet: 768,
  mobile: 375,
} as const;
```

```typescript
// apps/web/src/components/editor/DeviceToolbar.tsx
// Three toggle buttons: Desktop (monitor icon) | Tablet (tablet icon) | Mobile (phone icon)
// Switching breakpoint:
//   1. Changes canvas width to BREAKPOINTS[bp]
//   2. Property panel shows styles for active breakpoint
//   3. Element rendering merges styles: desktop → tablet → mobile (cascade)

// Style resolution function used by ElementRenderer
export function resolveStyles(
  styles: ResponsiveStyles,
  breakpoint: "desktop" | "tablet" | "mobile"
): Record<string, string> {
  const base = { ...styles.desktop };
  if (breakpoint === "tablet" || breakpoint === "mobile") {
    Object.assign(base, styles.tablet);
  }
  if (breakpoint === "mobile") {
    Object.assign(base, styles.mobile);
  }
  return base;
}
```

**Testing**:
- `Unit: resolveStyles at desktop returns only desktop styles`
- `Unit: resolveStyles at tablet merges desktop + tablet (tablet overrides desktop)`
- `Unit: resolveStyles at mobile merges desktop + tablet + mobile (mobile wins)`
- `E2E: switch to tablet view → canvas narrows to 768px → elements reflow`
- `E2E: edit font size in mobile breakpoint → only mobile styles updated; desktop unchanged`
- `Unit: property panel shows correct breakpoint styles when breakpoint changes`

#### 2.5 — Page Save & Auto-Save

**What**: Implement manual save, auto-save (debounced), and page version snapshots.

**Design**:

```typescript
// packages/api/src/routers/pages.ts (extended)
export const pageRouter = router({
  // ... existing routes

  get: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns full page with content, seo, settings, personalization
      // Returns: Page
    }),

  update: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      content: pageContentSchema.optional(),
      seo: pageSeoSchema.optional(),
      settings: z.record(z.unknown()).optional(),
      name: z.string().min(1).max(255).optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      // Validate content against element type schemas
      // Update page row (JSONB merge for partial updates)
      // Set updatedBy = ctx.user.id, updatedAt = now()
      // Returns: Page
    }),

  createVersion: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Snapshot current content into page_versions table
      // Auto-increment version_number
      // Pre-render snapshot_html for fast serving
      // Returns: PageVersion
    }),

  listVersions: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns: PageVersion[] (most recent first)
    }),

  restoreVersion: protectedProcedure
    .input(z.object({ pageId: z.string().uuid(), versionId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Copy version content/seo/personalization back to page row
      // Create a new version snapshot first (so restore is undoable)
      // Returns: Page
    }),
});
```

```typescript
// apps/web/src/hooks/useAutoSave.ts
export function useAutoSave(pageId: string) {
  const content = useEditorStore((s) => s.content);
  const isDirty = useEditorStore((s) => s.isDirty);
  const markClean = useEditorStore((s) => s.markClean);
  const updatePage = trpc.pages.update.useMutation();

  // Debounce: save 2 seconds after last change
  useEffect(() => {
    if (!isDirty) return;
    const timer = setTimeout(async () => {
      await updatePage.mutateAsync({ pageId, content });
      markClean();
    }, 2000);
    return () => clearTimeout(timer);
  }, [content, isDirty]);
}
```

**Testing**:
- `Unit: auto-save triggers 2 seconds after last content change`
- `Unit: rapid content changes reset the debounce timer (only one save)`
- `Unit: manual save immediately persists and resets isDirty`
- `Integration: save page content → re-fetch → content matches saved value`
- `Integration: createVersion snapshots current content with incremented version number`
- `Integration: restoreVersion copies version content back; creates new version first`
- `Unit: update with invalid content (malformed element) → validation error, page unchanged`

---

## Phase 3: Page Publishing & Rendering

### Purpose

Enable users to publish landing pages and serve them to visitors via optimised server-side rendering. After this phase, a user can build a page in the editor, click "Publish", and access it at a live URL with Core Web Vitals-compliant HTML, proper meta tags, and Open Graph support. This is the first time external visitors can see the output.

### Tasks

#### 3.1 — Server-Side Page Renderer

**What**: Build a rendering engine that converts the JSONB page content tree into optimised HTML + CSS for published pages.

**Design**:

```typescript
// packages/core/src/rendering/renderer.ts
export interface RenderOptions {
  content: PageContent;
  seo: PageSeo;
  settings: PageSettings;
  personalization?: PersonalizationContext;  // resolved DTR/rule replacements for this visitor
  variant?: PageContent;                     // A/B test variant content (overrides base content)
  trackingSnippet?: string;                  // analytics JS to inject
}

export interface RenderResult {
  html: string;           // Complete <!DOCTYPE html>...</html>
  css: string;            // Extracted scoped CSS
  structuredData: string; // JSON-LD script block(s)
  estimatedSize: number;  // bytes (for Core Web Vitals budgeting)
}

export function renderPage(options: RenderOptions): RenderResult {
  // 1. Resolve content: apply variant overrides if present, then personalization
  // 2. Extract all inline styles → generate scoped CSS classes
  //    (avoids inline style bloat; enables browser caching)
  // 3. Render sections → elements recursively into HTML
  // 4. Inject SEO: <title>, <meta description>, Open Graph tags, canonical URL
  // 5. Inject structured data as <script type="application/ld+json">
  // 6. Inject tracking snippet before </body>
  // 7. Inject consent banner HTML/JS if settings.consentMode === "banner"
  // 8. Minify HTML output
  // 9. Return { html, css, structuredData, estimatedSize }
}

// CSS extraction: convert inline styles to class-based styles
// Input:  element.styles.desktop = { fontSize: "36px", fontWeight: "700" }
// Output: .el_abc123 { font-size: 36px; font-weight: 700; }
//         @media (max-width: 768px) { .el_abc123 { font-size: 28px; } }
export function extractCSS(content: PageContent): string {
  // Walk element tree, generate unique class per element, emit CSS rules per breakpoint
}
```

```typescript
// packages/core/src/seo/structured-data.ts
export function generateStructuredData(content: PageContent, seo: PageSeo): string[] {
  const blocks: string[] = [];

  // Auto-generate Organization schema from workspace branding
  blocks.push(JSON.stringify({
    "@context": "https://schema.org",
    "@type": "Organization",
    name: seo.ogTitle,
    url: seo.canonicalUrl,
    logo: seo.ogImage,
  }));

  // Find FAQ sections and generate FAQPage schema
  const faqSections = content.sections.filter(s => s.type === "faq");
  for (const section of faqSections) {
    // Extract Q&A pairs from heading + paragraph element pairs
    blocks.push(JSON.stringify({
      "@context": "https://schema.org",
      "@type": "FAQPage",
      mainEntity: extractFaqPairs(section).map(({ question, answer }) => ({
        "@type": "Question",
        name: question,
        acceptedAnswer: { "@type": "Answer", text: answer },
      })),
    }));
  }

  return blocks;
}
```

**Testing**:
- `Unit: renderPage with single hero section → valid HTML with h1, button elements`
- `Unit: renderPage includes <title> and <meta name="description"> from seo`
- `Unit: renderPage includes Open Graph tags (<meta property="og:title">)`
- `Unit: extractCSS generates responsive media queries for tablet/mobile breakpoints`
- `Unit: renderPage with FAQ section → generates FAQPage JSON-LD structured data`
- `Unit: renderPage output is valid HTML (parse with htmlparser2)`
- `Unit: renderPage with consent mode "banner" → includes consent banner markup`
- `Integration: render a sample page → measure HTML size → under 50KB for a typical 5-section page`
- `Fixture: render fixture page "sample-hero-cta.json" → snapshot matches expected HTML output`

#### 3.2 — Published Page Routes & CDN Caching

**What**: Implement the Next.js dynamic route that serves published landing pages, with appropriate caching headers and CDN integration.

**Design**:

```typescript
// apps/web/src/app/(published)/[workspaceSlug]/[pageSlug]/page.tsx
// Server component that:
// 1. Looks up workspace by slug
// 2. Looks up published page by workspace + page slug
// 3. If not published → 404
// 4. Renders page using renderPage()
// 5. Returns HTML response with headers:
//    Cache-Control: public, s-maxage=60, stale-while-revalidate=300
//    Content-Security-Policy: <appropriate CSP for landing pages>

export async function generateMetadata({ params }): Promise<Metadata> {
  // Return Next.js metadata from page.seo
  return {
    title: page.seo.title,
    description: page.seo.metaDescription,
    openGraph: {
      title: page.seo.ogTitle,
      description: page.seo.ogDescription,
      images: page.seo.ogImage ? [{ url: page.seo.ogImage }] : [],
    },
    robots: {
      index: !page.seo.noindex,
      follow: !page.seo.nofollow,
    },
    alternates: { canonical: page.seo.canonicalUrl },
  };
}
```

```typescript
// packages/api/src/routers/pages.ts (extended)
export const pageRouter = router({
  publish: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // 1. Validate page content (all elements pass schema validation)
      // 2. Run accessibility pre-check (warn-only in this phase; enforced in Phase 8)
      // 3. Pre-render HTML snapshot and store in page_versions
      // 4. Set status = "published", publishedAt = now()
      // 5. Purge CDN cache for this page URL
      // Returns: { url: string, publishedAt: string }
    }),

  unpublish: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Set status = "draft", purge CDN cache
    }),
});
```

**Testing**:
- `Integration: publish page → GET /workspace-slug/page-slug → 200 with rendered HTML`
- `Integration: unpublished page → GET /workspace-slug/page-slug → 404`
- `Integration: published page response includes Cache-Control header with s-maxage`
- `Integration: published page response includes Content-Security-Policy header`
- `Unit: publish validates page content; page with invalid element → error, status unchanged`
- `E2E: build page in editor → publish → open published URL in new tab → page renders correctly`
- `Unit: Open Graph tags render correctly for social sharing`

#### 3.3 — Custom Domain Support

**What**: Enable workspaces to serve pages on their own custom domains.

**Design**:

```typescript
// Custom domain resolution:
// 1. Incoming request to custom domain → DNS CNAME points to builder's domain
// 2. Next.js middleware checks hostname against workspace custom_domain settings
// 3. If match found → rewrite to internal /(published)/[workspaceSlug]/[pageSlug] route
// 4. SSL via Cloudflare for SaaS (automatic certificate provisioning)

// apps/web/src/middleware.ts
export function middleware(request: NextRequest) {
  const hostname = request.headers.get("host") || "";
  const isCustomDomain = !hostname.endsWith(process.env.BASE_DOMAIN!);

  if (isCustomDomain) {
    // Look up workspace by custom_domain
    // Rewrite URL to internal route
    return NextResponse.rewrite(
      new URL(`/${workspaceSlug}/${pageSlug}`, request.url)
    );
  }
}
```

**Testing**:
- `Unit: middleware identifies custom domain vs. platform domain`
- `Unit: middleware rewrites custom domain request to correct internal route`
- `Integration: request with custom domain hostname → serves correct workspace's page`
- `Unit: request with unknown custom domain → 404`

#### 3.4 — Asset Upload & Image Management

**What**: Implement image upload to S3-compatible storage with CDN delivery, image optimisation, and asset library in the editor.

**Design**:

```typescript
// packages/api/src/routers/assets.ts
export const assetRouter = router({
  getUploadUrl: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      filename: z.string(),
      mimeType: z.enum(["image/jpeg", "image/png", "image/webp", "image/svg+xml", "image/gif", "video/mp4"]),
      fileSizeBytes: z.number().max(10 * 1024 * 1024), // 10MB max
    }))
    .mutation(async ({ input, ctx }) => {
      // Generate presigned S3 PUT URL (5 min expiry)
      // Insert asset record with status "uploading"
      // Returns: { uploadUrl: string, assetId: string, cdnUrl: string }
    }),

  confirmUpload: protectedProcedure
    .input(z.object({ assetId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Verify file exists in S3
      // Extract dimensions for images (via Sharp or S3 head metadata)
      // Generate alt text suggestion via AI (async job, optional)
      // Returns: Asset
    }),

  list: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      mimeType: z.string().optional(),
      cursor: z.string().uuid().optional(),
      limit: z.number().default(24),
    }))
    .query(async ({ input, ctx }) => {
      // Grid view of assets with thumbnail previews
      // Returns: { assets: Asset[], nextCursor: string | null }
    }),

  delete: protectedProcedure
    .input(z.object({ assetId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Delete from S3 and database
      // Warn if asset is referenced in any page content
    }),
});
```

**Testing**:
- `Unit: getUploadUrl generates valid presigned URL with correct content type`
- `Unit: getUploadUrl rejects files exceeding 10MB`
- `Unit: getUploadUrl rejects unsupported MIME types`
- `Integration: upload flow → presigned URL → PUT file → confirmUpload → asset accessible via CDN URL`
- `E2E: in editor, click image element → asset library opens → upload image → image appears in library → select → image renders on canvas`
- `Unit: delete warns when asset is referenced by page content`

---

## Phase 4: Lead Capture & Form Builder

### Purpose

Implement the form builder within the editor, lead capture on published pages, and lead management in the dashboard. After this phase, landing pages can collect leads with GDPR/CCPA-compliant consent, and workspace users can view, search, and export captured leads.

### Tasks

#### 4.1 — Visual Form Builder

**What**: Build the form element editor that allows users to add, configure, and arrange form fields within the page builder.

**Design**:

```typescript
// apps/web/src/components/editor/FormEditor.tsx
// When a form element is selected, the property panel shows:
// 1. Form settings: submit button text, success message, redirect URL, multi-step toggle
// 2. Field list: sortable list of form fields with drag handles
// 3. Per-field config: type, label, placeholder, required, validation pattern, options (for select/radio)
// 4. "Add Field" button with field type picker
// 5. Consent field preset: auto-generates GDPR consent checkbox with configurable legal text

// apps/web/src/components/elements/FormRenderer.tsx
// In editor mode: renders form with all fields visible, each field selectable for editing
// In published mode: renders functional <form> with client-side validation
//   - Required field validation
//   - Email format validation
//   - Phone format validation (E.164)
//   - Custom regex pattern validation
//   - Multi-step form navigation (next/previous buttons)
//   - GDPR consent checkbox with link to privacy policy

export interface FormSubmitPayload {
  pageId: string;
  formElementId: string;
  variantId?: string;
  fields: Record<string, string>;
  consents: Array<{
    type: "marketing" | "analytics" | "third_party";
    granted: boolean;
  }>;
  visitorId: string;
  sessionId: string;
  referrer: string;
  utmParams: Record<string, string>;
}
```

**Testing**:
- `Unit: FormEditor renders field list with correct labels and types`
- `Unit: adding a field to form updates page content JSONB`
- `Unit: reordering fields via drag-and-drop updates field order`
- `E2E: add form element → add email + name fields → set as required → preview shows form with validation`
- `Unit: multi-step form renders correct fields per step with navigation buttons`
- `Unit: consent checkbox field auto-generated with correct legal text`

#### 4.2 — Form Submission API & Lead Storage

**What**: Implement the public API endpoint for form submissions and the lead storage pipeline.

**Design**:

```typescript
// apps/web/src/app/api/submit/route.ts
// POST /api/submit
// Public endpoint (no auth required)
// Rate limited: 10 submissions per IP per minute
// Request body: FormSubmitPayload
// Response: { success: true, leadId: string } | { success: false, errors: ValidationError[] }

export async function POST(request: Request) {
  // 1. Parse and validate FormSubmitPayload with Zod
  // 2. Look up page and verify it's published
  // 3. Look up form element in page content to validate fields match schema
  // 4. Validate required fields, formats, patterns
  // 5. Rate limit check (Redis)
  // 6. Insert lead row with form_data JSONB
  // 7. Insert consent_records for each consent
  // 8. Record ConversionEvent (form_submit)
  // 9. Trigger webhook delivery jobs for "lead.created" event
  // 10. Return success with leadId
}
```

```typescript
// packages/api/src/routers/leads.ts
export const leadRouter = router({
  list: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      pageId: z.string().uuid().optional(),
      dateFrom: z.date().optional(),
      dateTo: z.date().optional(),
      search: z.string().optional(), // searches email and form_data
      cursor: z.string().uuid().optional(),
      limit: z.number().default(25),
    }))
    .query(async ({ input, ctx }) => {
      // Cursor-based pagination with JSONB search via GIN index
      // Returns: { leads: Lead[], nextCursor: string | null, totalCount: number }
    }),

  export: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      pageId: z.string().uuid().optional(),
      format: z.enum(["csv", "json"]),
    }))
    .mutation(async ({ input, ctx }) => {
      // Generate CSV/JSON export of leads
      // Include form_data fields as flattened columns
      // Returns: { downloadUrl: string }
    }),

  delete: protectedProcedure
    .input(z.object({ leadId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // GDPR right to erasure: delete lead + consent records
      // Returns: { deleted: true }
    }),
});
```

**Testing**:
- `Integration: POST /api/submit with valid payload → lead created, 200 returned`
- `Integration: POST /api/submit with missing required field → 400 with field-specific error`
- `Integration: POST /api/submit for unpublished page → 404`
- `Unit: rate limiter blocks 11th submission from same IP within 1 minute`
- `Integration: lead with consent records → consent_records table populated with correct legal basis`
- `Unit: lead list API searches email with ILIKE`
- `Unit: lead list API searches form_data JSONB with @> containment`
- `Integration: export CSV includes all form fields as columns; rows match lead data`
- `Unit: delete lead removes lead + consent records (CASCADE)`
- `E2E: publish page with form → submit form on published page → lead appears in dashboard`

#### 4.3 — GDPR/CCPA Consent Management

**What**: Implement cookie consent banner, consent tracking, and data subject rights endpoints.

**Design**:

```typescript
// packages/core/src/rendering/consent-banner.ts
// Generates consent banner HTML/CSS/JS injected into published pages
// Consent modes:
//   "banner" — show banner, block analytics/marketing cookies until consent
//   "implicit" — load all scripts, show informational banner
//   "none" — no banner (for non-EU traffic or pre-consented users)

// Google Consent Mode v2 integration:
// Sets window.gtag("consent", "default", { analytics_storage: "denied", ad_storage: "denied" })
// On consent: gtag("consent", "update", { analytics_storage: "granted" })

export function generateConsentBanner(config: {
  mode: "banner" | "implicit" | "none";
  googleConsentModeV2: boolean;
  tcfEnabled: boolean;
  privacyPolicyUrl?: string;
}): string {
  // Returns HTML string with:
  // - Banner UI (overlay at bottom of page)
  // - Accept All / Reject All / Customize buttons
  // - Cookie category toggles (necessary, analytics, marketing)
  // - JavaScript to store consent in first-party cookie
  // - Google Consent Mode v2 signal firing
  // - IAB TCF v2.2 TC string generation if tcfEnabled
}
```

**Testing**:
- `Unit: consent banner with mode "banner" includes Accept/Reject buttons`
- `Unit: consent banner with mode "none" returns empty string`
- `Unit: Google Consent Mode v2 signals fire correctly on accept`
- `Unit: TCF string generation produces valid IAB TCF v2.2 format`
- `Integration: form submission includes consent records → stored with correct legal basis`
- `Unit: GDPR delete endpoint removes all lead data including form_data and consents`

---

## Phase 5: A/B Testing & Experiment Engine

### Purpose

Implement the A/B testing and multivariate experiment system. After this phase, users can create experiments with multiple page variants, allocate traffic between them, monitor results with statistical significance, and automatically select winners. This is a key differentiator that directly drives the product's conversion-optimisation value proposition.

### Tasks

#### 5.1 — Experiment Management API

**What**: Build CRUD operations for experiments and variants.

**Design**:

```typescript
// packages/api/src/routers/experiments.ts
export const experimentRouter = router({
  create: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      name: z.string().min(1).max(255),
      experimentType: z.enum(["ab", "multivariate", "bandit"]),
      trafficPercentage: z.number().int().min(1).max(100).default(100),
      confidenceThreshold: z.number().min(0.8).max(0.99).default(0.95),
      primaryMetric: z.enum(["conversion_rate", "click_through_rate", "form_submit_rate"]).default("conversion_rate"),
    }))
    .mutation(async ({ input, ctx }) => {
      // Create experiment with control variant (uses page's current content)
      // Returns: Experiment with Variant[] (initially just control)
    }),

  addVariant: protectedProcedure
    .input(z.object({
      experimentId: z.string().uuid(),
      name: z.string().min(1).max(255),
      content: pageContentSchema, // full page content for this variant
      trafficWeight: z.number().int().min(1).max(100).default(50),
    }))
    .mutation(async ({ input, ctx }) => {
      // Add variant with full content copy
      // Rebalance traffic weights across all variants
      // Returns: Variant
    }),

  start: protectedProcedure
    .input(z.object({ experimentId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Validate: >=2 variants, page is published, traffic weights sum to 100
      // Set status = "running", startedAt = now()
      // Returns: Experiment
    }),

  pause: protectedProcedure
    .input(z.object({ experimentId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => { /* status → "paused" */ }),

  complete: protectedProcedure
    .input(z.object({
      experimentId: z.string().uuid(),
      winnerVariantId: z.string().uuid(),
    }))
    .mutation(async ({ input, ctx }) => {
      // Apply winner variant's content to the page's main content
      // Set status = "completed", endedAt = now()
      // Returns: Experiment
    }),

  getResults: protectedProcedure
    .input(z.object({ experimentId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns: ExperimentWithResults (variants with daily results, current significance)
    }),
});
```

```typescript
// packages/types/src/experiment.ts
export interface Experiment {
  id: string;
  pageId: string;
  name: string;
  experimentType: "ab" | "multivariate" | "bandit";
  status: "draft" | "running" | "paused" | "completed" | "archived";
  trafficPercentage: number;
  confidenceThreshold: number;
  primaryMetric: string;
  winnerVariantId: string | null;
  startedAt: string | null;
  endedAt: string | null;
  variants: Variant[];
}

export interface Variant {
  id: string;
  experimentId: string;
  name: string;
  isControl: boolean;
  trafficWeight: number;
  content: PageContent | null; // null for control (uses page content)
  results?: VariantResults;
}

export interface VariantResults {
  visitors: number;
  conversions: number;
  conversionRate: number;
  confidence: number;
  isSignificant: boolean;
  dailyResults: DailyResult[];
}
```

**Testing**:
- `Unit: create experiment → control variant auto-created with isControl=true`
- `Unit: addVariant rebalances weights (2 variants → 50/50, 3 variants → 33/33/34)`
- `Unit: start with <2 variants → validation error`
- `Unit: start on unpublished page → validation error`
- `Unit: complete applies winner content to page and sets status`
- `Integration: create → add variant → start → getResults returns empty results`

#### 5.2 — Traffic Allocation & Variant Serving

**What**: Implement visitor-to-variant assignment and variant-aware page rendering.

**Design**:

```typescript
// packages/core/src/experiments/traffic-allocator.ts
export interface TrafficAllocation {
  variantId: string;
  experimentId: string;
}

/**
 * Deterministic variant assignment using visitor hash.
 * Ensures same visitor always sees same variant (sticky sessions).
 * Uses MurmurHash3 of (visitorId + experimentId) → bucket → variant.
 */
export function allocateTraffic(
  visitorId: string,
  experiment: Experiment
): TrafficAllocation | null {
  if (experiment.status !== "running") return null;

  // Check if visitor is in the experiment's traffic percentage
  const experimentBucket = murmurHash3(visitorId + "exp_" + experiment.id) % 100;
  if (experimentBucket >= experiment.trafficPercentage) return null;

  // Assign to variant based on traffic weights
  const variantBucket = murmurHash3(visitorId + "var_" + experiment.id) % 100;
  let cumulative = 0;
  for (const variant of experiment.variants) {
    cumulative += variant.trafficWeight;
    if (variantBucket < cumulative) {
      return { variantId: variant.id, experimentId: experiment.id };
    }
  }
  return { variantId: experiment.variants[0].id, experimentId: experiment.id };
}

// Bandit algorithm (Thompson Sampling) for experiment_type === "bandit"
export function allocateTrafficBandit(
  experiment: Experiment
): string {
  // For each variant, sample from Beta(conversions + 1, visitors - conversions + 1)
  // Select the variant with the highest sampled value
  // This naturally explores undersampled variants while exploiting high-performers
  const samples = experiment.variants.map((v) => {
    const alpha = (v.results?.conversions ?? 0) + 1;
    const beta = (v.results?.visitors ?? 0) - (v.results?.conversions ?? 0) + 1;
    return { variantId: v.id, sample: betaSample(alpha, beta) };
  });
  return samples.sort((a, b) => b.sample - a.sample)[0].variantId;
}
```

```typescript
// packages/core/src/experiments/significance.ts
/**
 * Two-proportion z-test for statistical significance.
 * Compares each variant against the control.
 */
export function calculateSignificance(
  control: { visitors: number; conversions: number },
  treatment: { visitors: number; conversions: number }
): { confidence: number; isSignificant: boolean; pValue: number } {
  const p1 = control.conversions / control.visitors;
  const p2 = treatment.conversions / treatment.visitors;
  const pPooled = (control.conversions + treatment.conversions) / (control.visitors + treatment.visitors);
  const se = Math.sqrt(pPooled * (1 - pPooled) * (1 / control.visitors + 1 / treatment.visitors));
  const z = (p2 - p1) / se;
  const pValue = 2 * (1 - normalCDF(Math.abs(z)));
  const confidence = 1 - pValue;
  return { confidence, isSignificant: confidence >= 0.95, pValue };
}
```

**Testing**:
- `Unit: allocateTraffic returns same variant for same visitorId (deterministic)`
- `Unit: allocateTraffic with 50/50 weights → approximately equal distribution over 10000 visitors`
- `Unit: allocateTraffic with trafficPercentage=50 → ~50% of visitors get null (not in experiment)`
- `Unit: allocateTrafficBandit favours variant with higher conversion rate`
- `Unit: calculateSignificance with equal rates → not significant`
- `Unit: calculateSignificance with 5% vs 8% rate (n=1000 each) → significant at p<0.05`
- `Unit: calculateSignificance with small sample (n=10) → not significant`
- `Integration: published page with running experiment → different visitors see different variants`

#### 5.3 — Experiment Dashboard UI

**What**: Build the experiment management interface with results visualisation.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/experiments/page.tsx
// List view: experiment name, page name, status badge, variant count, duration, winner
// Filter by status (all, running, completed)

// apps/web/src/app/(dashboard)/experiments/[experimentId]/page.tsx
// Detail view:
// 1. Header: experiment name, status, start/pause/complete controls
// 2. Variant comparison table:
//    | Variant | Visitors | Conversions | Rate | Confidence | Status |
//    | Control | 1,234    | 56          | 4.5% | —          | —      |
//    | Var A   | 1,198    | 72          | 6.0% | 94.2%      | ⏳     |
//    | Var B   | 1,210    | 81          | 6.7% | 98.1%      | ✅     |
// 3. Time-series chart: conversion rate per variant per day
// 4. Variant preview: click variant row → shows rendered page preview
// 5. "Declare Winner" button (enabled when any variant is significant)
```

**Testing**:
- `E2E: create experiment from page editor → experiment appears in experiment list`
- `E2E: start experiment → status changes to "running" → results table shows`
- `E2E: declare winner → page content updated to winner variant → experiment status "completed"`
- `Unit: results table shows correct conversion rate calculation`
- `Unit: confidence badge shows correct colour (red <90%, yellow 90-95%, green >95%)`

---

## Phase 6: AI Copy Generation & Prompt-to-Page

### Purpose

Integrate LLM providers (Anthropic Claude, OpenAI) for AI copy generation and the flagship prompt-to-page feature. After this phase, users can generate complete landing pages from a text description, generate AI copy variants for individual elements, and receive AI-powered copy suggestions. This is the core AI differentiator.

### Tasks

#### 6.1 — AI Provider Abstraction Layer

**What**: Build a pluggable AI provider interface supporting Anthropic and OpenAI with streaming, token tracking, and error handling.

**Design**:

```typescript
// packages/core/src/ai/providers.ts
export interface AiProvider {
  name: "anthropic" | "openai";
  generateText(params: TextGenerationParams): Promise<TextGenerationResult>;
  generateStructured<T>(params: StructuredGenerationParams<T>): Promise<T>;
  streamText(params: TextGenerationParams): AsyncIterable<string>;
}

export interface TextGenerationParams {
  systemPrompt: string;
  userPrompt: string;
  maxTokens: number;
  temperature: number;
  model?: string;
}

export interface TextGenerationResult {
  text: string;
  inputTokens: number;
  outputTokens: number;
  modelName: string;
  latencyMs: number;
}

export interface StructuredGenerationParams<T> {
  systemPrompt: string;
  userPrompt: string;
  schema: z.ZodType<T>;
  maxTokens: number;
  temperature: number;
}

// packages/core/src/ai/anthropic-provider.ts
export class AnthropicProvider implements AiProvider {
  name = "anthropic" as const;
  private client: Anthropic;

  async generateStructured<T>(params: StructuredGenerationParams<T>): Promise<T> {
    // Uses Claude's tool_use for structured output
    // Validates result against Zod schema
    // Retries once on validation failure with error feedback
  }
}

// packages/core/src/ai/openai-provider.ts
export class OpenAIProvider implements AiProvider {
  name = "openai" as const;
  private client: OpenAI;

  async generateStructured<T>(params: StructuredGenerationParams<T>): Promise<T> {
    // Uses GPT-4o structured output (response_format: { type: "json_schema" })
  }
}

// Factory function
export function createAiProvider(config: AiProviderConfig): AiProvider {
  switch (config.provider) {
    case "anthropic": return new AnthropicProvider(config.apiKey);
    case "openai": return new OpenAIProvider(config.apiKey);
  }
}
```

**Testing**:
- `Unit: AnthropicProvider.generateText returns result with token counts`
- `Unit: OpenAIProvider.generateText returns result with token counts`
- `Integration (mocked): generateStructured with page content schema → returns valid PageContent`
- `Integration (mocked): generateStructured with invalid response → retries once with error context`
- `Unit: createAiProvider returns correct provider type based on config`
- `Unit: streamText yields incremental text chunks`

#### 6.2 — Prompt-to-Page Generation

**What**: Implement the flagship feature: generate a complete landing page from a text description of the offer and audience.

**Design**:

```typescript
// packages/core/src/ai/prompt-to-page.ts
export interface PromptToPageInput {
  description: string;           // "A SaaS tool that helps dentists manage appointments..."
  targetAudience?: string;       // "Independent dental practices with 2-10 dentists"
  conversionGoal?: string;       // "free_trial_signup" | "demo_request" | "lead_gen"
  industry?: string;             // "healthcare" | "saas" | "ecommerce"
  tone?: string;                 // "professional" | "friendly" | "urgent" | "authoritative"
  colorScheme?: {
    primary: string;             // "#2563eb"
    secondary: string;           // "#10b981"
  };
}

export async function generatePage(
  input: PromptToPageInput,
  provider: AiProvider
): Promise<PageContent> {
  const systemPrompt = `You are an expert landing page copywriter and conversion rate optimisation specialist.
Generate a complete, conversion-optimised landing page as a JSON structure.

The page should include these sections in order:
1. Hero: compelling headline (5-9 words), subheadline explaining the value proposition, primary CTA button, optional hero image placeholder
2. Social Proof: trust badges, customer count, or key metric
3. Features/Benefits: 3-4 key benefits with icons and descriptions (benefit-driven, not feature-driven)
4. How It Works: 3 simple steps explaining the process
5. Testimonials: 2-3 customer testimonials with names and roles
6. FAQ: 3-5 common questions and answers
7. Final CTA: repeated call to action with urgency element

Copy guidelines:
- Write at a 5th-7th grade reading level (shorter sentences, simpler words)
- Use benefit-driven copy (not feature-driven)
- Include specific numbers where possible
- Primary CTA should use action verbs ("Start", "Get", "Try")
- Headlines should create curiosity or promise a transformation

Output must conform to the PageContent JSON schema provided.`;

  const userPrompt = `Create a landing page for:
${input.description}

Target audience: ${input.targetAudience || "General"}
Conversion goal: ${input.conversionGoal || "lead_gen"}
Industry: ${input.industry || "general"}
Tone: ${input.tone || "professional"}
Primary color: ${input.colorScheme?.primary || "#2563eb"}
Secondary color: ${input.colorScheme?.secondary || "#10b981"}`;

  return provider.generateStructured<PageContent>({
    systemPrompt,
    userPrompt,
    schema: pageContentSchema,
    maxTokens: 8192,
    temperature: 0.7,
  });
}
```

```typescript
// packages/api/src/routers/ai.ts
export const aiRouter = router({
  generatePage: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      prompt: z.string().min(10).max(2000),
      targetAudience: z.string().max(500).optional(),
      conversionGoal: z.enum(["free_trial_signup", "demo_request", "lead_gen", "purchase", "download"]).optional(),
      industry: z.string().max(100).optional(),
      tone: z.enum(["professional", "friendly", "urgent", "authoritative", "playful"]).optional(),
      colorScheme: z.object({
        primary: z.string().regex(/^#[0-9a-fA-F]{6}$/),
        secondary: z.string().regex(/^#[0-9a-fA-F]{6}$/),
      }).optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      // 1. Create ai_generation_jobs record (status: "pending")
      // 2. Enqueue BullMQ job for async generation
      // Returns: { jobId: string }
    }),

  getJobStatus: protectedProcedure
    .input(z.object({ jobId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns: AiGenerationJob (with result if completed)
    }),

  acceptGeneration: protectedProcedure
    .input(z.object({
      jobId: z.string().uuid(),
      pageId: z.string().uuid().optional(), // if applying to existing page
    }))
    .mutation(async ({ input, ctx }) => {
      // If pageId provided: replace page content with generated content
      // If no pageId: create new page with generated content
      // Returns: { pageId: string }
    }),
});
```

**Testing**:
- `Integration (mocked LLM): generatePage with valid input → returns PageContent with hero, features, CTA sections`
- `Integration (mocked LLM): generated content passes pageContentSchema validation`
- `Unit: generated page includes required section types (hero, cta at minimum)`
- `Unit: generatePage job status transitions: pending → processing → completed`
- `Unit: acceptGeneration creates new page with generated content when no pageId`
- `Unit: acceptGeneration replaces existing page content when pageId provided`
- `E2E: enter prompt in generation dialog → loading state → preview generated page → accept → redirected to editor with generated content`
- `Unit: prompt under 10 characters → validation error`

#### 6.3 — AI Copy Generation for Individual Elements

**What**: Add AI copy generation for individual page elements (headlines, body copy, CTAs) with multiple copy angle variants.

**Design**:

```typescript
// packages/core/src/ai/copy-generator.ts
export interface CopyGenerationInput {
  elementType: "heading" | "paragraph" | "button";
  currentContent: string;
  pageContext: {
    pageName: string;
    industry?: string;
    conversionGoal?: string;
    surroundingContent: string; // text from adjacent elements for context
  };
  angles: ("benefit_driven" | "fomo" | "authority" | "social_proof" | "curiosity")[];
  count: number; // number of variants per angle
}

export interface CopyVariant {
  text: string;
  angle: string;
  confidence: number; // 0-1, model's self-assessed quality
}

export async function generateCopyVariants(
  input: CopyGenerationInput,
  provider: AiProvider
): Promise<CopyVariant[]> {
  // System prompt instructs the model to generate copy variants
  // Each variant tagged with copy angle and confidence score
  // Returns array of CopyVariant objects
}
```

```typescript
// UI integration:
// When user selects a heading/paragraph/button element, the property panel shows:
// "✨ Generate AI Copy" button
// Clicking opens a popover:
//   - Copy angle checkboxes (benefit, FOMO, authority, social proof, curiosity)
//   - "Generate" button
//   - Results: list of copy variants with "Apply" button on each
//   - Applying a variant updates the element content and records ai_copy_variants
```

**Testing**:
- `Integration (mocked LLM): generateCopyVariants for heading → returns 5 variants with different angles`
- `Unit: each variant has non-empty text, valid angle, confidence 0-1`
- `Unit: copy variants are distinct (no duplicates)`
- `E2E: select heading → click "Generate AI Copy" → select angles → generate → variants appear → click "Apply" → heading text updates`
- `Integration: accepted copy variant recorded in ai_copy_variants table with selected=true`

#### 6.4 — AI Generation Worker & Job Queue

**What**: Implement the BullMQ background worker that processes AI generation jobs.

**Design**:

```typescript
// workers/ai-generation.ts
import { Worker } from "bullmq";

const worker = new Worker("ai-generation", async (job) => {
  const { jobId, jobType, input, provider } = job.data;

  // Update job status to "processing"
  await db.update(aiGenerationJobs).set({ status: "processing", startedAt: new Date() }).where(eq(id, jobId));

  try {
    let result: unknown;
    switch (jobType) {
      case "prompt_to_page":
        result = await generatePage(input, createAiProvider(provider));
        break;
      case "copy_generation":
        result = await generateCopyVariants(input, createAiProvider(provider));
        break;
      case "layout_suggestion":
        result = await suggestLayout(input, createAiProvider(provider));
        break;
    }

    await db.update(aiGenerationJobs).set({
      status: "completed",
      result,
      completedAt: new Date(),
      inputTokens: result.inputTokens,
      outputTokens: result.outputTokens,
    }).where(eq(id, jobId));
  } catch (error) {
    await db.update(aiGenerationJobs).set({
      status: "failed",
      errorMessage: error.message,
      completedAt: new Date(),
    }).where(eq(id, jobId));
    throw error; // BullMQ retry handling
  }
}, {
  connection: redisConnection,
  concurrency: 5,
  limiter: { max: 10, duration: 60000 }, // max 10 jobs per minute per queue
});
```

**Testing**:
- `Integration: enqueue job → worker picks up → status transitions pending → processing → completed`
- `Integration: AI provider error → job status "failed" with error message`
- `Unit: worker concurrency limited to 5 simultaneous jobs`
- `Unit: rate limiter enforces 10 jobs per minute`
- `Integration: job result stored in ai_generation_jobs.result JSONB`

---

## Phase 7: Dynamic Text Replacement & Personalisation

### Purpose

Implement Dynamic Text Replacement (DTR) and rule-based visitor personalisation. After this phase, landing pages can dynamically adjust their content based on URL parameters (utm_source, keyword), visitor attributes (device, location, referrer), and configurable rules. This enables marketers to create a single page that adapts to different ad campaigns and audience segments.

### Tasks

#### 7.1 — DTR Engine

**What**: Implement the Dynamic Text Replacement engine that substitutes element content based on URL parameters.

**Design**:

```typescript
// packages/core/src/personalization/dtr-engine.ts
export interface DtrRule {
  parameter: string;     // URL parameter name (e.g., "keyword", "utm_term")
  elementId: string;     // Target element ID in the page content tree
  defaultText: string;   // Fallback when parameter is absent
}

export interface DtrContext {
  urlParams: Record<string, string>;
}

/**
 * Apply DTR rules to page content.
 * For each rule, if the URL parameter is present, replace the target element's content.
 * If absent, use the defaultText.
 * Sanitise replacement text to prevent XSS (strip HTML tags, encode entities).
 */
export function applyDtr(
  content: PageContent,
  rules: DtrRule[],
  context: DtrContext
): PageContent {
  const result = structuredClone(content);
  for (const rule of rules) {
    const replacement = context.urlParams[rule.parameter] || rule.defaultText;
    const sanitised = sanitiseText(replacement);
    // Walk sections → elements to find matching elementId and replace content
    replaceElementContent(result, rule.elementId, sanitised);
  }
  return result;
}

function sanitiseText(text: string): string {
  // Strip HTML tags, encode &, <, >, ", '
  return text.replace(/[<>&"']/g, (c) => `&#${c.charCodeAt(0)};`);
}
```

**Testing**:
- `Unit: applyDtr with matching URL parameter → element content replaced`
- `Unit: applyDtr with missing parameter → element uses defaultText`
- `Unit: applyDtr with HTML in parameter → tags stripped (XSS prevention)`
- `Unit: applyDtr with multiple rules → all matching elements replaced`
- `Unit: applyDtr with non-existent elementId → no error, content unchanged`
- `Integration: published page with ?keyword=dentist → headline shows "dentist"`

#### 7.2 — Rule-Based Personalisation Engine

**What**: Implement the personalisation rule engine that conditionally replaces page content based on visitor attributes.

**Design**:

```typescript
// packages/core/src/personalization/rule-engine.ts
export interface PersonalizationRule {
  id: string;
  name: string;
  priority: number;
  active: boolean;
  conditions: PersonalizationCondition[];
  replacements: PersonalizationReplacement[];
}

export interface PersonalizationCondition {
  type: "utm_source" | "utm_medium" | "utm_campaign" | "referrer" | "device" | "country" | "keyword";
  operator: "equals" | "contains" | "starts_with" | "regex" | "in";
  value: string;
}

export interface PersonalizationReplacement {
  elementId: string;
  field: "content" | "src" | "href" | "alt";
  value: string;
}

export interface VisitorContext {
  utmSource?: string;
  utmMedium?: string;
  utmCampaign?: string;
  referrer?: string;
  device: "desktop" | "tablet" | "mobile";
  country?: string;     // ISO 3166-1 alpha-2 from IP geolocation
  keyword?: string;
}

/**
 * Evaluate personalisation rules against visitor context.
 * Rules are sorted by priority (highest first).
 * First matching rule wins (no combining).
 */
export function evaluateRules(
  rules: PersonalizationRule[],
  context: VisitorContext
): PersonalizationReplacement[] | null {
  const activeRules = rules
    .filter((r) => r.active)
    .sort((a, b) => b.priority - a.priority);

  for (const rule of activeRules) {
    if (allConditionsMatch(rule.conditions, context)) {
      return rule.replacements;
    }
  }
  return null;
}

function allConditionsMatch(
  conditions: PersonalizationCondition[],
  context: VisitorContext
): boolean {
  return conditions.every((cond) => {
    const actualValue = getContextValue(cond.type, context);
    if (!actualValue) return false;
    switch (cond.operator) {
      case "equals": return actualValue === cond.value;
      case "contains": return actualValue.includes(cond.value);
      case "starts_with": return actualValue.startsWith(cond.value);
      case "regex": return new RegExp(cond.value, "i").test(actualValue);
      case "in": return cond.value.split(",").map(s => s.trim()).includes(actualValue);
    }
  });
}
```

**Testing**:
- `Unit: rule with utm_source equals "google" matches when context.utmSource is "google"`
- `Unit: rule with utm_source equals "google" does NOT match when utmSource is "facebook"`
- `Unit: rule with referrer contains "linkedin" matches "https://www.linkedin.com/feed"`
- `Unit: rule with regex operator validates correctly`
- `Unit: higher priority rule wins when multiple rules match`
- `Unit: inactive rules are skipped`
- `Unit: no matching rules → returns null`
- `Integration: published page with utm_source=google → personalised content rendered`

#### 7.3 — Personalisation Rule Editor UI

**What**: Build the UI for creating and managing personalisation rules within the page editor.

**Design**:

```typescript
// apps/web/src/components/editor/PersonalizationPanel.tsx
// Accessible from editor sidebar tab "Personalisation"
// Shows:
// 1. DTR rules list: parameter name ↔ element (click element to highlight on canvas)
// 2. Personalisation rules list with priority, conditions summary, active toggle
// 3. "Add Rule" dialog:
//    - Rule name
//    - Conditions builder (AND logic):
//      [utm_source] [equals] [google]  [+ Add condition]
//    - Replacements: select element on canvas → specify replacement text/src/href
//    - Priority number
// 4. Preview toggle: "Preview as Google traffic" dropdown showing visitor context presets
```

**Testing**:
- `E2E: add DTR rule linking URL parameter "keyword" to headline → preview with ?keyword=test shows replacement`
- `E2E: add personalisation rule for utm_source=facebook → preview as Facebook traffic shows personalised content`
- `Unit: condition builder validates operator compatibility with condition type`
- `Unit: replacement picker highlights the target element on canvas`

---

## Phase 8: Analytics, Heatmaps & Performance Monitoring

### Purpose

Implement visitor analytics (page views, conversions, UTM tracking), heatmap data collection, scroll depth tracking, and Core Web Vitals monitoring. After this phase, workspace users have a full analytics dashboard showing page performance, visitor behaviour, and Core Web Vitals compliance.

### Tasks

#### 8.1 — Analytics Event Collection

**What**: Build the client-side analytics SDK embedded in published pages and the server-side event ingestion endpoint.

**Design**:

```typescript
// packages/core/src/analytics/tracking-snippet.ts
// Generates a lightweight (<2KB gzipped) JavaScript snippet injected into published pages
// Collects:
// - Page view (on load)
// - Conversion events (form submit, button click with [data-track-conversion])
// - Scroll depth (25%, 50%, 75%, 100% milestones)
// - Time on page (on beforeunload)
// - Core Web Vitals (via web-vitals library, lazy-loaded)
// Sends events via navigator.sendBeacon to /api/analytics/collect

export function generateTrackingSnippet(config: {
  pageId: string;
  variantId?: string;
  consentRequired: boolean;
}): string {
  return `<script>
(function(){
  var c=${JSON.stringify(config)};
  var vid=localStorage.getItem('_lb_vid')||crypto.randomUUID();
  localStorage.setItem('_lb_vid',vid);
  var sid=sessionStorage.getItem('_lb_sid')||crypto.randomUUID();
  sessionStorage.setItem('_lb_sid',sid);

  function send(type,data){
    navigator.sendBeacon('/api/analytics/collect',JSON.stringify({
      pageId:c.pageId,variantId:c.variantId,visitorId:vid,sessionId:sid,
      type:type,data:data,ts:Date.now()
    }));
  }

  send('page_view',{referrer:document.referrer,url:location.href});

  // Scroll depth tracking
  var scrollMilestones=[25,50,75,100],scrollFired={};
  document.addEventListener('scroll',function(){
    var pct=Math.round((window.scrollY+window.innerHeight)/document.body.scrollHeight*100);
    scrollMilestones.forEach(function(m){if(pct>=m&&!scrollFired[m]){scrollFired[m]=1;send('scroll_depth',{depth:m});}});
  },{passive:true});

  // Time on page
  var startTime=Date.now();
  window.addEventListener('beforeunload',function(){send('time_on_page',{ms:Date.now()-startTime});});

  // Core Web Vitals (lazy load)
  import('/api/analytics/web-vitals.js').then(function(m){
    m.onLCP(function(v){send('cwv',{metric:'lcp',value:v.value});});
    m.onCLS(function(v){send('cwv',{metric:'cls',value:v.value});});
    m.onINP(function(v){send('cwv',{metric:'inp',value:v.value});});
  });
})();
</script>`;
}
```

```typescript
// apps/web/src/app/api/analytics/collect/route.ts
// POST /api/analytics/collect
// High-throughput endpoint (no auth, public)
// Rate limited: 100 events per visitor per minute
// Validates event schema, inserts into page_views / conversion_events / performance_scores
// Batch inserts via pg COPY or multi-row INSERT for performance
```

**Testing**:
- `Unit: tracking snippet generates valid JavaScript`
- `Unit: snippet size < 2KB uncompressed`
- `Integration: tracking snippet fires page_view on load → event stored in page_views`
- `Integration: scroll to 50% → scroll_depth event stored in conversion_events`
- `Unit: rate limiter blocks excessive events from single visitor`
- `Unit: consent-required mode delays tracking until consent granted`

#### 8.2 — Heatmap Data Collection

**What**: Collect click and mouse movement data for heatmap visualisation.

**Design**:

```typescript
// Heatmap tracking added to analytics snippet:
// - Click events: x, y, viewport dimensions, target element selector
// - Throttled to max 2 events/second to limit data volume
// - Data stored in conversion_events with event_type = "click" and metadata JSONB

// packages/core/src/analytics/heatmap-aggregator.ts
export interface HeatmapCell {
  x: number;           // 0-100 (percentage of viewport width)
  y: number;           // pixel offset from top of page
  count: number;       // click count in this cell
  intensity: number;   // 0-1 normalised intensity
}

export async function generateHeatmap(
  pageId: string,
  variantId: string | null,
  dateRange: { from: Date; to: Date }
): Promise<HeatmapCell[]> {
  // Query conversion_events where event_type = 'click'
  // Bucket x into 100 columns, y into 50px rows
  // Normalise counts to 0-1 intensity
  // Returns grid of HeatmapCell objects for rendering
}
```

**Testing**:
- `Unit: click events captured with correct x/y/viewport dimensions`
- `Unit: event throttling limits to 2 events per second`
- `Unit: generateHeatmap aggregates clicks into correct grid cells`
- `Unit: intensity normalisation maps max clicks to 1.0`
- `Integration: multiple clicks on page → heatmap shows concentrated areas`

#### 8.3 — Analytics Dashboard

**What**: Build the analytics dashboard with page performance metrics, visitor breakdowns, and conversion funnels.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/analytics/page.tsx
// Dashboard showing:
// 1. Top metrics cards: Total visitors, Unique visitors, Conversions, Conversion rate
// 2. Date range picker (7d, 30d, 90d, custom)
// 3. Time-series chart: visitors and conversions over time
// 4. Traffic sources breakdown (pie chart): utm_source
// 5. Device breakdown: desktop / tablet / mobile
// 6. Geographic breakdown: top countries (world map or table)
// 7. Core Web Vitals status: LCP / CLS / INP with pass/fail badges
// 8. Heatmap overlay view (toggle on specific page)

// packages/api/src/routers/analytics.ts
export const analyticsRouter = router({
  overview: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      pageId: z.string().uuid().optional(),
      dateFrom: z.date(),
      dateTo: z.date(),
    }))
    .query(async ({ input, ctx }) => {
      // Aggregated metrics from page_views and conversion_events
      // Returns: AnalyticsOverview
    }),

  timeSeries: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      pageId: z.string().uuid().optional(),
      dateFrom: z.date(),
      dateTo: z.date(),
      granularity: z.enum(["hourly", "daily", "weekly"]),
    }))
    .query(async ({ input, ctx }) => {
      // Time-bucketed visitor and conversion counts
      // Returns: TimeSeriesPoint[]
    }),

  heatmap: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      variantId: z.string().uuid().optional(),
      dateFrom: z.date(),
      dateTo: z.date(),
    }))
    .query(async ({ input, ctx }) => {
      // Returns: HeatmapCell[]
    }),

  coreWebVitals: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Latest CWV scores with pass/fail per metric
      // Returns: { lcp: MetricResult, cls: MetricResult, inp: MetricResult, passes: boolean }
    }),
});
```

**Testing**:
- `Integration: page with 100 views and 5 conversions → overview shows 5% conversion rate`
- `Unit: time series with daily granularity groups events by calendar date`
- `Unit: traffic source breakdown correctly groups by utm_source`
- `E2E: analytics page shows charts, metrics update on date range change`
- `Unit: CWV pass/fail thresholds: LCP<=2500ms, CLS<=0.1, INP<=200ms`

#### 8.4 — Automated Accessibility Auditing

**What**: Integrate axe-core for real-time WCAG 2.2 accessibility checking during page editing and automated audits on publish.

**Design**:

```typescript
// packages/core/src/accessibility/auditor.ts
import { AxeResults } from "axe-core";

export interface AccessibilityAuditResult {
  score: number;        // 0-100
  wcagLevel: "A" | "AA" | "AAA";
  totalViolations: number;
  totalPasses: number;
  issues: AccessibilityIssue[];
}

export interface AccessibilityIssue {
  ruleId: string;           // e.g., "color-contrast"
  impact: "critical" | "serious" | "moderate" | "minor";
  wcagCriteria: string[];   // e.g., ["1.4.3"]
  elementSelector: string;  // CSS selector
  description: string;
  autoFixable: boolean;     // can be fixed programmatically
  suggestedFix?: string;    // e.g., "Change color to #1a1a1a for 4.5:1 contrast ratio"
}

// Auto-fix capabilities:
// - Missing alt text: generate via AI (or set alt="" for decorative images)
// - Insufficient color contrast: adjust text color to meet 4.5:1 ratio
// - Missing form labels: associate label with field
// - Missing lang attribute: set from workspace settings
// - Heading hierarchy gaps: warn (cannot auto-fix without changing semantics)

export async function auditPage(html: string): Promise<AccessibilityAuditResult> {
  // Run axe-core on rendered HTML using jsdom or Playwright
  // Map violations to AccessibilityIssue format
  // Calculate score as (passes / (passes + violations)) * 100
}

export function autoFix(issue: AccessibilityIssue, content: PageContent): PageContent | null {
  // Attempt to fix the issue in the content tree
  // Returns modified content or null if not auto-fixable
}
```

**Testing**:
- `Unit: page with missing alt text → violation with ruleId "image-alt"`
- `Unit: page with low contrast text → violation with impact "serious"`
- `Unit: autoFix for missing alt text → adds alt attribute to image element`
- `Unit: autoFix for low contrast → adjusts colour to meet 4.5:1 ratio`
- `Integration: publish page → accessibility audit runs → results stored in accessibility_audits table`
- `E2E: editor shows accessibility warnings in real-time as user edits`
- `Fixture: audit fixture "accessible-page.html" → score 100, 0 violations`
- `Fixture: audit fixture "inaccessible-page.html" → score <80, violations found`

---

## Phase 9: Integrations & Webhooks

### Purpose

Build the integration layer connecting the builder to external CRM, marketing automation, and analytics platforms. After this phase, leads are automatically forwarded to connected systems, and workspace events trigger outbound webhooks following the Standard Webhooks specification.

### Tasks

#### 9.1 — Webhook Delivery System

**What**: Implement outbound webhook delivery with Standard Webhooks-compliant signing, retry logic, and delivery monitoring.

**Design**:

```typescript
// packages/core/src/integrations/webhook-manager.ts
import { createHmac } from "crypto";

export interface WebhookEvent {
  type: string;         // "lead.created", "page.published", "experiment.completed"
  timestamp: string;    // ISO 8601
  data: unknown;        // event-specific payload
}

/**
 * Standard Webhooks signing (https://www.standardwebhooks.com/)
 * Header: webhook-id, webhook-timestamp, webhook-signature
 */
export function signWebhookPayload(
  payload: string,
  secret: string,
  messageId: string,
  timestamp: number
): string {
  const toSign = `${messageId}.${timestamp}.${payload}`;
  const signature = createHmac("sha256", Buffer.from(secret, "base64"))
    .update(toSign)
    .digest("base64");
  return `v1,${signature}`;
}

// Retry policy: exponential backoff with jitter
// Attempts: 1, 2, 4, 8, 16 minutes (5 attempts total)
// After 5 failures, mark as failed and stop retrying
export const RETRY_DELAYS = [0, 60000, 120000, 240000, 480000, 960000]; // ms
```

```typescript
// workers/webhook-delivery.ts
const worker = new Worker("webhook-delivery", async (job) => {
  const { deliveryId, endpointUrl, payload, secret } = job.data;

  const messageId = `msg_${deliveryId}`;
  const timestamp = Math.floor(Date.now() / 1000);
  const body = JSON.stringify(payload);
  const signature = signWebhookPayload(body, secret, messageId, timestamp);

  const response = await fetch(endpointUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "webhook-id": messageId,
      "webhook-timestamp": String(timestamp),
      "webhook-signature": signature,
    },
    body,
    signal: AbortSignal.timeout(30000), // 30s timeout
  });

  await db.update(webhookDeliveries).set({
    responseStatus: response.status,
    attempts: sql`attempts + 1`,
    deliveredAt: response.ok ? new Date() : null,
    nextRetryAt: response.ok ? null : calculateNextRetry(job.attemptsMade),
  }).where(eq(id, deliveryId));

  if (!response.ok) throw new Error(`Webhook delivery failed: ${response.status}`);
}, {
  connection: redisConnection,
  attempts: 5,
  backoff: { type: "exponential", delay: 60000 },
});
```

**Testing**:
- `Unit: signWebhookPayload produces valid HMAC-SHA256 signature`
- `Unit: signature verification matches signed payload`
- `Integration: webhook delivery to httpbin.org → 200 → deliveredAt set`
- `Integration (mocked): webhook delivery to failing endpoint → retry scheduled`
- `Unit: after 5 failed attempts → no more retries scheduled`
- `Unit: exponential backoff delays match RETRY_DELAYS`

#### 9.2 — Zapier & CRM Integration Framework

**What**: Build the integration framework for connecting workspaces to external platforms (Zapier, Salesforce, HubSpot, Mailchimp, Klaviyo).

**Design**:

```typescript
// packages/core/src/integrations/integration-manager.ts
export interface IntegrationProvider {
  name: string;
  authenticate(credentials: unknown): Promise<IntegrationConnection>;
  sendLead(connection: IntegrationConnection, lead: Lead): Promise<void>;
  testConnection(connection: IntegrationConnection): Promise<boolean>;
}

// packages/core/src/integrations/providers/zapier.ts
export class ZapierProvider implements IntegrationProvider {
  name = "zapier";

  async sendLead(connection: IntegrationConnection, lead: Lead): Promise<void> {
    // POST lead data to Zapier webhook URL stored in connection.config.webhookUrl
    // Payload: flattened lead data (email, all form fields, UTM params, page name, timestamp)
  }
}

// packages/core/src/integrations/providers/hubspot.ts
export class HubSpotProvider implements IntegrationProvider {
  name = "hubspot";

  async authenticate(credentials: { apiKey: string }): Promise<IntegrationConnection> {
    // Verify API key via HubSpot API
    // Returns connection with encrypted credentials
  }

  async sendLead(connection: IntegrationConnection, lead: Lead): Promise<void> {
    // POST to HubSpot Contacts API: create or update contact
    // Map form fields to HubSpot contact properties
  }
}

// packages/api/src/routers/integrations.ts
export const integrationRouter = router({
  list: protectedProcedure
    .input(z.object({ workspaceId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Returns: Integration[] with status, last synced
    }),

  connect: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid(),
      provider: z.enum(["zapier", "hubspot", "salesforce", "mailchimp", "klaviyo"]),
      credentials: z.record(z.string()),
    }))
    .mutation(async ({ input, ctx }) => {
      // Authenticate with provider
      // Store encrypted credentials
      // Returns: Integration
    }),

  test: protectedProcedure
    .input(z.object({ integrationId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Test connection to provider
      // Returns: { success: boolean, message: string }
    }),

  disconnect: protectedProcedure
    .input(z.object({ integrationId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      // Remove credentials, set status = "disconnected"
    }),
});
```

**Testing**:
- `Integration (mocked): Zapier provider sends lead data to webhook URL → 200`
- `Integration (mocked): HubSpot provider creates contact via API`
- `Unit: credentials are encrypted before storage (not stored in plaintext)`
- `Unit: test connection verifies API key validity`
- `E2E: connect Zapier integration → submit form on published page → Zapier receives lead data`
- `Unit: disconnect removes encrypted credentials from database`

---

## Phase 10: Predictive Scoring & Smart Traffic

### Purpose

Implement the advanced AI features: predictive conversion scoring (estimate page performance before launch) and smart traffic routing (bandit-algorithm multivariate optimisation). These are the features that distinguish the product from competitors and represent the strongest AI-native advantage.

### Tasks

#### 10.1 — Predictive Conversion Scoring

**What**: Build an ML-powered system that estimates a page's expected conversion rate before launch, based on page structure, copy quality, and historical patterns from cross-workspace data.

**Design**:

```typescript
// packages/core/src/ai/predictive-scorer.ts
export interface PredictiveScoreInput {
  content: PageContent;
  industry?: string;
  conversionGoal?: string;
}

export interface PredictiveScore {
  estimatedConversionRate: number;     // e.g., 0.043
  confidenceIntervalLow: number;      // e.g., 0.031
  confidenceIntervalHigh: number;     // e.g., 0.056
  riskLevel: "low" | "medium" | "high";
  recommendations: Recommendation[];
}

export interface Recommendation {
  category: "copy" | "layout" | "cta" | "form" | "social_proof" | "performance";
  severity: "info" | "warning" | "critical";
  message: string;
  elementId?: string;
  suggestedAction?: string;
}

/**
 * Score a page by analysing:
 * 1. Structural signals: section count, element density, form field count, CTA count
 * 2. Copy signals: reading level (Flesch-Kincaid), headline word count, CTA verb strength
 * 3. Layout signals: hero section presence, social proof placement, CTA above/below fold
 * 4. Historical patterns: cross-workspace averages for similar page structures
 *
 * Uses LLM for copy quality analysis; rule-based heuristics for structural/layout signals.
 */
export async function scorePage(
  input: PredictiveScoreInput,
  provider: AiProvider,
  historicalData: HistoricalAverages
): Promise<PredictiveScore> {
  const structuralScore = analyseStructure(input.content);
  const copyScore = await analyseCopy(input.content, provider);
  const layoutScore = analyseLayout(input.content);
  const historicalBaseline = historicalData.getBaseline(input.industry, input.conversionGoal);

  // Weighted combination
  const estimatedRate = (
    structuralScore.score * 0.2 +
    copyScore.score * 0.35 +
    layoutScore.score * 0.25 +
    historicalBaseline * 0.2
  );

  const recommendations = [
    ...structuralScore.recommendations,
    ...copyScore.recommendations,
    ...layoutScore.recommendations,
  ];

  return {
    estimatedConversionRate: estimatedRate,
    confidenceIntervalLow: estimatedRate * 0.72,
    confidenceIntervalHigh: estimatedRate * 1.30,
    riskLevel: estimatedRate < 0.02 ? "high" : estimatedRate < 0.04 ? "medium" : "low",
    recommendations: recommendations.sort((a, b) =>
      severityOrder[b.severity] - severityOrder[a.severity]
    ),
  };
}
```

```typescript
// Structural heuristic examples:
function analyseStructure(content: PageContent): ScoringResult {
  const recommendations: Recommendation[] = [];

  // Check: does page have a hero section?
  if (!content.sections.find(s => s.type === "hero")) {
    recommendations.push({
      category: "layout", severity: "warning",
      message: "No hero section found. Pages with hero sections convert 23% higher on average.",
    });
  }

  // Check: social proof section placement
  const socialProofIndex = content.sections.findIndex(s =>
    s.type === "testimonials" || s.name?.toLowerCase().includes("social proof")
  );
  if (socialProofIndex > 3) {
    recommendations.push({
      category: "social_proof", severity: "info",
      message: "Social proof section is below the fold. Consider moving it higher for 12% average lift.",
    });
  }

  // Check: form field count
  const formElements = findElements(content, "form");
  for (const form of formElements) {
    if (form.config && form.config.fields.length > 5) {
      recommendations.push({
        category: "form", severity: "warning",
        message: `Form has ${form.config.fields.length} fields. Forms with 3 or fewer fields convert 25% higher.`,
        elementId: form.id,
        suggestedAction: "Reduce to name + email + one qualifier field",
      });
    }
  }

  return { score: calculateStructuralScore(content), recommendations };
}
```

**Testing**:
- `Unit: page with hero + social proof + CTA → high structural score`
- `Unit: page with no hero → lower score + recommendation to add hero`
- `Unit: page with 8-field form → recommendation to reduce fields`
- `Unit: copy analysis flags college-level readability → recommends simplification`
- `Integration (mocked LLM): scorePage returns valid PredictiveScore with recommendations`
- `Unit: risk level classification: <2% → high, 2-4% → medium, >4% → low`
- `E2E: click "Score Page" button in editor → score card shows estimated rate + recommendations`

#### 10.2 — Smart Traffic Routing (Thompson Sampling)

**What**: Upgrade the bandit experiment type to a fully automated smart traffic system that continuously reallocates traffic to higher-performing variants.

**Design**:

```typescript
// packages/core/src/experiments/smart-traffic.ts

/**
 * Thompson Sampling implementation for smart traffic routing.
 * Unlike A/B testing which waits for statistical significance,
 * Thompson Sampling continuously shifts traffic toward winning variants
 * while maintaining exploration of undersampled variants.
 */
export class SmartTrafficRouter {
  /**
   * Select variant for the next visitor using Thompson Sampling.
   * Samples from Beta(alpha, beta) for each variant where:
   *   alpha = conversions + 1
   *   beta = visitors - conversions + 1
   */
  selectVariant(variants: VariantResults[]): string {
    const samples = variants.map((v) => ({
      variantId: v.id,
      sample: betaSample(v.conversions + 1, v.visitors - v.conversions + 1),
    }));
    return samples.sort((a, b) => b.sample - a.sample)[0].variantId;
  }

  /**
   * Calculate the probability that each variant is the best.
   * Uses 10,000 Monte Carlo simulations.
   */
  calculateWinProbabilities(variants: VariantResults[]): Record<string, number> {
    const wins: Record<string, number> = {};
    variants.forEach(v => wins[v.id] = 0);

    for (let i = 0; i < 10000; i++) {
      const samples = variants.map(v => ({
        id: v.id,
        sample: betaSample(v.conversions + 1, v.visitors - v.conversions + 1),
      }));
      const winner = samples.sort((a, b) => b.sample - a.sample)[0];
      wins[winner.id]++;
    }

    const total = Object.values(wins).reduce((a, b) => a + b, 0);
    return Object.fromEntries(Object.entries(wins).map(([k, v]) => [k, v / total]));
  }

  /**
   * Auto-complete: declare winner when one variant has >95% win probability
   * and has at least 100 visitors.
   */
  shouldAutoComplete(variants: VariantResults[], threshold = 0.95): { complete: boolean; winnerId?: string } {
    const probs = this.calculateWinProbabilities(variants);
    for (const [id, prob] of Object.entries(probs)) {
      const variant = variants.find(v => v.id === id)!;
      if (prob >= threshold && variant.visitors >= 100) {
        return { complete: true, winnerId: id };
      }
    }
    return { complete: false };
  }
}

// Beta distribution sampling using Jondeau-Rockinger method
function betaSample(alpha: number, beta: number): number {
  const u = gammaSample(alpha);
  const v = gammaSample(beta);
  return u / (u + v);
}
```

**Testing**:
- `Unit: selectVariant with one high-converting variant → selects it most of the time`
- `Unit: selectVariant with equal conversion rates → roughly equal selection`
- `Unit: calculateWinProbabilities with dominant variant → probability > 0.9`
- `Unit: shouldAutoComplete with clear winner (>95% probability, >100 visitors) → true`
- `Unit: shouldAutoComplete with <100 visitors → false (insufficient data)`
- `Unit: Thompson Sampling naturally reduces traffic to underperforming variants over time`
- `Integration: smart traffic experiment running → visitors allocated proportionally to performance`

---

## Phase 11: Template Library & Vertical AI Training

### Purpose

Build the template library with industry-specific conversion-optimised templates and implement vertical-specific AI generation. After this phase, users can start from professional templates tailored to their industry (SaaS, E-commerce, Insurance, Real Estate), and AI generation produces industry-appropriate copy and layouts.

### Tasks

#### 11.1 — Template Management System

**What**: Build the template creation, categorisation, and selection system.

**Design**:

```typescript
// packages/api/src/routers/templates.ts
export const templateRouter = router({
  list: protectedProcedure
    .input(z.object({
      category: z.string().optional(),
      industry: z.string().optional(),
      conversionGoal: z.string().optional(),
      search: z.string().optional(),
      cursor: z.string().uuid().optional(),
      limit: z.number().default(12),
    }))
    .query(async ({ input, ctx }) => {
      // Filtered, paginated template listing
      // Returns: { templates: TemplateSummary[], nextCursor: string | null }
    }),

  get: protectedProcedure
    .input(z.object({ templateId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Full template with content for preview
      // Returns: Template
    }),

  createFromPage: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      name: z.string(),
      category: z.string(),
      industry: z.string().optional(),
      conversionGoal: z.string().optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      // Clone page content into a new template
      // Admin-only for platform templates; any user for workspace templates
      // Returns: Template
    }),
});
```

```typescript
// Template categories:
// - Lead Generation: optimised for form fills
// - Product Launch: focus on feature showcase + pre-order CTA
// - Webinar/Event: event details + registration form
// - App Download: mobile-first with app store badges
// - E-commerce: product gallery + buy CTA
// - SaaS Free Trial: benefit-driven with signup form
// - Quiz/Survey: multi-step interactive flow

// Industries: saas, ecommerce, insurance, real_estate, healthcare, finance, education, agency
```

**Testing**:
- `Unit: template list filters by category and industry correctly`
- `Unit: createFromPage clones page content into template (deep copy)`
- `E2E: browse templates → filter by industry → select template → page created with template content`
- `Integration: template usage count increments when used to create a page`
- `Unit: template content validates against pageContentSchema`

#### 11.2 — Vertical-Specific AI Prompt Engineering

**What**: Enhance prompt-to-page and copy generation with industry-specific prompt templates, terminology, and conversion patterns.

**Design**:

```typescript
// packages/core/src/ai/vertical-prompts.ts
export interface VerticalPromptConfig {
  industry: string;
  systemPromptAddendum: string;     // Industry-specific instructions
  sampleCopy: string[];             // Examples of high-converting copy in this vertical
  complianceNotes: string;          // Industry-specific compliance requirements
  commonSections: string[];         // Typical section types for this industry
  ctaPatterns: string[];            // Effective CTA patterns
}

export const verticalPrompts: Record<string, VerticalPromptConfig> = {
  insurance: {
    industry: "insurance",
    systemPromptAddendum: `You are writing for the insurance industry. Use reassuring, trust-building language.
Emphasise security, coverage, and peace of mind. Include compliance-safe language.
Never make specific claims about coverage amounts or pricing without disclaimers.
Common high-converting patterns: testimonial-first, comparison tables, "Get a free quote" CTAs.`,
    sampleCopy: [
      "Protect What Matters Most — Get Your Free Quote in 60 Seconds",
      "Compare Rates From Top Carriers. No Obligation.",
    ],
    complianceNotes: "Include disclaimer: 'Coverage and rates vary by state and carrier.'",
    commonSections: ["hero", "trust_badges", "coverage_comparison", "testimonials", "faq", "quote_form"],
    ctaPatterns: ["Get My Free Quote", "Compare Rates Now", "See My Options"],
  },
  saas: {
    industry: "saas",
    systemPromptAddendum: `You are writing for a SaaS product. Focus on solving pain points, saving time, and ROI.
Use specific metrics and social proof. Offer a free trial or demo as the primary CTA.
Emphasise ease of setup, integrations, and customer support.`,
    sampleCopy: [
      "Cut Your [Task] Time in Half — Start Your Free Trial",
      "Trusted by 10,000+ Teams. See Why They Switched.",
    ],
    complianceNotes: "",
    commonSections: ["hero", "social_proof", "features", "how_it_works", "pricing", "testimonials", "faq", "cta"],
    ctaPatterns: ["Start Free Trial", "Book a Demo", "Get Started Free"],
  },
  // ... ecommerce, real_estate, healthcare, finance, education
};
```

**Testing**:
- `Integration (mocked LLM): generate page with industry="insurance" → copy includes insurance terminology`
- `Unit: vertical prompt config includes compliance notes for regulated industries`
- `Unit: CTA patterns for each vertical are distinct and industry-appropriate`
- `Integration: generated insurance page includes disclaimer in footer`
- `Unit: unknown industry falls back to default prompt without error`

---

## Phase 12: MCP Server, API Keys & Developer Platform

### Purpose

Expose the landing page builder as a platform with a public REST API, API key management, and a Model Context Protocol (MCP) server that enables AI agents (Claude Desktop, Cursor, IDE assistants) to operate the builder programmatically. This is an underexplored differentiator in the landing page market.

### Tasks

#### 12.1 — Public REST API & API Key Authentication

**What**: Build a public REST API layer over the tRPC routers with API key authentication for programmatic access.

**Design**:

```typescript
// apps/web/src/app/api/v1/[...path]/route.ts
// Maps REST endpoints to tRPC router calls:
// GET    /api/v1/pages                    → pages.list
// POST   /api/v1/pages                    → pages.create
// GET    /api/v1/pages/:id                → pages.get
// PATCH  /api/v1/pages/:id                → pages.update
// POST   /api/v1/pages/:id/publish        → pages.publish
// POST   /api/v1/pages/:id/unpublish      → pages.unpublish
// GET    /api/v1/experiments              → experiments.list
// POST   /api/v1/experiments              → experiments.create
// GET    /api/v1/leads                    → leads.list
// POST   /api/v1/ai/generate-page        → ai.generatePage
// GET    /api/v1/ai/jobs/:id             → ai.getJobStatus

// Authentication: Bearer token (API key)
// Header: Authorization: Bearer lb_key_xxxxxxxxxxxx
// API keys scoped to workspace with permission scopes:
//   pages:read, pages:write, experiments:read, experiments:write,
//   leads:read, leads:write, ai:generate, analytics:read

// Rate limiting: 100 requests/minute for standard keys, 1000/minute for enterprise

// Response format follows JSON:API 1.1 conventions:
// {
//   "data": { "id": "...", "type": "page", "attributes": { ... } },
//   "meta": { "requestId": "...", "rateLimit": { "remaining": 99 } }
// }
```

```typescript
// packages/api/src/middleware/api-key-auth.ts
export async function authenticateApiKey(
  request: Request
): Promise<{ workspaceId: string; scopes: string[] } | null> {
  const authHeader = request.headers.get("Authorization");
  if (!authHeader?.startsWith("Bearer lb_key_")) return null;

  const keyPrefix = authHeader.slice(7, 15); // first 8 chars after "lb_key_"
  const keyHash = sha256(authHeader.slice(7));

  const apiKey = await db.query.apiKeys.findFirst({
    where: and(eq(apiKeys.keyHash, keyHash), isNull(apiKeys.expiresAt)),
  });

  if (!apiKey) return null;

  // Update last_used_at
  await db.update(apiKeys).set({ lastUsedAt: new Date() }).where(eq(apiKeys.id, apiKey.id));

  return { workspaceId: apiKey.workspaceId, scopes: apiKey.scopes };
}
```

**Testing**:
- `Integration: GET /api/v1/pages with valid API key → 200 with page list`
- `Integration: GET /api/v1/pages without API key → 401 Unauthorized`
- `Integration: GET /api/v1/pages with expired API key → 401`
- `Unit: API key with pages:read scope cannot POST to /api/v1/pages → 403`
- `Unit: rate limiter returns 429 after 100 requests in 1 minute`
- `Integration: POST /api/v1/ai/generate-page → 202 Accepted with jobId`

#### 12.2 — MCP Server

**What**: Build a Model Context Protocol server exposing the builder's tools and resources to AI agents.

**Design**:

```typescript
// packages/mcp-server/src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const server = new Server({
  name: "landing-page-builder",
  version: "1.0.0",
}, {
  capabilities: {
    tools: {},
    resources: {},
  },
});

// Tools exposed to AI agents:
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "create_page",
      description: "Create a new landing page in the workspace",
      inputSchema: { type: "object", properties: { name: { type: "string" }, prompt: { type: "string" } } },
    },
    {
      name: "update_section",
      description: "Update a section's content or elements in a page",
      inputSchema: { type: "object", properties: { pageId: { type: "string" }, sectionId: { type: "string" }, updates: { type: "object" } } },
    },
    {
      name: "generate_copy",
      description: "Generate AI copy variants for a page element",
      inputSchema: { type: "object", properties: { pageId: { type: "string" }, elementId: { type: "string" }, angles: { type: "array", items: { type: "string" } } } },
    },
    {
      name: "run_ab_test",
      description: "Create and start an A/B test on a page",
      inputSchema: { type: "object", properties: { pageId: { type: "string" }, variantContent: { type: "object" } } },
    },
    {
      name: "get_analytics",
      description: "Get analytics for a page including conversion rate, visitors, and CWV scores",
      inputSchema: { type: "object", properties: { pageId: { type: "string" }, dateRange: { type: "string" } } },
    },
    {
      name: "publish_page",
      description: "Publish a page to make it live",
      inputSchema: { type: "object", properties: { pageId: { type: "string" } } },
    },
    {
      name: "score_page",
      description: "Get a predictive conversion score for a page before publishing",
      inputSchema: { type: "object", properties: { pageId: { type: "string" } } },
    },
  ],
}));

// Resources exposed to AI agents:
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    { uri: "page://{id}", name: "Landing Page", description: "Full page content, SEO, and settings" },
    { uri: "template://{slug}", name: "Template", description: "Page template with default content" },
    { uri: "experiment://{id}", name: "Experiment", description: "A/B test with variants and results" },
    { uri: "analytics://{pageId}", name: "Page Analytics", description: "Visitor, conversion, and CWV data" },
  ],
}));
```

**Testing**:
- `Integration: MCP client connects to server → capabilities negotiated successfully`
- `Integration: call create_page tool → page created in database`
- `Integration: call generate_copy tool → AI generation job created`
- `Integration: read page:// resource → returns full page content`
- `Integration: call get_analytics tool → returns analytics summary`
- `Unit: MCP server handles unknown tool name → proper error response`
- `Unit: MCP server validates tool input against schema`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Infrastructure ─── required by everything
    │
Phase 2: Visual Page Builder ─── requires Phase 1
    │
Phase 3: Publishing & Rendering ─── requires Phase 2
    │
    ├── Phase 4: Lead Capture & Forms ─── requires Phase 3
    │       │
    ├── Phase 5: A/B Testing & Experiments ─── requires Phase 3
    │       │
    ├── Phase 6: AI Copy & Prompt-to-Page ─── requires Phase 2 (not Phase 3)
    │       │
    └── Phase 7: DTR & Personalisation ─── requires Phase 3
            │
Phase 8: Analytics & Heatmaps ─── requires Phase 3, Phase 4, Phase 5
    │
Phase 9: Integrations & Webhooks ─── requires Phase 4
    │
Phase 10: Predictive Scoring & Smart Traffic ─── requires Phase 5, Phase 6, Phase 8
    │
Phase 11: Templates & Vertical AI ─── requires Phase 2, Phase 6
    │
Phase 12: MCP Server & API ─── requires Phase 3, Phase 5, Phase 6, Phase 8
```

### Parallelism Opportunities

- **Phases 4, 5, 6, 7 can be developed concurrently** after Phase 3 (or Phase 2 for Phase 6). They are independent features that don't depend on each other.
- **Phase 8** can start once Phase 3 is complete; it does not strictly require Phase 4/5 but benefits from having conversion events to track.
- **Phase 9** can start once Phase 4 is complete (needs lead events to forward).
- **Phases 11 and 12** can be developed concurrently after their dependencies are met.

---

## Definition of Done (per phase)

1. All tasks implemented with complete functionality as specified in the Design section.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass against the test database (`pnpm test:integration`).
4. Biome lint and format pass with zero errors (`pnpm lint`).
5. TypeScript strict-mode type checking passes with zero errors (`pnpm typecheck`).
6. Docker build succeeds and application starts in Docker Compose environment.
7. Feature works end-to-end in the browser (manually verified or via Playwright E2E test).
8. Database migrations generated and applied without errors (`pnpm db:migrate`).
9. New API endpoints documented in OpenAPI spec (auto-generated from tRPC/Zod).
10. New environment variables documented in `.env.example` with sensible defaults.
11. No new WCAG 2.2 AA accessibility violations introduced in the builder UI.
12. Published page output passes Core Web Vitals thresholds (LCP <2.5s, CLS <0.1, INP <200ms) for standard-complexity pages.
