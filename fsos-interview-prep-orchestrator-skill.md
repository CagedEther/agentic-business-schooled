# FSOS Interview Prep Orchestrator Workflow

## Overview

This workflow rebuilds or operates a Blocks Network orchestrator agent that turns four interview-prep inputs into a markdown background report for the Future State of Streaming series. The orchestrator calls three specialist research agents in parallel, gathers their markdown and JSON artifacts, and uses OpenAI to synthesize the findings with the `fsos-one-pager.md` positioning document.

The result is a repeatable business workflow for interview preparation, content planning, account research, and executive briefing. The current Blocks agent is `fsos_interview_prep_orchestrator`, with the likely app URL `https://app.blocks.ai/agents/fsos_interview_prep_orchestrator`.

## When To Use This

- Build or reproduce an FSOS interview-prep agent that combines company research, LinkedIn company activity, LinkedIn person research, and an internal point-of-view document.
- Update the orchestrator when a faster or better downstream research agent becomes available.
- Deploy the orchestrator as a long-running Railway provider worker for Blocks.
- Run a smoke test that confirms all downstream agents complete and the final markdown report is synthesized.
- Adapt the same pattern for sales prep, executive briefings, content research, partnership research, or speaker prep.

## Prerequisites

- Blocks account and provider registration access, so the orchestrator and sub-agents can be registered and called by name.
- `BLOCKS_API_KEY`, configured locally and in Railway for the orchestrator.
- OpenAI API key with billing enabled, configured as `OPENAI_API_KEY` for final synthesis and for the company profile research sub-agent.
- Railway account, project access, and Railway CLI authentication for deploying the long-running provider service.
- Apify account and `APIFY_API_TOKEN` for the LinkedIn company and LinkedIn person sub-agents.
- Node.js 24 or later, npm, and the local project dependencies installed with `npm install`.
- Blocks CLI authentication for `blocks check`, `blocks run`, and `blocks register`.
- Local `fsos-one-pager.md`, copied into the orchestrator project or referenced with `FSOS_ONE_PAGER_PATH`.
- Access to the three downstream Blocks agents:
  - `company_profile_research_agent`
  - `linkedin_company_report_agent`
  - `linkedin_person_report_agent`

## Required Inputs

The orchestrator request must be JSON with these fields:

```json
{
  "companyName": "DAZN",
  "companyUrl": "https://www.dazn.com/",
  "personName": "Jane Doe",
  "linkedinUrl": "https://www.linkedin.com/in/janedoe/"
}
```

Optional request fields:

```json
{
  "maxPosts": 20,
  "postedLimit": "6months",
  "companyResearchAgentName": "company_profile_research_agent",
  "companyLinkedInReportAgentName": "linkedin_company_report_agent",
  "personResearchAgentName": "linkedin_person_report_agent",
  "downstreamTimeoutMs": 1800000,
  "includeRaw": false,
  "billingMode": "free"
}
```

## Tools And Services

- Blocks Network runs the orchestrator and lets it call other registered agents by name.
- Railway hosts the provider worker continuously with `npm start`, which runs `blocks run`.
- OpenAI synthesizes the final markdown report from the downstream research packet and FSOS one-pager.
- Apify powers the LinkedIn company and person research agents by collecting structured public LinkedIn data.
- GitHub stores reusable documentation, including this skill file, so another AI builder can reproduce the workflow.
- `company_profile_research_agent` creates a sourced company profile using OpenAI web search.
- `linkedin_company_report_agent` discovers or accepts a LinkedIn company page, fetches recent public posts through Apify, and returns messaging and activity signals.
- `linkedin_person_report_agent` fetches public LinkedIn profile details and recent activity through Apify and returns a concise person report.

## Workflow

### 1. Inspect The Existing Project

Work from the orchestrator directory:

```bash
cd fsos_interview_prep_orchestrator
```

Confirm these files exist:

