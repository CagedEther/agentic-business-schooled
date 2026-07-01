---
name: apify-linkedin-active-employees
description: Pull and qualify a list of company employees who are publicly active on LinkedIn using Apify. Use when asked to find active LinkedIn employees at a company, build a LinkedIn employee roster, filter company people by recent public posts, run Apify actors `harvestapi/linkedin-company-employees` and `harvestapi/linkedin-profile-posts`, normalize roster/post CSVs, apply activity thresholds, or produce audit-friendly Markdown/CSV outputs without email or phone enrichment.
---

# Apify LinkedIn Active Employees

## Overview

Use this skill to create a company employee roster from LinkedIn and identify which people have recent public LinkedIn activity. The core pattern is:

1. Pull public people associated with the company page.
2. Normalize and verify current-company fields.
3. Check each profile for recent public posts.
4. Return only employees with evidence of activity, plus audit files and caveats.

Default to a practical, low-cost definition of active: at least one recognizable public post, repost, or quote post in the selected activity window. Use `6months` as the default window, and widen to `year` only when the user wants a looser definition or the signal is sparse.

## Required Inputs

Collect or infer these before running Apify:

- Company name.
- LinkedIn company page URL, for example `https://www.linkedin.com/company/pubnub/`.
- Activity window: default `6months`; accepted Apify values often include `24h`, `week`, `month`, `3months`, `6months`, `year`.
- Roster limit: default 500; raise only with an explicit budget.
- Posts per employee: default 3 to 5.
- Optional filters: role families, geography, seniority, current employees only, or a maximum Apify spend.

Use `APIFY_API_TOKEN` from the environment or the Apify connector. Never print or commit token values.

## Actors

Always fetch actor details before running an actor because input schemas and pricing can change.

Preferred roster actor:

- `harvestapi/linkedin-company-employees`
- Purpose: pull public LinkedIn people associated with a company page.
- Use `profileScraperMode: "Short ($4 per 1k)"` when available.
- Do not request email, phone, or other contact enrichment.

Preferred activity actor:

- `harvestapi/linkedin-profile-posts`
- Purpose: fetch recent public posts from profile URLs.
- Use for per-employee activity verification.

Fallback activity actor:

- `unseenuser/linkedin-content`
- Use when profile-posts returns sparse data or its schema changes.

If Apify tools are not already available, use tool discovery for Apify, then search actors, fetch actor details, and call the selected actor with capped inputs.

## Workflow

### 1. Pull The Company Roster

Run the employee actor with the company LinkedIn URL. Keep the input small and explicit:

```json
{
  "profileScraperMode": "Short ($4 per 1k)",
  "companies": ["{{companyLinkedInUrl}}"],
  "maxItems": 500,
  "companyBatchMode": "all_at_once"
}
```

If the current schema uses a different key, such as `currentCompanies`, adapt after reading actor details. Do not add contact enrichment fields.

Save the raw dataset before transforming it:

- `research/{{company_slug}}_linkedin_people_{{YYYY-MM-DD}}.raw.json`
- `research/{{company_slug}}_linkedin_people_{{YYYY-MM-DD}}.csv`

### 2. Normalize The Roster

Create one row per person. Preserve raw JSON separately and keep the normalized CSV simple:

- `name`: `firstName + lastName`, falling back to `name`.
- `title_or_headline`: title from the current position matching the company, falling back to `headline`.
- `listed_company`: matched current position company, falling back to the requested company.
- `location`: `location.linkedinText` when available.
- `linkedin_url`: `linkedinUrl`.
- `current_position_detected`: `yes`, `not_in_returned_position_fields`, or `uncertain`.
- `position_start`: `startedOn.year-month` when available.

Verify current-company status by inspecting `currentPositions`. Prefer a position whose `companyName`, `companyLinkedinUrl`, or `companyId` matches the target. Keep board members, investors, consultants, and advisors if the actor returns them, but label them by title rather than assuming employee status.

Deduplicate by LinkedIn URL first, then normalized name. Some profile URLs may be LinkedIn internal IDs rather than vanity slugs; keep them if Apify returns them.

### 3. Check Recent Profile Activity

Run the activity actor against roster profile URLs. Use chunks if the actor supports multiple `targetUrls`; otherwise run per profile. Keep `maxPosts` low until the user asks for depth.

Recommended input for `harvestapi/linkedin-profile-posts`:

```json
{
  "targetUrls": ["{{linkedinProfileUrl}}"],
  "maxPosts": 5,
  "postedLimit": "6months",
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

Recommended fallback input for `unseenuser/linkedin-content`:

```json
{
  "profileUrls": ["{{linkedinProfileUrl}}"],
  "totalPostsPerProfile": 5,
  "postedLimit": "6months",
  "includeReposts": true,
  "removeEmptyFields": false
}
```

Keep a lookup from input profile URL to employee row. If the actor output includes `targetUrl`, `inputUrl`, `sourceUrl`, `profileUrl`, `authorUrl`, or `author_profile_url`, use those fields to map posts back to employees.

### 4. Normalize Activity Evidence

Do not trust one actor output shape. Normalize these aliases:

- Text: `text`, `postText`, `post_text`, `content`, `contentText`, `commentary`, `body`, `description`, `summary`, `headline`, `title`.
- Post URL: `postUrl`, `post_url`, `url`, `linkedinUrl`, `permalink`.
- Profile URL: `profileUrl`, `profile_url`, `authorUrl`, `author_profile_url`, `targetUrl`, `inputUrl`, `sourceUrl`.
- Author: `profileName`, `authorName`, `name`, or `author_first_name + author_last_name`.
- Date: `publishedAt`, `postedAt`, `posted_date`, `createdAt`, `datePublished`, `date`, `time`, or nested `postedAt.date`.
- Reactions: `reactionCount`, `likeCount`, `likes`, `totalReactions`, `stats_total_reactions`, or nested `engagement.likes`.
- Comments: `commentCount`, `commentsCount`, `comments`, `stats_comments`, or nested `engagement.comments`.
- Reposts/shares: `repostCount`, `repostsCount`, `shareCount`, `shares`, `stats_reposts`, or nested `engagement.shares`.

Use an engagement proxy only for sorting evidence, not for determining activity:

```text
score = reactions + 3 * comments + 5 * reposts
```

Avoid copying full LinkedIn post text into the final Markdown. Use links, dates, short excerpts, or summaries. Retain raw JSON locally for auditability when needed.

### 5. Classify Employees

Use these statuses:

- `active`: at least one public post/repost/quote post in the activity window.
- `strongly_active`: at least three public posts in the activity window, or at least one post in the last 90 days with meaningful engagement.
- `inactive_or_private`: no public posts returned in the activity window.
- `unverified`: profile URL missing, actor failed, output could not be mapped to the employee, or LinkedIn visibility prevented verification.

Do not state that someone is definitely inactive. Say "no public activity found by the actor in the selected window" when evidence is absent.

### 6. Produce Outputs

Save date-stamped files under `research/` unless the user asks for another location:

- `{{company_slug}}_linkedin_people_{{YYYY-MM-DD}}.csv`: normalized roster.
- `{{company_slug}}_linkedin_activity_{{YYYY-MM-DD}}.raw.json`: raw activity datasets or per-run metadata.
- `{{company_slug}}_linkedin_active_employees_{{YYYY-MM-DD}}.csv`: filtered active employees.
- `{{company_slug}}_linkedin_active_employees_{{YYYY-MM-DD}}.md`: short report.

Use these columns for the active employee CSV:

- `name`
- `title_or_headline`
- `listed_company`
- `location`
- `linkedin_url`
- `current_position_detected`
- `position_start`
- `activity_status`
- `activity_window`
- `posts_in_window`
- `last_post_at`
- `last_post_url`
- `sample_post_urls`
- `activity_actor`
- `activity_dataset_ids`
- `notes`

The Markdown report should include:

1. Method: company URL, pull date, actors, activity window, max posts per employee, and no-contact-enrichment note.
2. Summary counts: roster rows, verified current-company rows, active rows, strongly active rows, inactive/private rows, unverified rows.
3. Active employees table: name, title, location, last post date, posts in window, profile link, sample post link.
4. Caveats: LinkedIn visibility, actor schema changes, internal profile URLs, repost handling, and why lack of returned posts is not proof of inactivity.
5. Source files: raw JSON, roster CSV, active CSV, and Apify dataset IDs.

## Cost And Quality Controls

- Fetch actor details before every new project or after long gaps.
- Set `maxItems`, `maxPosts`, and `maxTotalChargeUsd` when the tool supports it.
- Use `maxPosts` 3 to 5 for activity qualification; increase only for content analysis.
- Keep comments and reactions disabled unless the user specifically needs audience-level data.
- Start with a small pilot chunk of 5 to 10 employees, inspect output shape, then run the full roster.
- Preserve raw data and dataset IDs so another person can audit the output.
- Label weak signals and unmapped profiles instead of filling gaps.
- Do not fabricate employees, post dates, post URLs, engagement counts, or dataset IDs.
- Do not request or include emails, phone numbers, or private profile data.
