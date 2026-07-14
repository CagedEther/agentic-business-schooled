# Build a PubNub Content Research and Technical Writing Workflow

## Overview

This workflow uses two Blocks Network provider agents to turn a broad topic into an evidence-backed editorial plan and then turn one selected idea into a publication-ready technical blog draft. The first agent coordinates SEO, external-blog, PubNub corpus, and LinkedIn research. The second agent runs a focused research pass through the planner, then drafts and edits the requested article with OpenAI and a reusable technical-writing standard.

The finished private Blocks agents are:

- [PubNub Content Plan Orchestrator](https://app.blocks.ai/agents/pubnub_content_plan_orchestrator)
- [PubNub Technical Blog Writer](https://app.blocks.ai/agents/pubnub_technical_blog_writer)

## When To Use This

- Build a repeatable research-to-draft workflow for developer marketing or technical content teams.
- Replace disconnected keyword, competitor, internal-content, and social research with one traceable content plan.
- Turn an approved article brief into a structured technical draft while keeping research and writing metadata inspectable.
- Reproduce the same two-stage pattern for another brand, audience, or evidence corpus.

## Prerequisites

- A Blocks account and organization with permission to register provider agents, plus a valid `BLOCKS_API_KEY`.
- Access to these downstream Blocks agents: `seo_analyst_agent`, `firecrawl_top_blogs`, `pubnub_top_blogs_rag`, and `linkedin_topic_content_ideas`. Private agents require an owner account or an accepted invitation.
- An OpenAI account with API billing enabled and an `OPENAI_API_KEY`. The default research and writing model is `gpt-5-mini`, but it is configurable.
- A Railway account and workspace access if the providers will be hosted continuously. Authenticate the Railway CLI before deployment.
- Node.js 24 or newer, npm, Git, the Blocks CLI, and the Railway CLI.
- Network access for Blocks tasks, OpenAI calls, package installation, registration, and deployment.
- No DataForSEO, Firecrawl, Apify, PubNub RAG database, or LinkedIn credentials are stored on these two providers. Those service credentials belong on the specialist downstream providers that use them.

Store secrets in a local ignored `.env` and in Railway service variables. Never commit them.

## Required Inputs

The content planner requires a topic:

```json
{
  "topic": "cart abandonment",
  "audience": "developers, engineering leaders, and product leaders",
  "businessGoal": "build organic authority and qualified demand",
  "brandContext": "PubNub real-time application infrastructure"
}
```

Optional planner fields include `topicDescription`, `tone`, `country`, `languageCode`, `locationCode`, `externalBlogLimit`, `linkedinIdeaCount`, `downstreamTimeoutMs`, `includeRawResearch`, and `synthesisMode`.

The writer requires an article overview:

```json
{
  "overview": "Write a production-focused guide showing developers how to detect cart abandonment events and trigger real-time recovery workflows.",
  "audience": "senior application developers",
  "readerIntent": "solve",
  "targetKeyword": "real-time cart abandonment monitoring",
  "desiredLengthWords": 1800,
  "callToAction": "Explore PubNub event-driven messaging."
}
```

Optional writer fields include `topic`, `businessGoal`, `brandContext`, `tone`, `downstreamTimeoutMs`, `includeRawResearchInTrace`, and `synthesisMode`.

## Tools And Services

- [Blocks Network](https://blocks.ai/) routes requests between the two provider agents and the four research specialists.
- `seo_analyst_agent` supplies keyword metrics, clusters, intent, and SERP context.
- `firecrawl_top_blogs` supplies ranked external articles and competitive patterns.
- `pubnub_top_blogs_rag` supplies grounded PubNub corpus coverage and internal-link candidates.
- `linkedin_topic_content_ideas` supplies current practitioner patterns, evidence, and social angles.
- [OpenAI](https://openai.com/) performs structured five-post synthesis and full-article drafting.
- [Railway](https://railway.com/) hosts each Blocks provider as an independent long-running worker.
- The local `technical-blog-writing/SKILL.md` file defines the writer's clarify, angle, outline, draft, verify, tighten, and distribution standards.

## Project Shape

Use one npm workspace with separate provider directories:

```text
README.md
package.json
content-plan-orchestrator/
  agent-card.json
  handler.ts
  trigger.ts
  package.json
  railway.json
  .env.example
technical-blog-writer/
  agent-card.json
  handler.ts
  trigger.ts
  smoke-test.ts
  smoke-live.ts
  package.json
  railway.json
  .env.example
technical-blog-writing/
  SKILL.md
```

Keep the agents independently runnable and deployable. A deployment rooted at the repository root can start the wrong provider, so each Railway service must use the matching provider directory as its service root or be deployed from that directory.

## Workflow

### 1. Scaffold The Workspace

Create a private npm workspace with both provider directories. Each provider uses TypeScript modules, Node 24+, `@blocks-network/sdk`, `@blocks-network/cli`, `dotenv`, `openai`, `tsx`, and TypeScript.

At the workspace root, provide scripts that run both checks and route triggers:

```json
{
  "scripts": {
    "typecheck": "npm run typecheck --workspace content-plan-orchestrator && npm run typecheck --workspace technical-blog-writer",
    "check": "npm run check --workspace content-plan-orchestrator && npm run check --workspace technical-blog-writer",
    "smoke:mock": "npm run smoke:mock --workspace technical-blog-writer",
    "smoke:live": "DOTENV_CONFIG_PATH=../.env npm run smoke:live --workspace technical-blog-writer",
    "trigger:plan": "DOTENV_CONFIG_PATH=../.env npm run trigger --workspace content-plan-orchestrator --",
    "trigger:writer": "DOTENV_CONFIG_PATH=../.env npm run trigger --workspace technical-blog-writer --"
  }
}
```

### 2. Build The Content Plan Orchestrator

Use the Blocks identity `pubnub_content_plan_orchestrator`. Its JSON request input requires `topic`; its guaranteed outputs are:

- `content_plan` as `text/markdown`
- `content_plan_json` as `application/json`
- `research_trace` as `application/json`

In the handler:

1. Parse and clamp all request values.
2. Create four schema-specific downstream payloads. Do not send one generic payload to every agent.
3. Start the independent Blocks tasks concurrently and wait with bounded timeouts.
4. Capture all artifact events, deduplicate artifacts by reference metadata, and download text-like outputs.
5. Continue with explicit warnings when one branch fails, but fail if all four branches fail.
6. Build a bounded research packet and ask OpenAI for a strict JSON schema with exactly five uniquely prioritized posts.
7. Render that structure into Markdown and return a separate trace.

The downstream request mapping is important:

- SEO receives `topic`, search locale, keyword limits, and SERP flags.
- External blogs receive `topic`, result limit, country, content-summary flag, and per-domain cap.
- PubNub RAG receives a grounded `query` and `prompt`, `action: "query"`, `topK`, and `includeAnswer`.
- LinkedIn receives `topic`, audience, brand context, tone, idea count, time window, and conservative post limits.

Provide `auto`, `openai`, and `deterministic` synthesis modes. In `auto`, use a deterministic five-post fallback if OpenAI is unavailable. The fallback must still satisfy the output contract and clearly identify its lower-confidence nature in trace metadata.

### 3. Build The Technical Blog Writer

Use the Blocks identity `pubnub_technical_blog_writer`. Its JSON request requires `overview`; its guaranteed outputs are:

- `blog_draft` as `text/markdown`
- `writing_trace` as `application/json`

The overview is editorial authority. Derive a concise research topic only when `topic` is omitted; never allow the downstream planner to replace the requested article with a different idea.

The writer should:

1. Send the overview, audience, intent, SEO, business, brand, and tone context to `pubnub_content_plan_orchestrator`.
2. Request raw planner research for the writing stage, but keep the final writer trace preview-only unless the caller explicitly enables raw research.
3. Collect the planner's Markdown plan, structured plan, and research trace.
4. Select supporting evidence relevant to the requested article rather than blindly choosing the planner's first recommendation.
5. Build a bounded source packet and draft with OpenAI using only supplied evidence.
6. Follow the local technical-writing standard: a specific thesis, skimmable structure, honest code, explicit tradeoffs, verified claims, one CTA, and a distribution kit.
7. Return a deterministic review draft in `auto` mode if OpenAI fails, or fail when `synthesisMode` is `openai`.

### 4. Configure Environment Variables

Use a shared ignored root `.env` for local workspace scripts. The planner needs:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_SYNTHESIS_MODEL=gpt-5-mini
OPENAI_SYNTHESIS_MAX_OUTPUT_TOKENS=9000
OPENAI_SYNTHESIS_INPUT_CHARS=120000
BLOCKS_BILLING_MODE=free
DOWNSTREAM_TASK_TIMEOUT_MS=1800000
SEO_AGENT_NAME=seo_analyst_agent
EXTERNAL_BLOGS_AGENT_NAME=firecrawl_top_blogs
PUBNUB_BLOGS_AGENT_NAME=pubnub_top_blogs_rag
LINKEDIN_IDEAS_AGENT_NAME=linkedin_topic_content_ideas
```

The writer needs:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_WRITING_MODEL=gpt-5-mini
OPENAI_WRITING_MAX_OUTPUT_TOKENS=12000
OPENAI_WRITING_INPUT_CHARS=160000
BLOCKS_BILLING_MODE=free
DOWNSTREAM_TASK_TIMEOUT_MS=1800000
BACKGROUND_RESEARCH_AGENT_NAME=pubnub_content_plan_orchestrator
```

### 5. Validate Locally

From the workspace root:

```bash
npm install
npm run typecheck
npm run check
npm run smoke:mock
```

The mock test should inject a fake completed planner task and confirm that the writer returns `blog_draft` and `writing_trace` without requiring live network calls.

When valid credentials and a planner provider are available, run the live writer handler test:

```bash
npm run smoke:live
```

### 6. Register The Providers

Validate each card before registration. Register private and free first unless the operator explicitly chooses public or paid access:

```bash
cd content-plan-orchestrator
blocks register

cd ../technical-blog-writer
blocks register
```

Agent names are global. If either name is unavailable or owned by another organization, choose a new stable underscore slug and update every matching environment default and trigger.

### 7. Deploy Separate Railway Workers

Create one Railway project or service for each provider. Set only `BLOCKS_API_KEY`, `OPENAI_API_KEY`, and that provider's optional tuning variables. Do not copy specialist-only secrets onto these orchestration services.

Each provider's `railway.json` should use Nixpacks and `npm start`, with restart-on-failure enabled. Deploy from the provider directory or configure it as the service root.

```bash
railway init --name pubnub-content-plan-orchestrator --json
railway add --service pubnub-content-plan-orchestrator --json
printf '%s' "$BLOCKS_API_KEY" | railway variable set BLOCKS_API_KEY --stdin --service pubnub-content-plan-orchestrator
printf '%s' "$OPENAI_API_KEY" | railway variable set OPENAI_API_KEY --stdin --service pubnub-content-plan-orchestrator
railway up --detach --json --service pubnub-content-plan-orchestrator
```

Repeat from `technical-blog-writer/` with `pubnub-technical-blog-writer`. Inspect deployment and runtime logs until each worker reports a successful Blocks agent instance registration.

### 8. Run End-To-End Tasks

Run the planner first:

```bash
npm run trigger:plan -- '{"topic":"cart abandonment","audience":"developers and product leaders"}'
```

Review the five recommendations, then send one selected brief to the writer:

```bash
npm run trigger:writer -- '{"overview":"Write a production-focused guide to real-time cart abandonment monitoring with PubNub.","audience":"senior developers","readerIntent":"solve"}'
```

Confirm that the writer launches a new focused planner task and then returns a complete Markdown article and trace.

## Outputs

- A five-post Markdown content strategy with executive summary, gaps, keywords, intent, outlines, evidence, URLs, CTAs, distribution ideas, cadence, and measurement plan.
- The same content plan as structured JSON.
- A research trace containing downstream task states, artifact metadata, synthesis metadata, and warnings.
- A publication-ready technical Markdown draft with title, meta description, thesis, article body, one CTA, distribution kit, and SME-review flags when needed.
- A writing trace containing planner task metadata, research artifact summaries, model metadata, word count, and warnings.

## Validation

- `npm run typecheck` passes for both workspaces.
- `npm run check` passes Blocks card schema validation for both providers.
- `npm run smoke:mock` passes without live services.
- Railway reports one running deployment per provider and logs a Blocks `agent_registered` event.
- Both registered cards are private/free unless intentionally changed.
- A live planner task returns exactly the three declared artifacts and exactly five posts.
- A live writer task returns both declared artifacts and its trace points to a completed planner task.
- Output IDs and MIME types match each `agent-card.json` exactly.

## Safety And Quality Notes

- Never print, commit, or place raw keys in commands, logs, traces, or documentation. Prefer stdin when setting Railway variables.
- Keep `.env`, generated logs, `node_modules`, and sensitive raw research out of commits and deployment archives.
- Treat external rankings, LinkedIn activity, and model-generated interpretations as research signals, not verified truth. Preserve URLs and label inference.
- Verify technical claims, API names, code, version numbers, and links before publication. Flag uncertain claims for subject-matter-expert review.
- Use only supplied research during synthesis. Never invent statistics, quotations, benchmarks, or URLs.
- Keep `includeRawResearch` and `includeRawResearchInTrace` off for ordinary consumer tasks. Raw source packets can be large and may contain text that should not be broadly redistributed.
- Bound downstream timeouts, input characters, output tokens, external result counts, and LinkedIn post counts. OpenAI and specialist providers may incur usage costs even when Blocks billing mode is free.
- Keep the planner and writer as separate Railway workers so deployments, failures, and scaling changes remain isolated.
- Do not place downstream service credentials on the orchestrators. Each specialist provider owns its DataForSEO, Firecrawl, Apify, database, and related secrets.
