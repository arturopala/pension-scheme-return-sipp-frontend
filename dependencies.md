# Pension Scheme Return SIPP Frontend – Dependencies

## Shared mechanism

Outbound HTTP calls go through the platform `HttpClientV2` (bootstrap-frontend-play-30). Most use `HeaderCarrier`, built from the incoming request and session (`HeaderCarrierConverter.fromRequestAndSession`), so the Government Gateway `Authorization` bearer token and the platform tracing headers are passed on automatically.

Base URLs come from `microservice.services.<name>.{protocol,host,port}`. These are loaded through `config.Service` (protocol defaults to `https` when it is not set) and exposed on `FrontendAppConfig`. Paths are hard-coded in each connector. Calls to the PSR backend, pensions scheme and pension administrator use the request timeout `ifs.timeout` (default 50s).

Query strings for the PSR backend are built by `utils.HttpUrl.makeUrl`, which concatenates strings (`key=value` joined with `&`) **without URL-encoding**.

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | `pension-scheme-return` (SIPP endpoints) | HTTP backend | `microservice.services.pensionSchemeReturn` | `PSRConnector` |
| 2 | `pensions-scheme` | HTTP backend | `microservice.services.pensionsScheme` | `SchemeDetailsConnector(Impl)` |
| 3 | `pension-administrator` | HTTP backend | `microservice.services.pensionAdministrator` | `MinimalDetailsConnector(Impl)`, `SessionDataCacheConnector` |
| 4 | `upscan-initiate` + upscan download URLs | HTTP platform service | `microservice.services.upscan`, `urls.upscan.*` | `UpscanConnector`, `UpscanDownloadStreamConnector` |
| 5 | `object-store` | Platform service (client lib) | `microservice.services.object-store`, `object-store.default-retention-period` | `UploadRepository` (`PlayObjectStoreClient`) |
| 6 | `internal-auth` | Platform service | `microservice.services.internal-auth`, `internal-auth.token`, `create-internal-auth-token-on-start` | `InternalAuthTokenInitialiserImpl` |
| 7 | `auth` | Platform service | `microservice.services.auth` | `IdentifierActionImpl` (`AuthorisedFunctions`) |
| 8 | MongoDB | Datastore | `mongodb.*` | `SessionRepository`, `UploadMetadataRepository` |
| 9 | Datastream (audit) | Platform service | bootstrap `auditing.*` | `AuditService` (`AuditConnector`) |
| 10 | Linked frontends (redirects/links only) | Browser navigation | `urls.*`, `microservice.services.contact-frontend`, `pensionSchemeReturnFrontend` | `FrontendAppConfig.urls` |

---

## 1. Pension Scheme Return backend (SIPP)

