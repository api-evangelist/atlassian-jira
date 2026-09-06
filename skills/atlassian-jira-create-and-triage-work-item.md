---
name: atlassian-jira-create-and-triage-work-item
description: Create a Jira work item in the right project with valid fields, assign it, and link it — without guessing at field names or blindly retrying a create that may already have succeeded.
api: Atlassian Jira Cloud platform REST API v3
generated: '2026-09-06'
method: generated
source: Grounded in openapi/atlassian-jira-platform-openapi.json (harvested first-party from developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json on 2026-09-06). Every operationId below was verified present in that document.
operations:
  - getCurrentUser
  - searchProjects
  - getCreateIssueMetaIssueTypes
  - getCreateIssueMetaIssueTypeId
  - createIssue
  - getIssue
  - findAssignableUsers
  - assignIssue
  - getIssueLinkTypes
  - linkIssues
  - addWatcher
  - deleteIssue
---

# Create and triage a Jira work item

Base URL: `https://api.atlassian.com/ex/jira/{cloudId}` with an OAuth 2.0 (3LO) bearer token, or
`https://your-domain.atlassian.net` with `Authorization: Basic base64(email:api-token)`.
Get `cloudId` from `GET https://api.atlassian.com/oauth/token/accessible-resources`.

## Before you write anything

**There is no idempotency key on this API.** A `createIssue` that times out may have succeeded. Do not
retry it blind — search for the item you were about to create first (see the JQL search skill), or
accept a duplicate. This is the single most important fact about writing to Jira.

## Steps

1. **Confirm who you are.** `getCurrentUser` — `GET /rest/api/3/myself`. Returns your `accountId`,
   which is the only user identifier this API accepts. Usernames and userKeys were removed.

2. **Find the project.** `searchProjects` — `GET /rest/api/3/project/search?query=...`. Prefer this over
   `getAllProjects`, which is deprecated and unpaginated. Keep the `key`.

3. **Ask what fields the project will actually accept.** This is the pre-flight step that replaces a
   dry run:
   - `getCreateIssueMetaIssueTypes` — `GET /rest/api/3/issue/createmeta/{projectIdOrKey}/issuetypes`
   - `getCreateIssueMetaIssueTypeId` — `GET /rest/api/3/issue/createmeta/{projectIdOrKey}/issuetypes/{issueTypeId}`

   These return the required fields, allowed values, and the custom field ids (`customfield_10014`
   style) for this project and issue type. Never hardcode a custom field id across projects — the same
   field has a different id on a different site.

4. **Create it.** `createIssue` — `POST /rest/api/3/issue`.
   - `fields.project.key`, `fields.issuetype.id`, `fields.summary` are the minimum.
   - `fields.description` **must be Atlassian Document Format**, not a string. v3 takes an ADF JSON
     document tree; a plain string is a 400. This is the number one cause of failed first calls.
   - The response carries `id`, `key` and `self`. Store the `key`.

5. **Assign it.** `findAssignableUsers` — `GET /rest/api/3/user/assignable/search?project=KEY&query=...`
   to resolve a name to an `accountId`, then `assignIssue` —
   `PUT /rest/api/3/issue/{issueIdOrKey}/assignee` with `{"accountId": "..."}`.
   Assigning to nobody is `{"accountId": null}`.

6. **Link it, if it belongs to something.** `getIssueLinkTypes` —
   `GET /rest/api/3/issueLinkType` for the legal link names on this site, then `linkIssues` —
   `POST /rest/api/3/issueLink`.

7. **Add watchers** with `addWatcher` — `POST /rest/api/3/issue/{issueIdOrKey}/watchers`, body is a
   bare JSON string containing the accountId.

8. **Verify** with `getIssue` — `GET /rest/api/3/issue/{issueIdOrKey}?expand=names,renderedFields`.

## Reversibility

`createIssue` is reversed by `deleteIssue` (`DELETE /rest/api/3/issue/{issueIdOrKey}`), but **deletion
is itself destructive and there is no REST operation that restores a deleted work item.** Jira's
project trash is a UI/admin surface with no documented retention window in the API reference. Treat
`deleteIssue` as one-way. Assignment, links and watchers are all freely reversible.

## Errors

Jira does **not** return RFC 9457 problem+json. Errors come back as
`{ "errorMessages": [...], "errors": { "<field>": "<message>" } }` with `application/json`.

- `400` — read `errors{}`; it names the offending field. Usually an ADF-vs-string description, a
  custom field id from the wrong project, or a value not in `allowedValues`.
- `403` — the account lacks the Jira permission or your token lacks the scope (`write:jira-work`).
- `404` — the project or issue does not exist **or you cannot see it**. Jira returns 404 rather than
  403 for objects the caller cannot browse.
- `429` — read `RateLimit-Reason`. `jira-per-issue-on-write` means you exceeded 20 writes in 2 seconds
  or 100 in 30 seconds against one issue; slow down per issue, not globally. Honor `Retry-After`.

Quote `atl-request-id` from the response headers in any support report.

See `conventions/atlassian-jira-conventions.yml`, `errors/atlassian-jira-problem-types.yml` and
`rate-limits/atlassian-jira-rate-limits.yml`.
