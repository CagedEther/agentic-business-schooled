# Write a Technical Blog Post From a Researched Content Brief

## Overview

This is Part Two of the PubNub content workflow. Part One uses the [PubNub Content Plan Orchestrator](https://app.blocks.ai/agents/pubnub_content_plan_orchestrator) to research a topic and recommend five posts. Part Two starts after a human selects one recommendation and uses the [PubNub Technical Blog Writer](https://app.blocks.ai/agents/pubnub_technical_blog_writer) to turn that approved brief into a reviewable Markdown draft.

The writing agent treats the brief as the editorial authority, reruns the content-plan orchestrator for article-specific background research, and returns the draft separately from an inspectable writing trace.

## When To Use This

- Turn an approved recommendation from the initial topic-research process into a full technical article.
- Draft an SEO-aware developer guide, tutorial, comparison, explainer, or technical thought-leadership post.
- Preserve the audience, angle, keywords, outline, evidence requirements, and business CTA chosen during editorial planning.
- Create a draft with enough trace information for an editor or subject-matter expert to review how it was produced.

Do not use this as the first step for a broad topic. Run the [content-research workflow](https://raw.githubusercontent.com/CagedEther/agentic-business-schooled/refs/heads/main/pubnub-content-research-orchestrator-skill.md) first, compare its five recommendations, and approve one article direction.

## Prerequisites

- A completed content-plan recommendation or an equivalent human-approved editorial brief.
- Access to the private Blocks agent `pubnub_technical_blog_writer`. Contact the agent owner for an organization invitation when needed.
- A Blocks account and valid consumer access. A scripted integration also needs a `BLOCKS_API_KEY` stored outside source control.
- Node.js and npm only when calling the agent through `@blocks-network/sdk`; the Blocks app can be used without a local Node project.
- An editor or subject-matter expert who can approve public claims, technical guidance, product statements, code, and links before publication.
- No personal OpenAI or Railway credentials are required to use the already-hosted agent. Those accounts and service variables are maintainer prerequisites, not writer prerequisites.

Never paste private customer data, credentials, embargoed information, or unapproved claims into the brief.

## Required Inputs

Only `overview` is technically required, but a strong brief should carry forward the approved fields from Part One:

- Working title.
- Primary and secondary keywords.
- Search intent and funnel stage.
- Target audience and assumed knowledge.
- Reader problem and desired payoff.
- Distinctive angle and reason the article should win.
- Required outline or must-cover points.
- One call to action.
- Source URLs or internal links that must appear.
- Claims, statistics, code, or product details that require SME review.

The published request also accepts these optional structured fields:

- `topic`
- `audience`
- `readerIntent`: `learn`, `solve`, `compare`, or `justify`
- `businessGoal`
- `brandContext`
- `tone`
- `targetKeyword`
- `desiredLengthWords`: 800 to 4,000
- `callToAction`
- `downstreamTimeoutMs`
- `includeRawResearchInTrace`
- `synthesisMode`: `auto`, `openai`, or `deterministic`

Use `synthesisMode: "openai"` when a deterministic fallback would not be acceptable for publication work. Use `auto` for ordinary drafting and `deterministic` only for workflow testing or continuity.

## Tools And Services

- [PubNub Technical Blog Writer](https://app.blocks.ai/agents/pubnub_technical_blog_writer) accepts the brief and returns the draft and trace.
- [PubNub Content Plan Orchestrator](https://app.blocks.ai/agents/pubnub_content_plan_orchestrator) runs again inside the writer to narrow SEO, external-blog, PubNub corpus, and LinkedIn research around the selected article.
- [Blocks Network](https://blocks.ai/) routes the hosted task and artifacts.
- [OpenAI](https://openai.com/) writes the normal model-generated draft from the brief and gathered research.
- [Railway](https://railway.com/) keeps the private writer provider available as a long-running worker.

The second research pass is intentional. Part One helps the team decide what to write; the writer's research pass checks the selected post in greater detail before drafting it.

## Workflow

### 1. Select One Recommendation From Part One

Review the initial content plan and choose one recommendation based on business priority, audience value, evidence quality, content gap, and ability to support the proposed CTA.

Do not send all five recommendations to the writer and ask it to choose. Editorial selection belongs to the team. The writing agent should receive one clear article direction.

### 2. Convert The Recommendation Into An Editorial Brief

Copy the selected recommendation into `overview`. Preserve its specific language where it represents an approved decision.

Use a structure like this:

```text
Working title: Cart Abandonment: A Developer's Guide to Causes, Metrics, and Recovery Strategies

Primary keyword: cart abandonment
Secondary keywords: shopping cart abandonment; cart abandonment rate; reduce cart abandonment
Intent / funnel: informational / awareness
Audience: Developers, engineering leaders, and product leaders looking for a technical yet holistic primer
Reader problem: Teams need one credible technical reference covering causes, metrics, and recovery channels.
Distinctive angle: Combine engineering tradeoffs in performance, identity, and instrumentation with email, push, in-app, and conversational recovery choices.
Required CTA: Read the three technical tutorials or request a PubNub architecture review.

Required outline:
1. Definition and why it matters to engineers.
2. Metrics and instrumentation.
3. Common technical causes.
4. Recovery-channel engineering tradeoffs.
5. Measurement and experimentation.
6. Product and engineering roadmap.
7. Links to tactical tutorials.
```

If the article must link to specific tutorials or product pages, include the exact approved URLs. The writer is instructed not to invent links that are absent from the brief or research.

### 3. Submit The Request

Open the [writer agent](https://app.blocks.ai/agents/pubnub_technical_blog_writer) and enter the brief, or send the JSON request through the Blocks SDK.

Example payload:

```json
{
  "overview": "Working title: Cart Abandonment: A Developer's Guide to Causes, Metrics, and Recovery Strategies\n\nPrimary keyword: cart abandonment\nSecondary keywords: shopping cart abandonment; cart abandonment rate; reduce cart abandonment\nAudience: Developers, engineering leaders, and product leaders\nReader problem: Explain causes, metrics, and recovery channels in one technical pillar.\nRequired outline: definition; metrics; technical causes; channel tradeoffs; experiments; roadmap; tutorial links.",
  "topic": "cart abandonment",
  "audience": "developers, engineering leaders, and product leaders",
  "readerIntent": "learn",
  "businessGoal": "create an authoritative awareness-stage pillar",
  "brandContext": "PubNub real-time application infrastructure",
  "tone": "practical, precise, technically credible, and developer-friendly",
  "targetKeyword": "cart abandonment",
  "desiredLengthWords": 1800,
  "callToAction": "Read the technical tutorials or request a PubNub architecture review.",
  "synthesisMode": "openai"
}
```

Minimal Node usage:

```ts
import { TaskClient, textPart } from '@blocks-network/sdk';

const client = await TaskClient.create({
  apiKey: process.env.BLOCKS_API_KEY!,
  billingMode: 'free',
});

const session = await client.sendMessage({
  agentName: 'pubnub_technical_blog_writer',
  taskKind: 'request',
  requestParts: [textPart(JSON.stringify(request), 'request')],
});

const terminal = await session.waitForTerminal(45 * 60 * 1000);
if (terminal.state !== 'completed') throw new Error('Writing task failed.');

const paths = await session.saveArtifacts('./output');
await session.asyncClose();
client.destroy();
console.log(paths);
```

### 4. Let The Article-Specific Research Finish

The writer calls `pubnub_content_plan_orchestrator` again using the approved brief as context. That downstream task normally checks:

1. Keyword demand and search intent.
2. Leading external coverage and competitive patterns.
3. Existing PubNub articles and internal-link opportunities.
4. Recent LinkedIn discussion and audience signals.

The writer then receives a Markdown content plan, structured content-plan JSON, and a research trace. It uses those materials as background without allowing them to replace the approved article direction.

### 5. Collect Both Outputs

The writer returns two guaranteed artifacts:

- `blog_draft`: Markdown containing the title, meta description, target keyword when applicable, thesis, full article, one CTA, and distribution kit.
- `writing_trace`: JSON containing the original request, downstream research task, artifact metadata, model metadata, warnings, and whether OpenAI or deterministic writing was used.

Save both. The Markdown is the editorial artifact; the trace is the audit and troubleshooting artifact.

### 6. Review The Draft Against The Brief

Check the draft before line editing:

- Does it preserve the approved title or improve it without changing the promise?
- Does it answer the stated reader problem and match the audience's knowledge level?
- Do the H2 headings cover the required outline in a logical order?
- Is the primary keyword used naturally in the title, opening, and relevant headings?
- Is there exactly one business CTA?
- Are code examples runnable or clearly labeled as pseudocode?
- Are statistics, benchmarks, product statements, APIs, and URLs supported by supplied research?
- Are missing links or unresolved claims explicitly flagged instead of invented?
- Is the meta description close to 155 characters?
- Does the distribution kit contain a useful social hook, standalone asset, and channel recommendation?

Do not treat `Publication draft complete` as human publication approval.

### 7. Inspect The Writing Trace

A healthy trace should show:

- `backgroundResearch.state` is `completed`.
- Research artifacts include `content_plan`, `content_plan_json`, and `research_trace`.
- `writing.mode` is `openai` for the normal publication path.
- `writing.wordCount` is reasonably close to `desiredLengthWords`.
- `warnings` is empty, or every warning has been reviewed and resolved.

Keep `includeRawResearchInTrace` false for normal work. Raw research can be large and may contain material that should not be redistributed.

### 8. Run The Human Accuracy And Editing Passes

Use separate passes:

1. Accuracy: verify claims, metrics, code, API behavior, product capabilities, and every public link.
2. Structure: remove sections that do not advance the thesis and confirm the argument works end to end.
3. Line edit: cut repetition, hedging, generic marketing language, and unnecessary words.
4. Introduction: rewrite it last so it accurately promises the article's real payoff.
5. Packaging: shorten the meta description, confirm one CTA, and approve the distribution kit.

Ask the relevant PubNub product or engineering owner to approve technical claims before publication.

### 9. Hand Off For Publishing

Move the approved Markdown into the normal CMS or editorial workflow. Preserve the writing trace with the working files so reviewers can understand the research coverage, model path, and outstanding warnings.

The agent prepares a draft for publishing; it does not publish automatically.

## Outputs

- A full technical blog draft in Markdown.
- A JSON writing trace with research and model metadata.
- A distribution kit containing a social hook, standalone asset idea, and channel suggestions.
- Explicit SME-review flags when evidence or product details remain uncertain.

## Validation

- The Blocks task reaches `completed` and the hosted provider remains online.
- Both `blog_draft` and `writing_trace` are present.
- The article covers every must-have section from the approved brief.
- The primary keyword and audience are reflected naturally.
- The CTA appears once and matches the approved business next step.
- The trace shows the downstream research task completed and lists all three research artifacts.
- Publication claims, code, and links pass human review.

A live `cart abandonment` test produced a 1,816-word draft, completed all four research branches, gathered three background artifacts, used OpenAI writing, returned no runtime warnings, and still surfaced editorial work: the meta description needed shortening, requested tutorial URLs were missing from the supplied sources, and one push-notification delivery claim needed technical verification. This is the expected operating model: automation creates a strong draft; humans approve the truth.

## Safety And Quality Notes

- Never include secrets, customer records, private analytics, or confidential roadmap details in the request.
- Treat SEO metrics, LinkedIn discussion, external articles, and model synthesis as research signals, not verified truth.
- Verify every public statistic and benchmark against a primary source or internal data.
- Verify product and API claims against current PubNub documentation and an SME.
- Do not invent missing internal links. Supply approved URLs in the brief or leave an explicit editorial placeholder.
- Use one CTA. More choices weaken the article and make attribution harder.
- Keep source and output limits bounded; the internal research pass may call several providers and incur model or data costs.
- A deterministic fallback is lower confidence than an OpenAI draft and should always receive additional editing.
- Preserve the trace for internal review, but do not publish it with the article.
