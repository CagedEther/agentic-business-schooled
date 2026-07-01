# Build And Operate A Keyword Space Analyst Blocks Agent

## Overview

This workflow builds a Blocks Network provider agent that turns a topic into a keyword-space report using DataForSEO. The agent accepts a topic or topic description, expands it into keyword ideas, clusters the keyword universe by intent and recurring modifiers, and adds a live SERP landscape showing top ranking sites for the seed topic. It runs as a long-lived Node/TypeScript worker on Railway and returns Markdown plus JSON artifacts.

Current reference implementation:

- Blocks agent name: `seo_analyst_agent`
- Blocks display name: `Keyword Space Analyst`
- Blocks app URL: `https://app.blocks.ai/agents/seo_analyst_agent`
- Railway project: `seo-analysis-agent`

## When To Use This

- Build a topic-first SEO research agent for content strategy, product marketing, or editorial planning.
- Analyze keyword demand, search intent, clusters, and recurring modifiers without requiring a target domain.
- Add page-one SERP context, including top ranking sites and content categories, to a keyword research workflow.
- Reproduce or adapt the `Keyword Space Analyst` Blocks provider for another niche, market, or DataForSEO account.

## Prerequisites

- Blocks account with provider registration access, so the agent can be registered and called through Blocks.
- `BLOCKS_API_KEY`, configured in local `.env` and Railway variables.
- DataForSEO account with API access and enough credits for keyword and SERP requests.
- `DATAFORSEO_USERNAME` and `DATAFORSEO_PASSWORD`, configured in local `.env` and Railway variables.
- Railway account and project/service access, so the provider worker can run continuously online.
- Railway CLI or Railway MCP access for deploys, variables, logs, and service status.
- Node.js 20 or newer, npm, TypeScript, and the package dependencies in `package.json`.
- GitHub access only if you are publishing reusable instructions or source code.

Do not paste credentials into chat or documentation. Keep `.env`, Railway variables, and any secret files out of git.

## Required Inputs

The Blocks task accepts one JSON input part with part id `request`.

Minimum input:

```json
{
  "topic": "chat moderation"
}
```

Typical input:

```json
{
  "topic": "chat moderation",
  "topicDescription": "Tools, services, and best practices for moderating chat in online communities and apps.",
  "locationCode": 2840,
  "languageCode": "en",
  "device": "desktop",
  "suggestionLimit": 50,
  "relatedLimit": 50,
  "serpDepth": 10,
  "includeRelatedKeywords": true,
  "includeSerpLandscape": true
}
```

Important fields:

- `topic`: required seed topic or short phrase.
- `topicDescription`: optional context for the market, product, audience, or angle.
- `locationCode` / `locationName`: DataForSEO market selection. `2840` is United States.
- `languageCode` / `languageName`: DataForSEO language selection. `en` is English.
- `device`: `desktop` or `mobile`.
- `suggestionLimit`: number of keyword suggestions to request, capped by the agent.
- `relatedLimit`: number of related keywords to request, capped by the agent.
- `serpDepth`: number of live Google organic results to request for the seed topic.
- `includeRelatedKeywords`: whether to expand with related keywords.
- `includeSerpLandscape`: whether to include top ranking sites and SERP features.

## Tools And Services

- Blocks Network: hosts the agent listing, task schema, request dispatch, and returned artifacts.
- `@blocks-network/sdk`: implements the provider handler and trigger client.
- `@blocks-network/cli`: validates the agent card, runs the local provider, and registers the agent.
- DataForSEO: provides keyword metrics, keyword ideas, related keywords, and live Google SERP data.
- Railway: runs the Blocks provider worker continuously.
- Node.js / TypeScript / npm: local runtime and build tooling.
- GitHub: optional place to store reusable workflow documentation or source.

## Workflow

### 1. Scaffold Or Inspect The Blocks Provider

Use a root-level Node/TypeScript provider with these files:

```text
agent-card.json
handler.ts
trigger.ts
package.json
tsconfig.json
railway.json
.env.example
.railwayignore
README.md
```

The package scripts should include:

```json
{
  "check": "blocks check",
  "start": "blocks run",
  "trigger": "tsx trigger.ts",
  "typecheck": "tsc --noEmit"
}
```

Keep `.env` out of source control and deployment bundles.

### 2. Define The Blocks Agent Card

Use a stable Blocks identity:

```json
{
  "identity": {
    "agentName": "seo_analyst_agent",
    "displayName": "Keyword Space Analyst",
    "description": "Analyzes a topic area with DataForSEO keyword metrics, clusters, SERP features, and top ranking sites.",
    "version": "0.2.0"
  }
}
```

The input should be `application/json` with required `topic`. The outputs should be:

- `keyword_report`, `text/markdown`, guaranteed.
- `keyword_data`, `application/json`, guaranteed.

Set `runtime.handler` to `./handler.ts`, `handlerExport` to `default`, and a reasonable `maxRunningTimeSec`, such as `180`.

### 3. Implement The DataForSEO Handler

The handler should:

1. Parse the `request` part as JSON.
2. Normalize and bound the topic, market, limits, and toggles.
3. Read DataForSEO credentials from environment variables only.
4. Call DataForSEO with a bounded set of requests.
5. Merge duplicate keywords by normalized keyword.
6. Build keyword clusters by intent and modifiers.
7. Extract live SERP top ranking sites when enabled.
8. Return both Markdown and JSON artifacts.

Use these DataForSEO endpoints:

```text
POST /v3/dataforseo_labs/google/keyword_overview/live
POST /v3/dataforseo_labs/google/keyword_suggestions/live
POST /v3/dataforseo_labs/google/related_keywords/live
POST /v3/serp/google/organic/live/advanced
```

Recommended outputs:

- Executive summary.
- Seed keyword metrics.
- Cluster table with keyword count, summed volume, average CPC, competition, and opportunity rating.
- Top keywords inside each cluster.
- Recurring modifiers.
- Top ranking sites and page-one content categories.
- Content strategy recommendations.
- Raw normalized JSON for downstream agents or dashboards.

### 4. Add A Local Trigger

Use `TaskClient` and `textPart` from `@blocks-network/sdk` to submit a request:

```typescript
const request = {
  topic: "chat moderation",
  locationCode: 2840,
  languageCode: "en",
  device: "desktop",
  suggestionLimit: 50,
  relatedLimit: 50,
  serpDepth: 10,
  includeRelatedKeywords: true,
  includeSerpLandscape: true
};
```

Send it to:

```typescript
agentName: "seo_analyst_agent"
```

Print task id, progress, artifact filenames, and a short preview of the Markdown report.

### 5. Configure Local Secrets

Create `.env` from `.env.example`:

```bash
cp .env.example .env
```

Required variables:

```text
BLOCKS_API_KEY=
DATAFORSEO_USERNAME=
DATAFORSEO_PASSWORD=
```

Never print these values. Use a secret-safe presence check instead:

```bash
node -r dotenv/config -e "for (const k of ['BLOCKS_API_KEY','DATAFORSEO_USERNAME','DATAFORSEO_PASSWORD']) console.log(k + '=' + (process.env[k] ? 'set' : 'missing'))"
```

### 6. Validate Locally

Run:

```bash
npm install
npm run typecheck
npm run check
```

Start the local provider only when you are ready to receive Blocks tasks:

```bash
npm start
```

In another terminal:

```bash
npm run trigger -- "chat moderation"
```

A good test returns:

- A completed Blocks task.
- `keyword-space-report.md`.
- `keyword-space-data.json`.
- Railway or local logs showing `Task <id> completed` when running as a provider.

### 7. Register Or Update The Blocks Agent

Fetch the latest Blocks guidance before live registry work when possible:

```bash
curl -fsSL https://config.blocks.ai/SKILL.md
```

Validate first:

```bash
npm run check
```

Register privately and free:

```bash
blocks register --org-name <your-org-name>
```

The reference implementation was registered as private/free under:

```text
https://app.blocks.ai/agents/seo_analyst_agent
```

Use `blocks publish` only when the user explicitly wants public listing or paid usage.

### 8. Deploy To Railway

Use `railway.json` with:

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

Use `.railwayignore` to exclude:

```text
.env
node_modules/
artifacts/
*.log
.DS_Store
```

Set Railway service variables without printing values. If the shell has a stale `RAILWAY_API_TOKEN`, unset it for CLI calls:

```bash
env -u RAILWAY_API_TOKEN railway whoami
```

Deploy:

```bash
env -u RAILWAY_API_TOKEN railway up --detach --json --message "Deploy Keyword Space Analyst"
```

Verify:

```bash
env -u RAILWAY_API_TOKEN railway service status --json
env -u RAILWAY_API_TOKEN railway logs --latest --lines 200
```

Look for:

```text
registered agent: seo_analyst_agent
Task <task-id> completed
```

### 9. Run A Hosted End-To-End Test

With the local provider stopped and Railway running:

```bash
npm run trigger -- "chat moderation"
```

Expected result:

- Blocks task state: completed.
- `keyword-space-report.md`, around several KB.
- `keyword-space-data.json`, often hundreds of KB because it includes normalized source data.
- Railway logs show the exact task id completed.

## Outputs

- `keyword_report`: Markdown report for business users and content strategists.
- `keyword_data`: JSON artifact with request, summary, keywords, clusters, SERP landscape, warnings, and source responses.
- Blocks listing or private agent page.
- Railway long-running provider deployment.

## Validation

Use this checklist:

- `npm run typecheck` passes.
- `npm run check` passes.
- DataForSEO credentials are present, not printed.
- Blocks registration succeeds for `seo_analyst_agent`.
- Railway service status is `SUCCESS` and not stopped.
- Railway logs show `registered agent: seo_analyst_agent`.
- A hosted trigger for `chat moderation` returns both artifacts.
- Railway logs show the hosted task id completed.

Reference validation from the completed build:

- Railway service id: `58822c90-dba4-43bb-a886-390e367b1777`.
- Railway deployment id: `493d59b0-e5ad-47c7-a39f-af2410490d1d`.
- Hosted test task id: `136613e7-753c-4558-b890-67ed0e8ce046`.

## Safety And Quality Notes

- DataForSEO calls spend credits. Keep default limits bounded and explain before broad runs.
- Do not include `.env`, raw API keys, tokens, or credential values in docs, commits, logs, or chat.
- Keyword volumes, SERP results, and CPC values are market and time sensitive. Treat reports as decision support, not permanent truth.
- The clustering is heuristic. Review clusters before using them as final information architecture.
- Separate job-seeker keywords from buyer or product-led content when the SERP is mixed.
- Register privately/free first. Publish publicly only after the user approves listing and pricing decisions.
- Keep Railway as a long-running worker process with `npm start`; do not convert this provider into an HTTP server unless a separate API is required.
