---
name: mitratech-tap-triage-assigned-work
description: Read a TAP user's assigned and action-required workflows, inspect a task, then continue or re-assign it.
api: Mitratech TAP Workflow Automation API
generated: '2026-09-13'
method: generated
source: openapi/_original/mitratech-tap-swagger.json
operations:
  - Users_GetCurrent
  - Workflows_GetMy
  - Workflows_GetActionRequired
  - Workflows_GetTaskValues
  - Workflows_GetWorkflowForm
  - Workflows_ContinueWorkflow
  - Workflows_Assign
  - Users_Get
---

# Triage assigned TAP work

## 1. Establish identity first

`Users_GetCurrent` — `GET /v1/users/current`. Every "my work" endpoint is scoped to the token's user,
and the TAP permission model degrades to silent partial results rather than a 403. Know who you are
before you report what you found.

## 2. List the queues

Two collections, same parameter shape, different meaning:

- `Workflows_GetMy` — `GET /v1/workflows/my` — workflows assigned to the token's user.
- `Workflows_GetActionRequired` — `GET /v1/workflows/actionrequired` — workflows waiting on an action.

Both **require** `page` and `pageSize`. Optional: `configurationId`, `search`, `orderBy`,
`orderDirection`, `advancedFilter`, `dynamicColumns`.

These are *not* OData endpoints — do not send `$top`/`$filter` here. TAP mixes two pagination idioms:
page/pageSize on workflow collections, `$top`/`$skip` on `/v1/users` and `/v1/templates/dashboard`.
See `conventions/mitratech-conventions.yml`.

`search` is ignored below three characters (a documented UI-side rule), and both collections return
`404` when empty — an empty result set is not distinguished from a missing resource.

## 3. Inspect one task

`Workflows_GetTaskValues` — `GET /v1/workflows/{taskId}/view?include=<...>`.
For the form definition behind a pending token, `Workflows_GetWorkflowForm` —
`GET /v1/workflows/form?tokenId=<tokenId>&additionalInfo=true`.

## 4. Act

**Continue the workflow:** `Workflows_ContinueWorkflow` — `PUT /v1/workflows/{tokenId}/form`.
Same three body encodings as initiate; returns `201`.

**Re-assign it:** `Workflows_Assign` — `PUT /v1/workflows/{tokenId}/assign/{userId}` with a
**required** `message` body. Resolve `userId` with `Users_Get` (`GET /v1/users?$filter=...`) — the
user collection is the OData-flavoured one, so `$filter` works there.

## Safety

`PUT` here is not the safe idempotent `PUT` you may expect. `Workflows_ContinueWorkflow` returns
`201 Created` and advances workflow state; there is no idempotency key and no dry-run parameter.
Read state, decide, act once, then verify by re-reading the queue.
