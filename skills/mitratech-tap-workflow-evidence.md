---
name: mitratech-tap-workflow-evidence
description: Assemble the complete audit record for a set of TAP workflows — audit trail, comments, collaboration discussion, attachments and signed documents.
api: Mitratech TAP Workflow Automation API
generated: '2026-09-13'
method: generated
source: openapi/_original/mitratech-tap-swagger.json
operations:
  - Workflows_GetAuditTrail
  - Workflows_GetAuditTrailDetailsWithFilter
  - Workflow_SearchAuditTrailDetails
  - Workflows_GetComments
  - Workflows_GetCollabCommentsById
  - Workflows_GetFiles
  - Files_Get
  - Workflows_GetDocuments
---

# Assemble TAP workflow evidence

This is the read-only flow, and it is the one worth automating: TAP is where legal, risk and
compliance intake lives, so "show me everything that happened on these requests" is the recurring
question.

Every endpoint below takes the same identifier: `resultSetId`, an opaque GUID. TAP identifiers carry
no type prefix, so a workflow id, a file id and a user id are indistinguishable by inspection — keep
them labelled.

## Audit trail

Three addressing modes, all `GET`, all returning the same shape:

| By | Operation | Path |
|---|---|---|
| id | `Workflows_GetAuditTrail` | `/v1/workflows/getaudittrail?resultSetIds=<id>&resultSetIds=<id>` |
| filter | `Workflows_GetAuditTrail` | `/v1/workflows/getaudittrail/filter` |
| name | `Workflows_GetAuditTrail` | `/v1/workflows/getaudittrail/name` |

`resultSetIds` is an **array** query parameter — repeat it, do not comma-join.

For detail rather than summary, prefer the **v2** operation:
`Workflow_SearchAuditTrailDetails` — `POST /v2/workflows/audittrail/details` with a JSON array of
workflow names in the body, e.g. `["Request # 00020998","CNDA0207"]`.

This is the only operation in the entire 49-operation contract with a real error vocabulary:
`400` invalid input, `401` authentication required, `403` not authorized for this tenant,
`500` unexpected. Everywhere else you get an opaque status. Use it when you need to distinguish
"not found" from "not allowed".

## Comments and discussion

TAP separates two comment streams and you need both:

- `Workflows_GetComments` — `GET /v1/workflows/comments?resultSetIds=<id>` (also `/comments/all` by
  filter id and `/comments/name` by workflow name).
- `Workflows_GetCollabCommentsById` — `GET /v1/workflows/discussion?resultSetIds=<id>` (also
  `/discussion/all` and `/discussion/name`).

## Attachments and signed documents

1. `Workflows_GetFiles` — `GET /v1/workflows/{resultSetId}/files` returns the file ids.
2. `Files_Get` — `GET /v1/files/{id}` returns the file body as an octet stream. There is no
   content-type or filename in the contract; read the response headers.
3. `Workflows_GetDocuments` — `GET /v1/workflows/{resultSetId}/docs` returns e-signature documents.

## Do not call these while gathering evidence

`DELETE /v1/workflows/{resultSetId}/docs` (`Workflows_CancelDocuments`) sits on the **same path** as
the signed-documents read, differing only by method. A wrong verb cancels sent signature requests.
There is no undo.

Likewise `POST /v1/workflows/delete/filter` permanently destroys every record matching a filter —
the contract's own words are "Deleted records are removed permanently." Archive
(`POST /v1/workflows/archive`) is the reversible one; it is undone by
`POST /v1/workflows/restore`, though no retention window is documented anywhere.
See `conventions/mitratech-conventions.yml` for the full reversibility map.
