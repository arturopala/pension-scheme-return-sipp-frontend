# Pension Scheme Return SIPP Frontend – Business Identifiers

This document lists every business and technical identifier the service handles: where it comes from, how it is used, and how it is validated. Validation rules come from three sources:

- **Local**: checks written in this repository (route binders, regexes in `Constants`, `TextFormProvider` and model constructors, normalisers).
- **Library**: checks inside dependencies, e.g. `uk.gov.hmrc.domain.Nino` (domain-play-30 13.0.0, inspected with `javap`). Its `isValid` regex is `[[A-Z]&&[^DFIQUV]][[A-Z]&&[^DFIQUVO]] ?\d{2} ?\d{2} ?\d{2} ?[A-D]{1}`, and it excludes the prefixes `BG, GB, NK, KN, TN, NT, ZZ`. The `Nino` constructor `require`s `isValid`.
- **Contract**: patterns from bundled JSON or OpenAPI schemas. **This repository bundles no schemas**, so contract rules are taken only from constants that evidently mirror ETMP formats (e.g. `Constants.inputRegexPSTR`) and are marked as such.

**Terminology:** "form bundle number", `fbNumber` and `formBundleNumber` all mean the ETMP return instance identifier. "Version" is `psrVersion`, and `periodStartDate` is the tax-year start (`taxYear` in the session). `userId` in code is the auth **internalId**. "PSA ID" and "PSAID" are the same thing, and so are "PSP ID" and "PSPID".

## Summary table

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| SRN | Pension scheme (registration number) | `S` + 10 digits | `^S[0-9]{10}$` path binder | Path (untrusted); session `SRN` (read only) | Every route; PSR `srn` header; pensions-scheme lookups; journey-store key; upload key; MPS/PSR links |
| PSTR | Pension scheme (tax reference) | `^[0-9]{8}[A-Z]{2}$` (`Constants.inputRegexPSTR`, not applied) | None (trusted upstream value) | pensions-scheme response | PSR backend URL path segment and submit body; audit |
| Form bundle number | One ETMP return instance | Not defined locally | **None** | Session `fbNumber` (set by this service from PSR responses); **query** `fbNumber` (untrusted) | PSR query param; submit body; session; download links |
| PSR version | Version of a submitted return | Not defined locally (default `"000"` constant, unused) | None | Session `version`; query `psrVersion` | PSR query param; `ReportDetails.version` |
| Tax year / period start | Return period | ISO date | `LocalDate.parse` (throws) | Session `taxYear`; query `periodStartDate`; form (which tax year) | PSR query param; date ranges; audit |
| PSA ID | Administrator | `A` + 7 digits | `^(A[0-9]{7})$` only in the PSP declaration form | Auth enrolment `HMRC-PODS-ORG`/`PSAID` (trusted); PSP declaration form (untrusted) | Pensions-scheme/pension-administrator headers; submit body `psaId`; audit |
| PSP ID | Practitioner | Not defined locally | None | Auth enrolment `HMRC-PODSPP-ORG`/`PSPID` (trusted) | Pensions-scheme/pension-administrator headers; `requestRole=PSP` |
| Authorising PSA ID | PSA that authorised the PSP | as PSA ID | Equality with the declaration input | pensions-scheme `pspDetails.authorisingPSAID` | PSP declaration check |
| internalId | Auth user (internal) | Opaque | Presence only | Auth retrieval (trusted) | Journey-store key; upload key; object-store path |
| externalId | Auth user (external) | Opaque | Presence only | Auth retrieval | Passed (unused) to the session-data-cache fetch log |
| Member NINO | Scheme member | Library `Nino` regex | `Nino.isValid` after upper-casing/whitespace removal (CSV, change-NINO form); **none** on query param | CSV cell; form; **query `nino`**; PSR member list | Transaction payloads; member update/delete payloads; member search |
| Member name + DOB | Scheme member (composite key) | – | Name regex/length, DOB date (CSV, forms); **none** on query (DOB parse only) | CSV; forms; **query** | Member update/delete bodies (identify the member upstream) |
| Company CRN | Company whose shares are held | 8 alphanumeric | `^[A-Za-z0-9 ]*$` + length 8 after removing spaces | CSV cell | Transaction payloads |
| UTR | Partnership/company (counterparty) | 10 digits | `^\d{5}\s*?\d{5}$` (defined; **not used in CSV flows**) | – | – |
| Upscan reference | One upload | Opaque | None | Upscan initiate response; callback body (untrusted) | `upload.reference` unique index; callback correlation |
| Upscan download URL | Uploaded file location | URL | `java.net.URL` parse only | Callback body (untrusted) | Fetched server-side for validation; sent as `fileReference` in audit |
| UploadKey | Upload slot per user/scheme/asset type | `{internalId}&&{srn}&&{tag}` | SRN regex when parsed | Derived | Mongo `upload.id`; object-store object name |
| User-answers id | Journey state | `{internalId}{srn}` | None | Derived | Mongo `user-answers._id` |
| CorrelationId | Outbound PSR call | UUID | – | Generated per call | `CorrelationId` header |

