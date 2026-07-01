# Company Profile Research Agent Workflow

## Overview

This workflow builds and deploys a Blocks provider agent that takes a company name, researches the company with the OpenAI Responses API web search tool, and returns a sourced Markdown company profile. The agent is designed for business research, sales preparation, partner diligence, and executive briefing workflows where users need a current, cited summary rather than a generic model answer.

The reference implementation is a Node.js and TypeScript Blocks agent hosted as a long-running worker on Railway. It is registered on Blocks as `company_profile_research_agent` and can be viewed at `https://app.blocks.ai/agents/company_profile_research_agent` when the user has access to the private agent.

## When To Use This

- Use this when you need a reusable Blocks agent that turns a company name into a structured research brief.
- Use this when business teams want sourced Markdown output that can be saved, reviewed, or passed into another agent.
- Use this when you want to deploy a provider agent on Railway and keep it online for other Blocks workflows.

## Prerequisites

- Blocks account and provider registration access, so the agent can be registered as a private or public Blocks agent.
- Blocks API key configured as `BLOCKS_API_KEY` locally and in Railway. This lets the provider worker authenticate and lets consumer scripts call private agents.
- Railway account and project access, so the provider worker can run continuously online.
- OpenAI API key with billing access configured as `OPENAI_API_KEY`. The handler uses live OpenAI Responses API calls with web search.
- Node.js 24 or later, npm, and a local shell.
- Local CLIs: `blocks`, `railway`, `gh`, and `git` when reproducing the full build, deploy, registration, and documentation workflow.
- GitHub access if publishing reusable instructions or code artifacts.
- No secrets should be committed. Keep `.env`, `.npmrc`, `node_modules`, logs, and generated outputs out of deploy archives and source control.

## Required Inputs

- `companyName`: the company to research.

Example Blocks request part:

```json
{
  "companyName": "PubNub"
}
```

