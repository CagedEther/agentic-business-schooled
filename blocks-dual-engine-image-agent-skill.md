# Build a Multi-Engine Text-to-Image Agent on the Blocks Network

## Overview

This workflow builds a hosted AI agent that turns a text description into a photo-realistic
image, and can render the same brief through more than one image model so you can compare
them. The agent runs 24/7 as a Docker worker on Railway, holds the image-provider API keys
once on the server, and is consumed over the Blocks Network by people who never need their
own OpenAI or Google credentials. Switching the default image engine is a single environment
variable, and any individual request can override it.

The reference implementation renders with Google `gemini-3-pro-image` by default, with
OpenAI `gpt-image-1` as an alternative, and a cheap `gpt-4o-mini` call that rewrites the
user's description into a photographic prompt before either model sees it.

## When To Use This

- You want a text-to-image capability that a marketing, content, or design team can use
  without distributing provider API keys to each person.
- You want to A/B two or more image models on real briefs before committing to one, and
  keep the ability to switch later without a rewrite.
- You need generated imagery on a repeatable brief (hero banners, listing photography,
  concept shots) where prompt quality and aspect ratio must be consistent.

## Prerequisites

Accounts and access, all required:

- **Blocks Network account** with an organization, and a `BLOCKS_API_KEY`. Get one with
  `blocks login --write-env`. The `provider.organization` in the agent card must match the
  org shown by `npx blocks whoami`, or registration fails.
- **OpenAI account with billing enabled** and an `OPENAI_API_KEY`. Live image calls are
  paid; there is no free tier.
- **Google AI Studio API key** (`GEMINI_API_KEY`) from https://aistudio.google.com/apikey.
  A plain AI Studio key is sufficient — no Google Cloud project, service account, or Vertex
  AI setup is needed. There is no free tier for the image models.
- **Railway account** with access to a project, plus either the Railway CLI authenticated
  (`railway login`) or Railway MCP tooling. Hosting a always-on worker is a paid resource.
- **Node.js 22+** and npm locally.

No GPU, database, inbound port, or domain is required. The agent is an outbound worker.

## Required Inputs

At build time:

- The three API keys above, placed in a local `.env` and mirrored as Railway service
  variables. Never commit `.env`; never paste key values into shell commands or logs.
- An agent name matching `^[a-zA-Z0-9_]+$`. Underscores only — hyphens are rejected.

At run time, per task:

- `description` (required, text) — what the image should show.
- `reference_image` (optional, JPEG/PNG/WebP up to 15 MB) — guides style, composition, or palette.
- `engine` (optional, text) — `google`, `openai`, or `both`. Blank uses the deployment default.
- `aspect_ratio` (optional, text) — `portrait`, `landscape`, `wide` (16:9), or `square`.
  Blank lets the prompt-rewrite step choose.

## Tools And Services

- **Blocks Network SDK + CLI** (`@blocks-network/sdk`, `@blocks-network/cli`) — agent card
  validation, registration, and the `blocks run` worker runtime that receives tasks.
- **Railway** — runs the Docker worker continuously and stores the API keys as service
  variables.
- **OpenAI Images API** — `gpt-image-1` for rendering; `gpt-4o-mini` for the prompt rewrite.
- **Google Gemini API** — `gemini-3-pro-image` for rendering, via the Interactions endpoint.

## Workflow

### 1. Scaffold the project

```bash
npx -y @blocks-network/cli init <agent_name> --yes --language node
cd <agent_name> && npm install && npm install -D tsx @types/node
```

Add `node_modules`, `.env`, and any results directory to `.gitignore`. The Railway upload
honors `.gitignore`; without this the deploy tarball is huge and may include secrets.

### 2. Design the agent card

Define the IO contract in `agent-card.json` before writing handler logic. Rules enforced by
`blocks check`:

- Text inputs use `text/plain` and must **not** carry a `schema`.
- File inputs need a **concrete** contentType (`image/jpeg`, never `image/*`), widened with
  an `accept` array and capped with `maxSizeBytes`.
