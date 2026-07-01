# Build A Product Marketing Interview Blocks Agent

## Overview

This workflow builds and operates a Blocks Network provider agent that generates Product Marketing Manager interview practice questions. The agent uses a built-in markdown background document as its source of truth, optionally personalizes questions from an interviewee profile, and returns questions, questions with answers, and structured JSON artifacts. It is designed to run as a long-lived Blocks worker, usually deployed on Railway.

## When To Use This

- Build a reusable interview-practice agent for a Product Marketing Manager role.
- Turn a role or exam background document into structured interview questions and answer rubrics.
- Deploy a Blocks provider worker to Railway and verify it with a hosted task.
- Adapt the same pattern to another role by replacing the built-in markdown background and card metadata.

## Prerequisites

- Blocks account with access to the target organization and permission to register provider agents.
- Blocks CLI authenticated for the correct organization. Verify with `blocks whoami --json`.
- Blocks API key for the provider runtime, configured as `BLOCKS_API_KEY` locally and on Railway.
- OpenAI API key with billing access for live model calls, configured as `OPENAI_API_KEY`.
- Optional `OPENAI_MODEL`, defaulting to `gpt-4.1-mini` in the current project.
- Railway account, project access, and Railway CLI auth for hosted deployment.
- Node.js 20 or newer and npm.
- GitHub access if publishing the workflow or related documentation.
- A markdown topic background file, currently `docs/prod-mkt-manager-bg.md`.
- Optional markdown interviewee profile file, currently shaped like `docs/profile-template.md` or `samples/profile.md`.

## Required Inputs

- `docs/prod-mkt-manager-bg.md`: built-in role and topic background used by the handler at runtime.
- `request` Blocks input, `application/json`, optional:

```json
{
  "questionCount": 1
}
```

- `profile_doc` Blocks input, `text/markdown`, optional. Include prior questions, areas of expertise, areas for improvement, and coaching notes.
- Local `.env` values:

```bash
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4.1-mini
```

## Tools And Services

- Blocks Network: registry, task routing, provider runtime, and artifact transport.
- `@blocks-network/sdk`: handler types, task client, artifact publishing, and trigger scripts.
- `@blocks-network/cli`: `blocks check`, `blocks register`, and `blocks run`.
- Railway: hosts the long-running `npm start` provider worker.
- OpenAI Chat Completions API: generates JSON question and answer content from the background and optional profile.
- TypeScript and `tsx`: local implementation and trigger execution.

## Workflow

### 1. Scaffold The Project

Use a simple Node/TypeScript Blocks provider shape:

```text
agent-card.json
handler.ts
trigger.ts
package.json
tsconfig.json
railway.json
.env.example
docs/prod-mkt-manager-bg.md
docs/profile-template.md
scripts/smoke-handler.ts
```

Use package scripts like:

```json
{
  "check": "blocks check",
  "start": "blocks run",
  "trigger": "tsx trigger.ts",
  "typecheck": "tsc --noEmit",
  "smoke:handler": "tsx scripts/smoke-handler.ts"
}
```

### 2. Write The Agent Card

Create `agent-card.json` with:

- `identity.agentName`: `abc_interview_agent`
- `identity.displayName`: `ABC Interview Agent`
- `capabilities.taskKinds`: `["request"]`
- `runtime.handler`: `./handler.ts`
- `runtime.maxRunningTimeSec`: about `180` for an LLM-backed request task
- `io.inputs`: optional `request` JSON and optional `profile_doc` markdown
- `io.outputs`: `questions`, `questions_and_answers`, and `questions_json`

Use `tags`, not the older `skills` key, if the current Blocks CLI expects it.

### 3. Add The Built-In Background

Put the source-of-truth role background in `docs/prod-mkt-manager-bg.md`. For this project it describes a B2B technology Product Marketing Manager interview, including:

- interviewer role and expertise
- candidate level
- PMM responsibilities
- positioning, messaging, launch, enablement, competitive intelligence, customer research, pricing, adoption, and metrics
- evaluation criteria and difficulty bands

To adapt the agent to another role, replace this markdown file and update the card description and examples.

### 4. Implement The Handler

In `handler.ts`:

1. Read the built-in topic background from `docs/prod-mkt-manager-bg.md`.
2. Parse optional `profile_doc` from `task.requestParts`.
3. Parse `questionCount` from the optional `request` part.
4. Clamp question count to a practical range, currently `1` to `20`.
5. Prompt OpenAI to return JSON with:
   - `title`
   - `questionCount`
   - `questions[]`
   - `nextFocusAreas[]`
