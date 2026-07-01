# Build A LinkedIn Topic Content Ideas Blocks Agent

## Overview

This workflow builds a Blocks Network provider agent that takes a topic, researches how that topic is being discussed in LinkedIn posts through Apify, and returns evidence-backed content ideas. The reference implementation is a Node/TypeScript worker deployed on Railway, registered privately on Blocks, and powered by the Apify `harvestapi/linkedin-post-search` actor plus optional OpenAI synthesis.

The agent is useful when a content, product marketing, founder-led sales, or thought-leadership workflow needs current LinkedIn conversation signals before writing posts or campaign angles.

## When To Use This

- Build a Blocks agent that turns a topic into LinkedIn-informed content ideas.
- Recreate a Railway-hosted provider worker that researches LinkedIn posts with Apify.
- Adapt the workflow to another social-research or content-strategy agent.
- Debug a Blocks provider that starts locally but fails on Railway because of missing registration or stale credentials.

## Prerequisites

- Blocks Network account with permission to create private/free agents.
- Blocks API key available as `BLOCKS_API_KEY`; use the key from the active Blocks profile that owns the target `identity.agentName`.
- Blocks CLI and SDK available through the project dependencies: `@blocks-network/cli` and `@blocks-network/sdk`.
- Railway account or workspace access with permission to create projects, services, deployments, and service variables.
- Railway CLI installed and authenticated. For workspace/account-scoped automation, use `RAILWAY_API_TOKEN`; reserve `RAILWAY_TOKEN` for valid project-scoped tokens.
- Apify account with billing/access to the chosen LinkedIn actor.
- Apify API token configured as `APIFY_API_TOKEN`.
- OpenAI API key configured as `OPENAI_API_KEY` when LLM synthesis is desired. Without it, the agent should fall back to deterministic ideas.
- Node.js 24 or newer, npm, and a local shell.
- GitHub access if publishing the resulting documentation or project code.

Do not paste raw API keys into logs, docs, prompts, or Git commits. When setting Railway variables, prefer stdin-based commands.

## Required Inputs

- A topic string, for example `AI chat moderation`.
- Optional audience, for example `B2B SaaS product leaders`.
- Optional brand or market context, for example `PubNub`.
- Optional explicit LinkedIn search queries.
- Optional LinkedIn author/profile/company URLs to constrain results.
- Runtime environment variables:
  - `BLOCKS_API_KEY`
  - `APIFY_API_TOKEN`
  - `OPENAI_API_KEY`
  - `OPENAI_MODEL`, default `gpt-4.1-mini`
  - `APIFY_LINKEDIN_ACTOR_ID`, default `harvestapi/linkedin-post-search`
  - `APIFY_DATASET_LIMIT`, default around `150`

## Tools And Services

- Blocks Network: registers the agent and routes tasks to the provider worker.
- Railway: hosts the long-running Blocks provider process.
- Apify: runs LinkedIn post-search actors and stores scraped post results in datasets.
- OpenAI: synthesizes discussion summaries and content ideas from the evidence.
- Node/TypeScript: implements the handler, trigger, smoke test, and helper modules.
- GitHub CLI: publishes documentation or project artifacts when needed.

## Workflow

### 1. Scaffold The Agent Project

Create a small Node/TypeScript provider with this shape:

```text
agent-card.json
handler.ts
trigger.ts
smoke.ts
package.json
tsconfig.json
railway.json
.env.example
src/linkedinResearch.ts
src/topicPlanner.ts
src/contentIdeas.ts
```

Use this script pattern in `package.json`:

```json
{
  "scripts": {
    "check": "blocks check",
    "smoke": "tsx smoke.ts",
    "start": "blocks run",
    "trigger": "tsx trigger.ts",
    "typecheck": "tsc --noEmit"
  },
  "type": "module",
  "engines": {
    "node": ">=24"
  }
}
```

Install the core dependencies:

```bash
npm install @blocks-network/cli@latest @blocks-network/sdk@latest apify-client dotenv
npm install -D typescript tsx @types/node
```

### 2. Define The Blocks Agent Card

Use a stable underscore slug for Blocks:

```json
{
  "identity": {
    "agentName": "linkedin_topic_content_ideas",
    "displayName": "LinkedIn Topic Content Ideas",
    "description": "Researches how a topic is discussed on LinkedIn using Apify, then returns evidence-backed content ideas.",
    "version": "1.0.0"
  },
  "capabilities": {
    "taskKinds": ["request"]
  },
  "runtime": {
    "handler": "./handler.ts",
    "handlerExport": "default",
    "concurrency": 2,
    "expectedInstances": 1,
    "maxRunningTimeSec": 600
  }
}
```

