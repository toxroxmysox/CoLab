# CoLab Power Query API Wrapper

## Objective

This repo provides a reusable Power Query (M language) wrapper around CoLab's enterprise REST API
(`https://app.colabsoftware.com/enterprise/v1/`). The goal is to pull CoLab data into Power BI
with zero hardcoded credentials, automatic token-based pagination, exponential-backoff retry, and
optional server-side filtering. Individual query files call the single core function to hydrate
separate tables in the Power BI data model.

---

## Repo File Inventory

| File | Type | Role |
|---|---|---|
| `CoLabGet` | Power Query function | Core function — defines and returns `CoLabEndpoint` |
| `CoLabTest` | Power Query function | Diagnostic probe — tests a single endpoint, returns a result record |
| `CoLabHealthCheck` | Power Query query | Runs `CoLabTest` on all known endpoints, returns a summary table |
| `GetReviews` | Power Query query | Data query — calls `CoLabEndpoint("reviews", false)` |
| `GetUsers` | Power Query query | Data query — calls `CoLabEndpoint("users", false)` |
| `GetFeedback` | Power Query query | Data query — calls `CoLabEndpoint("feedback", true)` |
| `GetFiles` | Power Query query | Data query — calls `CoLabEndpoint("files", false)` |
| `GetWorkspaces` | Power Query query | Data query — calls `CoLabEndpoint("workspaces", false)` |
| `GetPortals` | Power Query query | Data query — calls `CoLabEndpoint("portals", true)` |
| `GetAuditRecords` | Power Query query | Data query — calls `CoLabEndpoint("audit-records", true)` |
| `GetChecklistTemplates` | Power Query query | Data query — calls `CoLabEndpoint("checklist_templates", false)` |
| `GetAuditLogins` | Power Query query | Filtered audit query — login operations only via `extraQuery` |
| `ApiDocs` | JSON (OpenAPI 3.x) | Full API spec for `https://app.colabsoftware.com/enterprise/v1/` |
| `CLAUDE.md` | Markdown | This file — repo context for AI-assisted development |

> **Convention:** Files have no `.pq` extension in this repo. Each file is pasted directly into a
> Power BI query or function. The file name matches the Power Query query/function name inside
> Power BI Desktop.
>
> **Load order when setting up a new Power BI file:**
> 1. Create the three PQ Parameters: `ClientId`, `ApiKey`, `Filters`
> 2. Load `CoLabGet` as a function query named `CoLabGet`
> 3. Load `CoLabTest` as a function query named `CoLabTest`
> 4. Load `CoLabHealthCheck` as a regular query named `CoLabHealthCheck`
> 5. Load data queries (`GetReviews`, `GetUsers`, `GetFeedback`, `GetFiles`,
>    `GetWorkspaces`, `GetPortals`, `GetAuditRecords`, `GetChecklistTemplates`,
>    `GetAuditLogins`) last

---

## Power Query Parameters (already exist in the Power BI file)

These are **Power Query Parameters** — never hardcode their values in any query file.

| Parameter name | Used as | Notes |
|---|---|---|
| `ClientId` | `X-Client-Id` request header | Numeric or string company identifier |
| `ApiKey` | `Authorization` request header | Secret API key — treat as credential |
| `Filters` | `?filters=` query param (JSON string) | Optional; used when `applyFilters = true` |

`Filters` must be a valid JSON object serialised as a string, e.g.:
```json
{"status": {"operator": "equals", "value": "open"}}
```

---

## Core Function — `CoLabEndpoint`

**File:** `CoLabGet`

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

| Parameter | Required | Default | Description |
|---|---|---|---|
| `endpoint` | Yes | — | URL path segment appended to `enterprise/v1/`. E.g. `"reviews"`, `"users"`, `"feedback"` |
| `applyFilters` | No | `true` | When `true`, serialises the `Filters` PQ parameter and sends it as `?filters=<json>`. Pass `false` to skip filters entirely |
| `days` | No | `null` | Sends `?days=N` to the API. Useful for rolling-window queries on audit records |
| `extraQuery` | No | `null` | Record of additional query-string key/value pairs merged in last, e.g. `[ page_size = "500" ]` |

