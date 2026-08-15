---
name: Assemble a patient chart summary from Practice Fusion
description: Find a patient and pull their problems, medications, allergies, immunizations, vitals, labs and encounters as a single US Core summary.
api: conformance/practice-fusion-capability-statement.json
generated: '2026-08-14'
method: generated
source: conformance/practice-fusion-capability-statement.json
operations:
  - GET /Patient [search-type]
  - GET /Patient/{id} [read]
  - GET /Condition [search-type]
  - GET /MedicationRequest [search-type]
  - GET /AllergyIntolerance [search-type]
  - GET /Immunization [search-type]
  - GET /Observation [search-type]
  - GET /Encounter [search-type]
  - GET /DiagnosticReport [search-type]
---

# Assemble a patient chart summary

Every step below is a `search-type` interaction the Practice Fusion
CapabilityStatement declares, with the search parameters it actually names.
Do not invent parameters — an unsupported one is a `400 invalid`.

## 1. Find the patient

```
GET {base}/Patient?family=Chalmers&given=Peter&birthdate=1974-12-25
GET {base}/Patient?identifier=http://hospital.example.org/mrn|12345
```

Supported: `_id`, `active`, `address`, `address-city`, `address-country`,
`address-state`, `birthdate`, `death-date`, `deceased`, `family`, `gender`,
`given`, `identifier`, `language`, `link`, `name`, `phonetic`, `telecom`.

Search returns a `searchset` `Bundle`. If more than one patient matches and the
caller expected exactly one, stop and ask — do not guess. Take `Patient.id`
from `Bundle.entry[].resource.id`.

## 2. Pull each chart section, scoped to that patient

```
GET {base}/Condition?patient={id}&clinical-status=active
GET {base}/MedicationRequest?patient={id}&status=active
GET {base}/AllergyIntolerance?patient={id}&clinical-status=active
GET {base}/Immunization?patient={id}&status=completed
GET {base}/Observation?patient={id}&category=vital-signs
GET {base}/Observation?patient={id}&category=laboratory&date=ge2025-01-01
GET {base}/Encounter?patient={id}&date=ge2025-01-01
GET {base}/DiagnosticReport?patient={id}&category=LAB
```

Verified parameters per resource:

| Resource | Parameters declared |
|---|---|
| `Condition` | `_id`, `abatement-date`, `asserter`, `category`, `clinical-status`, `code`, `encounter`, `evidence`, `onset-date`, `patient`, `severity`, `stage`, `subject` |
| `MedicationRequest` | `_id`, `authoredon`, `code`, `date`, `encounter`, `identifier`, `intent`, `medication`, `patient`, `status`, `subject` |
| `AllergyIntolerance` | `_id`, `clinical-status`, `code`, `criticality`, `date`, `manifestation`, `onset`, `patient`, `recorder`, `severity`, `type` |
| `Immunization` | `_id`, `date`, `identifier`, `location`, `lot-number`, `manufacturer`, `patient`, `performer`, `reaction`, `reaction-date`, `status`, `vaccine-code` |
| `Observation` | `_id`, `category`, `code`, `date`, `encounter`, `patient`, `performer`, `specimen`, `status`, `subject`, `value-concept`, `value-date`, `value-quantity`, `value-string` |
| `Encounter` | `_id`, `appointment`, `class`, `date`, `diagnosis`, `episodeofcare`, `identifier`, `length`, `location`, `participant`, `participant-type`, `patient`, `practitioner`, `status`, `subject`, `type` |
| `DiagnosticReport` | `_id`, `category`, `code`, `date`, `encounter`, `identifier`, `issued`, `patient`, `performer`, `result`, `specimen`, `status`, `subject` |

## 3. Page every result set

Search returns a `Bundle` with `total` and `link[]`. Follow
`link[relation=next].url` until it is absent; use `_count` to size the page.
Never assume the first page is the whole chart — for `Observation` on an
established patient it will not be.

## 4. Resolve references, do not re-search blindly

`DiagnosticReport.result[]` points at `Observation`; `MedicationRequest.medicationReference`
points at `Medication`; `Encounter.location` points at `Location`. Use `_include`
where it saves a round trip rather than issuing one read per reference.

## Scopes required

`patient/Patient.rs patient/Condition.rs patient/MedicationRequest.rs
patient/AllergyIntolerance.rs patient/Immunization.rs patient/Observation.rs
patient/Encounter.rs patient/DiagnosticReport.rs` — or the `user/` or `system/`
prefix for provider and backend contexts respectively.

## Rules

- Read-only. There is no write interaction on any of these resources.
- On **429**, stop and back off; the limits are not published.
- On **403**, the token is missing a scope for that resource — request it at
  authorization time, not by retrying.
- This is PHI. Do not log resource bodies, and do not persist anything the
  caller did not ask you to persist.