- Outputs the handler does not always produce must be `"guaranteed": false`. With a
  switchable engine, every image output is conditional, so all of them are `false`.
- Set `runtime.maxRunningTimeSec` generously (420 works here); two concurrent image
  renders plus a rewrite call can take a minute or more.

Prefer several small optional text inputs over one JSON form, and make anything you may
want to tune later an environment variable with a default rather than a constant.

Validate after every edit:

```bash
npx -y @blocks-network/cli check
```

### 3. Write the handler

Core mechanics:

- Read inputs from `task.requestParts`, matched by `partId`. Text is on `.text`; files come
  from `await ctx.downloadInputArtifact(part)` as a Buffer.
- Call `ctx.reportStatus('...')` at each stage — it is the consumer's only progress signal.
- Pass `signal: ctx.cancelSignal` to every `fetch` so cancellation aborts in-flight work.
- Throw plain `Error`s with user-readable messages; validate cheap things (unknown engine,
  unknown aspect ratio, over-long description) **before** spending money on API calls.
- Never trust an uploaded part's declared contentType. Sniff magic bytes instead:

```ts
function sniffImageMime(bytes: Buffer): string | undefined {
  if (bytes.length < 12) return undefined;
  if (bytes[0] === 0x89 && bytes[1] === 0x50 && bytes[2] === 0x4e && bytes[3] === 0x47) return 'image/png';
  if (bytes[0] === 0xff && bytes[1] === 0xd8 && bytes[2] === 0xff) return 'image/jpeg';
  if (bytes.toString('ascii', 0, 4) === 'RIFF' && bytes.toString('ascii', 8, 12) === 'WEBP') return 'image/webp';
  return undefined;
}
```

**Engine selection.** Resolve the engine from the per-task input, falling back to an env
default, and guard the env value too so a typo cannot disable every engine and fail all
tasks:

```ts
const DEFAULT_ENGINES = (process.env.ENGINES ?? 'google').toLowerCase();
// per task: parse the `engine` input, else fall back to DEFAULT_ENGINES, else 'google'
```

**Run engines concurrently and settle, never race.** `Promise.allSettled` means wall-clock
is the slower engine rather than the sum, and one engine refusing or failing still returns
the other's image:

```ts
const [openaiResult, googleResult] = await Promise.allSettled([...]);
// push an artifact for each fulfilled result; if none fulfilled, throw with both reasons
```

Report partial success explicitly — say which engine dropped out and why, rather than
silently returning one image as if one was all that was expected.

**Prompt rewriting is the biggest realism lever**, larger than any API quality flag. Use a
cheap model to expand the description with camera/lens language, lighting direction, and
realism cues, ending with negatives such as "not an illustration, not CGI, no plastic
sheen, no text or watermarks". Two rules matter:

- Instruct it to preserve the user's subject and intent exactly, and to carry through every
  compositional requirement and exclusion already present. An already art-directed brief
  should survive almost verbatim. Without this, a careful brief loses its constraints.
- Make it best-effort: if the rewrite call fails, log a warning and fall back to the raw
  description rather than failing the task.

Log the final prompt so a disappointing render can be traced to the prompt that caused it.

### 4. Implement the OpenAI path

One model, two endpoints depending on whether a reference image was supplied:

- No reference: `POST https://api.openai.com/v1/images/generations`, JSON body with
  `model`, `prompt`, `size`, `quality`, `moderation`, `n: 1`.
- With reference: `POST https://api.openai.com/v1/images/edits`, `multipart/form-data`,
  same fields plus `input_fidelity: 'high'` and the file as `image[]`.

Sizes are discrete: `1024x1536` (portrait), `1536x1024` (landscape), `1024x1024` (square).
**There is no 16:9 size** — 3:2 is the widest available, so a widescreen request needs
cropping. Set `moderation: 'low'` to cut false-positive refusals. Read the image from
`data[0].b64_json`, with a `data[0].url` download as a fallback.

### 5. Implement the Google path

One endpoint handles both text-only and text-plus-reference:

