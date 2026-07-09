# Build a LinkedIn Tone Guide Blocks Agent

## Overview

This workflow builds a Blocks Network provider agent that accepts a LinkedIn profile URL or slug, fetches recent public LinkedIn posts through Apify, and turns the writing patterns into a portable markdown tone-of-voice guide. The output is designed for downstream content agents that need specific voice guidance before drafting posts, articles, campaigns, or executive ghostwriting.

The reference implementation is a Node TypeScript Blocks agent named `linkedin_tone_guide`. It runs as a long-lived provider process locally or on Railway, uses Apify for LinkedIn post collection, and uses OpenAI to generate the final markdown guide.

## When To Use This

- Build a reusable voice-analysis agent from public LinkedIn posts.
- Create downstream-agent guidance for executive ghostwriting, founder-led content, sales content, or thought leadership.
- Rebuild or adapt the `linkedin_tone_guide` Blocks agent in a new project.
- Debug an Apify-backed LinkedIn research workflow where actor output fields vary by actor.
- Deploy a non-HTTP Blocks provider worker to Railway.

## Prerequisites

- A Blocks Network account, `BLOCKS_API_KEY`, and permission to register or publish the target provider agent.
- A Railway account and project/service access if deploying the provider worker to Railway.
- An Apify account, API token, and actor access for LinkedIn post scraping.
- An OpenAI API key with billing access for live style-guide generation.
- Node.js 24 or newer and npm.
- The Blocks CLI and SDK via project dependencies: `@blocks-network/cli` and `@blocks-network/sdk`.
- Public LinkedIn profile URLs or slugs. The workflow should not depend on private LinkedIn sessions or browser cookies.
- Optional GitHub access if source control or documentation publishing is part of the build.

## Required Inputs

- `account`: LinkedIn profile URL, profile slug, username, or equivalent account identifier.
- Optional `maxPosts`: integer post limit. The reference handler defaults to 20 and caps at 50.
- Optional `focus`: the intended content use case, such as `thought-leadership posts for B2B audiences`.
- Optional `brandContext`: how the resulting guide will be used.
- Optional `includePsychology`: boolean. When true, include Big Five and empathy-map sections as writing-style heuristics only.
- Optional `includeExamples`: boolean. When true, include example phrasing patterns.
- Optional `language`: output language. Defaults to English.
- Optional `customInstructions`: caller-specific guidance for the final style guide.

Example request:

```json
{
  "account": "https://www.linkedin.com/in/williamhgates/",
  "maxPosts": 20,
  "focus": "thought-leadership posts for B2B audiences",
  "brandContext": "Use this guide for executive ghostwriting.",
  "includePsychology": true,
  "includeExamples": true,
  "language": "English",
  "customInstructions": ""
}
```

## Tools And Services

- Blocks Network: registers and runs the provider agent, receives task requests, and returns artifacts.
- Railway: hosts the long-lived Blocks provider worker with `npm start`.
- Apify: runs the LinkedIn posts actor and returns a default dataset.
- OpenAI: analyzes normalized post data and writes the markdown style guide.
- Node TypeScript: implements the handler, trigger script, validation, and local development flow.

## Workflow

### 1. Scaffold The Project

Create a Node TypeScript project with this basic shape:

```text
agent-card.json
handler.ts
trigger.ts
package.json
package-lock.json
tsconfig.json
railway.json
.env.example
.gitignore
.dockerignore
.railwayignore
```

Use package scripts like:

```json
{
  "check": "blocks check",
  "start": "blocks run",
  "trigger": "tsx trigger.ts",
  "typecheck": "tsc --noEmit"
}
```

Use Node `>=24`, ESM modules, and dependencies:

```json
{
  "@blocks-network/cli": "latest",
  "@blocks-network/sdk": "latest",
  "apify-client": "latest",
  "dotenv": "^16.4.5",
  "openai": "latest"
}
```

### 2. Define The Blocks Agent Card

