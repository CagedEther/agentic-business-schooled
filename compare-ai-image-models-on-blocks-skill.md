# Run a Fair Head-to-Head Comparison of AI Image Models on the Blocks Network

## Overview

This workflow sets up a repeatable bake-off between two or more AI image generation models,
using a single Blocks Network agent as the test harness. Because one agent owns the prompt
handling, the aspect-ratio mapping and the API keys, each model receives an identical brief,
so any difference in the output is attributable to the model rather than to how it was asked.
The result is a defensible model choice based on your own briefs instead of vendor marketing
or leaderboard scores.

The reference implementation compares OpenAI `gpt-image-1` against Google
`gemini-3-pro-image`, but the method generalizes to any number of engines.

## When To Use This

- You are choosing an image model for a recurring job (hero banners, listing photography,
  concept art) and want evidence from your own briefs.
- A new model launched and you need to know whether to switch, without rebuilding anything.
- Different teams disagree about which tool is better, and you want to replace opinion with
  a side-by-side on real work.
- You need to know a model's hard limits (aspect ratios, output formats, content policy)
  before committing a workflow to it.

## Prerequisites

- **A working multi-engine Blocks agent.** This workflow assumes an agent that accepts one
  description and can render it through either or both engines in a single submission. If you
  do not have one, build it first with the companion guide:
  https://raw.githubusercontent.com/CagedEther/agentic-business-schooled/refs/heads/main/blocks-dual-engine-image-agent-skill.md
- **Blocks Network account** with an organization and a `BLOCKS_API_KEY`.
- **OpenAI account with billing enabled** and an `OPENAI_API_KEY`. Image calls are paid.
- **Google AI Studio API key** (`GEMINI_API_KEY`) from https://aistudio.google.com/apikey.
  A plain AI Studio key is enough; no Google Cloud project or Vertex AI setup is required.
- **A host for the agent worker**, such as a Railway account with project access, if the
  comparison should be runnable by people other than you.
- **Node.js 22+** locally to run the trigger script.
- Optional: Python with Pillow, only for the objective composition measurement in step 6.

## Required Inputs

- **A test set of 5-10 real briefs**, written the way your team actually writes them, spanning
  the subject categories you care about. Categories matter more than volume; see step 2.
- **An explicit success checklist per brief** — the things that make the image usable, such as
  required aspect ratio, where copy must sit, what must not appear.
- The three API keys above, in a gitignored `.env` and mirrored as host variables.

## Tools And Services

- **Blocks Network agent** — the test harness. It normalizes the prompt and aspect ratio
  across engines, so the comparison is fair by construction.
- **OpenAI Images API** (`gpt-image-1`) — candidate engine.
- **Google Gemini image API** (`gemini-3-pro-image`) — candidate engine.
- **A cheap text model** (for example `gpt-4o-mini`) — rewrites the description into a
  photographic prompt once, before either engine sees it. This is what makes the test fair.
- **Pillow (optional)** — measures how busy a region of the image is, for briefs that need
  clear space for overlaid copy.

## Workflow

### 1. Make the comparison fair before you run it

A comparison is only meaningful if the models get the same instruction. Three things must be
shared, not per-engine:

- **One prompt.** Rewrite the user's description into a photographic prompt **once**, then
  send that same string to every engine. If each engine gets its own prompt variant, you are
  comparing prompts, not models.
- **One orientation mapping.** Engines express shape differently. Keep a single mapping table
  from a logical orientation to each engine's native parameter, so `wide` means the same
  intent everywhere even though the parameter differs.
- **One submission.** Fire the engines concurrently from the same request with
  `Promise.allSettled`, so both see identical inputs and one failing does not lose the other.

Record for each render: engine, model id, pixel dimensions, file format, and cost.

### 2. Build a test set organized by subject category, not volume

**This is the most important step, because results differ by category, not overall.** Ten
briefs of the same kind will teach you less than five spanning different kinds. Cover at
minimum:

- **People / portraits** — skin texture and realism are the bar here.
- **Products / single objects** — clean edges, even lighting, catalogue feel.
- **Architecture / interiors** — geometry, materials, whether the whole subject reads.
- **Editorial / atmospheric scenes** — mood, depth of field, "looks like a photograph".
- **Briefs with hard compositional rules** — a required aspect ratio, empty space for a
  headline, explicit exclusions.

Write each brief once and reuse it verbatim across engines and across future model launches.
The test set is the durable asset; the models are replaceable.

### 3. Establish the hard constraints first

Some differences are capability limits, not taste, and they decide the question before any
aesthetic judgment. Check these before scoring anything:

- **Aspect ratio.** `gpt-image-1` offers only `1024x1024`, `1536x1024`, and `1024x1536`.
  Its widest is 3:2 (ratio 1.500); it cannot produce 16:9 at all. Gemini takes an aspect
  ratio string including `16:9` and produced 2752x1536 (ratio 1.792) at its 2K tier. If a
  brief requires true widescreen, that is decided on capability alone.
- **Resolution.** At the settings above, Gemini returned roughly 2.75x the pixel count of
  gpt-image-1's widest output. Relevant for print or large-format use.
- **Output format.** The Gemini Interactions API only emits JPEG; `image/png` is rejected.
  gpt-image-1 returns PNG. If you need lossless or transparency, that narrows the field.
