---
name: poplar
description: Operate Poplar — the AI social-media platform — from a coding agent or assistant. Use when the user mentions Poplar, wants to draft, schedule or review LinkedIn/social posts in their brand voice, pull post analytics, read their content strategy or ideas, feed meeting notes into their knowledge base, or connect an agent to Poplar's MCP server or REST API. Teaches the MCP connection, the write-with-Poplar's-writer workflow (start_draft → get_draft → create_draft → schedule_post → confirm_action), media (generate or attach images), editing, approvals, comments, per-provider composition (polls, Instagram, Twitter threads), the workspace/timezone rules, and where every guardrail is.
license: Proprietary. See LICENSE.md.
metadata:
  author: Poplar-AI
  version: "1.0"
  homepage: https://usepoplar.com
---

# Poplar

Poplar writes and schedules social posts in a customer's own voice. This skill teaches you how to drive it **as an outside agent** through its MCP server. It contains no prompts, models, or writing rules — those run inside Poplar. Your job is to orchestrate; Poplar's writer writes.

## Connect

Poplar is a remote MCP server:

- **Endpoint:** `https://usepoplar.com/api/mcp` (Streamable HTTP, stateless)
- **Auth, option A — OAuth:** add the URL as a connector in Claude.ai / Claude Desktop; the user signs in and approves scopes. Nothing to paste.
- **Auth, option B — API key:** `Authorization: Bearer pop_live_…` — the user creates one in Poplar → Settings → API keys, choosing scopes. Works in Claude Code, Cursor, and any client that sends headers.

Claude Code / Cursor config:

```json
{ "mcpServers": { "poplar": { "url": "https://usepoplar.com/api/mcp", "headers": { "Authorization": "Bearer pop_live_..." } } } }
```

stdio-only clients: `npx mcp-remote https://usepoplar.com/api/mcp --header "Authorization: Bearer pop_live_..."`

The server sends `instructions` at initialize. Follow them; they take precedence over this file where they differ.

You may already have this file without installing anything: the server publishes it as the MCP resource `poplar://skill` (and the tool reference as `poplar://skill/tools`). If your client can read resources, read `poplar://skill` before your first write in a session. Installing the skill only changes *when* you have it (before the connection exists) and where it lives (on disk).

## The one rule that matters

**When the user wants a post written, do not write it. Call `start_draft`.** Poplar's writer applies their brand voice, the preferences learned from how they edit, their strategy and knowledge base. Text you write yourself is stored as-is, labelled `written_by: caller`, and gets none of that. Only use `create_draft` with your own text when the user hands you copy or explicitly asks after being offered Poplar's writer — and say so plainly.

## Core workflow: write → save → schedule

```
1. list_workspaces                       → pick by NAME; never show or ask for a UUID
2. start_draft { workspace_id, brief }   → { run_id }  (returns in ~1s)
3. get_draft   { run_id }                → waits up to ~20s; repeat until status "done"
4. list_social_accounts { workspace_id } → pick the account by name; needs connection_id
5. create_draft { workspace_id, content: <text from step 3>, run_id, connection_id }
6. schedule_post { workspace_id, post_id, scheduled_at }
      → a PREVIEW with confirmation_id + will_publish_local. Nothing is scheduled yet.
7. Show the user will_publish_local. Only on a clear yes:
   confirm_action { confirmation_id }
```

- `scheduled_at` **without a timezone** (`2026-09-18T09:00:00`) is read in the **workspace's timezone** — that is what "9am on the 18th" means. Add `Z` only for an exact UTC instant.
- A confirmation is single-use and expires in minutes. If it fails, propose again; never retry blindly.
- If scheduling is refused with *"turned off for this workspace"*, a workspace owner must enable it in Poplar → Settings → Workspace. Say that; do not work around it.
- There is **no publish-now tool**. Scheduling is the furthest an agent can go.
- To move a scheduled post, call `schedule_post` again with the new time (preview → confirm as usual). To take it off the calendar, `unschedule_post`; it goes back to draft with everything intact.
- **Approvals**: `request_approval` sets the reviewers and whether it is blocking; there is no edit-in-place — to change either, `cancel_approval` (requester only) and request again, exactly as the app does.

## Beyond a plain draft

