---
name: Retrieve clinical documents and provenance from Practice Fusion
description: Locate a patient's DocumentReference entries, fetch the underlying Binary content, and record who authored the data via Provenance.
api: conformance/practice-fusion-capability-statement.json
generated: '2026-08-14'
method: generated
source: conformance/practice-fusion-capability-statement.json
operations:
  - GET /DocumentReference [search-type]
  - GET /DocumentReference/{id} [read]
  - GET /Binary/{id} [read]
  - GET /Provenance [search-type]
  - GET /Practitioner/{id} [read]
  - GET /Organization/{id} [read]
---

# Retrieve clinical documents and their provenance

## 1. Search DocumentReference

```
GET {base}/DocumentReference?patient={id}&type=http://loinc.org|34133-9
GET {base}/DocumentReference?patient={id}&created=ge2025-01-01&status=current
```

Declared parameters: `_id`, `authenticator`, `author`, `class`, `created`,
`custodian`, `description`, `encounter`, `event`, `facility`, `format`,
`identifier`, `language`, `location`, `patient`, `period`, `relatesto`,
`relation`, `relationship`, `status`, `subject`, `type`.

## 2. Fetch the content

`DocumentReference.content[].attachment` gives you either inline `data`
(base64) or a `url`. When it is a URL it points at a `Binary`:

```
GET {base}/Binary/{id}
Accept: application/fhir+json
```

`Binary` supports `read` only — there is no search on it, so you must arrive via
a `DocumentReference`. Respect `attachment.contentType` (commonly
`application/pdf`, `text/plain` or `application/xml` for C-CDA); do not assume
JSON.

## 3. Establish provenance

```
GET {base}/Provenance?target=DocumentReference/{id}
GET {base}/Provenance?patient={patientId}&recorded=ge2025-01-01
```

Declared parameters: `_id`, `entity`, `patient`, `recorded`, `target`, `when`.
`Provenance.agent[].who` references a `Practitioner`, `PractitionerRole`,
`Organization` or `Device`; resolve it with a `read` to attribute the record.
US Core requires Provenance on the resources it profiles, so use it rather than
inferring authorship from a narrative.

## Scopes required

`patient/DocumentReference.rs patient/Binary.rs patient/Provenance.rs`
(or `user/` / `system/` for provider and backend contexts).

## Rules

- Documents are unstructured PHI. Summarize in place; do not copy content into
  logs, caches or prompts that outlive the request.
- `410 Gone` means the instance was deleted — do not retry it.
- `406 Not Acceptable` means you asked for a representation the server does not
  serve; Practice Fusion's CapabilityStatement declares `format: ["json"]`.
- Page `DocumentReference` searches via `Bundle.link[relation=next]`.
