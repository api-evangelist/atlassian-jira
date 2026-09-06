---
name: atlassian-jira-log-work-and-comment
description: Add comments and worklogs to a Jira work item in Atlassian Document Format, and reverse them cleanly — the two write operations an agent performs most often and the two most likely to be duplicated on retry.
api: Atlassian Jira Cloud platform REST API v3
generated: '2026-09-06'
method: generated
source: Grounded in openapi/atlassian-jira-platform-openapi.json (harvested first-party from developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json on 2026-09-06). Every operationId below was verified present in that document.
operations:
  - getIssue
  - getComments
  - addComment
  - updateComment
  - deleteComment
  - getIssueWorklog
  - addWorklog
  - updateWorklog
  - deleteWorklog
  - addAttachment
  - removeAttachment
---

# Comment on and log work against a Jira work item

## Atlassian Document Format is mandatory

In the v3 API, `body` on a comment and `comment` on a worklog are **ADF JSON document trees**, not
strings. The minimal valid body is:

```json
{"type":"doc","version":1,"content":[{"type":"paragraph","content":[{"type":"text","text":"..."}]}]}
```

Passing a string is the most common `400` on these endpoints. (The v2 API took wiki-markup strings;
v2 is deprecated.)

## Comments

1. **Read existing comments first** — `getComments` —
   `GET /rest/api/3/issue/{issueIdOrKey}/comment?startAt=0&maxResults=50`. This is also how you avoid
   posting a duplicate after a timed-out retry, because there is no idempotency key.
2. **Add** — `addComment` — `POST /rest/api/3/issue/{issueIdOrKey}/comment`. Returns the comment with
   its `id`. Keep the id.
3. **Edit** — `updateComment` — `PUT /rest/api/3/issue/{issueIdOrKey}/comment/{id}`.
4. **Remove** — `deleteComment` — `DELETE /rest/api/3/issue/{issueIdOrKey}/comment/{id}`. Fully
   reverses step 2, at any time.
5. Restrict visibility with the `visibility` object (`{"type":"role","value":"Administrators"}`) — use
   this rather than assuming a comment is internal.

## Worklogs

1. **Read** — `getIssueWorklog` — `GET /rest/api/3/issue/{issueIdOrKey}/worklog`.
2. **Add** — `addWorklog` — `POST /rest/api/3/issue/{issueIdOrKey}/worklog` with
   `timeSpentSeconds` (or `timeSpent` as `"3h 30m"`), `started` as an ISO 8601 timestamp with offset,
   and optionally an ADF `comment`.
   - `adjustEstimate` (`new`, `leave`, `manual`, `auto`) controls what happens to the remaining
     estimate. `leave` is the safe choice for an agent that should not silently reshape a plan.
3. **Edit** — `updateWorklog` — `PUT /rest/api/3/issue/{issueIdOrKey}/worklog/{id}`.
4. **Remove** — `deleteWorklog` — `DELETE /rest/api/3/issue/{issueIdOrKey}/worklog/{id}`. Set
   `adjustEstimate` on the delete too, or the estimate will not be restored.

## Attachments

`addAttachment` — `POST /rest/api/3/issue/{issueIdOrKey}/attachments` requires **two** unusual headers:
`X-Atlassian-Token: no-check` and `Content-Type: multipart/form-data`, with the file in a part named
`file`. Reversed by `removeAttachment` — `DELETE /rest/api/3/attachment/{id}`.

## Reversibility summary

| Write | Reversal | Window |
|---|---|---|
| `addComment` | `deleteComment` | unbounded |
| `addWorklog` | `deleteWorklog` | unbounded |
| `addAttachment` | `removeAttachment` | unbounded |

All three are cleanly reversible with a documented inverse operation and no stated expiry — unlike
`deleteIssue`, which has no REST restore.

## Errors

- `400` — almost always a string where ADF was expected, or a malformed `started` timestamp (it needs
  the offset, e.g. `2026-09-06T10:00:00.000+0000`).
- `403` — missing Add Comments / Work On Issues permission, or missing `write:jira-work` scope.
- `413` — the attachment exceeds the site's attachment size limit.
- `429` with `RateLimit-Reason: jira-per-issue-on-write` — 20 writes / 2s or 100 writes / 30s against
  one issue. Comment and worklog loops hit this before any global quota.

See `conventions/atlassian-jira-conventions.yml` and `errors/atlassian-jira-problem-types.yml`.