```
POST https://generativelanguage.googleapis.com/v1beta/interactions
Headers: x-goog-api-key: <GEMINI_API_KEY>, Content-Type: application/json
Body: { model, input: [{type:'text', text}, {type:'image', mime_type, data:'<base64>'}?],
         response_format: { type:'image', mime_type:'image/jpeg', aspect_ratio, image_size } }
```

Four things that are easy to get wrong, all verified against the live API:

1. **`response_format.mime_type` must be `image/jpeg`.** `image/png` is rejected with
   `invalid_request`. Declare that output as JPEG in the agent card rather than mislabelling it.
2. **There is no top-level `output_image` field.** That accessor exists only in Google's
   SDK. On the raw REST response the image is at **`steps[1].content[0]`** as
   `{type:'image', mime_type:'image/jpeg', data:'<base64>'}`.
3. **`steps[0]` is a `thought` step containing a multi-megabyte base64 `signature` blob.**
   Any extractor that grabs "the longest base64-looking string" returns that instead of the
   image. Require an image-ish `type` or `mime_type` alongside a `data` field.
4. Successful responses are a top-level **object**; errors come back as a top-level
   **array**. Write the extractor to walk either.

Because the documented shape and the real shape differ, prefer a defensive recursive
extractor over betting on one path, and log the unrecognised response shape on failure.

Aspect ratio is a string here (`'16:9'`, `'3:2'`, `'2:3'`, `'1:1'`) with a separate
`image_size` tier (`512px`, `1K`, `2K`, `4K`). Since the two providers express shape
differently, keep a small mapping table so a given orientation produces comparable
framing on both.

Avoid the Imagen models (`imagen-4.0-*`): they are deprecated with an August 17, 2026
shutdown, and their `:predict` endpoint cannot accept a reference image at all.

### 6. Adapt the test trigger

Drive the card's inputs from `process.argv`. Three traps:

- `filePartFromPath(...)` returns a Promise — it must be awaited.
- **Artifact-write race**: `onArtifact` downloads asynchronously. If `onTerminal` calls
  `process.exit()` immediately, large artifacts are silently lost. Collect the download
  promises and `await Promise.allSettled(...)` inside `onTerminal` before exiting.
- Name saved files by engine and real extension (OpenAI PNG, Google JPEG) so comparison
  runs do not overwrite each other.

Typecheck both files after every change:

```bash
npx tsc --noEmit --strict --target es2022 --module nodenext --moduleResolution nodenext --skipLibCheck handler.ts trigger.ts
```

### 7. Register and test locally

```bash
npx -y @blocks-network/cli check
npx -y @blocks-network/cli register     # private + free
npm start                                # terminal 1: worker
npx tsx trigger.ts "<description>"      # terminal 2: end-to-end
```

Startup is healthy when the log shows `starting "<Display Name>" (<agent_name>)` followed
by an `agent_registered` event. Iterate locally — worker restarts are seconds, Railway
deploys are minutes.

When restarting between edits, kill with `pkill -f "blocks-run"` (hyphenated). The pattern
`"blocks run"` misses the real process, leaving a stale worker that keeps serving tasks and
making your fixes appear to do nothing. Confirm with `pgrep -f blocks-run`.

Where a response shape is uncertain, probe the provider with `curl` and dump the structure
before wiring it in — and probe with a deliberately invalid key first. An auth error proves
the URL and body parse, while a 404 or unknown-field error reveals a mistake, all for free.

### 8. Containerize

Use a Node 22 base image. **The one line that matters:**

```dockerfile
RUN npm install -g @blocks-network/cli
ENV PATH="/root/.blocks/bin:${PATH}"
RUN blocks --version
```

The npm package publishes a dangling bin shim; its postinstall puts the real binary in
`/root/.blocks/bin` and only edits shell profiles, which Docker's exec-form `CMD` never
sources. Without the `ENV` line the container crash-loops with
`Error: Cannot find module '/app/blocks'`. The `RUN blocks --version` layer turns that into
a build-time failure instead of a runtime one. Finish with `CMD ["blocks", "run"]` and
document every env var in comments.

### 9. Deploy to Railway