- **Base URL:** `microservice.services.pensionSchemeReturn` (local `http://localhost:10704`), plus `/pension-scheme-return-sipp/psr`.
- **Headers:** `srn` (from the route `Srn`) and `requestRole` (`PSP` or `PSA`, from the auth enrolment) on every call. `CorrelationId` (a random UUID per call) on the create, submit, PUT and asset GET calls. **Not** on `getPSRSubmission`, `getMemberDetails`, `getPsrVersions`, `getPsrAssetCounts` or `getPsrAssetDeclarations`. The user's bearer token goes through `HeaderCarrier`.
- **Return identification:** each call names the return with **either** `fbNumber` **or** `periodStartDate` + `psrVersion`. These come from the Play session (`createQueryParamsFromSession`) or from explicit/query parameters (`createQueryParams`). If neither is present, an `EtmpServerError` is **thrown synchronously** before any request is made.
- **Timeout:** `ifs.timeout`.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Create empty return | `POST /psr/empty/sipp` | `PSRConnector.createEmptyPsr` | Body `ReportDetails`. Only `201` is success; anything else fails |
| Member transactions held | `PUT /psr/report-details/member-transactions/{pstr}?journeyType=Standard&<id>` | `updateMemberTransactions` | Body `{memberTransactions: Yes/No}` |
| Accounting periods | `PUT /psr/accounting-periods/{pstr}?journeyType=Standard&<id>` | `updateAccountingPeriodsDetails` | |
| Submit interest in land/property | `PUT /psr/land-or-connected-property?journeyType={jt}&<id>` | `submitLandOrConnectedProperty` | Audited PUT (see below) |
| Submit arm's-length land/property | `PUT /psr/land-arms-length?journeyType={jt}&<id>` | `submitLandArmsLength` | Audited PUT |
| Submit outstanding loans | `PUT /psr/outstanding-loans?journeyType={jt}&<id>` | `submitOutstandingLoans` | Audited PUT |
| Submit assets from connected party | `PUT /psr/assets-from-connected-party?journeyType={jt}&<id>` | `submitAssetsFromConnectedParty` | Audited PUT |
| Submit tangible moveable property | `PUT /psr/tangible-moveable-property?journeyType={jt}&<id>` | `submitTangibleMoveableProperty` | Audited PUT |
| Submit unquoted shares | `PUT /psr/unquoted-shares?journeyType={jt}&<id>` | `submitUnquotedShares` | Audited PUT |
| Get asset data (×6) | `GET /psr/{land-arms-length,land-or-connected-property,outstanding-loans,assets-from-connected-party,tangible-moveable-property,unquoted-shares}/{pstr}?<id>` | `get*` | Land-or-connected-property converts country code → country name for non-UK addresses |
| Get return | `GET /psr/sipp/{pstr}?<id>` | `getPSRSubmission` | |
| Member list | `GET /psr/member-details/{pstr}?<id>` | `getMemberDetails` | |
| Update member | `PUT /psr/member-details/{pstr}?journeyType={jt}&<id>` | `updateMemberDetails` | Body `{current, updated}` |
| Delete member | `PUT /psr/delete-member/{pstr}?journeyType={jt}&<id>` | `deleteMember` | Body: the `MemberDetails` |
| Delete assets for a journey | `PUT /psr/delete-assets/{pstr}?journey={j}&journeyType={jt}&<id>` | `deleteAssets` | Body is JSON `null` |
| Submit return | `POST /psr/sipp?journeyType={jt}` | `submitPsr` | Body `PsrSubmissionRequest` (pstr, fbNumber/periodStartDate/psrVersion, psaId, taxYear, schemeName, checkReturnDates). Response `{emailSent}` |
| Versions | `GET /psr/versions/{pstr}?startDate=yyyy-MM-dd` | `getPsrVersions` | |
| Asset counts | `GET /psr/asset-counts/{pstr}?<id>` | `getPsrAssetCounts` | Response wrapped in `OptionalResponse` |
| Asset declarations | `GET /psr/asset-declarations/{pstr}?<id>` | `getPsrAssetDeclarations` | |

`<id>` = `fbNumber=…` or `periodStartDate=…&psrVersion=…`. `{pstr}` comes from the pensions-scheme details for the authorised scheme.

**All PUTs (`submitRequest`):**
- The JSON body size is checked against `etmpConfig.maxRequestSize` (default 512000 bytes). If it is bigger, a `PensionSchemeReturnFileUpload` audit event is sent with type of error "size limit" (only for the audited asset submissions), and the call fails with `EtmpRequestDataSizeExceedError` **without** sending.
- Asset submissions add an `auditContext` object to the body, built from the upload metadata (file name, reference = upscan download URL, size, tax year).
- `200` and `201` count as success. Anything else becomes `UpstreamErrorResponse`.

**Error mapping (`handleError`):** `404` → `NotFoundException`; `413` → `EtmpRequestDataSizeExceedError`; other upstream status → `EtmpServerError`; any other exception → `InternalServerException`. `ErrorHandler.onServerError` then redirects the user to `/service-unavailable` or `/files-exceeds-limit`.

**Callers:** `WhatYouWillNeedController`, `AssetsHeldController`, `AccountingPeriodListController` (via services), `JourneyContributionsHeldController`, `ValidateUploadService`, `RemoveFileController`, `DownloadCsvController`, `DeclarationController`, `ViewTaskListController`, `ChangeTaskListController`, `RemoveMemberController`, `ViewChangePersonalDetailsController`, `ReportDetailsService`, `SchemeDateService`, `PsrVersionsService`.

