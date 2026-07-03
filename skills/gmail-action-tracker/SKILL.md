---
name: gmail-action-tracker
description: Build a live Cowork artifact that scans someone's connected Gmail for unread emails that actually need an action from them (schedule/accept a meeting, submit or continue a job application, fill out a form or questionnaire, RSVP, reply because someone's waiting). Use this whenever the user wants a "tracker", "dashboard", "board", or "digest" for their inbox, wants to know which unread emails need a reply or action, wants to stop missing things buried in a cluttered inbox, or asks for anything like "what's actually waiting on me in Gmail". Also use it if the user asks to make a live/persistent view of their email rather than a one-time summary. Requires the user to have a Gmail connector active in Cowork — if they don't, help them connect one first rather than skipping this skill.
---

# Gmail Action-Item Tracker

## What this builds

A self-contained Cowork live artifact: a filterable board of unread Gmail threads that need something from the user, grouped into Meetings & Scheduling / Applications / Forms & Questionnaires / Other Actions. Each card links to the real thread in Gmail and has a "Mark as read" button; there's also a bulk "Mark all as read" for the current filter. Everything not actionable (newsletters, job-alert digests, receipts, automated notices) is filtered out by an AI triage pass, so what's left is short and worth looking at.

This came out of building the same thing for one user by hand and hitting two real bugs along the way — both are called out below because they'll bite you again if skipped.

## Step 1: Confirm Gmail is connected, and find the real tool names

Every Cowork install has its own connector instances, so the exact tool names look like `mcp__<some-uuid>__search_threads` — that UUID is unique to *this* user's Gmail connection and will be different for every person who runs this skill. **Never hardcode a tool name you saw in a previous session or example.** Always rediscover it fresh:

1. Look at (or `ToolSearch` for) the available tools for ones that read like a Gmail connector: something that searches/lists email threads (usually named like `search_threads`), something that fetches one thread's full body (`get_thread`), and something that adds/removes Gmail labels on a thread (`label_thread` / `unlabel_thread`, or it may be a single `modify_thread`-style tool — read the actual tool descriptions, don't assume names).
2. If nothing like this exists, the user doesn't have Gmail connected yet. Don't fail silently — search the connector registry for Gmail and suggest connecting it, then stop until they do.
3. Read each candidate tool's full schema/description carefully. Field names and parameter shapes genuinely differ between connector implementations (e.g. `pageSize` vs `maxResults`, `threadId` vs `thread_id`, whether label removal takes label IDs or names). Don't guess — the description tells you.

## Step 2: Quick preferences, not a long interview

Ask at most two short multiple-choice questions so this stays fast:

- What should count as "actionable"? Default to the four categories above, but let them narrow it (e.g. only job-related, or organize by Gmail label instead).
- Time scope: unread-only (no time limit) is a good default; offer "last N days" as an alternative for people with huge unread counts.

Don't over-ask. The goal is a working artifact in one pass, not a requirements document.

## Step 3: Verify the real response shape before building

Call the search tool once for real (e.g. query `is:unread`, small page size) and look at the actual JSON that comes back. This is a read-only call, so it's safe to run — do it rather than assuming the shape matches any example you've seen. Use what you observe to write the parsing code, not a remembered schema.

**Do not** call the label-removal/mark-as-read tool as a "test" during setup. That's a write action on the person's real inbox — it should only ever fire when a human clicks the button inside the finished artifact, never during your own verification pass.

## Step 4: Build from the template

Start from `assets/action_items_template.html`. It has three placeholders to fill in with the tool names you found in Step 1:

- `__SEARCH_THREADS_TOOL__`
- `__GET_THREAD_TOOL__`
- `__UNLABEL_THREAD_TOOL__`

If the discovered tools use different parameter names than the template assumes (see comments near the top of the `<script>` block), adjust the calls accordingly — don't force-fit the response into the template's assumptions.

### The bug that will bite you if you skip this: snippet truncation

Gmail's thread-search API returns a short snippet (~200 characters), not the full email body. This is fine for obvious cases, but real emails often put the actual ask several sentences in — a Hebrew government-employment-service email said "Welcome to your digital career track" in the visible snippet and only mentioned "you must update your CV and join the smart-agent this month, or your benefit eligibility is affected" further down, past the cutoff. An AI classifier working only off snippets will confidently call that "not actionable."

The template already handles this with a second pass: after the first classification round, anything Gmail marked Important, or that comes from an official/government-looking sender, or whose snippet/subject contains action-hinting keywords (see `IMPORTANT_SENDER_RE` / `KEYWORD_RE` near the top of the script), but which the first pass called non-actionable, gets its full body fetched once via the thread-detail tool and reclassified. Keep this pass — it's there because the naive version genuinely missed real, important emails.

### The other bug that will bite you: dead references after renaming

If you rename or restructure any JS function while customizing the template, grep the whole file for the old name afterward. A stray `window.oldName = oldName;` left over from a rename throws a `ReferenceError` the instant the script runs, which silently kills the entire page before it ever calls its own init function — the artifact will just look permanently empty, with no error shown to the user. This exact bug happened once already; it's an easy one to reintroduce.

## Step 5: Publish

Use `create_artifact` (or `update_artifact` if one already exists for this user), listing only the tool names you actually called and verified this session in `mcp_tools`. Give it a clear `id` and a `description` that says what it shows and where the data comes from.

## Step 6: Tell the user what you built, briefly

One or two sentences: what it tracks, that it auto-refreshes via the artifact's own Reload button, and that "Mark as read" / "Mark all as read" are real Gmail actions (not just local dismissals) — worth flagging since it's a live write action, not just cosmetic.