## Scheme identifiers

### SRN
- **Issuer:** the Pensions Scheme Registration service. It reaches this service in the URL that Manage Pension Schemes and the main PSR frontend link to.
- **Sources:** route path `/:srn/...` (untrusted). `Srn.fromSession` reads the session key `SRN` only for sign-out and keep-alive clean-up. This service never writes that key.
- **Validation:** `Srn.apply` regex `^S[0-9]{10}$`. Failure → `400`.
- **Authorisation:** `AllowAccessAction` calls `GET /pensions-scheme/scheme/{srn}` (PSA, headers `idNumber`, `schemeIdType=srn`, `psaId`) or `GET /pensions-scheme/psp-scheme/{srn}` (PSP, headers `pspId`, `srn`). The upstream decides whether the user is associated with the scheme. A `404` → `None` → `/unauthorised`.
- **Uses:** PSR `srn` header, journey-store key, `UploadKey`, MPS dashboard URL templates (`%s` formatted), PSR overview URL, `SchemeDetailsService` filters the scheme list by SRN.
- **Mismatch:** the SRN that was authorised (path) and the SRN used for sign-out clean-up (session `SRN`) come from different sources. If the session key is missing, `Srn.fromSession` returns `""`, so `clear(internalId + "")` silently does nothing.

### PSTR
- **Issuer:** HMRC. Obtained **only** from the pensions-scheme `SchemeDetails.pstr` for the authorised SRN (trusted, never taken from user input).
- **Uses:** path segment in every PSR GET/PUT (`/psr/.../{pstr}`), the `PsrSubmissionRequest.pstr` body field, `ReportDetails.pstr`, and the `pensionSchemeTaxReference` audit field.
- **Validation:** none locally. `Constants.inputRegexPSTR` (`^([0-9]{8}[A-Z]{2})$`) exists but is unused. 

## Return identifiers (form bundle number, version, tax year)

- **Issuer:** ETMP, through the PSR backend. `fbNumber` is returned by `createEmptyPsr` and by every PUT (`SippPsrJourneySubmissionEtmpResponse.formBundleNumber`).
- **Lifecycle:** after each successful write, this service **writes** `fbNumber` to the Play session (`WhatYouWillNeedController`, `AssetsHeldController`, `AccountingPeriodListController`, `JourneyContributionsHeldController`, `RemoveFileController`, `RemoveMemberController`, `ViewChangePersonalDetailsController`, `LoadingPageController` via `PendingFileActionService`). It **never writes** `taxYear` or `version`. Those are expected in the shared session cookie, presumably set by the main PSR frontend.
- **Sources:**
  - Session (signed cookie, set by the platform): `fbNumber`, `taxYear`, `version`. Used by `createQueryParamsFromSession` for all PUTs, and by the `*RequiredAction` refiners.
  - **Query string:** `fbNumber` on `/declaration`, `/view-task-list` and `/view-change-question` (POST); `fbNumber` / `periodStartDate` / `psrVersion` on the six `/download-csv-*` routes; `taxYear: Int` on `/view-change-question`.
- **Uses:** PSR query parameters (`fbNumber=` or `periodStartDate=&psrVersion=`), the submit body, `ReportDetails.version`, CSV download links on the declaration page.
- **Validation:** none locally. `TaxYear.optFromSession`, `VersionTaxYear.optFromSession` and `TaxYearService.fromRequest` call `LocalDate.parse` and throw on bad input → `303` `/service-unavailable`.

## User and enrolment identifiers

### PSA ID / PSP ID
- **Trusted source:** `IdentifierActionImpl` retrieves `allEnrolments`: `HMRC-PODS-ORG` → `PSAID`, `HMRC-PODSPP-ORG` → `PSPID`. A user with both enrolments has the role chosen from pension-administrator session data.
- **Uses:**
  - pension-administrator `get-minimal-details-self` headers `psaId`/`pspId` and `loggedInAsPsa`
  - pensions-scheme headers (`psaId`, `pspId`, `idValue` + `idType`)
  - PSR `requestRole` header (role only; the ID itself is not sent as a header)
  - `PsrSubmissionRequest.psaId`
  - audit events (`AuthorizedAuditEvent`)
  - choice of MPS dashboard URL
- **PSP declaration:** the PSP types a PSA ID into a form (untrusted). It is checked against `^(A[0-9]{7})$`, max 8, and **must equal** `SchemeDetails.authorisingPSAID` from upstream. The submitted `psaId` is the typed value, which has been proven equal to the authorising PSA. For a PSA, `psaId` = the enrolment PSA ID.
- **Validation:** no local format check on enrolment values (they are trusted). Local `models.PensionSchemeId.PsaId`/`PspId` wrappers have no `require`.

