# Terminus

Campaign naming and link governance: one shared governance model of fields, picklists and rules that campaign names, UTM-tagged URLs, custom query parameters, short links and QR codes are built from and checked against.

## When to Use

Use when a team needs every campaign name and tagged link to follow one convention: allowed values for `utm_source` and `utm_medium`, cascading dropdowns (a channel limits which sources are valid), campaign names assembled from those values, and review before anything goes live. An agent can build names and links that pass the rules, look up ones that already exist, and change the convention itself when the user is an owner or admin.

For one person tagging a handful of links, a shared sheet of allowed values (see the analytics skill) is enough. For link shortening and click analytics without naming rules, see Dub.co.

## What a Taxonomy Can Produce

Each taxonomy is a naming structure the account defines. What one record contains depends on the fields placed in it, and `describe_taxonomy` shows which fields Terminus computes and from what. Common outputs:

- **Campaign names**: a Concatenation field joins other fields with a separator (`us_email_spring-launch_2026-03`). Auto Number, Random and Auto-fill fields add sequence numbers, unique suffixes and derived values. The same structure works for names that never become a URL, such as ad platform campaign, ad set and ad names, or email and file names.
- **Tagged URLs**: a Tagged URL field appends query parameters to a destination URL. Keys are not limited to UTMs: `cid`, `s_kwcid`, `ref` or any custom key, and several keys can read the same field (`utm_campaign` and `cid` from one campaign name). A parameter can carry an ad platform macro such as `{campaignid}` unencoded, or the record's Terminus ID (for example as `utm_id`). The base can be a fixed URL, such as a mobile attribution link.
- **Short links**: a Short URL field mints a branded short link for each record, on the account's custom domain if one is set up, and records every click.
- **QR codes**: a QR Code field encodes another field, usually the short link, as a downloadable image in the account's QR style. Scans count as clicks on the short link.
- **Terminus ID**: every record gets a unique `trm0...` ID that can be passed in a URL parameter and used to look the record up later.

## Picklists and Validation Rules

Dropdown fields draw their allowed values from picklists. A picklist has one of four structures:

- **Simple**: one flat list, such as seven allowed mediums.
- **Linear cascade**: each level narrows the next (Channel, then Source, then Placement). The same code can live under two parents, and changing a parent clears the values below it.
- **Branching cascade**: one parent narrows several sibling lists at once (Campaign type sets Objective, Asset type and Budget owner).
- **Conditional**: rules swap in a different list when conditions on other fields hold (`region=emea AND channel=display` uses its own placements list).

Options are static (maintained by hand) or automated (taken live from another taxonomy's approved records, so an approved campaign becomes a value later links can pick). A taxonomy or workspace can narrow a dropdown to an include or exclude list. Some dropdowns allow several choices with a minimum and maximum.

Other fields carry their own rules, which `create_submission` enforces on every row:

- **Text**: lowercase or uppercase transforms; spaces removed or replaced with a dash, underscore or custom character; allowed character types (lowercase, uppercase, numbers, spaces, dashes, underscores) plus extra allowed or disallowed characters; minimum and maximum length; contains, starts with, ends with, excludes and regex patterns, each with its own error message; uniqueness across all records; default values and suggestions.
- **Date**: output format built from day, month, year and week parts with a separator, and earliest and latest allowed dates relative to today.
- **URL**: allowed protocols, a domain allowlist, and a check that the domain is well formed and its host resolves to a public address.
- **Required and hidden** settings per field, overridable per taxonomy.

`describe_taxonomy` returns each field's rules and help text, and the preview reports which row and field broke which rule, so the agent can fix the value instead of guessing.

## Batches, Workspaces and Approvals

**A submission is a batch.** One `create_submission` call takes up to 1,000 rows, so a whole campaign's links (every channel, source, placement and creative) can be built, checked and approved together. The account's plan caps how many records one submission can create; the cap is checked when the submission advances, and a larger batch still saves as a draft. In the Terminus app, grid mode does the same by pasting from a spreadsheet, round-tripping a CSV, or generating every combination of a few values.

**Lifecycle:**

1. **Draft**: saves even with row errors, so work is never lost. Errors are attached to the exact row and field.
2. **In review** (only when the workspace requires it): a workspace admin approves or rejects. Editing a submission in review takes it back to draft; a rejected one can be edited and resubmitted.
3. **Approved (live)**: every row must be valid. Each row becomes a permanent record and its short link starts redirecting. Approval is final: to change a record later, submit a new submission seeded from it. Records can be archived.

Submissions always build against the governance model's latest published revision, so an admin's unpublished edits never affect people building links.

**Workspaces** divide an account by team, brand, region or client. Every submission and record belongs to one workspace, taxonomies are connected per workspace, and each member has a role per workspace: Admin (settings and approvals), Member (creates submissions and edits their own) or Viewer (read-only). A workspace can narrow a dropdown's options further, so one region or client sees only its own sources or markets.

**Approvals** are set per taxonomy in each workspace with Require review (Business plan or higher). `describe_taxonomy` with a `workspace_id` returns the review rule, `submit_submission` then sends the batch to review instead of live, and `get_submission` reports where it stands.

## Analytics and Delivery

Every short link click and QR scan is recorded from the moment its submission is approved, with the visitor's approximate country, device, browser, OS, referrer and the UTM parameters on the link. In the Terminus app:

- **Overview**: total clicks, unique visitors, active links and QR share for a period, with change against the prior period, and a per-link table (trend, top country, link-versus-QR split, top device) that can be grouped by any field in the governance model.
- **Drill-in**: filter and break down clicks by UTM campaign, source, medium, content and term, country, device, link click versus QR scan, and any governance model field, such as business unit or region.
- **Pivot**: cross-tab two of those dimensions by clicks, unique visitors or unique rate.
- **Per-link stats**: one short link's trend and its geo, device, browser, OS, referrer and link-versus-QR breakdowns.

Because the analytics dimensions are the governance model's own fields, results read in the team's naming convention rather than in raw parameter strings. Analytics are read in the app; the MCP server and the API do not return them yet.

Approved records can also be pushed to your own HTTPS endpoint with an outbound webhook connector, with signed, idempotent deliveries.

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
- **Taxonomy**: one kind of record, such as "Paid social link", "Campaign" or "Ad name", with the fields placed in it and their rules.
- **Workspace**: where a taxonomy is turned on, with its own members, value restrictions and review rule.
- **Submission**: a draft of one or more rows. Nothing goes live until it is submitted, and a workspace can require a reviewer.
- **Record**: a published row. Each has a Terminus ID (`trm0...`) and its computed values, such as the campaign name, tagged URL, short link and QR code.

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

### Build names and tagged links that pass the convention

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

5. Show the user the preview (computed names, URLs and any row errors). Fix errors with `update_submission`. Short links go live when the submission does.
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
- [Tagged URL field](https://docs.terminus.app/reference/fields/tagged-url)
- [Short URL field](https://docs.terminus.app/reference/fields/short-url)
- [QR Code field](https://docs.terminus.app/reference/fields/qr-code)
- [Concatenation field](https://docs.terminus.app/reference/fields/concatenation)
- [Picklist structures](https://docs.terminus.app/concepts/picklist-structures)
- [Link and QR analytics](https://docs.terminus.app/guides/analytics)
- [Public API overview](https://docs.terminus.app/api-reference/overview)
- [API keys](https://docs.terminus.app/account/api-keys)
- [Pagination, errors and rate limits](https://docs.terminus.app/api-reference/pagination-errors)
