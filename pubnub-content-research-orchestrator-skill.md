# Build a PubNub Content Research Orchestrator

## Overview

This workflow builds a Blocks Network provider agent that accepts a topic, runs four specialist research agents concurrently, and returns an evidence-backed content plan with exactly five recommended blog posts. It combines SEO demand, leading external content, PubNub's existing corpus, and recent LinkedIn discussion while preserving task and artifact metadata for review.

The finished private agent is the [PubNub Content Plan Orchestrator](https://app.blocks.ai/agents/pubnub_content_plan_orchestrator).

## When To Use This

- Build a repeatable topic-research workflow for content, product marketing, SEO, or developer relations teams.
- Compare search demand, competitor coverage, internal content, and professional discussion before choosing an editorial direction.
- Produce a five-post content cluster with keywords, intent, evidence, outlines, distribution ideas, and measurement guidance.
- Adapt the same orchestration pattern for another brand, corpus, market, or set of research agents.

## Prerequisites

- A Blocks account and organization with permission to register provider agents, plus a valid `BLOCKS_API_KEY`.
- Access to these downstream Blocks agents: `seo_analyst_agent`, `firecrawl_top_blogs`, `pubnub_top_blogs_rag`, and `linkedin_topic_content_ideas`. Private agents require owner access or an accepted invitation.
- An OpenAI account with API billing enabled and an `OPENAI_API_KEY` for structured synthesis. The default model is `gpt-5-mini`.
- A Railway account and workspace access if the provider will run continuously. Authenticate the Railway CLI before deployment.
- Node.js 24 or newer, npm, the Blocks CLI, and the Railway CLI.
- Network access for package installation, Blocks tasks, OpenAI calls, registration, and deployment.
- No DataForSEO, Firecrawl, Apify, LinkedIn, or PubNub RAG database credentials belong on this orchestrator. Those secrets stay on the specialist providers that directly use them.

Store local secrets in an ignored `.env` file and remote secrets in Railway service variables. Never commit them.

## Required Inputs

Only `topic` is required:

```json
{
  "topic": "cart abandonment",
  "topicDescription": "Recovery patterns for real-time commerce experiences.",
  "audience": "developers, engineering leaders, and product leaders",
  "businessGoal": "build organic authority and qualified demand",
  "brandContext": "PubNub real-time application infrastructure",
  "tone": "practical, technically credible, and developer-friendly"
}
```

Optional fields are:

- `topicDescription`
- `audience`
- `businessGoal`
- `brandContext`
- `tone`
- `country`
- `languageCode`
- `locationCode`
- `externalBlogLimit`
- `linkedinIdeaCount`
- `downstreamTimeoutMs`
- `includeRawResearch`
- `synthesisMode`: `auto`, `openai`, or `deterministic`

## Tools And Services

- [Blocks Network](https://blocks.ai/) routes the main request to the orchestrator and its four downstream agents.
- `seo_analyst_agent` supplies keyword metrics, search intent, clusters, and a live SERP landscape.
- `firecrawl_top_blogs` supplies ranked external articles, descriptions, and competitive patterns.
- `pubnub_top_blogs_rag` supplies grounded coverage from PubNub's existing content and internal-link candidates.
- `linkedin_topic_content_ideas` supplies recent practitioner patterns, evidence, and social-content angles.
- [OpenAI](https://openai.com/) converts the collected research into a strict five-post plan.
- [Railway](https://railway.com/) hosts the orchestrator as a long-running Blocks worker.

## Project Shape

Use a small Node and TypeScript provider:

```text
content-plan-orchestrator/
  agent-card.json
  handler.ts
  trigger.ts
  package.json
  package-lock.json
  tsconfig.json
  railway.json
  .env.example
  .gitignore
  .railwayignore
  README.md
```

Use `@blocks-network/sdk`, `@blocks-network/cli`, `dotenv`, `openai`, `tsx`, and TypeScript. The provider is a worker; it does not need an HTTP server.

## Workflow

### 1. Define The Agent Contract

Use the Blocks identity `pubnub_content_plan_orchestrator` and display name `PubNub Content Plan Orchestrator`.

Declare one required `request` input with `application/json`. Declare three guaranteed outputs:

- `content_plan` as `text/markdown`
- `content_plan_json` as `application/json`
- `research_trace` as `application/json`

Set `runtime.handler` to `./handler.ts`, `handlerExport` to `default`, concurrency and expected instances to `1`, and `maxRunningTimeSec` high enough for four external tasks plus synthesis. The working implementation uses `2400` seconds.

### 2. Parse And Bound The Request

Parse the `request` part explicitly. Accept plain text as a topic for trigger convenience, but keep the published form contract as JSON.

Apply practical bounds:

- Topic: 300 characters.
- Topic context: 2,000 characters.
- External blogs: 5 to 25.
- LinkedIn ideas: 5 to 20.
- Per-agent timeout: 60,000 to 3,600,000 milliseconds.
- OpenAI synthesis input: 20,000 to 250,000 characters.
- OpenAI output: 2,000 to 20,000 tokens.

Default the audience, business goal, brand context, tone, US search locale, free Blocks billing mode, and a 30-minute downstream timeout when they are omitted.

### 3. Map The Topic To Native Downstream Schemas

Do not send the same generic object to every research agent. Build one minimal payload per contract.

SEO request:

```json
{
  "topic": "cart abandonment",
  "topicDescription": "Audience, business, and brand context",
  "device": "desktop",
  "serpDepth": 10,
  "languageCode": "en",
  "locationCode": 2840,
  "relatedLimit": 50,
  "suggestionLimit": 50,
  "includeSerpLandscape": true,
  "includeRelatedKeywords": true
}
```

External-blog request:

```json
{
  "topic": "cart abandonment",
  "limit": 10,
  "country": "US",
  "timeoutMs": 60000,
  "includeContent": true,
  "maxBlogsPerDomain": 2
}
```

PubNub RAG request:

```json
{
  "action": "query",
  "query": "cart abandonment",
  "prompt": "Identify relevant PubNub coverage, claims, patterns, gaps, and internal-link targets. Return a grounded answer with citations.",
  "topK": 10,
  "includeAnswer": true
}
```

LinkedIn request:

```json
{
  "topic": "cart abandonment",
  "audience": "developers, engineering leaders, and product leaders",
  "brandContext": "PubNub real-time application infrastructure",
  "tone": "practical, technically credible, and developer-friendly",
  "ideaCount": 10,
  "maxQueries": 6,
  "postedLimit": "3months",
  "sortBy": "relevance",
  "includeComments": false,
  "includeReactions": false,
  "maxPostsPerQuery": 20
}
```

Keep downstream agent names configurable through environment variables.

### 4. Run Fan-Out And Fan-In

Create a fresh authenticated Blocks `TaskClient`. Start all four independent requests concurrently with settled outcomes so one failed branch does not cancel the others.

For every branch:

1. Attach progress and artifact listeners immediately.
2. Wait for a bounded terminal event.
3. Inspect terminal-time events and accumulated artifacts to catch buffered outputs.
4. Deduplicate by artifact reference metadata rather than output ID alone.
5. Download text-like artifacts and record output ID, MIME type, filename, and size.
6. Close every session and destroy the task client.

Convert individual failures into warnings and continue with the successful evidence. Fail the main task when all four research branches fail or no readable artifacts remain.

### 5. Synthesize Exactly Five Recommendations

Build a source packet containing the business brief, successful research artifacts, source labels, task IDs, and warnings. Divide the configured character budget across successful sources and mark truncation.

Use OpenAI Structured Outputs with a strict JSON schema. Require exactly five posts and unique priorities from 1 through 5. Each post should include:

- Working title.
- Primary and secondary keywords.
- Search intent and funnel stage.
- Target audience and reader problem.
- Distinctive angle and reason it can win.
- Detailed outline.
- Supporting evidence and source URLs.
- One call to action.
- LinkedIn repurposing ideas.

Top-level plan fields should include an executive summary, strategic thesis, audience insight, content gaps, keyword strategy, internal-linking strategy, cadence, sequencing rationale, and measurement plan.

Instruct the model to use only the supplied packet and never invent metrics, quotations, claims, or URLs. Permit an empty URL list when the evidence does not contain one.

### 6. Add A Deterministic Fallback

Support three synthesis modes:

- `auto`: use OpenAI, then fall back if the call or parsing fails.
- `openai`: require a successful OpenAI response.
- `deterministic`: skip the model and render a predictable five-post cluster.

The fallback should always return five posts and the same JSON shape. Record the fallback reason in synthesis metadata so consumers can distinguish model synthesis from a lower-confidence deterministic result.

### 7. Return Business Output And Trace Separately

Render the structured plan into readable Markdown. Return the Markdown, JSON plan, and trace as separate artifacts with stable output IDs.

The trace should include:

- Sanitized request values.
- All four intended branches.
- Downstream agent names, task IDs, terminal states, and durations.
- Artifact metadata and short previews by default.
- Synthesis mode, model, response ID, and input size.
- Branch, artifact, and synthesis warnings.

Include full source text only when `includeRawResearch` is explicitly true.

### 8. Configure Environment Variables

Use this `.env.example` shape:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_SYNTHESIS_MODEL=gpt-5-mini
OPENAI_SYNTHESIS_MAX_OUTPUT_TOKENS=9000
OPENAI_SYNTHESIS_INPUT_CHARS=120000
BLOCKS_BILLING_MODE=free
DOWNSTREAM_TASK_TIMEOUT_MS=1800000
ORCHESTRATOR_AGENT_NAME=pubnub_content_plan_orchestrator
SEO_AGENT_NAME=seo_analyst_agent
EXTERNAL_BLOGS_AGENT_NAME=firecrawl_top_blogs
PUBNUB_BLOGS_AGENT_NAME=pubnub_top_blogs_rag
LINKEDIN_IDEAS_AGENT_NAME=linkedin_topic_content_ideas
```

Only `BLOCKS_API_KEY` is required for downstream tasks. `OPENAI_API_KEY` is required for normal OpenAI synthesis but optional when deterministic mode is acceptable.

### 9. Validate, Register, And Deploy

Install and validate locally:

```bash
npm install
npm run typecheck
npm run check
```

Register private and free first unless the operator explicitly chooses public or paid access:

```bash
blocks register
```

Deploy the directory as its own Railway project and service:

```bash
railway init --name pubnub-content-plan-orchestrator --json
railway add --service pubnub-content-plan-orchestrator --json
printf '%s' "$BLOCKS_API_KEY" | railway variable set BLOCKS_API_KEY --stdin --service pubnub-content-plan-orchestrator
printf '%s' "$OPENAI_API_KEY" | railway variable set OPENAI_API_KEY --stdin --service pubnub-content-plan-orchestrator
railway up --detach --json --service pubnub-content-plan-orchestrator
```

Inspect the latest deployment and runtime logs. A healthy worker reports a successful Railway deployment followed by Blocks `agent_instance_started` and `agent_registered` events.

### 10. Run An End-To-End Topic Test

After the hosted worker is online:

```bash
npm run trigger -- '{"topic":"cart abandonment","audience":"developers and product leaders"}'
```

Confirm that all available research branches reach terminal states, the plan contains exactly five unique priorities, and all three declared artifacts are present. Save returned artifacts to an `output/` directory when a durable local copy is needed.

## Outputs

- `content_plan`: readable Markdown strategy with five detailed post recommendations.
- `content_plan_json`: the same plan as structured JSON.
- `research_trace`: downstream task, artifact, synthesis, and warning metadata.
- Optional local copies such as `content-plan-cart-abandonment.md`, `content-plan-cart-abandonment.json`, and `research-trace-cart-abandonment.json`.

## Validation

- TypeScript compilation passes with `tsc --noEmit`.
- `blocks check` validates the card, handler path, schema, and unique IDs.
- The Railway deployment reports `SUCCESS` and its worker remains running.
- Blocks can route a real task to `pubnub_content_plan_orchestrator`.
- A live request attempts all four branches and completes when at least one useful branch succeeds.
- OpenAI mode returns exactly five posts with priorities `1,2,3,4,5`.
- Output IDs and MIME types match `agent-card.json`.
- Trace warnings are empty on a fully successful run or explicit when research is partial.

## Safety And Quality Notes

- Never print, commit, or embed API keys, tokens, cookies, or raw secret values. Prefer stdin when setting Railway variables.
- Keep `.env`, `node_modules`, generated logs, and sensitive raw research out of commits and deployment archives.
- Treat keyword metrics, SERP rankings, public LinkedIn activity, and model interpretations as research signals rather than verified truth.
- Preserve evidence URLs and distinguish sourced facts from strategic interpretation.
- Verify public claims, statistics, links, and technical recommendations before acting on or publishing the plan.
- Keep raw research disabled for ordinary tasks. Source packets can be large and may include text that should not be broadly redistributed.
- Bound timeouts, input characters, output tokens, result counts, and LinkedIn queries. OpenAI and specialist providers may incur costs even when Blocks billing mode is free.
- Keep downstream service credentials on their specialist providers. The orchestrator should contain only the credentials it directly needs.
- Keep synthesis fallbacks explicit. A deterministic plan is useful for continuity, but it should not be presented as equivalent to a fully researched OpenAI synthesis.