- `agent-card.json`
- `handler.ts`
- `trigger.ts`
- `package.json`
- `railway.json`
- `fsos-one-pager.md`

Check that `agent-card.json` identifies the agent as `fsos_interview_prep_orchestrator` and that the required inputs are `companyName`, `companyUrl`, `personName`, and `linkedinUrl`.

### 2. Configure The Orchestrator Environment

Use these environment variables locally and in Railway:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_SYNTHESIS_MODEL=gpt-4.1
OPENAI_SYNTHESIS_MAX_OUTPUT_TOKENS=7000
OPENAI_SYNTHESIS_INPUT_CHARS=90000
BLOCKS_BILLING_MODE=free
COMPANY_RESEARCH_AGENT_NAME=company_profile_research_agent
COMPANY_LINKEDIN_REPORT_AGENT_NAME=linkedin_company_report_agent
PERSON_RESEARCH_AGENT_NAME=linkedin_person_report_agent
DOWNSTREAM_TASK_TIMEOUT_MS=1800000
FSOS_ONE_PAGER_PATH=./fsos-one-pager.md
```

Do not print secret values in logs, documentation, commits, or chat responses.

### 3. Validate The Project Locally

Install dependencies and run the static checks:

```bash
npm install
npm run typecheck
npm run check
```

`npm run check` runs the Blocks CLI validation against `agent-card.json` and the handler configuration.

### 4. Confirm Downstream Agent Contracts

The orchestrator runs three downstream tasks concurrently.

For `company_profile_research_agent`, send:

```json
{
  "companyName": "DAZN",
  "companyUrl": "https://www.dazn.com/"
}
```

For `linkedin_company_report_agent`, send:

```json
{
  "companyName": "DAZN",
  "maxPosts": 20,
  "postedLimit": "6months",
  "includeReposts": true,
  "includeComments": false,
  "includeReactions": false,
  "includeRaw": false
}
```

For `linkedin_person_report_agent`, send:

```json
{
  "linkedinUrl": "https://www.linkedin.com/in/janedoe/",
  "maxPosts": 20,
  "postedLimit": "6months",
  "includeProfileDetails": true,
  "includeReposts": true,
  "includeComments": false,
  "includeReactions": false,
  "includeRaw": false
}
```

The orchestrator can still support legacy prompt-based company agents such as `gemini_deep_research_rw`, but the current default is `company_profile_research_agent` because it completes faster.

### 5. Register Or Update The Blocks Provider

After Blocks CLI authentication, register the orchestrator from the orchestrator directory:

```bash
blocks register
```

Use private/free registration unless the project owner explicitly asks to publish differently. If the registration step asks for a provider or agent update, keep the existing agent name `fsos_interview_prep_orchestrator` unless the user requests a new agent.

### 6. Deploy To Railway

Link or create the Railway project from the orchestrator directory, then set variables without exposing values:

```bash
railway link
railway variables --set "BLOCKS_API_KEY=$BLOCKS_API_KEY"
railway variables --set "OPENAI_API_KEY=$OPENAI_API_KEY"
railway variables --set "OPENAI_SYNTHESIS_MODEL=gpt-4.1"
railway variables --set "OPENAI_SYNTHESIS_MAX_OUTPUT_TOKENS=7000"
railway variables --set "OPENAI_SYNTHESIS_INPUT_CHARS=90000"
railway variables --set "BLOCKS_BILLING_MODE=free"
railway variables --set "COMPANY_RESEARCH_AGENT_NAME=company_profile_research_agent"
railway variables --set "COMPANY_LINKEDIN_REPORT_AGENT_NAME=linkedin_company_report_agent"
railway variables --set "PERSON_RESEARCH_AGENT_NAME=linkedin_person_report_agent"
railway variables --set "DOWNSTREAM_TASK_TIMEOUT_MS=1800000"
railway variables --set "FSOS_ONE_PAGER_PATH=./fsos-one-pager.md"
railway up --detach
```

Check status and logs:

```bash
railway status
railway logs
```

The expected Railway state is one running provider instance with no crashed replicas.

### 7. Run A Smoke Test Through Blocks

Use the local trigger script after the provider is registered and running:

```bash
npm run trigger -- '{"companyName":"PubNub","companyUrl":"https://pubnub.com","personName":"Daryl Pereira","linkedinUrl":"https://www.linkedin.com/in/darylpereira/"}'
```

For a business-facing production smoke test, choose a real company, company URL, interviewee name, and LinkedIn `/in/` URL.

### 8. Inspect The Outputs

The provider returns two artifacts:

- `background_report`: markdown report for interview prep and content generation.
- `research_json`: structured metadata, downstream task IDs, warnings, source artifact summaries, matched FSOS pillars, and synthesis metadata.

The markdown report should include:

- Executive brief
- Interview target
- Company background
- Interviewee snapshot
- Why the guest fits Future State of Streaming
- Strongest FSOS alignment
- Interview prep angles
- Suggested questions
- Content generation hooks
- Data notes

### 9. Troubleshoot Common Failures

If Railway shows a crashed service, inspect logs first. Common causes are missing environment variables, a failed Blocks registration, incompatible Node version, or a provider worker started from the wrong directory.

If the orchestrator starts but downstream research fails, check that each sub-agent is deployed, registered, and has its required variables:

- `company_profile_research_agent`: `BLOCKS_API_KEY`, `OPENAI_API_KEY`
- `linkedin_company_report_agent`: `BLOCKS_API_KEY`, `APIFY_API_TOKEN`
- `linkedin_person_report_agent`: `BLOCKS_API_KEY`, `APIFY_API_TOKEN`

If OpenAI synthesis fails or `OPENAI_API_KEY` is absent, the handler uses its deterministic markdown fallback. The fallback is useful for continuity, but the best business report comes from the OpenAI synthesis stage.

## Outputs

- A registered Blocks provider agent named `fsos_interview_prep_orchestrator`.
- A Railway service running the orchestrator provider worker.
- A Blocks app page, likely `https://app.blocks.ai/agents/fsos_interview_prep_orchestrator`.
- Markdown interview prep reports returned as `background_report`.
- JSON trace artifacts returned as `research_json`.
- Optional local output files if the trigger script or test harness stores artifacts in an `outputs/` directory.

