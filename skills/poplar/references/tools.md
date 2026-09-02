# Poplar MCP tools — argument reference

41 tools. Every workspace-scoped tool accepts `workspace_id` (a UUID from `list_workspaces`); it may be omitted only when the account/key has exactly one workspace, otherwise the call returns `WORKSPACE_REQUIRED` with the list to choose from. Required arguments are marked `*`.

Responses are compact JSON: ids appear only where a follow-up call needs them, and an absent field means "not set / not measured", never zero.

## Workspaces & team

| Tool | Args | Notes |
|---|---|---|
| `list_workspaces` | — | id, name, your_role, timezone, and `current_time` per workspace. Call it first. |
| `create_workspace` | `name`*, description, timezone | caller becomes owner |
| `get_workspace_settings` | — | members with roles |
| `update_workspace_settings` | name, timezone | admin |
| `get_workspace_allowance` | — | workspaces/accounts/members used vs limit |
| `invite_member` | `email`*, role, agentAccess | sends an email — confirm first |

## Deciding what to post (read-only)

| Tool | Args | Notes |
|---|---|---|
| `list_ideas` | limit, status | ranked angles from the customer's industry, each with its source |
| `get_strategy` | — | pillars, do-more, do-less, bottleneck, per LinkedIn account |
| `get_post_analytics` | connection_id, since, until, limit, cursor | omit workspace_id to span all workspaces |
| `list_learned_preferences` | limit | rules learned from real edits (read-only) |
| `get_brand_identity` | — | voice, tone, audiences, offer; lists `editable_fields` |
| `list_known_mentions` | query, limit | people tagged before, each with a paste-ready `@[Name](urn)` token |

## Writing

| Tool | Args | Notes |
|---|---|---|
| `start_draft` | `brief`*, post_id | Poplar's writer (voice + preferences + strategy). Returns `run_id`; poll `get_draft`. THE way to get a voice-true post. |
| `get_draft` | `run_id`* | long-polls ~20s; status queued/running/done/failed; `text` when done |
| `create_draft` | `content`*, title, connection_id, run_id | store text. Pass `run_id` when it came from start_draft (records provenance). For a Twitter/X thread, separate tweets with a line of only `---`. |
| `update_post` | post_id*, content, title, connection_id | edit text/title/account. Cannot edit a published post; blocked while an approval is pending/approved (cancel it first). Media is separate. `updated` lists what the database actually holds after the write, not what you sent; a write that did not take is a `CONFLICT` error. A content edit is also pushed into the post's live editor, so a customer with the post open sees it land; `editor_sync: "failed"` means that push did not happen and they should reopen the post. |
| `get_post` | `post_id`* | the FULL post: content, attachments, poll, mentions, labels, notes, approval state, Instagram options, schedule, account, provenance. Read before scheduling or editing. |
| `list_posts` | statuses, range_days, limit | draft/scheduled/published/failed |

## Media

| Tool | Args | Notes |
|---|---|---|
| `generate_image` | post_id*, instructions | design an image FROM THE POST (its text + the workspace's default template, brand colours and image model — the app's own pipeline) and attach it. `instructions` = only the customer's specific ask, else omit. Needs post text first. Returns a preview image to show them. Costs AI credit. |
| `create_media_upload` | post_id*, filename*, content_type* | the customer's own file: returns a presigned PUT + `media_key`. Upload the bytes, then `attach_media`. |
| `attach_media` | post_id*, media_key OR url, content_type | records an uploaded `media_key`, or fetches a public image `url`. LinkedIn supports multiple images, a video, or a document. |
| `remove_attachment` | post_id*, position* | drop the attachment at a position (from get_post) |
| `reorder_attachments` | post_id*, order* | which image is first, second… — pass the current positions (from get_post) in the wanted order, every one exactly once; renumbered from 0 |

A LinkedIn post is EITHER a poll OR media, never both (the Posts API `content` is one type). `set_poll` refuses while media is attached; `generate_image` / `create_media_upload` / `attach_media` refuse while a poll is set. Each refusal says which to remove.

## Per-provider composition

