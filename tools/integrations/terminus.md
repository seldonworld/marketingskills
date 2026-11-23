# Terminus

UTM and campaign governance: a shared governance model of fields, picklists and naming rules that every tagged URL, short link and QR code is built from and checked against.

## When to Use

Use when a team needs every tagged link to follow one naming convention: allowed values for `utm_source` and `utm_medium`, cascading dropdowns (a channel limits which sources are valid), computed campaign names, and review before links go live. An agent can build links that pass the rules, look up links that already exist, and change the convention itself when the user is an owner or admin.

For one person tagging a handful of links, a shared sheet of allowed values (see the analytics skill) is enough. For link shortening and click analytics without naming rules, see Dub.co.

## Capabilities

| Integration | Available | Notes |
|-------------|-----------|-------|
| API | ✓ | Read-only REST API for workspaces, governance models, submissions and records |
| MCP | ✓ | Hosted at `https://mcp.terminus.app/mcp`, OAuth sign-in; builds links, reads records, edits governance models |
| CLI | - | Not available |
| SDK | - | Not available; use a standard HTTP client |

The MCP server is the only way for an agent to create links. The REST API cannot write.

## Concepts

- **Governance model**: the convention itself. Picklists (allowed values), fields (text, dropdown, date, concatenation, tagged URL and more) and taxonomies.
- **Taxonomy**: one kind of record, such as "Paid social link" or "Campaign", with the fields placed in it and their rules.
- **Workspace**: where a taxonomy is turned on, with its own members, value restrictions and review rule.
- **Submission**: a draft of one or more rows. Nothing goes live until it is submitted, and a workspace can require a reviewer.
- **Record**: a published row. Each has a Terminus ID (`trm0...`) and its computed values: the tagged URL, short link and QR code.

## Authentication

- **MCP**: OAuth. The user signs in to Terminus and approves the app for one account. API keys do not work for the MCP server. If the client asks for an OAuth client ID and secret, leave both empty.
- **API**: Bearer API key, prefixed `thub_`. Account owners and admins create keys under Settings, then API Keys. A key is shown once.
- **Header**: `Authorization: Bearer $TERMINUS_API_KEY`
- Keep the key in an environment variable or secret manager, never in the repo or in client-side code.

An account owner or admin can turn off app connections for members (Allow members to connect apps, on the Security page). A connected app can only do what the user's own role allows: a viewer's app cannot create submissions.

## MCP Setup

**Claude Code**

```bash
claude mcp add --transport http terminus https://mcp.terminus.app/mcp
```

Then run `/mcp`, choose `terminus` and sign in. Claude Code sees a newly added server only after a restart.

**Claude on the web and Claude Desktop**: Customize, then Connectors, then Add custom connector; name it Terminus and paste `https://mcp.terminus.app/mcp`. On Team and Enterprise plans an organization owner adds the connector first.

**ChatGPT**: add it as a custom app in developer mode. The ChatGPT plan decides whether the app may take write actions such as creating submissions.

