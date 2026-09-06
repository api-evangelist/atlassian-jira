---
name: atlassian-jira-transition-a-work-item
description: Move a Jira work item through its workflow safely — discover the legal transitions first, supply required screen fields, and know that the move is reversible only if the workflow allows the inverse.
api: Atlassian Jira Cloud platform REST API v3
generated: '2026-09-06'
method: generated
source: Grounded in openapi/atlassian-jira-platform-openapi.json (harvested first-party from developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json on 2026-09-06). Every operationId below was verified present in that document.
operations:
  - getIssue
  - getTransitions
  - doTransition
  - getChangeLogs
  - getStatuses
  - editIssue
---

# Transition a Jira work item

Never guess a transition id. They are per-workflow, per-project and per-issue-type, and the same
"Done" can be id 31 on one project and 41 on the next.

## Steps

1. **Read the current state.** `getIssue` — `GET /rest/api/3/issue/{issueIdOrKey}?fields=status,summary`.

2. **Ask what is legal right now.** `getTransitions` —
   `GET /rest/api/3/issue/{issueIdOrKey}/transitions?expand=transitions.fields`.
   The response lists only transitions this user can perform on this issue in its current status, with
   the id, the destination status, and — because of the `expand` — the fields the transition screen
   requires. This is a genuine pre-flight: it tells you whether the move will be rejected before you
   attempt it.

3. **Perform the transition.** `doTransition` — `POST /rest/api/3/issue/{issueIdOrKey}/transitions`
   with `{"transition": {"id": "<id from step 2>"}}`.
   - If step 2 showed required screen fields, include them in `fields` in the same body.
   - A comment can ride along in `update.comment[0].add.body` and must be **Atlassian Document Format**,
     not a string.
   - A successful transition returns `204` with no body.

4. **Confirm.** Re-read with `getIssue`, or read the audit trail with `getChangeLogs` —
   `GET /rest/api/3/issue/{issueIdOrKey}/changelog`, which shows the from/to status and who did it.

`getStatuses` — `GET /rest/api/3/status` lists every status on the site if you need to map names to ids
across projects.

## Reversibility

A transition is reversed by performing the **inverse transition**, and only if the workflow defines
one. Call `getTransitions` again after the move: what comes back is exactly the set of reversals
available. There is no undo endpoint and no time window — a workflow that is one-way (a "Close" with no
"Reopen") is genuinely one-way, and no API call will take it back.

Field edits made during the transition are separately reversible with `editIssue`
(`PUT /rest/api/3/issue/{issueIdOrKey}`), which is not automatic — reverting the status does not revert
the fields.

## Errors

- `400` — the transition id is not valid **from the current status**, or a required screen field is
  missing. Re-run step 2; the state may have changed under you.
- `403` — the user lacks the Transition Issues permission, or a workflow condition or validator blocked
  the move. The message in `errorMessages[]` is the validator's.
- `404` — the issue does not exist or is not visible to this account.
- `429` with `RateLimit-Reason: jira-per-issue-on-write` — you are transitioning one issue too fast
  (limit: 20 writes / 2s, 100 writes / 30s per issue). Back off per issue.

There is no idempotency key: a retried `doTransition` after a timeout may perform the transition twice,
which for a workflow with a loop means two changelog entries. Re-read the status before retrying.

See `conventions/atlassian-jira-conventions.yml` and `errors/atlassian-jira-problem-types.yml`.