### Example calls

```powerquery
// All reviews, no filter
CoLabEndpoint("reviews", false)

// Users, applying the Filters PQ parameter
CoLabEndpoint("users", true)

// Last 30 days of audit records, no filter
CoLabEndpoint("audit-records", false, 30)

// Feedback with filters and a sort override
CoLabEndpoint("feedback", true, null, [ sort = "{""date_created"": {""order"": ""desc""}}" ])
```

### Internal helpers (all inside `CoLabGet`)

| Helper | What it does |
|---|---|
| `FetchWithRetry` | Wraps `Web.Contents`; retries on 429/500/502/503 up to 4 times with exponential backoff (1 s, 2 s, 4 s, 8 s). Hard-errors on 400/401/403/404/409/422 |
| `BuildQuery` | Assembles the query record for each page: `next_page` token + optional `days` + optional `filters` + `extraQuery` |
| `ExtractList` | Pulls the record array from `json[data]`, `json[records]`, or `json[items]` |
| `ExtractNextToken` | Reads `json[next_page]` or `json[next]` for the continuation token |
| `TokenPagedFetch` | `List.Generate` loop — fetches pages until `next_page` is null or a repeated token (cycle guard) |
| `ListToTable` | Converts the flat list of records to a `table`; falls back gracefully if records are not uniform |

---

## API Fundamentals

- **Base URL:** `https://app.colabsoftware.com`
- **Path prefix:** `enterprise/v1/`
- **Auth:** Two headers on every request:
  - `Authorization: <ApiKey>`
  - `X-Client-Id: <ClientId>`
- **Pagination:** Cursor-based. Each response includes `"next_page": "<base64 token or null>"`.
  Pass the token back as `?next_page=<token>` to get the next page.
- **Response envelope:**
  ```json
  { "data": [...], "next_page": "abc123==", "status_code": 200 }
  ```

---

## Available GET Endpoints

These are the endpoints suitable for `CoLabEndpoint`. All return paginated `data` arrays.

| Endpoint string | API path | Notes |
|---|---|---|
| `"reviews"` | `/enterprise/v1/reviews` | Core review records |
| `"users"` | `/enterprise/v1/users` | Company user roster |
| `"users/roles"` | `/enterprise/v1/users/roles` | Permission role definitions |
| `"feedback"` | `/enterprise/v1/feedback` | Feedback items / issues |
| `"files"` | `/enterprise/v1/files` | File/revision records |
| `"workspaces"` | `/enterprise/v1/workspaces` | Workspace list |
| `"portals"` | `/enterprise/v1/portals` | Portal list |
| `"audit-records"` | `/enterprise/v1/audit-records` | Audit log |
| `"checklist_templates"` | `/enterprise/v1/checklist_templates` | Checklist template library |

Endpoints with path parameters (e.g. `/portals/{uuid}/users`) require constructing the path string
manually and passing it as `endpoint`, e.g. `CoLabEndpoint("portals/42/users", false)`.

---

## Response Fields by Endpoint

### `reviews`
`checklists`, `completed_date`, `date_created`, `description`, `due_date`, `file_id`, `id`,
`name`, `owner_emails`, `owner_ids`, `package`, `package_files`, `portal`, `review_creator`,
`reviewer_ids`, `reviewers`, `revision_id`, `start_date`, `status`, `total_feedback`,
`unresolved_feedback`, `url`, `workspace`

### `users`
`company_role`, `created_time`, `default_workspace_role`, `email`, `first_name`, `id`,
`is_admin`, `job_title`, `last_name`, `modified_time`, `status`, `username`

### `feedback`
`date_closed`, `date_created`, `description`, `details`, `file`, `id`, `issue_assignee`,
`issue_creator`, `key`, `resolution`, `review`, `revision_id`, `status`, `tags`, `title`,
`type`, `url`, `workspace`

### `files`
`creator`, `date_created`, `directory_parent_id`, `feedback_count`, `file_id`, `name`, `path`,
`revision_id`, `upload_request_id`, `url`, `workspace`