---

## 2. Pensions scheme

- **Base URL:** `microservice.services.pensionsScheme` (local `http://localhost:8203`).
- **Auth:** the user's bearer token via `HeaderCarrier`, plus identity headers.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Scheme details (PSA) | `GET /pensions-scheme/scheme/{srn}` | `SchemeDetailsConnectorImpl.details(psaId, schemeId)` | Headers `idNumber`, `schemeIdType=srn`, `psaId`. `404` → `None` |
| Scheme details (PSP) | `GET /pensions-scheme/psp-scheme/{srn}` | `details(pspId, srn)` | Headers `pspId`, `srn` |
| List user's schemes | `GET /pensions-scheme/list-of-schemes-self` | `listSchemeDetails` | Headers `idValue`, `idType` (`PSA`/`PSP`). `404` → `None` |

**Usage:** `AllowAccessAction` uses the details to authorise access to the scheme and to get the `pstr`, scheme status and `authorisingPSAID`. `SchemeDetailsService.getMinimalSchemeDetails` filters the scheme list by SRN to get the open date and wind-up date (the wind-up date feeds CSV date validation), for the declaration and member removal. A `403` from the details call becomes a redirect to `/unauthorised`. Other failures are logged and re-thrown.

---

## 3. Pension administrator

- **Base URL:** `microservice.services.pensionAdministrator` (local `http://localhost:8205`).

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Minimal details | `GET /pension-administrator/get-minimal-details-self` | `MinimalDetailsConnectorImpl.fetch` | Headers `psaId` or `pspId`, `loggedInAsPsa=true/false`. `404` with "no match found" → `DetailsNotFound`; `403` with "DELIMITED_PSAID" → `DelimitedAdmin`; other errors are re-thrown |
| Role choice for dual-enrolled users | `GET /pension-administrator/journey-cache/session-data-self` | `SessionDataCacheConnector.fetch` | No identifying header; the `cacheId` argument (externalId) is only used in the log message. `404` → `None` |

**Usage:** `AllowAccessAction` checks `rlsFlag`, `deceasedFlag` and `DelimitedAdmin`, and only lets in users with both flags false. `PsrVersionsController` uses the individual or organisation name for display. `IdentifierAction` uses the session data (`Administrator` / `Practitioner`) when the user holds both enrolments.

---

## 4. Upscan

- **Initiate URL:** `microservice.services.upscan` baseUrl + `urls.upscan.initiate` (`/upscan/v2/initiate`; local `http://localhost:9570`).
- **Headers:** `Content-Type: application/json`. No auth.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Initiate | `POST /upscan/v2/initiate` | `UpscanConnector.initiate` | Body: `callbackUrl` (absolute URL of `/fileupload/upscan-callback`; `https` unless `microservice.services.upscan.secure=false`), `successRedirect` (loading page), `errorRedirect` (upload page), `minimumFileSize=1`, `maximumFileSize=maxFileSize MB × 1 MiB` |
| Download (stream) | `GET {downloadUrl}` | `UpscanDownloadStreamConnector.stream` | Uses the URL that arrived in the upscan callback |
| Download (buffered) | `GET {downloadUrl}` | `UpscanConnector.download` | ⚠️ **Observation:** not called anywhere (`UploadService.downloadFromUpscan` has no callers) |

**Callback into the service:** `POST /pension-scheme-return-sipp/fileupload/upscan-callback`. See *Inbound callers*.

---

## 5. Object store

- Client: `object-store-client-play-30`, enabled through `ObjectStoreModule`. Base URL from `microservice.services.object-store` (local port 8464). Auth uses the `internal-auth.token` config value.
- Directory `uploads/`. Object name = `UploadKey.value` = `{internalId}&&{srn}&&{uploadRedirectTag}`. Retention is `object-store.default-retention-period = 1-day`.
- `UploadRepository.save` writes a stream of length-prefixed (4-byte big-endian) records. Each record is a JSON `CsvRowState` wrapped in `SensitiveCsvRow`, so it is encrypted when crypto is enabled. `retrieve` reads them back for submission (`ValidateUploadService`) and for the "download with errors" CSV (`DownloadCsvController`). Frames can be at most 256 000 bytes.

