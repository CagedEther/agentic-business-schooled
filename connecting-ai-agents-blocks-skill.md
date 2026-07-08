# Connecting AI Agents With Blocks

## Overview

This workflow helps another LLM explain and reproduce the business pattern behind AI agents that can call each other. It uses Blocks as the practical agent network example, A2A as the broader interoperability concept, and the 5-Year-Old Translator as a small concrete Blocks agent. The goal is to show how a narrow agent can become a reusable business capability when it has clear inputs, outputs, credentials, validation, and a way for other agents to call it.

## When To Use This

- Teach business students why agent-to-agent workflows matter.
- Turn a simple AI capability into a Blocks-compatible provider agent.
- Create a beginner-friendly article about Blocks, A2A, and reusable agents.
- Show how a specialist agent can be called by another agent instead of living inside one chatbot.

## Prerequisites

- Blocks account and Blocks API key, so the agent can be registered, run, and called through the Blocks network.
- Node.js and npm, so the agent can be built with the Blocks SDK. The required SDK install command is:

```bash
npm install @blocks-network/sdk
```

- Blocks CLI if you are scaffolding, checking, registering, or running a provider agent locally:

```bash
npm install -g @blocks-network/cli
```

- LLM provider account and API key, such as an OpenAI API key, if the agent needs live model calls.
- Railway account and project access if the agent should run continuously as a hosted worker.
- GitHub access if you are publishing reusable instructions, examples, or documentation.
- A clear privacy boundary for what data the agent may receive from users or other agents.

## Required Inputs

- Article title. For the reference education page, use `Connecting AI agents and the power of Blocks`.
- Call to action URL. For the reference education page, use `https://agentic.business-schooled.com/quick-start/`.
- Agent example. Use the Blocks agent `5_year_old_translator`.
- Agent input schema. The translator accepts one text input inside the request payload: `original_text`.
- Agent output schema. The translator returns JSON with `translated_text` and `short_summary`.
- Business audience. Assume capable business professionals who may not know APIs, CLIs, SDKs, hosted workers, or agent protocols yet.

## Tools And Services

- Blocks: A network and operating layer for registering, running, discovering, and calling AI agents.
- `@blocks-network/sdk`: The Node SDK used by provider or consumer code to handle Blocks tasks and artifacts.
- Blocks CLI: Used for commands such as `blocks check`, `blocks register`, and `blocks run`.
- A2A, or [Agent2Agent](https://github.com/a2aproject/A2A): An open protocol concept for letting agents built by different vendors, frameworks, or teams discover capabilities and collaborate through standard task and message patterns.
- OpenAI or another LLM provider: Supplies the model that transforms input into useful output.
- Railway: Hosts the provider worker so the agent can stay online.
- GitHub: Stores reusable instructions and articles that other builders or AI assistants can reference.

## Workflow

### 1. Frame The Business Lesson

Start with the business idea, not the technology. Explain that the opportunity is moving from one large chatbot to a network of small specialist agents. Each agent should do one valuable job, publish a clear capability, accept structured inputs, and return structured outputs.

Use this framing:

- The old pattern is a standalone app or chatbot that tries to own the whole workflow.
- The agentic pattern is a set of callable capabilities that can be combined as work changes.
- Blocks gives a practical place to package and call those capabilities.
- A2A explains why standard agent communication matters across vendors, frameworks, and hosting environments.

### 2. Use A Narrow Agent Example

Use the 5-Year-Old Translator as the running example. It is intentionally small, which makes the agent-to-agent idea easier to see.

Agent name:

```text
5_year_old_translator
```

Input:

```json
{
  "original_text": "We need to leverage our core competencies to optimize synergistic deliverables before the end of the fiscal year."
}
```

Output:

```json
{
  "translated_text": "We need to use our best block-building skills to make the coolest sandbox castle before recess is over!",
  "short_summary": "The team should use its strengths to improve important work before the fiscal year ends."
}
```

Explain why this matters: a research agent, training agent, or customer-success agent could call this translator whenever it needs to turn confusing language into plain English.

### 3. Build Or Inspect The Blocks Agent Shape

For a Blocks-compatible Node project, make sure the project includes these core files:

- `agent-card.json`, which describes the agent identity, input schema, output schema, and runtime handler.
- `handler.ts`, which receives the Blocks task, validates `original_text`, calls the LLM, and returns an artifact.
- `trigger.ts`, which submits a test task to the local or registered agent.
- `.env.example`, which names required secrets without exposing real values.
- `railway.json`, if the agent should later run on Railway.

The package should include the Blocks SDK:

```bash
npm install @blocks-network/sdk
```

If using an LLM provider, also install and configure that provider SDK. Keep model keys in environment variables, not in source code.

### 4. Configure Secrets

Use environment variables for credentials:

```text
BLOCKS_API_KEY=
OPENAI_API_KEY=
OPENAI_MODEL=
```

`BLOCKS_API_KEY` lets the provider connect to Blocks. `OPENAI_API_KEY` or another model key lets the handler make live model calls. `OPENAI_MODEL` should be optional and should match the model available to the builder's account.

### 5. Validate The Agent Locally

Run the checks before treating the agent as reusable:

```bash
npm run typecheck
blocks check
blocks register
blocks run
```

In another terminal, trigger the sample task:

```bash
npm run trigger
```

A good local test proves that the task reaches the handler and returns the expected output artifact. If it fails because a model key is missing, report that clearly and do not hide the failure.

## Outputs

- A reusable skill-style Markdown file that tells another LLM how to reproduce the explanation and example.
- A beginner-friendly education article titled `Connecting AI agents and the power of Blocks`.
- Optional Blocks provider agent files if the builder is reproducing the 5-Year-Old Translator example.
- A likely Blocks app URL for the example agent: `https://app.blocks.ai/agents/5_year_old_translator`.

## Validation

- Confirm the skill file has a visible `Prerequisites` section.
- Confirm the education article links to the raw GitHub skill file.
- Confirm the education article includes the required CTA URL.
- Confirm the education article includes a Blocks app URL or clearly notes that publication/access needs confirmation.
- Confirm the education article explains A2A as interoperability across agents built in different places.
- Confirm no secrets, API keys, private customer data, or raw confidential records are included.
- If a Blocks agent is built, run `blocks check` and a local trigger test.

## Safety And Quality Notes

- Treat agents outside your direct control as untrusted unless you have reviewed their data handling, permissions, and outputs.
- Do not send confidential business data to a third-party agent unless the right agreements, access controls, and privacy rules are in place.
- Keep each agent narrow enough to test. A small, reliable agent is easier to connect than a vague general assistant.
- Use structured inputs and outputs so another agent can safely call the capability.
- Log task status and failures clearly. Silent failures make multi-agent workflows hard to trust.
- Budget for model and hosting costs before making an agent available to a team.
- Keep humans in the review loop for decisions that affect customers, money, compliance, hiring, legal terms, or safety.