## Validation

- `npm run typecheck` succeeds.
- `npm run check` succeeds.
- `railway status` shows the orchestrator service deployed and running.
- `railway logs` shows the provider waiting for Blocks tasks, not repeatedly exiting or crashing.
- A smoke test returns both `background_report` and `research_json`.
- In `research_json`, the downstream sections for `company`, `companyLinkedIn`, and `person` each contain a completed task ID or an explicit warning.
- The synthesis metadata reports `mode: "openai"` when `OPENAI_API_KEY` is configured correctly.
- The markdown report is specific to the target company and interviewee, names source limitations clearly, and does not invent facts that are absent from downstream outputs.

## Safety And Quality Notes

- Treat API keys and tokens as private credentials. Never commit `.env` files or paste secrets into chat, logs, screenshots, or documentation.
- LinkedIn data should be limited to public, business-relevant information. Do not use the workflow for sensitive personal profiling or private-data inference.
- Apify usage can incur cost. Keep comments and reactions disabled by default unless the business need justifies larger pulls.
- OpenAI synthesis should use only the source packet returned by the sub-agents and the FSOS one-pager. The report should separate evidence from interpretation.
- Company and LinkedIn research can be incomplete or stale. Include data notes and use the JSON trace when business decisions require source review.
- Keep the orchestrator modular. Swap downstream agents through environment variables or request overrides instead of hard-coding new dependencies whenever possible.
- For business users, position the output as a preparation brief, not as a final fact-checked research dossier.