6. Normalize and validate the model output.
7. Return three artifacts:
   - `questions.md`
   - `questions-and-answers.md`
   - `questions.json`

The handler should fail clearly when `OPENAI_API_KEY` is missing, unless `INTERVIEW_AGENT_DRY_RUN=true` is set for local smoke tests.

### 5. Add Local Smoke And Trigger Scripts

Use `scripts/smoke-handler.ts` to call the handler directly. If no local `OPENAI_API_KEY` exists, set `INTERVIEW_AGENT_DRY_RUN=true` so the smoke test can still verify artifact shape without spending model tokens.

Use `trigger.ts` to send a real Blocks task:

```bash
npm run trigger
npm run trigger -- 1
npm run trigger -- samples/profile.md 3
```

The trigger should save artifacts under `outputs/<task-id>/`.

### 6. Validate Locally

Run:

```bash
npm ci
npm run typecheck
npm run check
npm run smoke:handler
```

Expected results:

- TypeScript compiles.
- `blocks check` passes card schema validation and finds `handler.ts`.
- Smoke test writes `questions.md`, `questions-and-answers.md`, and `questions.json`.

### 7. Register With Blocks

Confirm the correct Blocks account and organization before registry work:

```bash
blocks whoami --json
```

If the wrong org is active, switch the CLI profile or login again before registering. Register privately and free:

```bash
blocks register --org-name <org-name>
```

Re-run `blocks register` after changing `agent-card.json`, especially after changing input or output schemas. If the registry says the agent is owned by another org, delete the wrong registration or choose a new `identity.agentName`.

### 8. Deploy To Railway

Create or link a Railway project and service. The worker command is:

```bash
npm start
```

Set service variables without printing secret values:

```bash
printf '%s' "$BLOCKS_API_KEY" | railway variable set BLOCKS_API_KEY --stdin
printf '%s' "$OPENAI_API_KEY" | railway variable set OPENAI_API_KEY --stdin
railway variable set OPENAI_MODEL=gpt-4.1-mini
```

Deploy:

```bash
railway up --detach --json --message "Deploy interview Blocks provider"
```

Inspect logs:

```bash
railway logs --latest --lines 120
```

Look for an `agent_instance_started` event followed by `agent_registered`.

### 9. Run A Hosted Task

After the Railway worker is running and the registry card matches the deployed code:

```bash
npm run trigger -- 1
```

Verify artifacts are saved under `outputs/<task-id>/`. Read `questions.md` for the interview question and `questions-and-answers.md` for the expected answer and rubric.

## Outputs

- Blocks agent registered as a private/free provider, for example `abc_interview_agent`.
- Railway service running `npm start` as a long-lived Blocks worker.
- `questions.md`: generated interview questions.
- `questions-and-answers.md`: questions, expected answers, follow-up probes, and evaluation criteria.
- `questions.json`: structured payload for a separate frontend.
- Optional education or handoff docs explaining how to rebuild the workflow.

## Validation

- `npm run typecheck` exits successfully.
- `npm run check` reports all Blocks card checks passing.
- `npm run smoke:handler` writes all three artifacts locally.
- Railway logs show the provider instance is running and registered.
- A hosted `npm run trigger -- 1` task completes and saves artifacts.
- Generated questions are grounded in `docs/prod-mkt-manager-bg.md` and respect `profile_doc` when provided.

## Safety And Quality Notes

- Never commit `.env`, raw API keys, Railway variables, Blocks API keys, OpenAI keys, or generated private interview outputs.
- Keep `outputs/`, `node_modules/`, local logs, and local videos out of deploy or commit contexts unless they are intentionally published.
- Verify `blocks whoami --json` before registration. Registering under the wrong org can create ownership conflicts for `identity.agentName`.
- Re-register Blocks after changing `agent-card.json`; redeploy Railway after changing `handler.ts`, `docs/prod-mkt-manager-bg.md`, dependencies, or runtime config.
- Keep `questionCount` bounded to avoid surprise OpenAI cost and overly large artifacts.
- Use `INTERVIEW_AGENT_DRY_RUN=true` only for local shape checks. Do not treat dry-run questions as production-quality interview content.
- Avoid putting candidate private data, confidential company information, or sensitive interview notes into `profile_doc` unless the environment is approved for that data.
- If Railway CLI reports expired auth, refresh with `railway login` or use the approved Railway token flow before deploy.