Expose one required input part:

- `request`, `application/json`, with required `topic`.

Expose three output artifacts:

- `report`, `text/markdown`
- `ideas`, `application/json`
- `evidence`, `application/json`

### 3. Normalize The Request

In `topicPlanner.ts`, accept either plain text or JSON:

```json
{
  "topic": "AI chat moderation",
  "audience": "B2B SaaS product leaders",
  "brandContext": "PubNub",
  "postedLimit": "3months",
  "maxPostsPerQuery": 10,
  "maxQueries": 4,
  "ideaCount": 8,
  "includeComments": false,
  "includeReactions": false,
  "sortBy": "relevance"
}
```

Apply conservative limits:

- Topic length: 220 characters.
- `maxPostsPerQuery`: clamp between 5 and 50.
- `maxQueries`: clamp between 1 and 10.
- `ideaCount`: clamp between 3 and 20.
- `maxComments` and `maxReactions`: clamp between 1 and 25.
- Allowed `postedLimit` values: `any`, `1h`, `24h`, `week`, `month`, `3months`, `6months`, `year`.

Generate search queries from the topic, quoted topic, audience, strategy, lessons, challenges, trends, use cases, and brand context unless the user provides `searchQueries`.

### 4. Research LinkedIn Posts With Apify

In `linkedinResearch.ts`, use `apify-client` and default to:

```text
harvestapi/linkedin-post-search
```

Build actor input for post-search mode:

```json
{
  "searchQueries": ["AI chat moderation", "\"AI chat moderation\""],
  "maxPosts": 10,
  "postedLimit": "3months",
  "sortBy": "relevance",
  "scrapeReactions": false,
  "maxReactions": 5,
  "postNestedReactions": false,
  "scrapeComments": false,
  "maxComments": 5,
  "postNestedComments": false
}
```

If `authorUrls` or target URLs are provided, pass them as `authorUrls`. Read the actor dataset using `run.defaultDatasetId`, then normalize each result into a common post shape:

- Text/body
- Post URL
- Author name/headline/profile URL when present
- Published date when present
- Engagement counts
- Optional raw item when explicitly requested

Fail clearly when Apify returns no default dataset, an empty dataset, or items without recognizable post text/URLs.

### 5. Synthesize Ideas

In `contentIdeas.ts`, rank posts by a simple visible engagement score:

```text
reactions + 3 * comments + 5 * reposts
```

Extract pattern signals:

- Top recurring terms
- Hashtags
- Format signals such as question hooks, lists, announcements, contrarian posts, frameworks, and customer stories
- A short engagement note

If `OPENAI_API_KEY` is available, call the configured OpenAI model to create:

- A discussion summary
- Audience pain points
- Common frames
- White space
- Content ideas with title, hook, angle, format, outline, target audience, rationale, and supporting post URLs

If OpenAI is unavailable or returns unusable output, keep the task useful with deterministic fallback summaries and ideas.

### 6. Return Blocks Artifacts

In `handler.ts`, return three artifacts with stable output ids:

- `report`: Markdown with topic, audience, generated timestamp, Apify actor, dataset id, normalized post count, search queries, discussion summary, ideas, evidence highlights, and pattern signals.
- `ideas`: JSON with structured ideas and synthesis metadata.
- `evidence`: JSON with request, search queries, Apify dataset metadata, ranked posts, and pattern signals.

Use `ctx.reportStatus(...)` for user-visible progress:

- Prepared search queries.
- Running Apify actor.
- Normalized post count.
- Synthesizing content ideas.

### 7. Validate Locally

Run:

```bash
npm install
npm run typecheck
npm run check
```

Run a direct handler smoke test before Blocks registration:

```bash
npm run smoke
```

Run a small live smoke request to control Apify cost:

```bash
npm run smoke -- '{"topic":"AI agents for customer support","audience":"B2B SaaS product leaders","postedLimit":"month","maxQueries":1,"maxPostsPerQuery":5,"ideaCount":3,"includeComments":false,"includeReactions":false}'
```

Expected local smoke output includes saved files under `outputs/` and a summary such as `ideas=3 usedOpenAI=true`.

### 8. Register On Blocks

Authenticate the CLI:

```bash
blocks profile list
blocks whoami --json
```

If `blocks register` rejects `BLOCKS_API_KEY` but `blocks whoami` works, try the saved profile that owns the agent:

```bash
env -u BLOCKS_API_KEY blocks register --profile abc
```

Normally, register privately and free first:

```bash
blocks register
```

Expected success includes a private/free listing and a page like:

```text
https://app.blocks.ai/agents/linkedin_topic_content_ideas
```

