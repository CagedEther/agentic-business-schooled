# Build And Deploy A LinkedIn Company Report Agent

## Overview

This workflow builds a Blocks Network provider agent that accepts a company name, finds the company's LinkedIn company page, fetches recent public LinkedIn posts through Apify, and returns a concise markdown company report plus structured JSON evidence. It is useful for repeatable company research, sales preparation, marketing review, and partner or account intelligence workflows.

The reference implementation is a Node.js TypeScript Blocks agent named `linkedin_company_report_agent`. It runs as a long-lived Railway worker with `npm start`, uses Apify for LinkedIn post collection, and keeps all tokens in environment variables.

## When To Use This

- Use this when a team needs a reusable agent that turns a company name into a LinkedIn activity report.
- Use this when you want public LinkedIn company page discovery and post analysis without requiring the user to provide the LinkedIn URL.
- Use this when a Blocks workflow needs a private/free provider agent that returns both a human-readable markdown report and machine-readable research evidence.
- Use this as a pattern for adapting a person-profile LinkedIn report agent into a company-report agent.

## Prerequisites

- Blocks account and provider registration access, so the agent can be registered as `linkedin_company_report_agent` and called by Blocks tasks.
- `BLOCKS_API_KEY`, configured locally in `.env` for local runs and as a Railway service variable for hosted provider runs.
- Railway account, project access, and authenticated Railway CLI, so the provider worker can run continuously online.
- Apify account with API access and billing/usage capacity for the LinkedIn actor.
- `APIFY_API_TOKEN`, configured locally or in Railway service variables.
- Access to the Apify actor `harvestapi/linkedin-profile-posts`. This is the default actor used for public LinkedIn company-page posts.
- Optional SerpAPI or Bing Search credentials if you want a paid/controlled search provider for LinkedIn company page discovery:
  - `SERPAPI_API_KEY`
  - `BING_SEARCH_API_KEY` or `AZURE_BING_SEARCH_API_KEY`
  - `BING_SEARCH_ENDPOINT`
- Node.js 24 or newer, npm, TypeScript, and the local project dependencies.
- GitHub access only if publishing the reusable documentation or source to a repository.

No OpenAI API key is required for the reference implementation. The report rendering is deterministic TypeScript based on normalized LinkedIn post data.

## Required Inputs

The Blocks input part is named `request` and uses `application/json`.

Minimum request:

```json
{
  "companyName": "OpenAI"
}
```

Common request:

```json
{
  "companyName": "OpenAI",
  "maxPosts": 20,
  "postedLimit": "6months",
  "includeReposts": true,
  "includeComments": false,
  "includeReactions": false
}
```

Optional fields:

- `linkedinUrl`: full LinkedIn `/company/` URL. When supplied, web search discovery is skipped.
- `searchQuery`: override for the web search query.
- `searchResultLimit`: number of search results to inspect, capped at 20.
- `includeRaw`: include raw Apify items in the JSON artifact.
- `activityActorId`: request-level override for the Apify actor.
- `activityActorInputTemplate`: request-level override for the Apify actor input shape.

## Tools And Services

- Blocks Network: hosts the agent registry and task network. The provider is registered as private/free first.
- Railway: runs the provider worker continuously with `npm start`.
- Apify: runs the LinkedIn data actor and returns datasets of public LinkedIn posts.
- HarvestAPI LinkedIn profile posts actor: default Apify actor, `harvestapi/linkedin-profile-posts`, used for company page posts.
- DuckDuckGo HTML search: no-key fallback for LinkedIn company page discovery.
- SerpAPI or Bing Search: optional configured search providers for discovery.
- Node.js and TypeScript: runtime and implementation language.
- GitHub CLI and git: optional publishing workflow for reusable documentation or source.

## Workflow

### 1. Scaffold Or Copy The Agent Shape

Start from a working Node.js Blocks provider layout:

```text
agent-card.json
handler.ts
src/companySearch.ts
src/linkedinResearch.ts
trigger.ts
package.json
tsconfig.json
railway.json
.env.example
README.md
```

Use these package scripts:

```json
{
  "check": "blocks check",
  "start": "blocks run",
  "trigger": "tsx trigger.ts",
  "typecheck": "tsc --noEmit"
}
```

Use these dependencies:

```json
{
  "@blocks-network/cli": "latest",
  "@blocks-network/sdk": "latest",
  "apify-client": "latest",
  "dotenv": "^16.4.5"
}
```

### 2. Configure The Blocks Agent Card

Set `identity.agentName` to an underscore-safe Blocks name:

```json
"identity": {
  "agentName": "linkedin_company_report_agent",
  "displayName": "LinkedIn Company Report Agent",
  "description": "Creates a markdown report on a company from a company name, LinkedIn company page discovery, and recent LinkedIn activity using Apify.",
  "version": "1.0.0"
}
```

