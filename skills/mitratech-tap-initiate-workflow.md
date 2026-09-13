---
name: mitratech-tap-initiate-workflow
description: Authenticate against a Mitratech TAP tenant and initiate a workflow from a template, then confirm it started.
api: Mitratech TAP Workflow Automation API
generated: '2026-09-13'
method: generated
source: openapi/_original/mitratech-tap-swagger.json + https://success.mitratech.com/TAP/TAP_Solutions/APIs_and_Integrations/TAP_API_Documentation
operations:
  - Users_GetCurrent
  - Templates_Get
  - Workflows_GetWorkflowForm
  - Workflows_InitiateWorkflow
  - Workflows_GetMy
---

# Initiate a TAP workflow

TAP is multi-tenant in two places at once: the tenant name is both the **subdomain** and the **first
path segment**. Production is `https://{tenant}.tap.thinksmart.com/{tenant}/api`; staging is
`https://{tenant}.stagingtap.thinksmart.com/{tenant}/api`. Get both wrong and you get a bare 404 with
no body.

## 1. Get a token

`POST /{tenant}/auth/identity/connect/token` — note this is **outside** `/api`.

```
Content-Type: application/x-www-form-urlencoded

grant_type=password&scope=api&redirect_uri=tapredirect&username=<user>&password=<password>&client_id=<id>&client_secret=<secret>
```

Response: `{"access_token": "...", "expires_in": 3600, "token_type": "Bearer"}`.
Send it as `Authorization: Bearer <access_token>` on every subsequent call.

**Permission trap.** Mitratech's own documentation warns that the token inherits the TAP user's
permissions and that an under-privileged account returns *fewer rows*, not a 403. Confirm who you are
with `Users_GetCurrent` (`GET /v1/users/current`) before trusting any list you read.

## 2. Find the template

`Templates_Get` — `GET /v1/templates/dashboard`. This is one of the endpoints that accepts the OData
query vocabulary: `$top`, `$skip`, `$filter`, `$select`, `$orderby`, `$inlinecount`. Use `$filter` to
find the template by name rather than paging the whole list.

Optionally read the form definition first with `Workflows_GetWorkflowForm`
(`GET /v1/workflows/form?templateId=<id>&additionalInfo=true`) so you know which field names the
template expects.

## 3. Initiate

`Workflows_InitiateWorkflow` — `POST /v1/workflows/{templateId}/form?title=<title>`.

Three body encodings are accepted, and the contract names them explicitly:

- `application/json` as `{ "fieldName": "fieldValue" }`
- `application/x-www-form-urlencoded` as `fieldName=fieldValue`
- `multipart/form-data` as `fieldName:fieldValue` — **the only way to submit files**

Success is `201 Created` with a `WorkflowSubmitResult`.

## 4. STOP AND READ BEFORE YOU RETRY

**There is no idempotency key on this API.** No `Idempotency-Key` header, no client-supplied request
id, no ETag/If-Match. A retry after a timeout starts a *second* workflow instance — a second legal
intake, contract request or compliance review — and nothing in the API will collapse the duplicate.

If a call times out, do **not** re-POST. Instead:

1. Call `Workflows_GetMy` (`GET /v1/workflows/my?page=1&pageSize=25&orderBy=__DateCreated__&orderDirection=desc`).
   `page` and `pageSize` are **required**, not optional.
2. Look for your `title` in the returned `__WorkflowName__` / `__WorkflowDescription__` main fields —
   TAP main fields are wrapped in double underscores.
3. Only initiate again if it genuinely is not there.

## Error handling

The contract declares almost nothing. `Workflows_InitiateWorkflow` declares only `201`. There is no
`application/problem+json`, no error schema, and 48 of the 49 operations declare no `401` at all even
though every one of them requires a token. Treat any non-2xx as opaque and log the raw body. See
`errors/mitratech-problem-types.yml`.

## Rate limits

None published. No `429` appears anywhere in the contract and no `RateLimit-*` or `Retry-After`
header is documented. Choose your own conservative pacing; there is no runtime signal to back off on.