- **Images** — `generate_image` designs one **from the post** the way the app's Generate Image button does: Poplar's infographic prompt + the workspace's default template and brand colours + the post text. You do not write an art prompt; pass `instructions` only for what the customer specifically asked ("dark background", "chart the 20%") and nothing otherwise. It returns a small preview: show it, let them judge, and if they want a change, `remove_attachment` then generate again with their words. For the customer's own file use `create_media_upload` then `attach_media`, or pass a public `url` to `attach_media`. `remove_attachment` drops one; `reorder_attachments` sets which is first, second, and so on. To replace a bad image: `get_post` → `remove_attachment` → `generate_image` → `reorder_attachments` if it should not be last.
  - A file the customer drops into a **chat** (Claude.ai) never reaches you as bytes, and the chat sandbox cannot PUT to Poplar's storage — offer a public URL or attaching in Poplar instead. From **Claude Code / Cursor** the upload works: `curl -T <file> <upload_url>`, then `attach_media` with the `media_key`.
- **Editing** — `update_post` changes text/title/account. It refuses a published post, and is blocked while an approval is pending/approved (cancel it first). Its `updated` list is what the database holds after the write, not an echo of your request; a `CONFLICT` means the post changed underneath you — `get_post` and decide again. A content edit also lands live in the customer's open editor. `get_post` shows everything a post currently holds.
- **Composition** — `set_poll` (LinkedIn), `set_instagram_options` (collaborators, tags, alt text…), and Twitter/X threads by putting `---` between tweets. A LinkedIn post is **either a poll or media, never both** — `set_poll` refuses while media is attached and the media tools refuse while a poll is set; ask the customer which they want.
- **Approvals** — `request_approval` to send for review; an approver uses `list_pending_approvals` → `get_post` → `submit_approval_decision` (approve / reject / needs_changes, with a comment). Approving does NOT publish; it clears the gate.
- **Comments** — `list_comments`, `add_comment` (reply with `reply_to`), `resolve_comment`.

## Deciding what to post

Before drafting, read what Poplar already knows:

- `list_ideas` — ranked angles harvested from the customer's industry news, each with its source
- `get_strategy` — pillars, do-more, do-less, biggest bottleneck, from real performance
- `get_post_analytics` — what actually worked; "write more like the top one" flows straight into `start_draft`
- `get_brand_identity` / `list_learned_preferences` — the voice and the rules; read-only for preferences

## Tools at a glance (41)

| Area | Tools |
|---|---|
| Workspaces | `list_workspaces`, `create_workspace`, `get_workspace_settings`, `update_workspace_settings`, `get_workspace_allowance`, `invite_member` |
| Writing & editing | `start_draft`, `get_draft`, `create_draft`, `update_post`, `get_post`, `list_posts` |
| Media | `generate_image`, `create_media_upload`, `attach_media`, `remove_attachment`, `reorder_attachments` |
| Composition | `set_poll`, `remove_poll`, `set_instagram_options` (+ Twitter threads via `---` in content) |
| Publishing | `list_social_accounts`, `schedule_post`, `confirm_action`, `unschedule_post` |
| Approvals | `request_approval`, `list_pending_approvals`, `get_approval`, `submit_approval_decision`, `cancel_approval` |
| Comments | `list_comments`, `add_comment`, `resolve_comment` |
| Intelligence | `list_ideas`, `get_strategy`, `get_post_analytics`, `list_learned_preferences`, `list_known_mentions` |
| Voice & knowledge | `get_brand_identity`, `update_brand_identity`, `push_knowledge_item` |
| Budget | `get_usage` |

Full argument reference: [references/tools.md](references/tools.md). REST equivalents and scopes: [references/rest-and-scopes.md](references/rest-and-scopes.md).

## Conduct

- Talk in product terms. Never recite tool names, scopes, or field names to the user.
- Refer to workspaces, accounts and people by name. IDs are for tool calls only.
- Names are not unique. Two workspaces, accounts or contacts sharing a name come back already told apart — "My's Workspace (UTC)" — so show them exactly as written and map the customer's pick to the id yourself. Never guess between two same-named things; a reviewer name that matches two people returns `AMBIGUOUS_REVIEWER` and wants an email instead.
- Every recoverable error carries a `next_step`. Do what it says rather than retrying the same call.
- Summarize results; do not echo JSON.
- Anything irreversible (scheduling, inviting) gets a plain-language confirmation first.
- `list_social_accounts` returns a `problem` on accounts whose token expired — tell the user to reconnect that account in Poplar before scheduling to it.

## Limits you will hit

- Per-key rate limit: 100 requests/minute.
- AI spend is capped per workspace per day and month; `start_draft` returns `SPEND_LIMIT_EXCEEDED` when exhausted. `get_usage` shows remaining budget — check it before a batch.
- One connection sees every workspace the user can access. Pass `workspace_id` on every workspace-scoped call unless the account has exactly one.