### internalId / externalId
- From auth retrievals (trusted). A missing value → `/unauthorised`.
- `internalId` is used as `userId`, the prefix of the journey-store `_id` (`internalId + srn`) and of `UploadKey` (`internalId&&srn&&tag`), and it appears in object-store object names.
- `externalId` is passed to `SessionDataCacheConnector.fetch`, which only puts it in a log message. The upstream works out the user from the bearer token.

## Party identifiers (members and counterparties)

| Identifier | Frontend key(s) | Upstream key (per flow) | Local rule | Contract rule |
|---|---|---|---|---|
| Member NINO | CSV column "member NINO"; form `value` (change NINO); query `nino` | `nino` inside `nameDOB`/`NinoType` (transactions); `nino` in `MemberDetails` (update/delete); `current.nino`, `updated.nino` | CSV/form: upper-case, remove whitespace (CSV only), `Nino.isValid`. Query: **none** | Library regex above (none bundled) |
| Reason for no NINO | CSV column; form; query `reasonNoNINO` | `reasonNoNino` / `reasonNoNINO` | CSV: free text, no CR/LF, ≤160; form: text-area regex, ≤160; query: **none** | – |
| Member first/last name | CSV; forms; query `firstName`/`lastName` | `firstName`, `lastName` | `^[a-zA-Z\-' ]+$`, ≤35 (CSV/forms); query: **none** | – |
| Member DOB | CSV `dd/MM/yyyy` or with `-`; forms; query `dateOfBirth` | `dob` / `dateOfBirth` | Valid date, not after the wind-up date or today; query: `LocalDate.parse` (throws) | – |
| Company CRN (shares) | CSV CRN column, or "reason no CRN" | `companySharesCRN` / `reasonNoCRN` | `^[A-Za-z0-9 ]*$` + 8 characters after removing spaces, upper-cased | – |
| UTR | – | – | `^\d{5}\s*?\d{5}$` (unused in flows) | – |

**NINO normalisation (CSV):** `n.toUpperCase.filterNot(_.isWhitespace)` → `Mappings.nino` (`Nino.isValid(s.toUpperCase)`) → `Nino(s.toUpperCase)`. Forms skip the whitespace removal, so the library regex's optional spaces may stay in a NINO entered through a form.

**Member identity in amend flows:** the members list page links to `/update-member` and `/remove-member` with `firstName`, `lastName`, `dateOfBirth`, `nino` and `reasonNoNINO` in the **GET query string**. The controller builds `MemberDetails` from these values, stores them in the journey store, and later sends them upstream as the key that identifies the member (`deleteMember` body; `UpdateMemberDetailsRequest.current`).

- ⚠️ **Observation:** `ChangeMembersNinoController.onPageLoad` builds `Nino(...)` from the stored `updated.nino`. If that value came from a malformed query string, the library `require` throws → `303` `/service-unavailable` instead of a form error.

## Technical identifiers

| Identifier | Format | Lifecycle | Validation | Notes |
|---|---|---|---|---|
| Upscan `reference` | Opaque string | Created by upscan initiate; stored as unique `upload.reference`; returned in the callback | None | |
| Upscan `downloadUrl` | URL | Callback → encrypted `UploadStatus.Success.downloadUrl` → streamed by `UpscanDownloadStreamConnector` | `new URL(s)` | Also sent as the audit `fileReference` |
| `UploadKey` | `{internalId}&&{srn}&&{uploadRedirectTag}` | Derived per request | On read: split on `&&` + SRN regex | Object-store path `uploads/{UploadKey}`; Mongo `upload.id` unique |
| User-answers `_id` | `{internalId}{srn}` (no separator) | Derived | None | TTL 1h, refreshed on read |
| `CorrelationId` | UUID v4 | New per PSR call (not on all GETs) | – | Not tied to the platform `X-Request-ID` |
| Accounting-period index | 1–3 | Path | `refineUnsafe` (throws outside the Future → `/service-unavailable`) | |

## Validation coverage matrix

| Identifier | Format checked locally? |
|---|---|
| SRN | ✅ regex |
| PSTR | ❌ |
| Form bundle number (session) | ❌ |
| Form bundle number (query) | ❌ |
| PSR version / period start (query) | ❌ |
| Tax year (session) | ⚠️ parse only (throws) |
| PSA ID (enrolment) | ❌ (trusted) |
| PSA ID (PSP declaration) | ✅ regex + length |
| PSP ID | ❌ (trusted) |
| Member NINO (CSV / form) | ✅ library regex |
| Member NINO / name / DOB (query) | ❌ (DOB parse only) |
| CRN | ✅ |
| Upscan reference (callback) | ❌ |
| Upscan download URL | ⚠️ URL syntax only |

## Recommendations

1. Validate member NINOs taken from upstream or query data with `Nino.isValid` before calling the `Nino(...)` constructor, to avoid `require` failures.
2. Apply `Constants.inputRegexPSTR` (or remove it) and decide whether UTR validation is needed in the CSV flows (`validateUtr` is currently dead code).
3. Write `SRN` into the session (or derive it from the route) so that sign-out clean-up of the journey store works reliably.