### `workspaces`
`created_by_user_id`, `created_time`, `id`, `is_downloading_allowed`, `is_restricted`,
`modified_time`, `name`, `url`

### `portals`
`created_by_user_id`, `created_time`, `is_downloading_allowed`, `modified_time`, `name`, `url`,
`uuid`, `workspace`

### `audit-records`
`action_type`, `api_key_id`, `api_key_title`, `company_id`, `company_name`, `details`,
`event_time`, `log_message`, `operation`, `result`, `source_ip`, `user_agent`, `user_email`,
`user_id`, `uuid`, `workspace_id`, `workspace_name`

### `checklist_templates`
`date_created`, `id`, `name`

---

## Filtering

When `applyFilters = true`, the `Filters` PQ parameter value is sent as `?filters=<json>`.

### Filter object shape
```json
{
  "<field>": { "operator": "<op>", "value": <value> }
}
```

### Available operators
| Operator | Meaning |
|---|---|
| `equals` / `not_equals` | Exact match / exclusion |
| `in` / `not_in` | Match against a list of values |
| `contains` / `not_contains` | Substring match |
| `like` / `not_like` | Pattern match |
| `gt` / `gte` / `lt` / `lte` | Numeric / date comparisons |
| `between` | Range (value is a two-element array) |
| `not_null` | Field is present and non-null |

### Filterable fields per endpoint

| Endpoint | Filterable fields |
|---|---|
| `reviews` | `completed_date`, `date_created`, `description`, `due_date`, `file_id`, `id`, `name`, `portal_uuid`, `start_date`, `status`, `workspace_id` |
| `users` | `created_time`, `email`, `first_name`, `id`, `job_title`, `last_name`, `modified_time`, `status`, `username` |
| `feedback` | `asset_id`, `date_closed`, `date_created`, `description`, `id`, `key`, `priority`, `resolution`, `review_id`, `revision_id`, `status`, `title`, `workspace_id` |
| `files` | *(none documented in spec — use `extraQuery` for path-param filtering)* |
| `workspaces` | `created_by_user_id`, `created_time`, `id`, `is_downloading_allowed`, `modified_time`, `name` |
| `portals` | `created_by_user_id`, `created_time`, `modified_time`, `name`, `uuid`, `workspace_id` |
| `audit-records` | `action_type`, `api_key_id`, `api_key_title`, `event_time`, `log_message`, `operation`, `result`, `source_ip`, `user_agent`, `user_email`, `user_id`, `workspace_id`, `workspace_name` |
| `checklist_templates` | `date_created`, `id`, `name` |

### Example `Filters` parameter value
```json
{"status": {"operator": "equals", "value": "open"}, "workspace_id": {"operator": "in", "value": [1, 2, 3]}}
```

---

## Sorting

Sorting is not directly exposed as a `CoLabEndpoint` parameter yet. Pass it via `extraQuery`:

```powerquery
CoLabEndpoint(
    "reviews",
    false,
    null,
    [ sort = "{""date_created"": {""order"": ""desc""}}" ]
)
```

Sort order values: `"asc"` | `"desc"`

---

## Testing and Diagnostics

### `CoLabTest` — probe a single endpoint

**File:** `CoLabTest`
**Depends on:** PQ Parameters `ClientId`, `ApiKey`, `Filters` (does **not** depend on `CoLabGet`)

```powerquery
CoLabTest(endpoint as text, optional applyFilters as nullable logical) as record
```

Makes exactly **one** request (no pagination) and returns a diagnostic record. Use this to:
- Verify credentials are working before running full queries
- Check what columns an endpoint returns before building a data query
- Get a sample record to understand nested field structure
- Diagnose 401/403/404 errors without running the full paginator

**Return record fields:**

| Field | Type | Meaning |
|---|---|---|
| `endpoint` | text | Endpoint string that was tested |
| `success` | logical | `true` if HTTP status < 400 and no M exception |
| `status_code` | number | HTTP status (null on network-level exception) |
| `record_count` | number | Records on the first page (0 on failure) |
| `has_more_pages` | logical | `true` if a `next_page` token was present |
| `columns` | list | Field names from the first record |
| `sample_record` | record | First record returned (null on failure or empty result) |
| `error_message` | text | Human-readable error detail (null on success) |