Required environment variables:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
```

Optional environment variables:

```bash
OPENAI_MODEL=gpt-5.5
OPENAI_SEARCH_CONTEXT_SIZE=medium
```

`OPENAI_SEARCH_CONTEXT_SIZE` accepts `low`, `medium`, or `high`.

## Tools And Services

- Blocks Network: registers the provider agent, receives tasks, and routes artifacts back to callers.
- Railway: hosts the long-running Blocks provider worker.
- OpenAI Responses API: researches current company facts with the hosted `web_search` tool and drafts the Markdown profile.
- Node.js and TypeScript: implement the provider handler, trigger script, and local validation.
- GitHub: stores reusable documentation or source code when needed.

## Workflow

### 1. Create The Blocks Agent Project

Use a Node.js provider agent structure:

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

Recommended package scripts:

```json
{
  "check": "blocks check",
  "start": "blocks run",
  "trigger": "tsx trigger.ts",
  "typecheck": "tsc --noEmit"
}
```

The reference package uses:

- `@blocks-network/cli`
- `@blocks-network/sdk`
- `dotenv`
- `openai`
- `tsx`
- `typescript`

### 2. Define The Agent Card

Set the Blocks identity and IO contract in `agent-card.json`.

Important fields:

- `identity.agentName`: `company_profile_research_agent`
- `identity.displayName`: `Company Profile Research Agent`
- `capabilities.taskKinds`: `["request"]`
- input part id: `request`
- input content type: `application/json`
- required input field: `companyName`
- output artifact ids: `company_profile` and `research_json`
- runtime handler: `./handler.ts`
- runtime max time: `300` seconds

The Blocks app URL for the deployed private agent is:

```text
https://app.blocks.ai/agents/company_profile_research_agent
```

### 3. Implement The Handler

The handler should:

1. Load `OPENAI_API_KEY` from the environment.
2. Parse `task.requestParts` and accept either JSON or plain text.
3. Validate `companyName`, keeping it reasonably short. The reference implementation caps it at 160 characters.
4. Call `client.responses.create()` with:
   - `model`: default `gpt-5.5`, overridable with `OPENAI_MODEL`
   - `tools`: `[{ type: "web_search", search_context_size: "medium", external_web_access: true }]`
   - `tool_choice`: `required`
   - `include`: `["web_search_call.action.sources"]`
   - `store`: `false`
5. Ask OpenAI to return Markdown with these sections:
   - Snapshot
   - What The Company Does
   - Products, Customers, And Use Cases
   - Business Model
   - Market Position And Competitors
   - Recent Signals
   - Leadership And Footprint
   - Risks And Open Questions
   - Sources
6. Extract cited URLs and web search sources from the OpenAI response.
7. Return two Blocks artifacts:
   - `company_profile`: `text/markdown`
   - `research_json`: `application/json`

### 4. Add A Trigger Script

Create `trigger.ts` so the agent can be called from a local terminal or through Railway variables:

```bash
npm run trigger -- "PubNub"
```

For private agents, the consumer key must belong to the owning Blocks org or an invited org. If local auth is in a different org, use Railway's service variables for the smoke test:

```bash
railway run --service company-profile-research-agent --environment production -- npm run trigger -- "PubNub"
```

### 5. Validate Locally

Run:

```bash
npm install
npm run typecheck
npm run check
```

`npm run check` should confirm:

- `agent-card.json` exists
- JSON is valid
- schema validation passes
- the handler exists
- IDs are unique

### 6. Configure Railway

Use `railway.json` to make Railway run the provider as a worker:

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

Add `.railwayignore` so secrets and local dependencies are not uploaded:

```text
.env
.env.*
.npmrc
node_modules
outputs
*.log
pip.conf
```

Create or link the project:

```bash
railway init --name company-profile-research-agent --json
railway add --service company-profile-research-agent --json
```

Set Railway variables without printing their values:

```bash
railway variables --set "BLOCKS_API_KEY=$BLOCKS_API_KEY"
railway variables --set "OPENAI_API_KEY=$OPENAI_API_KEY"
railway variables --set "OPENAI_MODEL=gpt-5.5"
railway variables --set "OPENAI_SEARCH_CONTEXT_SIZE=medium"
```

Deploy:

```bash
railway up --service company-profile-research-agent --environment production --detach --json
```

### 7. Register The Blocks Agent

Authenticate with Blocks and register the agent private and free:

```bash
blocks login --write-env
blocks register
```

If the Railway worker starts before registration, logs may show that the agent is not found in the registry. Register first, then restart the Railway service:

```bash
railway service restart --service company-profile-research-agent --environment production --yes --json
```

### 8. Verify The Hosted Provider

Check Railway:

```bash
railway status --json
railway deployment list --json
railway logs --latest --lines 120
```

Healthy logs should show the Blocks agent instance started and registered, for example:

```text
agent_instance_started
registered agent: company_profile_research_agent
```

Run an end-to-end task:

```bash
railway run --service company-profile-research-agent --environment production -- npm run trigger -- "PubNub"
```

The reference build completed a PubNub smoke test through Blocks and returned the `company_profile` Markdown artifact.

## Outputs

- A deployed Blocks provider agent named `company_profile_research_agent`.
- Blocks app URL: `https://app.blocks.ai/agents/company_profile_research_agent`.
- Railway project/service named `company-profile-research-agent`.
- `company_profile` artifact: sourced Markdown company profile.
- `research_json` artifact: model, source, usage, and generated Markdown metadata.

## Validation

- `npm run typecheck` passes.
- `npm run check` passes Blocks card validation.
- Railway latest deployment status is `SUCCESS`.
- Railway instance status is `RUNNING`.
- Railway logs show the Blocks agent instance started.
- A Blocks task for `PubNub` completes and returns a Markdown profile artifact.

## Safety And Quality Notes

- Do not commit `.env`, API keys, Railway variables, Blocks keys, OpenAI keys, `.npmrc`, logs, or generated outputs.
- The OpenAI web search tool can incur cost and latency. Keep `max_output_tokens`, `maxRunningTimeSec`, and search context size bounded.
- Require citations and source links in the output. Treat generated company facts as research drafts for review, not as guaranteed truth.
- Private Blocks agents can only be called by the owning org or invited orgs. Use `blocks invite send` if another organization needs access.
- If `blocks register` is re-run after a public or paid publish, it can reset visibility to private and billing to free. Use `blocks publish` for later public or paid updates.
- Avoid using this workflow for confidential target lists or private customer data unless the relevant data policies allow external model and search usage.
