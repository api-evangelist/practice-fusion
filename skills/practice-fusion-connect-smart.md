---
name: Connect to a Practice Fusion FHIR endpoint with SMART on FHIR
description: Resolve a practice's FHIR service base URL, read its SMART configuration, and obtain an access token for provider/system or patient access.
api: conformance/practice-fusion-capability-statement.json
generated: '2026-08-14'
method: generated
source: >-
  https://api.practicefusion.com/fhir/r4/v1/{organizationId}/.well-known/smart-configuration ,
  well-known/practice-fusion-well-known.yml , scopes/practice-fusion-scopes.yml
operations:
  - GET /.well-known/smart-configuration
  - GET /metadata [CapabilityStatement read]
  - POST /authorize [SMART authorization_code]
  - POST /token [SMART token exchange]
  - POST /introspect
---

# Connect to a Practice Fusion FHIR endpoint

Practice Fusion has **no single API base URL**. Every practice organization gets its
own FHIR service base URL, and you must resolve the right one before anything else.

## 1. Resolve the service base URL

The full directory of live endpoints is published as a FHIR `Bundle` of
`Organization` + `Endpoint` resources:

```
GET https://www.practicefusion.com/assets/static_files/ServiceBaseURLs.json
```

Two surfaces exist per organization:

| Surface | Base URL | Who it is for |
|---|---|---|
| Provider / System | `https://api.practicefusion.com/fhir/r4/v1/{organizationId}` | apps acting for a clinician or a backend service |
| Patient (FMH) | `https://api.practicefusion.com/fhir/fmh/r4/v1/{organizationId}` | apps acting for a patient |

## 2. Read the SMART configuration — never hard-code endpoints

```
GET {base}/.well-known/smart-configuration
```

Take `authorization_endpoint`, `token_endpoint`, `introspection_endpoint` and
`jwks_uri` from that document. The provider surface issues them under
`api.practicefusion.com`; the patient surface delegates them to
`muauthentication.followmyhealth.com` (Veradigm FollowMyHealth). Hard-coding
either one will break the other.

Advertised capabilities include `launch-ehr`, `launch-standalone`,
`client-public`, `client-confidential-symmetric`, `client-confidential-asymmetric`,
`sso-openid-connect`, `permission-v1`, `permission-v2`, `permission-patient`,
`permission-user`, `permission-offline` and `authorize-post`.

## 3. Pick the grant

`grant_types_supported` is `authorization_code`, `refresh_token`,
`client_credentials`; `code_challenge_methods_supported` is `S256` only.

- **User-facing app** — authorization code + PKCE (`S256`). Request
  `launch` (EHR launch) or `launch/patient` (standalone), plus `openid fhirUser`
  and `offline_access` if you need a refresh token.
- **Backend service** — `client_credentials` with an asymmetric (private key JWT)
  client assertion, signed against the key you published in your JWKS.

## 4. Ask for the smallest scope that works

Scopes follow SMART v2 grammar `<context>/<Resource>.<cruds>`. The API is
read/search only for every resource except `Group`, so `.rs` is the operative
access — for example `patient/Observation.rs`, `user/Condition.rs`,
`system/Patient.rs`. See `scopes/practice-fusion-scopes.yml`.

## 5. Confirm what the endpoint actually supports

```
GET {base}/metadata
Accept: application/fhir+json
```

The CapabilityStatement is authoritative per deployment: 47 resource types,
`read` on all of them, `search-type` on 45, and `create`/`update` plus
`$export` on `Group` only. Do not plan a write that is not in it.

## Rules

- All requests and responses are `application/fhir+json`.
- Errors come back as a FHIR `OperationOutcome`, not `application/problem+json`
  — read `issue[].code` and `issue[].diagnostics`. See `errors/practice-fusion-problem-types.yml`.
- Rate limiting is enforced but the numbers are not published. Treat **429** as
  the only signal and back off; no `X-RateLimit-*` or `Retry-After` header is
  documented. See `rate-limits/practice-fusion-rate-limits.yml`.
- Registration is manual: apply at https://pfpds.practicefusion.com/s/Registration
  and accept https://www.practicefusion.com/pds-api/termsofservice/ before you
  get credentials. There is no self-serve key and no sandbox.