Declare one JSON input part named `request` with `companyName` required. Declare these outputs:

- `company_report` as `text/markdown`
- `research_json` as `application/json`

Set runtime values appropriate for a scraping-backed request:

```json
"runtime": {
  "handler": "./handler.ts",
  "handlerExport": "default",
  "concurrency": 2,
  "expectedInstances": 1,
  "maxRunningTimeSec": 600
}
```

### 3. Add LinkedIn Company Page Discovery

Implement `src/companySearch.ts` with this behavior:

- Accept `companyName`, optional `query`, and optional `resultLimit`.
- Build a default query such as `"OpenAI LinkedIn company page"`.
- Try providers in this order:
  - SerpAPI when `SERPAPI_API_KEY` is set.
  - Bing when `BING_SEARCH_API_KEY` or `AZURE_BING_SEARCH_API_KEY` is set.
  - DuckDuckGo HTML search as a no-key fallback.
- Normalize and accept only URLs under `https://www.linkedin.com/company/{slug}/`.
- Rank candidates by matching company tokens against title, snippet, and LinkedIn slug.
- Return selected URL, candidates, provider, query, and warnings.

Expose `normalizeLinkedInCompanyUrl(value)` so the handler can validate direct `linkedinUrl` inputs.

### 4. Add Apify LinkedIn Post Research

Implement `src/linkedinResearch.ts` around `apify-client`.

Default actors:

```ts
const DEFAULT_COMPANY_POSTS_ACTOR = 'harvestapi/linkedin-profile-posts';
const DEFAULT_POST_SEARCH_ACTOR = 'harvestapi/linkedin-post-search';
```

Build actor input with cost controls:

```json
{
  "targetUrls": ["{{target}}"],
  "maxPosts": "{{maxPosts}}",
  "postedLimit": "{{postedLimit}}",
  "includeQuotePosts": true,
  "includeReposts": true,
  "scrapeReactions": false,
  "maxReactions": 5,
  "postNestedReactions": false,
  "scrapeComments": false,
  "maxComments": 5,
  "postNestedComments": false
}
```

Keep actor and input shape configurable:

- `APIFY_LINKEDIN_ACTOR_ID`
- `APIFY_INPUT_TEMPLATE_JSON`
- `APIFY_DATASET_LIMIT`

Normalize common post fields across actor output shapes:

- Text: `text`, `postText`, `post_text`, `content`, `contentText`, `commentary`, `body`, `description`, `summary`
- URL: `postUrl`, `post_url`, `url`, `linkedinUrl`, `permalink`
- Author: `profileName`, `authorName`, `name`
- Date: `publishedAt`, `postedAt`, `posted_date`, `createdAt`, `datePublished`, `date`, `time`
- Engagement: reactions, comments, reposts, likes, shares, and common stats fields

### 5. Implement The Handler

In `handler.ts`:

1. Parse `task.requestParts` and read the JSON `request` part.
2. Require `companyName` unless a LinkedIn URL is provided and a company name can be inferred.
3. Clamp `maxPosts` to `1..50`.
4. Default `postedLimit` to `6months`.
5. Default `includeReposts` to true.
6. Keep `includeComments` and `includeReactions` false by default.
7. Find or validate the LinkedIn company URL.
8. Call `runLinkedInResearch` in `company-posts` mode.
9. Extract repeated messaging themes from post text.
10. Select up to three notable posts by engagement score.
11. Return markdown and JSON artifacts.

The markdown report should include:

- Snapshot
- Activity Signals
- Messaging Themes
- Content Strategy Signals
- Notable Recent Posts
- Follow-Up Angles
- Data Notes when warnings exist

The JSON artifact should include:

- Generated timestamp
- Request metadata without secret values
- Discovery result and candidates
- Normalized Apify activity result
- Report metadata and warnings

### 6. Configure Environment Variables

Use `.env.example` like this:

```bash
BLOCKS_API_KEY=
APIFY_API_TOKEN=

SERPAPI_API_KEY=
BING_SEARCH_API_KEY=
BING_SEARCH_ENDPOINT=https://api.bing.microsoft.com/v7.0/search

APIFY_LINKEDIN_ACTOR_ID=harvestapi/linkedin-profile-posts
APIFY_INPUT_TEMPLATE_JSON={"targetUrls":"{{targets}}","maxPosts":"{{maxPosts}}","postedLimit":"{{postedLimit}}","includeQuotePosts":true,"includeReposts":"{{includeReposts}}","scrapeReactions":"{{includeReactions}}","maxReactions":"{{maxReactions}}","postNestedReactions":false,"scrapeComments":"{{includeComments}}","maxComments":"{{maxComments}}","postNestedComments":false}
APIFY_DATASET_LIMIT=100
```

Never commit `.env`, secrets, API keys, or raw token values.

