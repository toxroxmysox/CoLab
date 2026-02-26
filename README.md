# CoLab Power Query API Wrapper

A reusable [Power Query (M language)](https://learn.microsoft.com/en-us/powerquery-m/) wrapper around the [CoLab](https://app.colabsoftware.com) enterprise REST API for use in Power BI Desktop. Provides automatic token-based pagination, exponential-backoff retry, and optional server-side filtering — with zero hardcoded credentials.

---

## Prerequisites

- **Power BI Desktop** with the Power Query editor
- A CoLab account with API access
- Your **Client ID** and **API Key** from CoLab

---

## Setup

### 1. Create Power Query Parameters

In Power BI Desktop, open the Power Query editor and create these three parameters (**Home → Manage Parameters → New Parameter**):

| Parameter name | Type | Description |
|---|---|---|
| `ClientId` | Text or Number | Your company identifier (`X-Client-Id` header) |
| `ApiKey` | Text | Your secret API key (`Authorization` header) |
| `Filters` | Text | Optional JSON filter string — can be left blank |

> **Never hardcode these values in any query file.** Always reference the named parameters.

### 2. Load Queries in Order

Paste each file's contents into a new blank query in the Power Query editor, naming each query to match its file name exactly:

1. `CoLabGet` — load as a **function** query
2. `CoLabTest` — load as a **function** query
3. `CoLabHealthCheck` — load as a regular query
4. Data queries — load in any order:
   - `GetReviews`, `GetUsers`, `GetFeedback`, `GetFiles`
   - `GetWorkspaces`, `GetPortals`, `GetAuditRecords`
   - `GetChecklistTemplates`, `GetAuditLogins`

### 3. Verify the Connection

After loading `CoLabHealthCheck`, run it. A fully green table (all `Success = true`) confirms credentials are working before loading full data queries.

---

## File Reference

| File | Role |
|---|---|
| `CoLabGet` | Core function — defines `CoLabEndpoint`, the paginated API fetcher |
| `CoLabTest` | Diagnostic probe — tests a single endpoint, returns a result record |
| `CoLabHealthCheck` | Runs `CoLabTest` on all known endpoints, returns a summary table |
| `GetReviews` | Fetches all review records |
| `GetUsers` | Fetches all user records |
| `GetFeedback` | Fetches feedback items (with filters applied) |
| `GetFiles` | Fetches file/revision records |
| `GetWorkspaces` | Fetches workspace list |
| `GetPortals` | Fetches portal list (with filters applied) |
| `GetAuditRecords` | Fetches the full audit log |
| `GetChecklistTemplates` | Fetches checklist template library |
| `GetAuditLogins` | Filtered audit query — login operations only |
| `ApiDocs` | Full OpenAPI 3.x spec for the CoLab enterprise API |
| `CLAUDE.md` | AI-assisted development context for this repo |

> **Note:** Files have no `.pq` extension. Each file is pasted directly into a Power BI query. File names must match the query/function names inside Power BI Desktop.

---

## Core Function — `CoLabEndpoint`

All data queries call this single function defined in `CoLabGet`.

### Signature

```powerquery
CoLabEndpoint(
    endpoint    as text,
    optional applyFilters as nullable logical,
    optional days         as nullable number,
    optional extraQuery   as nullable record
) as table
```

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `endpoint` | — | URL path segment, e.g. `"reviews"`, `"audit-records"` |
| `applyFilters` | `true` | When `true`, sends the `Filters` PQ parameter as `?filters=<json>` |
| `days` | `null` | Sends `?days=N` for rolling-window queries (useful for audit records) |
| `extraQuery` | `null` | Additional query-string key/value pairs, e.g. `[ page_size = "500" ]` |

### Examples

```powerquery
// All reviews, no filter
CoLabEndpoint("reviews", false)

// Users with the Filters parameter applied
CoLabEndpoint("users", true)

// Last 30 days of audit records
CoLabEndpoint("audit-records", false, 30)

// Feedback with a sort override
CoLabEndpoint("feedback", true, null, [ sort = "{""date_created"": {""order"": ""desc""}}" ])
```

---

## Available Endpoints

| Endpoint string | API path |
|---|---|
| `"reviews"` | `/enterprise/v1/reviews` |
| `"users"` | `/enterprise/v1/users` |
| `"users/roles"` | `/enterprise/v1/users/roles` |
| `"feedback"` | `/enterprise/v1/feedback` |
| `"files"` | `/enterprise/v1/files` |
| `"workspaces"` | `/enterprise/v1/workspaces` |
| `"portals"` | `/enterprise/v1/portals` |
| `"audit-records"` | `/enterprise/v1/audit-records` |
| `"checklist_templates"` | `/enterprise/v1/checklist_templates` |

---

## Filtering

Set the `Filters` PQ parameter to a JSON object and pass `applyFilters = true`:

```json
{"status": {"operator": "equals", "value": "open"}}
```

```json
{"status": {"operator": "in", "value": ["open", "in_progress"]}, "workspace_id": {"operator": "equals", "value": 42}}
```

### Available operators

`equals`, `not_equals`, `in`, `not_in`, `contains`, `not_contains`, `like`, `not_like`, `gt`, `gte`, `lt`, `lte`, `between`, `not_null`

### Filterable fields by endpoint

| Endpoint | Filterable fields |
|---|---|
| `reviews` | `completed_date`, `date_created`, `description`, `due_date`, `file_id`, `id`, `name`, `portal_uuid`, `start_date`, `status`, `workspace_id` |
| `users` | `created_time`, `email`, `first_name`, `id`, `job_title`, `last_name`, `modified_time`, `status`, `username` |
| `feedback` | `asset_id`, `date_closed`, `date_created`, `description`, `id`, `key`, `priority`, `resolution`, `review_id`, `revision_id`, `status`, `title`, `workspace_id` |
| `workspaces` | `created_by_user_id`, `created_time`, `id`, `is_downloading_allowed`, `modified_time`, `name` |
| `portals` | `created_by_user_id`, `created_time`, `modified_time`, `name`, `uuid`, `workspace_id` |
| `audit-records` | `action_type`, `api_key_id`, `api_key_title`, `event_time`, `log_message`, `operation`, `result`, `source_ip`, `user_agent`, `user_email`, `user_id`, `workspace_id`, `workspace_name` |
| `checklist_templates` | `date_created`, `id`, `name` |

---

## Diagnostics

### `CoLabTest` — probe a single endpoint

Makes one request (no pagination) and returns a diagnostic record. Useful for verifying credentials, previewing column names, and inspecting a sample record before running full queries.

```powerquery
// Paste into a blank query to inspect results interactively
CoLabTest("reviews", false)

CoLabTest("reviews", false)[columns]       // column list
CoLabTest("reviews", false)[sample_record] // first row
CoLabTest("reviews", false)[error_message] // error detail on failure
```

Return record fields: `endpoint`, `success`, `status_code`, `record_count`, `has_more_pages`, `columns`, `sample_record`, `error_message`, `raw_body_snippet`

> `applyFilters` defaults to `false` in `CoLabTest` so baseline probes always succeed even if `Filters` is misconfigured.

### `CoLabHealthCheck` — check all endpoints at once

Load as a regular query. Returns a summary table with one row per endpoint showing `Success`, `Status_Code`, `Record_Count`, `Column_Count`, `Columns`, and `Error_Message`.

Run this after any credential change, after updating `CoLabGet`, or when setting up a new Power BI file.

---

## Adding a New Query

1. Create a new file in the repo (e.g. `GetUserRoles`).
2. Reference `CoLabEndpoint` by name — do not copy `CoLabGet` contents into the query.
3. Use the minimal template:

```powerquery
let
  Source = CoLabEndpoint("users/roles", false)
in
  Source
```

4. Paste the contents into a new Power BI query and name it to match the file.
5. Add the endpoint string to `KnownEndpoints` in `CoLabHealthCheck` to include it in future health checks.

---

## Key Behaviours

- **Retry logic:** Transient errors (429, 500, 502, 503) are retried up to 4 times with exponential backoff (1s, 2s, 4s, 8s). Rate-limit (429) waits 60s. Hard errors (400, 401, 403, 404, 409, 422) are not retried.
- **Pagination:** Cursor-based via `next_page` token. A cycle guard prevents infinite loops if a token repeats.
- **`applyFilters` defaults to `true`** in `CoLabEndpoint` when omitted. Pass `false` explicitly to skip filtering.
- **Non-uniform records** across pages merge safely — missing fields become `null` rather than erroring.
- **Power BI gateway compatibility:** `Web.Contents` uses `RelativePath` and `Query` parameters as required by the on-premises data gateway.
