---
name: Run a Practice Fusion FHIR Bulk Data group export
description: Create or select a Group, kick off an asynchronous $export, poll to completion, and download the NDJSON output.
api: conformance/practice-fusion-capability-statement.json
generated: '2026-08-14'
method: generated
source: conformance/practice-fusion-capability-statement.json
operations:
  - GET /Group [search-type]
  - POST /Group [create]
  - PUT /Group/{id} [update]
  - GET /Group/{id}/$export [operation]
  - GET /Binary/{id} [read]
---

# Run a Bulk Data group export

`Group` is the **only** resource in the Practice Fusion CapabilityStatement with
write interactions (`create`, `update`) and the only one carrying an operation
(`$export`). Everything else is read/search. Practice Fusion implements FHIR
Bulk Data Access IG v1.0.1.

## 1. Find or create the Group

```
GET  {base}/Group?type=person
POST {base}/Group
```

`Group` declares only `_id` and `type` as search parameters. Creation is a
conditional-create candidate: send `If-None-Exist` with your search criteria so a
retried request does not produce a duplicate cohort.

## 2. Kick off the export

```
GET {base}/Group/{id}/$export
Accept: application/fhir+json
Prefer: respond-async
```

Optional IG parameters: `_type` (comma-separated resource types), `_since`
(an instant), `_outputFormat` (`application/fhir+ndjson`).

A successful kick-off returns **202 Accepted** with a `Content-Location` header
pointing at a status URL. It does not return data.

## 3. Poll the status URL

`GET` the `Content-Location` value. While the job runs it answers **202** with an
`X-Progress` hint; on completion it answers **200** with a manifest containing
`transactionTime`, `request`, `requiresAccessToken` and `output[]` — one entry
per `{type, url}`. Honour any `Retry-After` on the 202; do not tight-loop.

## 4. Download the NDJSON

Each `output[].url` is newline-delimited JSON, one FHIR resource per line.
When `requiresAccessToken` is true, send the same bearer token. Stream and parse
line by line — a full export of a practice will not fit in memory.

## 5. Clean up

Issue `DELETE` on the status URL when you are done, so the server can release
the generated files.

## Scopes required

`system/Group.c system/Group.u system/Group.rs` plus `system/*.rs` (or the
explicit per-resource `system/<Resource>.rs` set matching your `_type`). Bulk
export is a backend-services flow: `client_credentials` with a private key JWT
assertion, not an interactive login.

## Rules

- Async only. Anything that returns immediately with data is not the export.
- The output is bulk PHI. Verify the destination before you start, and never
  write it anywhere the caller did not name.
- **429** applies here too and the limits are unpublished — a poll loop is the
  most likely way to hit it.
- Errors are `OperationOutcome`; a failed job reports its problems in the
  manifest's `error[]` array, also as `OperationOutcome` NDJSON.