Create `agent-card.json` with a stable agent name. The reference project uses:

```json
{
  "identity": {
    "agentName": "linkedin_tone_guide",
    "displayName": "LinkedIn Tone Guide",
    "description": "Analyzes LinkedIn posts from a requested profile and returns a markdown tone-of-voice style guide for downstream content agents."
  },
  "capabilities": {
    "taskKinds": ["request"]
  },
  "runtime": {
    "handler": "./handler.ts",
    "handlerExport": "default",
    "concurrency": 2,
    "expectedInstances": 1,
    "maxRunningTimeSec": 300
  }
}
```

Define one required input part named `request` with `contentType` set to `application/json`. Define one guaranteed output named `style_guide` with `contentType` set to `text/markdown`.

The likely Blocks app URL for the reference agent is `https://app.blocks.ai/agents/linkedin_tone_guide`. Confirm publication and access in Blocks before sharing it externally.

### 3. Configure Environment Variables

Create `.env.example`:

```bash
BLOCKS_API_KEY=
APIFY_API_TOKEN=
OPENAI_API_KEY=

# Optional overrides
OPENAI_MODEL=gpt-5.1
OPENAI_MAX_OUTPUT_TOKENS=6000
APIFY_LINKEDIN_POSTS_ACTOR=unseenuser/linkedin-content
APIFY_INPUT_TEMPLATE_JSON={"profileUrls":["{{account}}"],"totalPostsPerProfile":"{{maxPosts}}","includeReposts":true,"removeEmptyFields":false}
ANALYSIS_CHAR_LIMIT=45000
```

Keep `.env`, `node_modules/`, generated `outputs/`, and local coverage/log artifacts out of git, Docker, and Railway build contexts.

### 4. Implement The Handler

The handler should:

1. Parse the Blocks `request` part.
2. Accept either JSON or a plain account string.
3. Validate that an `account` is present.
4. Clamp `maxPosts` to a safe range.
5. Fetch posts through Apify.
6. Normalize post text and metadata.
7. Fail clearly when the dataset has no usable text.
8. Ask OpenAI for a markdown-only style guide.
9. Return one `text/markdown` artifact with output id `style_guide`.

Keep Apify actor selection configurable:

```ts
const actorId = process.env.APIFY_LINKEDIN_POSTS_ACTOR ?? 'unseenuser/linkedin-content';
```

Use this default actor input:

```json
{
  "profileUrls": ["{{account}}"],
  "totalPostsPerProfile": "{{maxPosts}}",
  "includeReposts": true,
  "removeEmptyFields": false
}
```

When hydrating `APIFY_INPUT_TEMPLATE_JSON`, preserve numbers. If the whole string is `"{{maxPosts}}"`, return a number, not a string. This matters because some Apify actors reject numeric fields passed as strings.

Normalize multiple actor output shapes. At minimum support:

- Text: `text`, `postText`, `post_text`, `content`, `contentText`, `commentary`, `body`, `description`, `summary`, `headline`, `title`, `postTitle`
- Post URL: `postUrl`, `post_url`, `url`, `linkedinUrl`
- Date: `publishedAt`, `postedAt`, `posted_date`, `createdAt`, `datePublished`, `date`, `time`
- Profile: `profileName`, `authorName`, `name`, `profileUrl`, `profile_url`, `authorUrl`, `author_profile_url`
- Engagement: `reactionCount`, `likeCount`, `likes`, `totalReactions`, `stats_total_reactions`, `commentCount`, `stats_comments`, `repostCount`, `stats_reposts`

### 5. Prompt The Style Guide

Build the OpenAI instructions so the output is a practical markdown file for another agent. Require sections like:

- Source Snapshot
- Executive Voice Summary
- Semantic Patterns
- Personality And Communication Signals
- Empathy Map
- Signature Content Moves
- Syntax, Cadence, And Formatting
- Lexicon: Use / Avoid
- Do / Do Not Guidance
- Downstream Agent Prompt Block
- Confidence Notes And Data Limits