- **Content policy.** OpenAI refuses reference-image requests that ask it to reproduce a real
  person's likeness, even with `moderation: 'low'`. Style and palette references are fine.

Document these as facts about the engines, separate from your quality scores.

### 4. Run each brief through both engines in one submission

```bash
npx tsx trigger.ts "<brief>" --shape wide --engine both
```

Save outputs named by engine and true extension so nothing overwrites and formats stay
visible, for example `results/<slug>-google.jpg` and `results/<slug>-openai.png`.

Log the exact rewritten prompt for every run. Without it you cannot tell whether a weak
result came from the model or from a prompt that lost the brief.

### 5. Score against the brief's checklist, not on vibes

For each image, mark each checklist item pass or fail: right aspect ratio, subject placed
correctly, required empty space present, nothing from the exclusion list visible, reads as a
photograph. Count passes. "I prefer this one" is not transferable to a colleague; "this one
satisfied five of six requirements" is.

Judge realism on specifics rather than overall impression: skin texture and pores on people,
material and fabric behavior on objects, whether depth of field is plausible for the implied
lens, and whether lighting direction is consistent across the frame.

### 6. Measure composition objectively where copy has to sit

For briefs that reserve space for a headline, pixel variation in that region is a usable
proxy for how easy it is to overlay text. Lower means flatter and cleaner:

```python
from PIL import Image, ImageStat
im = Image.open(path).convert('L')
w, h = im.size
left_third = im.crop((0, 0, w // 3, h))
print(ImageStat.Stat(left_third).stddev[0])
```

Treat this as a proxy, not a verdict: it measures busyness, not whether the composition is
good. Always confirm by eye.

### 7. Interpret the results honestly, then re-test

Two findings from the reference run, and the second one matters more than the first:

- On a 16:9 editorial brief that **explicitly asked** for empty space on the left, Gemini
  produced true widescreen with a flatter left third (stddev 31.8 versus 39.3) and a more
  documentary, shallow-depth-of-field look. gpt-image-1 centred the subject and could not
  deliver 16:9.
- On a 3:2 architectural brief that **did not ask** for negative space, the ranking reversed:
  Gemini's left third measured 63.9 against gpt-image-1's 33.0. Gemini had chosen a tight,
  atmospheric crop, while gpt-image-1 produced a cleaner, centred, catalogue-style view of the
  whole subject.

The transferable lesson is not "model X wins". It is that **the winner changes with the
subject and with how explicit the brief is**, and that Gemini responded more strongly to
explicit compositional instructions. So: do not generalize from one category, and re-run the
set when either model updates. Note honestly how many head-to-head pairs your conclusion
rests on — two pairs is a signal to investigate, not a finding to standardize on.

### 8. Turn the decision into configuration, not code

Set the winner as the agent's default engine through an environment variable, keeping a
per-request override so anyone can still call the other engine or ask for both. Changing the
default must not require a code change or retrain your users; that is what keeps the
comparison worth repeating when the next model ships.

## Outputs

- A results directory of paired renders, one per engine per brief, named by engine and format.
- A scoring table per brief: checklist passes, dimensions, format, cost, and notes.
- A recorded set of hard constraints per engine (aspect ratios, formats, policy limits).
- A chosen default engine, set as configuration on the running agent.
- A reusable test set of briefs for the next model launch.

## Validation

- **Fairness check:** confirm from the logs that both engines received the identical rewritten
  prompt and the same logical orientation. If they did not, the comparison is invalid.
- **Dimension check:** confirm returned pixel dimensions match the requested shape per engine,
  and that any ratio shortfall is a known engine limit rather than a bug.
- **Resilience check:** with one engine deliberately misconfigured, a `both` request must still
  return the working engine's image plus a clear reason for the other's absence.
- **Reproducibility check:** re-run one brief and confirm the conclusion is stable. Image models
  are stochastic; if a verdict flips between runs on the same brief, you need more samples
  before deciding.
- **Cost check:** confirm actual spend matches expectation before running the full set.

## Safety And Quality Notes

- **Cost.** Every render is billed. At the time of writing: `gemini-3-pro-image` about $0.134
  per image at 1K/2K and $0.24 at 4K; `gemini-3.1-flash-image` about $0.067 at 1K;
  `gpt-image-1` roughly $0.19-$0.25 at high quality. A `both` submission is therefore roughly
  $0.33-$0.39. A 10-brief comparison across two engines is a few dollars, but keep the default
  single-engine and make `both` an explicit opt-in so routine use is not double-billed.
- **Sample size.** Image models are stochastic. A single pair per category is an anecdote.
  State how many runs a conclusion rests on, and prefer re-running a brief over trusting one
  output.
- **Do not compare across different prompts.** The most common way this exercise produces a
  wrong answer is per-engine prompt tuning. Keep one shared prompt.
- **Privacy.** Process uploads in memory, return the artifacts, store nothing, and say so in
  the agent's description. Request base64 outputs where offered so images do not land on a
  third-party CDN.
- **Generated images of people.** Do not use reference images to reproduce a real person's
  likeness without their consent; keep references to style, palette, and composition. Label
  generated imagery as generated wherever an audience could mistake it for documentary
  photography.
- Secrets live only in a gitignored `.env` and in host service variables. Never commit them,
  print them, or bake them into an image.