## 6. Internal auth

`InternalAuthTokenInitialiserImpl` runs only when `create-internal-auth-token-on-start=true`. It calls `GET {internal-auth}/test-only/token` with `Authorization: <internal-auth.token>`. On `404` it calls `POST /test-only/token` to create a token with `READ/WRITE/DELETE` on the `object-store` resource `pension-scheme-return-sipp-frontend`. It retries (`internal-auth.retry.delay` / `max-attempts`) and **blocks start-up for up to 5 minutes** (`Await.result`).

## 7. Auth

See the *Auth* section below.

## 8. Datastore (MongoDB)

| Collection | Repository | Key | Expiry | Purpose |
|---|---|---|---|---|
| `user-answers` | `SessionRepository` | `_id` = `internalId + srn` (concatenated, no separator) | `lastUpdated` TTL `mongodb.timeToLiveInSeconds` (3600s); `get` also refreshes `lastUpdated` | Journey answers (`UserAnswers`) |
| `upload` | `UploadMetadataRepository` | `id` = `UploadKey` string (unique); `reference` = upscan reference (unique) | `lastUpdated` TTL `mongodb.upload.timeToLiveInSeconds` (3600s) | Upload status (InProgress / Success / Failed) and validation state |

**Encryption:** `config.Crypto` is bound to `CryptoImpl` (AES-GCM through `SymmetricCryptoFactory.aesGcmCryptoFromConfig("mongodb.encryption")`, key config `mongodb.encryption.key`) when `mongodb.encryption.enabled=true`. Otherwise it is a **no-op** that stores plaintext. The no-op is the default in `application.conf`. The encrypted fields are the `UserAnswers` format, the upload `status` and `validationState`, and the object-store CSV rows.

## 9. Audit

`AuditService.sendEvent` / `sendExtendedEvent` send events with `auditSource = appName`:

| Audit type | Sent from | When |
|---|---|---|
| `PensionSchemeReturnStarted` | `WhatYouWillNeedController.onPageLoad` | No existing return found for the user |
| `PensionSchemeReturnFileUpload` | `FileUploadSuccessController`, `FileUploadErrorSummaryController`, `FileUploadTooManyErrorsController`, `PSRConnector.submitRequest` (size limit) | Upload outcome (success / validation errors / size limit) |
| *(embedded)* `auditContext` | `PSRConnector.submitRequest` | Sent inside the asset PUT body for the backend to audit |

## Auth

| Action | Requires | Retrieves / produces |
|---|---|---|
| `IdentifierActionImpl` | Active GG session; enrolment `HMRC-PODS-ORG` (identifier `PSAID`) or `HMRC-PODSPP-ORG` (identifier `PSPID`); `internalId` and `externalId` present | `AdministratorRequest` / `PractitionerRequest` (internalId as `userId`, externalId, PSA/PSP ID). With both enrolments, the role comes from pension-administrator session data. No session → login redirect; other auth failure → MPS register page |
| `AllowAccessAction(srn)` | Pensions-scheme details found for this user and SRN, status in {Open, WoundUp, Deregistered}; minimal details with `rlsFlag=false` and `deceasedFlag=false` | `AllowedAccessRequest` (schemeDetails, minimalDetails, srn) |
| `DataRetrievalAction` / `DataCreationAction` / `DataRequiredAction` | – | Load, create (empty) or require `UserAnswers` keyed `internalId+srn` |
| `FormBundleRequiredAction` | `fbNumber` in session | `FormBundleRequest` |
| `TaxYearRequiredAction` / `VersionTaxYearRequiredAction` | `taxYear` (+ `version`) in session | Tax-year date range |
| `FormBundleOrVersionTaxYearRequiredAction`, `FormBundleOrTaxYearRequiredAction` | One of the above | Combined request |
| `IdentifyAndRequireData` | Composes identify → allowAccess → getData → requireData (→ optional refiners) | |

## Inbound callers and callbacks