Do not publish publicly or set paid pricing unless the user explicitly asks.

### 9. Deploy The Provider Worker To Railway

Create and link a Railway project and service:

```bash
railway init --name linkedin-topic-content-ideas --json
railway add --service linkedin-topic-content-ideas --json
```

Use `railway.json`:

```json
{
  "build": {
    "builder": "NIXPACKS"
  },
  "deploy": {
    "startCommand": "npm start",
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```

Set variables without printing raw values:

```bash
printf "%s" "$BLOCKS_API_KEY" | railway variable set BLOCKS_API_KEY --stdin --service linkedin-topic-content-ideas --environment production --skip-deploys
printf "%s" "$APIFY_API_TOKEN" | railway variable set APIFY_API_TOKEN --stdin --service linkedin-topic-content-ideas --environment production --skip-deploys
printf "%s" "$OPENAI_API_KEY" | railway variable set OPENAI_API_KEY --stdin --service linkedin-topic-content-ideas --environment production --skip-deploys
railway variable set APIFY_LINKEDIN_ACTOR_ID=harvestapi/linkedin-post-search --service linkedin-topic-content-ideas --environment production --skip-deploys
railway variable set OPENAI_MODEL=gpt-4.1-mini --service linkedin-topic-content-ideas --environment production --skip-deploys
railway variable set APIFY_DATASET_LIMIT=150 --service linkedin-topic-content-ideas --environment production --skip-deploys
```

Deploy:

```bash
railway up --detach --json --service linkedin-topic-content-ideas --environment production
```

If Railway uses a workspace/account token, prefer `RAILWAY_API_TOKEN`; if both `RAILWAY_API_TOKEN` and `RAILWAY_TOKEN` are set and auth acts strangely, unset the wrong one for that command.

### 10. Verify The Hosted Provider

Check Railway:

```bash
railway service status --service linkedin-topic-content-ideas --environment production --json
railway logs --latest --lines 160 --service linkedin-topic-content-ideas --environment production
```

Healthy logs should include:

```text
Agent instance ... started
registry billingMode: free
registered agent: linkedin_topic_content_ideas
```

Then call the registered Blocks agent:

```bash
npm run trigger -- '{"topic":"AI chat moderation","audience":"B2B SaaS product leaders","postedLimit":"3months","maxPostsPerQuery":10,"maxQueries":4,"ideaCount":8,"includeComments":false,"includeReactions":false}'
```

The reference run produced:

- Task id: `def6b14b-fac1-4a53-ba07-e94fc53b140b`
- Dataset id: `GDelreqRFNxpKIGAf`
- Normalized LinkedIn posts: `40`
- Synthesis: OpenAI `gpt-4.1-mini`

## Outputs

- A private/free Blocks agent named `linkedin_topic_content_ideas`.
- A Railway worker service named `linkedin-topic-content-ideas`.
- Markdown report artifacts for each topic run.
- Structured `ideas.json` with content ideas, hooks, angles, outlines, and supporting post URLs.
- Structured `evidence.json` with query and dataset metadata, ranked posts, and pattern signals.
- Local smoke artifacts under `outputs/`.

## Validation

Confirm all of the following:

- `npm run typecheck` passes.
- `npm run check` passes and validates `agent-card.json`.
- `npm run smoke` creates report, ideas, and evidence files.
- `blocks register` succeeds or confirms the agent is already registered under the intended org/profile.
- Railway deployment status is `SUCCESS` and `stopped` is false.
- Railway logs show the agent instance started and registered.
- `npm run trigger -- '{"topic":"AI chat moderation"}'` completes and emits `report`, `ideas`, and `evidence` artifacts.

## Safety And Quality Notes

- LinkedIn scraping through Apify can consume credits. Keep `maxQueries`, `maxPostsPerQuery`, `includeComments`, and `includeReactions` low during tests.
- Do not store raw secrets in Markdown files, Git commits, shell history, task artifacts, or logs.
- Railway variable list output can include raw values. Avoid sharing `railway variable list --json` or `--kv` output.
- Check for trailing whitespace in API tokens. A copied `APIFY_API_TOKEN` with a trailing space can break live calls.
- If Railway starts but Blocks logs say the agent is not found, make sure the agent was registered first and that Railway has the same `BLOCKS_API_KEY` as the Blocks profile that registered the agent.
- Treat LinkedIn evidence as directional research. Do not claim exhaustive market coverage.
- Avoid copying LinkedIn post text verbatim into generated marketing content. Use links and paraphrased evidence.
- Make OpenAI synthesis optional and keep deterministic fallbacks so the agent can still produce useful output during model/API failures.