Full steps: [Connect AI apps](https://docs.terminus.app/guides/connect-ai-apps).

## MCP Tools

| Tool | What it does | Writes |
|------|--------------|:------:|
| `list_taxonomies` | Taxonomies the user can use and the workspaces each is active in | - |
| `describe_taxonomy` | Fields by handle, which are required, their rules, defaults, fill order, and which fields Terminus computes | - |
| `list_allowed_values` | Values a dropdown accepts, given the values already chosen in the row | - |
| `create_submission` | Draft rows; returns a preview with computed links and errors | Draft only |
| `update_submission` | Replace a draft's rows (send every row) | Draft only |
| `submit_submission` | Make a confirmed draft live, or send it to review | ✓ |
| `get_submission` | Status and values of a submission: links, short links, QR content, errors | - |
| `list_records` | Published records in one taxonomy, newest first, with exact-match filters | - |
| `get_record` | One record by record ID or Terminus ID | - |
| `render_qr_code` | QR image from a record's QR field; view and download links valid 7 days | - |
| `list_governance_models` | Governance models in the account and whether each has unpublished changes | - |
| `get_governance_model` | A model's draft: picklists and options, fields and settings, taxonomies | - |
| `get_governance_guide` | Terminus best practices by topic, such as `utm-parameters` and `naming-conventions` | - |
| `get_pending_changes` | What publishing the draft would change, blocking errors and warnings | - |
| `apply_governance_changes` | Batch edits to a model's draft; supports `dry_run` | Draft only |
| `publish_governance_model` | Publish the draft so new links follow it | ✓ |
| `discard_governance_draft` | Throw away all unpublished changes, including teammates' edits | ✓ |

Terminus has no confirmation step of its own before `submit_submission`, `publish_governance_model` or `discard_governance_draft`. Show the user the preview or pending changes and call these only after they confirm.

## Common Agent Operations (MCP)

### Build tagged links that pass the convention

1. `list_taxonomies` to find the taxonomy and a workspace it is active in.
2. `describe_taxonomy` with that `taxonomy_id` and `workspace_id`. Fill fields in `fill_order` and leave computed fields out.
3. `list_allowed_values` for each dropdown, passing every field in its `depends_on` as `row_values`. A code is only valid under the parent values it was listed for.
4. `create_submission` with one row per link, keyed by field handle:

```json
{
  "taxonomy_id": "TAXONOMY_ID",
  "workspace_id": "WORKSPACE_ID",
  "name": "Spring launch newsletter",
  "rows": [
    {
      "destination_url": "https://example.com/spring",
      "channel": "email",
      "source": "newsletter",
      "campaign": "spring-launch"
    }
  ]
}
```

Field handles differ per account; take them from `describe_taxonomy`, never from this example.

5. Show the user the preview (computed URLs and any row errors). Fix errors with `update_submission`.
6. After the user confirms, `submit_submission` with the `version` from the preview they saw. If the draft changed since, nothing is submitted.

### Make links like an existing one

`list_records` with a `where` filter (for example `{"channel": "paid"}`) or `get_record` with a `trm0...` ID. A record's `values` are the raw inputs `create_submission` takes: copy them, change what the user asked for, and leave out anything under `generated`.

### Find an existing link or short link

`list_records` for the taxonomy, filtered by workspace, status (`active` or `archived`), creation date or exact field values. Paginate with `next_cursor`.

### Change the convention (owners and admins)

1. `list_governance_models`, then `get_governance_model` to read the draft.
2. Read the `get_governance_guide` topics that apply before proposing changes.
3. `apply_governance_changes` with `dry_run: true`, show the user the result, then apply:

```json
{
  "model": "Marketing",
  "dry_run": true,
  "operations": [
    {
      "op": "add_options",
      "picklist": "Sources",
      "options": [{ "code": "reddit", "label": "Reddit" }]
    }
  ]
}
```

4. `get_pending_changes`, show the user everything it lists, and call `publish_governance_model` with its `digest` only when they say to publish.

Chat can create text, URL, dropdown, date, constant, concatenation and tagged URL fields and static picklists. Other field types and automated picklists are changed in the Terminus app; the tool returns a link to the right page. An agent cannot create a new governance model.

## API Operations (read-only)

Base URL: `https://api.terminus.app/api/v1`. Every endpoint is a `GET`.

### List workspaces

```bash
curl "https://api.terminus.app/api/v1/workspaces" \
  -H "Authorization: Bearer $TERMINUS_API_KEY"
```

### List records in a taxonomy

```bash
curl "https://api.terminus.app/api/v1/records?workspace_id=$WORKSPACE_ID&taxonomy_id=$TAXONOMY_ID&limit=500" \
  -H "Authorization: Bearer $TERMINUS_API_KEY"
```

### Get one record

```bash
curl "https://api.terminus.app/api/v1/records/$RECORD_ID" \
  -H "Authorization: Bearer $TERMINUS_API_KEY"
```

API records key field values by field ID, not by handle.

### List submissions in a workspace

```bash
curl "https://api.terminus.app/api/v1/submissions?workspace_id=$WORKSPACE_ID" \
  -H "Authorization: Bearer $TERMINUS_API_KEY"
```

### List governance models

```bash
curl "https://api.terminus.app/api/v1/governance_models" \
  -H "Authorization: Bearer $TERMINUS_API_KEY"
```

List endpoints return `{ "items": [...], "has_more": bool, "total_count": n }`. Paginate with `page` (1-indexed) and `limit` (default 100, maximum 10000) until `has_more` is `false`.

## Rate Limits

- API: 100 requests per minute per key.

## Account

A Terminus account is required. New accounts start on a trial of the Business plan with no card; API access and connected apps are included on every plan.

## Relevant Skills

- analytics
- attribution
- ads
- influencer-marketing

## Links

- [Connect AI apps (MCP)](https://docs.terminus.app/guides/connect-ai-apps)
- [Public API overview](https://docs.terminus.app/api-reference/overview)
- [API keys](https://docs.terminus.app/account/api-keys)
- [Pagination, errors and rate limits](https://docs.terminus.app/api-reference/pagination-errors)