| Endpoint | Caller | Auth |
|---|---|---|
| All `/pension-scheme-return-sipp/:srn/...` pages | Browser, linked from Manage Pension Schemes and the main PSR frontend | GG + enrolment + scheme access |
| `POST /pension-scheme-return-sipp/fileupload/upscan-callback` | Upscan | `reference` must match an existing upload record |
| `GET /pension-scheme-return-sipp/download-template-file/:file` | Browser | None (serves static templates from `conf/`) |
| `GET /refresh-session`, sign-out, `/unauthorised`, `/there-is-a-problem`, `/service-unavailable`, `/files-exceeds-limit` | Browser | Mixed (sign-out and keep-alive require identify) |

The service registers the upscan callback URL with each initiate request. No other webhooks are registered.

**Session keys shared with sibling services:** `fbNumber`, `taxYear`, `version` and `SRN` are read from the Play session. This service only **writes** `fbNumber`. `taxYear`, `version` and `SRN` are never written here, so they are presumably set by the main PSR frontend, which shares the session cookie.

## Libraries

| Library | Version | Used for |
|---|---|---|
| `bootstrap-frontend-play-30` | 10.7.0 | HttpClientV2, auth, auditing, error handler, filters |
| `play-frontend-hmrc-play-30` | 13.14.0 | GOV.UK / HMRC UI components |
| `hmrc-mongo-play-30` | 2.13.0 | Mongo repositories and TTL indexes |
| `object-store-client-play-30` | 2.6.0 | Object store client |
| `crypto-json-play-30` | 8.4.0 | `Sensitive` JSON encryption |
| `domain-play-30` | 13.0.0 | `Nino` validation |
| `tax-year` | 6.0.0 | Tax year calculations |
| `pekko-connectors-csv` | 1.3.0 | Streaming CSV parsing and formatting |
| `fs2-reactive-streams`, `cats-core`/cats-effect | 3.14.0 / 2.13.0 | Background validation and submission (`IO`) |
| `refined`, `enumeratum-play-json`, `quicklens` | 0.11.4 / 1.9.8 / 1.9.15 | Types, enums, updates |
| `play-conditional-form-mapping-play-30` | 3.5.0 | Conditional forms |

## Local development

- `sm2 -start PSR_ALL` starts every dependency, this frontend included. Stop `PENSION_SCHEME_RETURN_SIPP_FRONTEND` and run `sbt run` (port **10703**) to use local code.
- Local defaults: auth 8500, pension-administrator 8205, pensions-scheme 8203, pension-scheme-return 10704, main PSR frontend 10701, upscan 9570 (`secure=false` so the callback is `http`), object-store 8464, internal-auth 8470, contact-frontend 9250, MongoDB `localhost:27017/pension-scheme-return-sipp-frontend`.
- Login goes through `auth-login-stub` (`urls.login`, port 9949). `create-internal-auth-token-on-start=true` creates the object-store token through the internal-auth **test-only** endpoint.

### Other observations

- ⚠️ **Observation:** `FrontendAppConfig.urls.pensionPractitioner.updateContactDetails` is built from the **`urls.pension-administrator.updateContactDetails`** key (`/update-contact-address`). The PSP-specific key `urls.pension-practitioner.updateContactDetails` (`/update-contact-address-psp`) is never read, so PSPs with the RLS flag are sent to the wrong path.
- ⚠️ **Observation:** `SessionDataCacheConnector.fetch(cacheId)` does not send the cache id. It relies on the `-self` endpoint working the id out from the bearer token.
- ⚠️ **Observation:** `PSRConnector.createQueryParams*` throw synchronously, outside the returned `Future`. In practice this still reaches `ErrorHandler.onServerError`, which redirects to the service-unavailable page.
- ⚠️ **Observation:** `WhichTaxYearController.allDates` and `form` are computed once, when the controller is constructed (`taxYearService.current`). After 6 April the list of tax years stays stale until the application restarts.
- ⚠️ **Observation:** route `POST /:srn/land-or-property-contributions-made=amend` has `=` where `-` is expected (`conf/app.routes:86`).