Every major claim should cite post indices such as `[P4]`. Frame psychology sections as writing-style heuristics only, never clinical or definitive claims about the author.

### 6. Add A Local Trigger

Create `trigger.ts` so a builder can run a real Blocks task from the command line:

```bash
npm run trigger -- https://www.linkedin.com/in/williamhgates/
```

The trigger should create a `TaskClient`, send a `request` part, print progress events, download the returned artifact, and close the session on terminal events.

### 7. Validate Locally

Run:

```bash
npm install
npm run typecheck
npm run check
```

For the reference project, `blocks check` passes and validates:

- `agent-card.json` exists.
- JSON is valid.
- Schema validation passes.
- `identity.agentName` is `linkedin_tone_guide`.
- Handler exists at `./handler.ts`.
- IDs are unique.

### 8. Register, Publish, And Run With Blocks

Use current Blocks CLI instructions when doing live registry work. A typical flow is:

```bash
blocks login --write-env
blocks publish --billing-mode free --listing public --accept-terms
npm start
```

If using a hosted Railway provider, ensure the provider process is online before relying on hosted task execution.

### 9. Deploy To Railway

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

Set Railway service variables without printing secret values:

- `BLOCKS_API_KEY`
- `APIFY_API_TOKEN`
- `OPENAI_API_KEY`

Optional Railway variables:

- `OPENAI_MODEL`
- `OPENAI_MAX_OUTPUT_TOKENS`
- `APIFY_LINKEDIN_POSTS_ACTOR`
- `APIFY_INPUT_TEMPLATE_JSON`
- `ANALYSIS_CHAR_LIMIT`

Deploy the service and inspect logs until the Blocks provider is registered and waiting for tasks.

### 10. Run An End-To-End Test

Send a request for a public LinkedIn profile. Confirm that progress includes:

- Fetching recent LinkedIn posts.
- Running the configured Apify actor.
- Analyzing the normalized posts.

Confirm the returned artifact is a markdown file named like:

```text
linkedin-tone-guide-samanthasmitte.md
```

The reference test produced a style guide with 20 sampled posts and sections for source snapshot, executive voice summary, semantic patterns, personality and communication signals, empathy map, lexicon guidance, do/do-not rules, and downstream-agent prompt guidance.

## Outputs

- A Blocks provider agent named `linkedin_tone_guide`.
- A markdown artifact with output id `style_guide`.
- A portable tone-of-voice guide that can be pasted into another agent's prompt or used as a writing reference.
- Optional Railway deployment running the provider with `npm start`.

## Validation

- `npm run typecheck` succeeds.
- `npm run check` succeeds.
- Railway deployment logs show a healthy provider process when deployed.
- A Blocks task returns exactly one markdown artifact.
- The guide cites sampled post numbers for major claims.
- The guide includes confidence notes and data limits.
- The guide clearly frames Big Five and empathy-map content as writing-style heuristics.

## Safety And Quality Notes

- Do not commit `.env` or print API keys.
- Do not use private LinkedIn data or private browser sessions. Use public LinkedIn profile data returned by Apify.
- Apify actors can change names, permissions, pricing, and output fields. If the agent returns no usable text, inspect the raw Apify dataset before changing the OpenAI prompt.
- Keep `maxPosts` capped for cost and prompt-size control.
- Keep `ANALYSIS_CHAR_LIMIT` and `OPENAI_MAX_OUTPUT_TOKENS` configurable.
- Treat the resulting guide as writing guidance, not a psychological diagnosis or factual profile of the person.
- Do not use the guide to impersonate someone deceptively. Use it for approved ghostwriting, brand consistency, or style adaptation with human review.
- If using another Apify actor, update `APIFY_INPUT_TEMPLATE_JSON` and add field aliases to the normalizer rather than hard-coding a one-off path.