```bash
railway link --project <project-id> --environment production --service <service>
railway variables --set "BLOCKS_API_KEY=$BLOCKS_API_KEY" --set "..." --skip-deploys
railway up --detach
```

Set values by `source`-ing the local `.env` and expanding `$VARS` — never paste key values
into commands. Verify names only with `railway variables --kv | awk -F= '{print $1}'`.

Poll until a terminal state rather than assuming success:

```bash
railway deployment list --json   # newest first; wait for SUCCESS
```

Variables changed after the last deploy with `--skip-deploys` need `railway redeploy -y`.
Code changes need another `railway up --detach`. If Railway MCP tooling returns
Unauthorized, pass an explicit workspace id or fall back to the CLI.

### 10. Publish or invite

Registration defaults to private. To let others use it without their own provider keys,
either invite them:

```bash
npx -y @blocks-network/cli invite send <agent_name> --email <email>
```

or publish a public listing:

```bash
npx -y @blocks-network/cli publish --billing-mode free --listing public --accept-terms
```

Confirm pricing explicitly before publishing a paid listing, since that changes what
consumers are charged.

## Outputs

- A registered Blocks agent at `https://app.blocks.ai/agents/<agent_name>`.
- One or two generated images per task, returned as artifacts: a JPEG from the Google
  engine and/or a PNG from the OpenAI engine.
- A running Railway service holding the provider keys, so consumers need none.
- Local `trigger.ts` results for comparison runs.

## Validation

- `blocks check` passes and typecheck is clean.
- Railway deployment reaches `SUCCESS` and its logs contain an `agent_registered` event.
  `CRASHED` plus module-not-found means the Dockerfile PATH line is missing;
  `fatal: BLOCKS_API_KEY is required` means variables are unset or were set without a redeploy.
- **Positive test:** with no local worker running (`pgrep -f blocks-run` to confirm), run
  the trigger. The deployed instance must serve it end to end and return a usable image.
- **Engine test:** run with no `engine` input and confirm only the default engine's artifact
  comes back; run with each explicit value and confirm the override is honored.
- **Negative test:** send an unknown engine or aspect ratio. It must refuse with a clear
  message and without calling a paid API.
- **Aspect check:** confirm returned pixel dimensions match the requested shape on both
  engines, and remember OpenAI cannot exceed 3:2.

## Safety And Quality Notes

- **Cost.** Every task spends real credits. At the time of writing: `gemini-3-pro-image`
  about $0.134 per image at 1K/2K and $0.24 at 4K; `gemini-3.1-flash-image` about $0.067 at
  1K; `gpt-image-1` roughly $0.19-$0.25 at high quality. Rendering both engines is therefore
  roughly $0.33-$0.39 per submission versus $0.134 for a single default render. Default to
  one engine, make `both` an explicit opt-in, and use a flash-tier model while iterating.
- **Content policy.** OpenAI refuses reference-image requests that ask it to reproduce a
  real person's likeness, even with `moderation: 'low'`. Style, palette, and composition
  references are fine. Surface that distinction in the error message.
- Detect genuine safety refusals narrowly. Matching the bare word `moderation` also matches
  ordinary invalid-parameter errors about the `moderation` field and mislabels bugs as
  policy blocks. Log the raw provider response body while showing users a short message.
- **Privacy.** Process uploads in memory, return the artifacts, store nothing, and say so
  in the card description — then honor it. Request base64 outputs where a provider offers
  the choice, so images do not land on a third-party CDN.
- Cap unbounded inputs (description length, file size, retries) and expose the caps as env
  vars so they can be tightened without a code change.
- Retry only 429 and 5xx responses, honoring `retry-after`; never retry a 400.
- **Known platform gap:** a thrown error message reaches the consumer's terminal event for
  locally-served tasks, but Railway-served failures arrive as a bare `{state:'failed'}` with
  the reason only in `railway logs`. If consumer-visible failure reasons matter, return an
  explanatory text artifact instead of throwing.
- Secrets live only in a gitignored `.env` and in Railway service variables. Never commit
  them, print them, or bake them into an image.