### 7. Validate Locally

Install and validate:

```bash
npm install
npm run typecheck
npm run check
```

Test company discovery without Apify:

```bash
npx tsx -e "import { findLinkedInCompanyUrl } from './src/companySearch.ts'; void (async () => { const result = await findLinkedInCompanyUrl({ companyName: 'OpenAI', resultLimit: 5 }); console.log(JSON.stringify({ provider: result.provider, selectedUrl: result.selectedUrl, warnings: result.warnings }, null, 2)); })();"
```

Expected discovery result:

```json
{
  "provider": "DuckDuckGo",
  "selectedUrl": "https://www.linkedin.com/company/openai/",
  "warnings": []
}
```

After Blocks registration and secrets are available, run a small live smoke test:

```bash
npm run trigger -- '{"companyName":"OpenAI","maxPosts":3,"postedLimit":"month","includeReposts":true,"includeComments":false,"includeReactions":false}'
```

Confirm that both `company_report` and `research_json` artifacts are returned.

### 8. Deploy To Railway

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

Create or link the Railway project and service:

```bash
railway init --name linkedin-company-report-agent --json
railway add --service linkedin-company-report-agent --json
```

Set non-secret defaults:

```bash
railway variable set --service linkedin-company-report-agent --environment production --skip-deploys \
  APIFY_LINKEDIN_ACTOR_ID=harvestapi/linkedin-profile-posts \
  APIFY_DATASET_LIMIT=100 \
  BING_SEARCH_ENDPOINT=https://api.bing.microsoft.com/v7.0/search \
  'APIFY_INPUT_TEMPLATE_JSON={"targetUrls":"{{targets}}","maxPosts":"{{maxPosts}}","postedLimit":"{{postedLimit}}","includeQuotePosts":true,"includeReposts":"{{includeReposts}}","scrapeReactions":"{{includeReactions}}","maxReactions":"{{maxReactions}}","postNestedReactions":false,"scrapeComments":"{{includeComments}}","maxComments":"{{maxComments}}","postNestedComments":false}'
```

Set secrets in Railway without printing values:

- `BLOCKS_API_KEY`
- `APIFY_API_TOKEN`
- Optional `SERPAPI_API_KEY`
- Optional `BING_SEARCH_API_KEY`

Deploy:

```bash
railway up --detach --json --service linkedin-company-report-agent --environment production --message "Deploy LinkedIn company report Blocks provider"
```

### 9. Register The Blocks Agent

Register privately and free before starting or restarting the worker:

```bash
blocks register
```

If using Railway-injected variables for CLI registration:

```bash
railway run --service linkedin-company-report-agent --environment production -- blocks register
```

The Blocks app URL for this agent is:

```text
https://app.blocks.ai/agents/linkedin_company_report_agent
```

After registration, restart the Railway service:

```bash
railway service restart --service linkedin-company-report-agent --environment production --yes --json
```

Check status and logs:

```bash
railway service status --json --service linkedin-company-report-agent --environment production
railway logs --latest --lines 120 --service linkedin-company-report-agent --environment production
```

Healthy logs should include a message like:

```text
registered agent: linkedin_company_report_agent
```

## Outputs

- `company_report`: markdown report with snapshot, activity signals, messaging themes, content strategy signals, notable posts, follow-up angles, and data notes.
- `research_json`: JSON evidence with request metadata, discovery result, normalized activity data, report metadata, actor IDs, dataset IDs, and warnings.
- Blocks provider registration: `linkedin_company_report_agent`, private/free by default.
- Railway worker service: `linkedin-company-report-agent`.

## Validation

- `npm run typecheck` passes.
- `npm run check` passes Blocks card validation.
- Company discovery resolves a LinkedIn `/company/` URL for a known company.
- Railway service status is `SUCCESS` and not stopped.
- Railway logs show the Blocks instance started and registered.
- A live trigger with `maxPosts: 3` returns both artifacts.
- The JSON artifact includes discovery, actor ID, dataset ID, warnings, and normalized posts.

## Safety And Quality Notes

- Do not print, commit, log, or include `BLOCKS_API_KEY`, `APIFY_API_TOKEN`, SerpAPI keys, or Bing Search keys in docs or output artifacts.
- Keep comments and reactions disabled by default because they increase Apify cost and output size.
- Cap `maxPosts` in the handler. The reference handler caps requests at 50 posts.
- Treat LinkedIn data as public web data that can be incomplete, stale, rate-limited, or removed.
- Include `research_json` so future debugging does not require re-running Apify unnecessarily.
- If discovery picks the wrong company page, pass `linkedinUrl` directly or tune `searchQuery`.
- If Apify returns sparse records, inspect the raw dataset with `includeRaw: true` before swapping actors.
- Re-register or republish the Blocks agent after changing `agent-card.json` input/output schemas.
