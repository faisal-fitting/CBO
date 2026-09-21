# CBO.AI solution architecture

**Current system and repository plans**

**[Open the illustrated document](solution-architecture.html)** — full text with six diagrams.

## About this document

- **Scope:** application structure, responsibilities, report-generation flow, AI orchestration, data movement, integrations, security facts, build behavior, and repository plans.
- **Current system:** statements are based on source in the `soft-ai` repository. Items marked **not verified** are not established by repository code.
- **Repository plans:** planned behavior is labelled and is not described as current behavior.
- **Security:** credential values are not reproduced.
- **Sources:** the [source index](#11-source-index) links to the main implementation files.

## 1. System overview

- **Business purpose:** CBO.AI creates Arabic business-intelligence reports for Saudi food-and-beverage businesses.
- **Primary users:** owners and managers of cafes, restaurants, cloud kitchens, and fine-dining businesses.
- **Main input:** business identity, Google Place, social-media handles, monthly financial values, and menu-item economics.
- **Main output:** a structured report with a strategic directive, financial analysis, digital analysis, market analysis, and an action plan.
- **Follow-up:** a report-scoped chat answers questions using stored workflow and report data.

[Diagram: system context](solution-architecture.html#system-context)

### Responsibility boundaries

| Part | Owns | Does not establish |
| --- | --- | --- |
| Browser application | Arabic-first form, generation progress, report presentation, charts, map, and follow-up chat | Server authorization or durable business records |
| Next.js server | API routes, server actions, workflow streaming, report restoration, and chat transport | Independent service isolation from Mastra |
| Mastra runtime | Workflow execution, agents, tools, memory, run state, and observability | Deterministic correctness of model-generated analysis |
| Deterministic financial module | Revenue, cost, margin, break-even, menu-item, and capacity calculations | Market, review, or social-media interpretation |
| External providers | Places, reviews, social data, web search, model inference, and managed storage | Application access control or report ownership |

- **Runtime shape:** Next.js and Mastra are imported into the same application source. No separate Mastra deployment boundary is defined in this repository.
- **Language and direction:** the product is Arabic-first and renders the document root right-to-left.

## 2. Repository and technology

### Repository map

| Path | Purpose |
| --- | --- |
| `src/app/` | App Router pages, report routes, API routes, and server actions |
| `src/components/` | Business form, report shell, progress UI, chat, maps, charts, and shadcn-based controls |
| `src/mastra/workflows/` | End-to-end business-analysis workflow |
| `src/mastra/agents/` | CBO, financial, digital, market, semantic, and social agents |
| `src/mastra/tools/` | Google Places, reviews, social scraping, web search, manual input, and report retrieval |
| `src/mastra/shared/` | Zod schemas and deterministic financial calculations |
| `src/store/` | Browser-persisted report and form state |
| `public/` | Product logo and static assets |

### Technology choices

| Technology | Role |
| --- | --- |
| Next.js 16 / React 19 | Pages, client UI, route handlers, and server actions |
| Mastra | Workflow, agents, tools, memory, storage abstraction, and observability |
| AI SDK / Mastra AI SDK | Browser chat hooks and stream conversion between Next.js and Mastra |
| Zod | Workflow, tool, structured-model-output, and report validation |
| TypeScript | Strict source typing; the production build is configured to ignore type errors |
| Tailwind CSS 4 / shadcn / Base UI | RTL interface and component styling |
| Zustand | Browser state persisted in local storage |
| Recharts / MapLibre / React Flow | Charts, competitor mapping, and visual report components |
| Motion | Form, progress, report, and navigation transitions |
| LibSQL / Turso | Default Mastra persistence configured by a fixed database URL and environment token |
| DuckDB | Local observability-domain storage |

- **Single package:** the repository contains one application package and one pnpm lockfile; no workspace definition is present.
- **Installed but not active:** `better-auth`, LiveKit React Native packages, and some other dependencies have no application path found in `src/`.

## 3. Runtime and request paths

[Diagram: request paths](solution-architecture.html#request-paths)

### Report generation

1. The browser validates and submits the business form.
2. The browser creates a random Thread identifier and calls `POST /api/report/stream`.
3. The route invokes Mastra's `business-analysis-workflow` and streams workflow events through the AI SDK protocol.
4. The browser records the returned Run identifier, navigates to `/report/{runId}`, and renders progress events.
5. On completion, the assembled report manifest and collected workflow-step data are rendered.

### Restore and follow-up

- **Report route:** `/report/{runId}` reads persisted workflow state through server actions.
- **Running workflow:** `POST /api/report/observe` reconnects to an existing Run stream.
- **Chat history:** `GET /api/chat?threadId=...` recalls messages from CBO agent memory.
- **Chat response:** `POST /api/chat` streams a CBO agent response against the supplied Thread.
- **Report lookup tool:** the CBO agent can load manifest or selected workflow-step outputs using a Run identifier stored in working memory.
- **Seed endpoint:** `POST /api/report/seed` validates a report manifest and writes it into CBO working memory.

- **Execution limit:** report, observe, seed, and chat routes declare a 300-second maximum duration where configured.
- **Client continuity:** Run identifier, Thread identifier, business name, form step, and form data persist under `cbo-report`; an independent form draft persists under `cbo-form-draft`.

## 4. Business-analysis workflow

[Diagram: report workflow](solution-architecture.html#report-workflow)

### Processing stages

- **Financial calculation:** deterministic TypeScript calculates net revenue, variable and fixed costs, profit, margins, contribution margin, break-even values, and menu-item metrics.
- **Place enrichment:** Google Places supplies location, rating, reviews count, business type, opening information, photos, and service-style signals.
- **External collection:** target reviews, nearby competitors, and social profiles are collected in parallel.
- **Competitor enrichment:** the six strongest nearby listings by rating and review volume receive review samples and heuristic revenue estimates.
- **Signal extraction:** semantic review analysis and social engagement audit run in parallel with structured-output fallbacks.
- **Strategic directive:** the CBO agent sets the report theme, North Star metric, focus areas, and overall status.
- **Expert analysis:** financial, digital, and market agents produce report sections in parallel.
- **Action plan:** the CBO agent synthesizes expert findings into phased tasks and expected outcomes.
- **Assembly:** the workflow returns metadata, directive, and four report sections as a validated report manifest.

### Calculation and model boundary

| Deterministic application code | Model-generated output |
| --- | --- |
| Financial totals and ratios | Strategic theme and North Star framing |
| Product revenue, cost, margin, and menu category | Narrative conclusions and risks |
| Location radius and competitor selection rules | Semantic themes and critical weakness |
| Reputation-weighted market-share inputs | Digital and market interpretation |
| Health-score formula | Action tasks and expected targets |

- **Structured outputs:** agent calls use Zod schemas and fallback values for directive, section, semantic, and social-audit generation.
- **Partial-provider behavior:** social scraping degrades to an error-bearing empty profile; competitor review calls use `Promise.allSettled`; several other provider failures can stop their workflow step.

## 5. AI agents and tools

[Diagram: agent collaboration](solution-architecture.html#agent-collaboration)

| Agent | Model route | Responsibility |
| --- | --- | --- |
| CBO agent | OpenRouter → Anthropic Claude Opus 4.6 | Strategic directive, action-plan synthesis, and report Q&A |
| Financial expert | OpenRouter → Anthropic Claude Opus 4.6 | Profitability, costs, break-even, menu economics, and revenue positioning |
| Digital expert | OpenRouter → Anthropic Claude Opus 4.6 | Reviews, sentiment, social performance, and digital recommendations |
| Market expert | OpenRouter → Anthropic Claude Opus 4.6 | Direct competitors, reputation share, market context, and opportunities |
| Semantic analysis agent | OpenRouter → Anthropic Claude Sonnet 4.6 | Arabic themes, sentiment score, strengths, weaknesses, and critical complaint |
| Social engagement auditor | OpenRouter → Anthropic Claude Sonnet 4.6 | Platform benchmarks, engagement health, content gaps, and top content |

### Tool boundaries

- **Google Places tools:** place details, nearby search, photo resolution, and autocomplete.
- **SerpAPI tools:** Google Maps review collection and market web search.
- **ScrapeCreators tool:** Instagram and TikTok profile and recent-content collection.
- **Report-data tool:** reads a completed workflow Run and returns a selected report or step-data section.
- **Memory:** the CBO agent uses observational memory, generated Thread titles, messages, and working memory.
- **Language:** expert narrative output is directed to professional Saudi Arabic; the social signal extractor emits English intermediate strings that the digital expert consumes.

## 6. Data, state, and retention

[Diagram: data and state](solution-architecture.html#data-state)

| Data class | Current store or destination | Access key in source |
| --- | --- | --- |
| Form draft and current-report UI state | Browser local storage through direct draft writes and Zustand | Browser profile; `cbo-form-draft` and `cbo-report` keys |
| Workflow Run state and outputs | Mastra default LibSQL store on Turso | Run identifier |
| Conversation messages and working memory | Mastra Memory through the default store | Thread identifier plus constant resource `user` |
| Local observability records | DuckDB file `mastra-observability.duckdb` | Application runtime filesystem |
| Platform traces | Mastra Platform exporter | Platform configuration outside this repository |
| Provider request data | Google, SerpAPI, ScrapeCreators, OpenRouter/model provider | Provider credentials and request payloads |

- **No application database model:** the repository defines workflow schemas and storage adapters, not business-account, organization, user, or report-owner entities.
- **Global resource:** server actions and chat routes use the literal Mastra resource identifier `user`.
- **Report recovery:** report pages and the report-data tool load workflow state directly by Run identifier.
- **Retention:** no time-based deletion, user deletion flow, archival policy, or retention period is implemented in source.
- **Backups and restore:** backup coverage and restore behavior for Turso, DuckDB, browser state, and Mastra Platform are not verified.

## 7. Integrations and information flow

[Diagram: integration landscape](solution-architecture.html#integration-landscape)

| Service | Application path | Information handled |
| --- | --- | --- |
| OpenRouter / Anthropic models | All six agents | Financial values, place/review/social context, generated sections, and chat prompts |
| Google Places API | Form autocomplete, details, nearby search, photos, and static maps | Search text, Place identifiers, location, listing metadata, and image requests |
| SerpAPI | Google Maps reviews and market web search | Place identifiers, search queries, reviews, topics, and result links |
| ScrapeCreators | Instagram and TikTok collection | Public handles, profile metadata, content statistics, captions, and URLs |
| Turso LibSQL | Default Mastra storage | Workflow state, memory, messages, and related runtime records |
| Mastra Platform | Observability exporter | Traces after sensitive-data processing configured by Mastra |
| DuckDB | Observability storage domain | Runtime observability records on local disk |
| CARTO basemaps | MapLibre style source | Browser map tile/style requests |

- **Direct image delivery:** Google, Instagram, Facebook, TikTok, Google user-content, and avatar hosts are allowed by Next.js image configuration.
- **Client-visible provider requests:** generated Google photo and static-map URLs can contain a Google API key in the query string.
- **Mock mode:** reviews, web search, and social scraping include `MOCK_TOOLS=true` paths with synthetic data.
- **Not verified:** provider account ownership, quotas, billing limits, processing regions, retention, and production enablement.

## 8. Identity, access, and security

### Access facts

| Boundary | Current source behavior |
| --- | --- |
| User authentication | No active authentication, session lookup, middleware, or route authorization path was found |
| Report ownership | A supplied Run identifier is used directly to load workflow state |
| Conversation ownership | A supplied Thread identifier and shared resource `user` are used directly to recall or stream chat |
| Workflow creation | The report stream accepts caller-provided business and financial input |
| Memory seeding | The seed route accepts caller-provided manifest, Thread, and resource identifiers |
| Abuse controls | No request-rate, provider-budget, payload-size, or per-user workflow limits were found |
| Request validation | The seed route validates the manifest; other routes mainly destructure JSON and rely on downstream handlers or schemas |
| Origin controls | No application-level CSRF or request-origin validation was found for mutation routes |

### Security facts

- **Credential fallbacks:** Google Places and SerpAPI credentials have literal fallback values in tracked source.
- **Browser exposure:** Google photo and static-map URLs are assembled with the API key as a query parameter.
- **Sensitive business data:** financial inputs, social metrics, reviews, and report content can reach model and data providers.
- **Application logging:** routes and server actions log Run and Thread identifiers, workflow status, input keys, and a report-manifest sample.
- **Trace filtering:** Mastra observability configures `SensitiveDataFilter`; this does not cover ordinary `console` or Pino application messages by itself.
- **Dependency versus implementation:** `better-auth` is installed, but no authentication implementation was found in application source.
- **Transport and infrastructure:** TLS termination, network controls, secret injection, key restrictions, administrative access, and production log access are not verified.

## 9. Build, release, and reliability

### Build and release facts

- **Commands:** package scripts provide `dev`, `build`, `start`, `lint`, and `mastra`.
- **Package management:** the repository contains a pnpm lockfile but no `packageManager` or Node engine declaration.
- **Type checking:** TypeScript strict mode is enabled; Next.js is configured to ignore build-time TypeScript errors. The current standalone type check reports errors in server actions, chat streaming, UI components, and Mastra workflow typing.
- **Linting:** the current repository-wide lint command reports existing errors and warnings across application and generated-style UI code.
- **Tests:** no test script or tracked test files were found.
- **Continuous integration:** no tracked GitHub Actions workflow or other repository CI definition was found.
- **Deployment:** no Dockerfile, deployment manifest, or checked-in Vercel project configuration was found. Production host, region, replicas, runtime limits, and active commit are not verified.
- **Documentation:** the README remains the generated Next.js starter guide and does not define the application deployment process.

### Failure and recovery boundaries

- **External dependency chain:** report completion depends on managed storage, model access, and several data providers.
- **Long-running requests:** report generation uses streamed HTTP requests with a declared 300-second maximum duration.
- **Fallbacks:** structured model outputs have fallback values; social and competitor-review paths can continue with partial data. A fallback report section can therefore represent an upstream model failure as completed workflow output.
- **Run restoration:** persisted Run state supports report reload and running-stream observation.
- **Retry safety:** no application idempotency key, durable retry policy, provider circuit breaker, or workflow cancellation policy was found.
- **Memory update:** final CBO working-memory update is best-effort and ignores errors.
- **Observability:** Pino logging, local DuckDB storage, Mastra storage export, Mastra Platform export, and sensitive-span filtering are configured.
- **Not verified:** alerting, uptime checks, backup schedule, restore tests, RTO, RPO, incident response, and provider-failure monitoring.

## 10. Repository plans

### Report history and comparison

- **State:** `PLAN.md` describes Plan Mode work and explicitly awaits agreement; the behavior is not implemented in current source.
- **History:** retain past report manifests and expose report date, business identity, health score, and status.
- **Comparison:** compare financial, digital, market, and action-plan results against a selected baseline.
- **Proposed persistence:** browser local storage with a twenty-report limit.
- **Proposed routes:** history listing, deletion, and baseline selection endpoints.
- **Non-goals in the plan:** real-time collaboration, AI-generated comparison analysis, statistical forecasting, and cross-business benchmarking.

- **No current history model:** the present Zustand store keeps one current Run, Thread, business name, form step, and form payload.
- **No current account boundary:** the repository plan does not supply implemented user identity or server-side ownership for history records.

## 11. Source index

The links below describe repository source at `main` commit `0605bd9042f43153172c3bddc02553ad33cf2be1`.

| Area | Main references |
| --- | --- |
| Product and repository conventions | [AGENTS.md](../../AGENTS.md), [package manifest](../../package.json), [TypeScript](../../tsconfig.json), [Next.js config](../../next.config.ts) |
| Entry pages and UI state | [form page](../../src/app/page.tsx), [report page](../../src/app/report/[runId]/page.tsx), [report session](../../src/components/report-session.tsx), [browser store](../../src/store/report-store.ts) |
| Server paths | [server actions](../../src/app/actions.ts), [workflow stream](../../src/app/api/report/stream/route.ts), [workflow observe](../../src/app/api/report/observe/route.ts), [report seed](../../src/app/api/report/seed/route.ts), [chat](../../src/app/api/chat/route.ts) |
| Mastra composition | [Mastra runtime](../../src/mastra/index.ts), [business workflow](../../src/mastra/workflows/main-workflow.ts) |
| Data contracts and calculations | [schemas](../../src/mastra/shared/schemas.ts), [financial calculations](../../src/mastra/shared/financials.ts), [frontend types](../../src/lib/types.ts) |
| Agents | [CBO](../../src/mastra/agents/CBO-agent.ts), [financial](../../src/mastra/agents/financial-expert-agent.ts), [digital](../../src/mastra/agents/digital-expert-agent.ts), [market](../../src/mastra/agents/market-expert-agent.ts), [semantic](../../src/mastra/agents/semantic-analysis.ts), [social audit](../../src/mastra/agents/social-engagement-auditor.ts) |
| External tools | [Google Places](../../src/mastra/tools/google-places.ts), [Google reviews](../../src/mastra/tools/google-maps-reiews.ts), [social scraping](../../src/mastra/tools/social-media-scrape.ts), [web search](../../src/mastra/tools/web-search.ts), [report data](../../src/mastra/tools/report-data.ts) |
| Interface styling | [root layout](../../src/app/layout.tsx), [brand tokens](../../src/app/globals.css), [report view](../../src/components/report-view.tsx), [chat sidebar](../../src/components/chat-sidebar.tsx) |
| Repository plan | [PLAN.md](../../PLAN.md) |
