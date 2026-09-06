---
name: atlassian-jira-manage-sprints-and-boards
description: Drive Jira Software boards and sprints over the Agile REST API — find the board, read the sprint, move work into it, and avoid the deprecated issue-listing endpoints.
api: Jira Software Cloud API (Agile REST API 1.0)
generated: '2026-09-06'
method: generated
source: Grounded in openapi/atlassian-jira-software-openapi.json (harvested first-party from developer.atlassian.com/cloud/jira/software/swagger.v3.json on 2026-09-06). Every operationId below was verified present in that document, including its deprecated flag.
operations:
  - getAllBoards
  - getConfiguration
  - getAllSprints
  - getSprint
  - createSprint
  - updateSprint
  - partiallyUpdateSprint
  - moveIssuesToSprintAndRank
  - deleteSprint
---

# Manage Jira sprints and boards

This is a **different API surface** from the platform REST API. Base path is `/rest/agile/1.0`, not
`/rest/api/3`, on the same host and with the same credentials. Scopes are
`read:board-scope:jira-software`, `read:sprint:jira-software`, `write:sprint:jira-software` and friends.

## Steps

1. **Find the board.** `getAllBoards` — `GET /rest/agile/1.0/board?projectKeyOrId=ABC`. Paginate with
   `startAt`/`maxResults`; the response sets `isLast`.

2. **Read its configuration.** `getConfiguration` —
   `GET /rest/agile/1.0/board/{boardId}/configuration` returns the board's filter, columns, estimation
   field and ranking field. You need the estimation field id before you can read or write story points.

3. **List sprints.** `getAllSprints` — `GET /rest/agile/1.0/board/{boardId}/sprint?state=active,future`.
   `getSprint` — `GET /rest/agile/1.0/sprint/{sprintId}` for one.

4. **Create a sprint.** `createSprint` — `POST /rest/agile/1.0/sprint` with `originBoardId`, `name`,
   and optionally `startDate`/`endDate`. A sprint is created in the `future` state.

5. **Start or close it.** `updateSprint` — `PUT /rest/agile/1.0/sprint/{sprintId}` (full replace) or
   `partiallyUpdateSprint` — `POST /rest/agile/1.0/sprint/{sprintId}` (partial) with
   `{"state": "active"}` or `{"state": "closed"}`. Starting a sprint requires `startDate` and `endDate`.

6. **Move work into the sprint.** `moveIssuesToSprintAndRank` —
   `POST /rest/agile/1.0/sprint/{sprintId}/issue` with `{"issues": ["ABC-1","ABC-2"]}`. Maximum 50
   issues per call. The same body accepts `rankBeforeIssue` / `rankAfterIssue` to control ordering.

## Do not use the deprecated issue-listing endpoints

The Jira Software spec marks eight operations `deprecated: true`, and they are exactly the ones a
naive integration reaches for first:

`getIssuesForBacklog`, `getIssuesForBoard`, `getIssuesForSprint`, `getIssuesForEpic`,
`getIssuesWithoutEpic`, `getBoardIssuesForSprint`, `getBoardIssuesForEpic`, `getIssuesWithoutEpicForBoard`.

Read sprint and board contents through JQL on the platform API instead —
`GET /rest/api/3/search/jql` with `sprint = <id>` or `board = <id>` — which is the supported path and
gives you cursor pagination. Atlassian's REST API policy guarantees a deprecated endpoint for six
months from its changelog notice, so anything bound to these has a clock on it.

## Reversibility

- `createSprint` → `deleteSprint` (`DELETE /rest/agile/1.0/sprint/{sprintId}`), unbounded.
- A state change (`future` → `active` → `closed`) is reversed by another `partiallyUpdateSprint`,
  subject to Jira's own rules about reopening a closed sprint.
- `moveIssuesToSprintAndRank` is reversed by moving the issues to another sprint or to the backlog —
  there is no "undo move", only another move.

## Errors

- `400` — starting a sprint without both dates, or moving more than 50 issues in one call.
- `403` — the board is not visible, or the account lacks Manage Sprints permission.
- `404` — the board or sprint does not exist, or is not visible to this account.
- `429`/`503` — the Software API returns 44 documented `429` and 44 documented `503` responses across
  its 105 operations. Back off on both; a `503` here is usually transient tenant load.

See `lifecycle/atlassian-jira-lifecycle.yml` for the full deprecated-operation list.
