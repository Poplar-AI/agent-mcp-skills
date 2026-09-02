# Scopes and the REST API

## Scopes

An API key (or an OAuth grant) carries scopes. Tools refuse with `MISSING_SCOPE` when the key lacks one. Some scopes are plan-gated.

| Scope | Unlocks |
|---|---|
| `workspaces:read` | list_workspaces, get_workspace_settings, get_workspace_allowance, list_social_accounts |
| `workspaces:create` | create_workspace |
| `workspaces:update` | update_workspace_settings |
| `members:invite` | invite_member |
| `knowledge:write` | push_knowledge_item |
| `analytics:read` | get_post_analytics, get_strategy |
| `usage:read` | get_usage |
| `agent:run` | start_draft, get_draft (Pro / Agency — these spend AI credit) |
| `identity:read` | get_brand_identity, list_learned_preferences |
| `identity:write` | update_brand_identity |
| `posts:read` | list_posts, list_ideas, list_known_mentions |
| `posts:write` | create_draft |
| `posts:schedule` | schedule_post, confirm_action |

There is deliberately no `posts:publish` scope exposed to agents.

## REST

Everything the MCP tools do is also available as REST under `https://usepoplar.com/api/v1/*` with the same Bearer key; the interactive reference lives at `https://usepoplar.com/api-docs`. REST differs in one way: workspace-scoped endpoints take `X-Workspace-Id: <uuid>` as a header instead of a tool argument.

Prefer MCP from an agent. Use REST for scripts, CI, or webhooks.

## OAuth (for connector-style clients)

Discovery: `/.well-known/oauth-protected-resource` → `/.well-known/oauth-authorization-server`. Dynamic client registration is open; PKCE S256 is required; redirect URIs must be `https://` (or `http://localhost` for development). The access token you receive is an ordinary Poplar API key limited to the approved scopes.