| Tool | Args | Notes |
|---|---|---|
| `set_poll` | post_id*, question*, options* (2–4), duration | LinkedIn poll. duration ONE/THREE/SEVEN/FOURTEEN_DAYS. |
| `remove_poll` | post_id* | remove it |
| `set_instagram_options` | post_id*, collaborators, tagged_accounts, share_to_feed, alt_text, location_id | Instagram collaboration; only the fields you pass change |

Twitter/X **threads** need no tool: put `---` on its own line between tweets in the content.

## Publishing

| Tool | Args | Notes |
|---|---|---|
| `list_social_accounts` | — | the accounts you can publish to, with the `connection_id` each needs; a `problem` marks an expired token |
| `schedule_post` | post_id*, scheduled_at* | returns a PREVIEW + `confirmation_id` + `will_publish_local`; writes nothing yet |
| `confirm_action` | confirmation_id* | performs the previewed schedule; single-use, short TTL |
| `unschedule_post` | post_id* | scheduled → draft, content and media kept; reversible so no confirm. To change the time, just call `schedule_post` again — it reschedules. |

`scheduled_at`: ISO 8601. No zone ⇒ the workspace's timezone. `Z`/offset ⇒ exact instant. Must be in the future; the post needs a `connection_id`. Scheduling must be enabled by an owner in Settings → Workspace. There is no publish-now tool.

## Approvals

| Tool | Args | Notes |
|---|---|---|
| `request_approval` | post_id*, reviewers* (names/emails), blocking, message, due_date | send for review; the post goes `approval_status: pending` |
| `list_pending_approvals` | — | posts awaiting YOUR sign-off; omit workspace_id to span all workspaces |
| `get_approval` | post_id* | who requested it, who it waits on, every decision + comment |
| `submit_approval_decision` | post_id*, decision* (approved/rejected/needs_changes), comment | only a current-step reviewer, once. Approving the last step marks the post approved — it does NOT publish it. |
| `cancel_approval` | post_id* | requester withdraws it |

## Comments

| Tool | Args | Notes |
|---|---|---|
| `list_comments` | post_id* | the internal review thread, each with author, resolved, reply-to, anchored text |
| `add_comment` | post_id*, content*, reply_to, on_text | a comment, or a reply when reply_to is a comment_id |
| `resolve_comment` | comment_id*, resolved | mark resolved (default) or reopen |

## Voice, knowledge & budget

| Tool | Args | Notes |
|---|---|---|
| `update_brand_identity` | any editable field | only the listed `editable_fields` are writable |
| `push_knowledge_item` | `type`*, `title`*, text_content, description, source_url, external_id | type ∈ document, link, website, fireflies_transcript, notion_page, notion_database, granola_meeting |
| `get_usage` | — | today/this month: used, limit, remaining, resets_at |

## Error codes

`UNAUTHORIZED` · `MISSING_SCOPE` · `WORKSPACE_REQUIRED` · `NOT_FOUND` (also for workspaces/posts you cannot access — don't probe) · `NO_ACCOUNT` · `WRONG_PROVIDER` (e.g. a poll on a non-LinkedIn post) · `BLOCKED_BY_APPROVAL` (editing under active review) · `NOT_A_MEMBER` / `AMBIGUOUS_REVIEWER` (assigning approvals) · `NOT_ASSIGNED` / `ALREADY_DECIDED` (approvals) · `ALREADY_PUBLISHED` · `AGENT_ACCESS_DENIED` · `SPEND_LIMIT_EXCEEDED` · `RATE_LIMITED` (100/min) · `SCHEDULING_DISABLED`.

A recoverable error carries a `next_step` telling you what to do about it — follow it rather than retrying the same call. Two of them hand you a list to choose from (`WORKSPACE_REQUIRED`, `NOT_A_MEMBER`); the names in those lists are already made unique, so show them to the customer exactly as written and map their answer back to the id yourself.

**Names are not unique.** Two workspaces, two social accounts or two saved contacts can share a name, and a name that collides comes back with the smallest detail that separates it — "My's Workspace (owner)". Never guess between two same-named things: `AMBIGUOUS_REVIEWER` exists because assigning the wrong approver is worse than asking. One exception to display: a `mention_token` always contains the raw name, so paste it verbatim and never rebuild it from the label.
