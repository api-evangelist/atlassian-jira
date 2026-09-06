---
name: atlassian-jira-search-work-items-with-jql
description: Search Jira with JQL and page through the results correctly — validating the query before running it, choosing the right search endpoint, and keeping the rate-limit cost down.
api: Atlassian Jira Cloud platform REST API v3
generated: '2026-09-06'
method: generated
source: Grounded in openapi/atlassian-jira-platform-openapi.json (harvested first-party from developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json on 2026-09-06). Every operationId below was verified present in that document.
operations:
  - parseJqlQueries
  - searchAndReconsileIssuesUsingJql
  - searchForIssuesUsingJql
  - searchForIssuesUsingJqlPost
  - bulkFetchIssues
  - getFields
  - getIssuePickerResource
---

# Search Jira work items with JQL

## Steps

1. **Validate the query before you run it.** `parseJqlQueries` — `POST /rest/api/3/jql/parse`.
   Body: `{"queries": ["project = ABC AND status != Done ORDER BY created DESC"]}`. It returns the
   parse errors without executing anything. This is the read-side equivalent of a dry run and it costs
   almost nothing.

2. **Resolve field names once.** `getFields` — `GET /rest/api/3/field` returns every system and custom
   field with its id, name and searchable clause names. JQL matches on clause names, and a custom
   field's id is site-specific.

3. **Run the search.** Prefer `searchAndReconsileIssuesUsingJql` — `GET /rest/api/3/search/jql`.
   - Paginate with `nextPageToken` and `maxResults`; the response sets `isLast`.
   - This is the cursor-based endpoint. The older offset-based `searchForIssuesUsingJql`
     (`GET /rest/api/3/search`) and `searchForIssuesUsingJqlPost` (`POST /rest/api/3/search`) still work
     and take `startAt`/`maxResults`, but the cursor endpoint is the one to build on.
   - Use `POST` when the JQL is long enough to run into URL length limits.

4. **Ask for only the fields you need.** `fields=summary,status,assignee`. Negation (`-description`) and
   the keywords `*all` and `*navigable` are supported. `expand=renderedFields,changelog,names` pulls in
   extra sections.

5. **Hydrate in bulk, not one by one.** If you have a list of keys already, `bulkFetchIssues` —
   `POST /rest/api/3/issue/bulkfetch` fetches up to 1000 issues in one request (raised from 100 on
   2026-08-28). One bulk call is dramatically cheaper than 1000 `getIssue` calls.

6. For a typeahead over issue keys and summaries, `getIssuePickerResource` —
   `GET /rest/api/3/issue/picker?query=...` is purpose-built and much lighter than a JQL search.

## Cost — this is the part agents get wrong

Jira meters an **hourly points budget**, not requests per month. A read costs 1 point plus 1 point per
object returned. `maxResults=100` costs about 101 points; `maxResults=10` costs about 11. Ten pages of
10 cost roughly the same as one page of 100 — but requesting fields you throw away costs nothing extra,
while requesting objects you throw away costs a point each.

- Default app quota is 65,000 points/hour shared across **all** tenants (the Global Pool).
- Watch `X-RateLimit-NearLimit: true`, which appears at under 20% remaining, before any 429.
- On `429`, read `RateLimit-Reason`: `jira-burst-based` clears in a second,
  `jira-quota-tenant-based` may not clear for the rest of the hour.

## Errors

- `400` with an `errorMessages[]` entry naming the JQL clause — you skipped step 1.
- `403` — missing `read:jira-work` scope or the Browse Projects permission.
- `410`/`404` on a project in the query — Jira hides what you cannot see rather than saying so.

See `rate-limits/atlassian-jira-rate-limits.yml` and `conventions/atlassian-jira-conventions.yml`.