**Usage — paste into a blank query to debug interactively:**

```powerquery
// Paste into a blank query, run, then inspect the record fields
CoLabTest("reviews", false)
```

To inspect a specific field from the returned record:
```powerquery
CoLabTest("reviews", false)[columns]      // see column list
CoLabTest("reviews", false)[sample_record] // see first row
CoLabTest("reviews", false)[error_message] // see error if failed
```

**`applyFilters` defaults to `false`** in `CoLabTest` (opposite of `CoLabEndpoint`) so baseline
probes always succeed even if the `Filters` parameter is misconfigured.

---

### `CoLabHealthCheck` — check all endpoints at once

**File:** `CoLabHealthCheck`
**Depends on:** `CoLabTest` (must be loaded first)

Load as a regular query. Returns a table with one row per endpoint:

| Column | Meaning |
|---|---|
| `Endpoint` | Endpoint string |
| `Success` | `true` / `false` |
| `Status_Code` | HTTP status |
| `Record_Count` | Records on first page |
| `Has_More_Pages` | `true` if data continues past first page |
| `Column_Count` | Number of fields per record |
| `Columns` | Comma-separated field names |
| `Error_Message` | null on success; error detail on failure |

**Workflow:** Run `CoLabHealthCheck` after any credential change, after updating `CoLabGet`, or
when setting up a new Power BI file. A fully green table (all `Success = true`) confirms the
connection is working before loading full data queries.

**To add a new endpoint to the health check**, add its string to `KnownEndpoints` inside
`CoLabHealthCheck`.

---

## How to Add a New Query

1. Create a new file (e.g. `GetUsers`) in the repo.
2. Reference `CoLabEndpoint` — it is available as a shared function inside the Power BI file.
3. Minimal template:

```powerquery
let
  Source = CoLabEndpoint("users", false)
in
  Source
```

4. Load the file content into a new Power BI query. Set the query name to match the file name.
5. Do **not** paste `CoLabGet` contents into query files — reference it by name.

---

## Known Behaviours and Gotchas

- **`applyFilters` defaults to `true`** when omitted. Pass `false` explicitly if you do not want
  server-side filtering applied, or if `Filters` is empty and you don't want a stray `?filters=`
  param sent.
- **`Filters` PQ parameter must be valid JSON** when `applyFilters = true`. An unparseable value
  falls back to sending the raw string, which may cause a 400 error.
- **`days` is not in the OpenAPI spec** for most endpoints but is documented in the `audit-records`
  sort schema. It works as a rolling-window shortcut. Test per endpoint before relying on it.
- **Pagination cycle guard:** `List.Generate` tracks seen tokens and stops if a `next_page` token
  repeats — prevents infinite loops on a misbehaving API.
- **`ExtractList` fallback order:** `data` → `records` → `items` → `{}`. The CoLab API
  consistently uses `data`, so the others are defensive fallbacks.
- **Retry scope:** Only 429, 500, 502, 503 are retried. 401/403 are not retried — fix credentials
  in the PQ parameters, not in code.
- **Power BI refresh:** `Web.Contents` with `RelativePath` and `Query` is the correct pattern for
  Power BI gateway compatibility. Do not move the base URL into `RelativePath`.

---

## Planned / Future Work

- Filter merging: enhance `BuildQuery` to merge `extraQuery` filters with the `Filters` PQ
  parameter instead of overriding, enabling audit operation queries to also apply shared
  workspace exclusions automatically
- Additional audit operation queries (e.g. `GetAuditFileUploads`, `GetAuditReviewCreations`)
  following the `GetAuditLogins` pattern
- Sort parameter surfaced as a dedicated `CoLabEndpoint` argument
- `page_size` exposed as an optional argument (API supports it via `extraQuery` already)
- Relationship table linking `reviews[workspace][id]` → `workspaces[id]`, etc., for the
  Power BI data model star schema
- Incremental refresh integration using `date_created` / `event_time` filter fields
- `CoLabTest` extended to optionally probe with `applyFilters = true` so filter JSON can also
  be validated before running full paginated queries
