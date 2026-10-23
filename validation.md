# Pension Scheme Return SIPP Frontend – Data Validation

This service is a browser frontend, so it checks data at two very different boundaries:

- **User-entered data** (HTML forms and uploaded CSV files) gets **rich semantic validation**: regexes, lengths, value ranges, date ranges, NINO/UTR/CRN formats and cross-field rules. Every failure becomes a localised error message (a `400` page re-render for forms, or a per-cell error list for CSVs).
- **Identifiers and parameters in URLs, sessions and callbacks** get **mostly structural checks**. The SRN is checked against a regex and enums are bound by name. Return and member identifiers are validated by the PSR backend / ETMP.

Server-side failures of any kind (`EtmpServerError`, a parse exception, an unknown exception) are not shown as 4xx/5xx. `ErrorHandler.onServerError` turns **every** unhandled exception into a `303` redirect to `/service-unavailable` (or `/files-exceeds-limit` when the request was too large).

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Route path/query binding | `PathBindable` / `QueryStringBindable`, Play built-in `Int` | `config.Binders`, routes | `400` (Play client error page), except where noted |
| 2 | Authentication / enrolment | `authorised(Enrolment(...))` | `IdentifierActionImpl` | `303` to login, register, unauthorised or role-choice page |
| 3 | Scheme access | pensions-scheme lookup, status allow-list, minimal-details flags | `AllowAccessAction` | `303` to `/unauthorised` or an MPS page |
| 4 | Session preconditions | `fbNumber` / `taxYear` / `version` presence | `*RequiredAction` refiners, `DataRequiredAction` | `303` to `/there-is-a-problem` |
| 5 | HTML forms | Play `Form` mappings (`Mappings`, `Formatters`, `Constraints`) | `forms.*FormProvider` | `400` re-render with errors |
| 6 | Upload pre-checks | upscan min/max size; `.csv` filename suffix | `UpscanConnector.initiate`, `PendingFileActionService.checkFileFormat` | `303` back to the upload page with `errorCode`/`errorMessage` |
| 7 | CSV row validation | Form mappings applied per cell, accumulated with `ValidatedNel` | `*CsvRowValidator`, `ValidationsService`, `*ValidationsService` | Errors stored; error summary page or "too many errors" page; error CSV download |
| 8 | Outbound size check | JSON body byte length vs `etmpConfig.maxRequestSize` | `PSRConnector.submitRequest` | Request not sent; audit; `303` to `/files-exceeds-limit` |
| 9 | Upstream response parsing | Play JSON `Reads` | models in `models.*`, `models.requests.*` | `JsResultException` → `InternalServerException` → `303` to `/service-unavailable` |
| 10 | Upscan callback | `parse.json` + `withJsonBody[CallbackBody]` | `UploadCallbackController` | `400` (bad JSON / failed read) |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `srn` (path) | `^S[0-9]{10}$` | `Binders.srnBinder` → `Srn.apply`. Mismatch → `400` "Invalid scheme reference number" |
| `journey` | enumeratum `withNameOption`. In practice it is fixed by route defaults and never taken from the URL | `Binders.journeyPathBinder` |
| `journeyType` (query binder) | `JourneyType.withName`, which **throws** on an unknown value | `Binders.journeyTypeQueryStringBinder`. Every route fixes it with a default (`= Standard/Amend`), so it is never bound from user input |
| `file` (template) | Case-insensitive name match. An unknown name falls back to `Unknown`, which **never fails binding** | `Binders.templateFileTypeBinder`. `Unknown` makes `sendFile("conf/unknown")` throw → `303` to `/service-unavailable` |
| `index` (accounting period) | Play `Int`, then `refineUnsafe[Int, OneToThree]` | Called **while the `Action` is built**, outside any `Future`. An index outside 1–3 throws → `303` to `/service-unavailable` (not `400`) |
| `page` | Play `Int`; no lower or upper bound | `ViewChangeMembersController.onPageLoad` |
| `search` | Optional string; not validated (used only in-memory, case-insensitive `contains`) | |
| `fbNumber` (query, `Option[String]` / `String`) | **None** | Declaration, view task list, view/change question, CSV downloads |
| `periodStartDate`, `psrVersion` (query) | **None** | CSV downloads |
| `taxYear` (query, `Int`) | Play `Int` only | `ViewChangeQuestionController.onSubmit` |
| `firstName`, `lastName`, `nino`, `reasonNoNINO` (query) | **None** | `ViewChangeMembersController.redirectTo*` |
| `dateOfBirth` (query) | `LocalDate.parse(_, CSV_DATE_TIME)`, called **synchronously** | Bad value → `DateTimeParseException` → `303` to `/service-unavailable` (not `400`) |
| `continueUrl` | `RedirectUrl` (bootstrap) | `JourneyRecoveryController` |

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| All authenticated pages | Session cookie, GG bearer token | Checked by `auth` |
| Upscan callback | `Content-Type: application/json` | `parse.json` (`400` otherwise) |

No business data is read from inbound headers.

## Request bodies

### HTML forms (summary of the rules)

| Field family | Rule | Source |
|---|---|---|
| Required text | Non-blank after `trim` | `Formatters.stringFormatter` |
| Names | `^[a-zA-Z\-' ]+$`, max 35 | `TextFormProvider.name`, `NameDOBFormProvider` |
| Text area | `^[a-zA-Z0-9\-'" \t\r\n,.@/]+$`, max 160 | `textArea`, `text` |
| Free text (CSV) | No CR/LF, max 160 | `freeText` |
| Address line / town | `^[a-zA-Z0-9\-'" \t\r\n]+$`, max 35 | `addressLine` |
| UK postcode | `^([A-Z]{1,2}[0-9][0-9A-Z]?\s?[0-9][A-Z]{2}\|BFPO\s?[0-9]{1,3})$` (case-sensitive), max 160 | `postcode` |
| Country | Must match a label in `Country.countries` (case-insensitive). Stored as a country code | `Mappings.selectCountry` |
| Yes/No (CSV) | `(?i)^(yes\|no)$` | `yesNo` |
| Enumerated text (CSV) | `INDIVIDUAL\|COMPANY\|PARTNERSHIP\|OTHER`, `CONNECTED\|UNCONNECTED`, `MARKET VALUE\|COST VALUE` (case-insensitive) | `TextFormProvider` |
| NINO | `uk.gov.hmrc.domain.Nino.isValid(upper-cased)` | `Mappings.nino` |
| UTR | `^\d{5}\s*?\d{5}$` (whole-string match) | `Utr.isValid` |
| CRN | `^[A-Za-z0-9 ]*$` and exactly 8 characters once spaces are removed | `Crn.isValid`, `Crn.isLengthInRange` |
| PSA ID (PSP declaration) | `^(A[0-9]{7})$`, max 8, **must equal the scheme's `authorisingPSAID`** | `Mappings.psaId` |
| Money | ±999,999,999.99, at most 2 decimal places | `MoneyFormProvider`, `Constants` |
| Percentage | -999.99 … 999.99 | `DoubleFormProvider` |
| Dates | Valid day/month/year. Ranges depend on the context (e.g. not before 1900-01-01, not after the scheme wind-up date) | `LocalDateFormatter`, `DatePageFormProvider` |
| Accounting periods | Up to 3; inside the tax year; no overlap with existing periods; no gaps | `DateRangeFormatter` |
| Tax year | Must be one of the 7 offered tax years | `RadioListFormProvider` + `Enumerable` |

On failure: `400` with the form re-rendered and messages from `messages.en`/`messages.cy`.

### CSV uploads

**Reader:** `CsvDocumentValidator` streams `CsvParsing.lineScanner()`. Row 1 is the header row and row 2, the helper row, is skipped. Data rows are numbered from 3. Each row is validated in parallel (up to 8 at once) by the journey's `CsvRowValidator`. Columns are found **by header name** (`getCSVValue` / `getOptionalCSVValue`).

**What is checked:**
- Every mandatory column is present in the header. Otherwise the row gets `invalidFileFormat` (a single "invalid row format" error for that row).
- Each cell is checked with the form rules above. Errors are **accumulated per row** (`ValidatedNel`) and each one carries its cell reference (e.g. `C5`).
- Member identity: first name, last name and date of birth are required. The date of birth must not be after the scheme wind-up date when there is one. The NINO, **or** a reason for having no NINO, is required.
- Conditional fields, for example: arrears amount only when present; UK vs non-UK address (UK needs a postcode, non-UK needs a country); share company **CRN or a reason for no CRN** (`UnquotedSharesValidationsService`, `AssetsFromConnectedPartyValidationsService`); land and property specifics (`LandOrPropertyValidationsService`, `TangibleMoveablePropertyValidationsService`).
- Member date of birth must not be after the scheme wind-up date, or after today when there is no wind-up date (`validateNameDOB`).
- ⚠️ **Observation:** `ValidationsService.validateUtr` exists but no CSV validator calls it, so UTRs are never checked in uploads.

**What is *not* checked:**
- Header names or order beyond the mandatory columns being present. Extra columns are ignored.
- Uniqueness of members or transactions inside the file, and consistency of the same member's name and date of birth across rows.
- The total row count. The only limit is the JSON size check before submission.
- File encoding and character set. Bytes are decoded as UTF-8 by `utf8String`.
- Whether the file is really a CSV, beyond the `.csv` name suffix and upscan's MIME/virus scan.

**Normalisation and derivation:**

| Input | Rule |
|---|---|
| NINO | `toUpperCase.filterNot(_.isWhitespace)` before validation |
| NINO / UTR / CRN in forms | Upper-cased before validation and in the stored value |
| Yes/No | `YesNo.withNameInsensitive` |
| Country | Label → ISO code (`Country.getCountryCode`). Turned back into a label when reading land/property data from upstream |
| Dates | `-` or `/` separator detected (`validateDate`) |
| Empty optional cells | Become `None` |
| Row number | Added to each transaction as `row` |

**Error threshold:** `validation.csv.error-limit` (25) only affects **navigation** (`SippNavigator`): 25 or fewer errors shows the summary, more shows the "too many errors" page. Every error is still stored in the `upload` document (`CsvDocumentInvalid.errors`), so the document is unbounded.

**Empty file:** `CsvDocumentEmpty` → upload error page with "empty csv".

**Exceptions during validation** (download, parse, object-store write) → `ValidationException` state → `/there-is-a-problem`.

### Upscan callback body

`CallbackBody.reads` dispatches on `fileStatus`:
- `READY`: requires `reference`, `downloadUrl` (any string that `java.net.URL` can parse), and `uploadDetails{uploadTimestamp, checksum, fileMimeType, fileName, size}`.
- `FAILED`: requires `reference` and `failureDetails{failureReason, message}`.
- Any other value, or a missing discriminator → `JsError` → `400`.

A reference that is not in the database is a silent no-op (`findOneAndUpdate` without upsert, `200`).

## External validation services

| Service | Use | Fail mode |
|---|---|---|
| Upscan | Virus scan, file-size limits (`minimumFileSize=1`, `maximumFileSize=maxFileSize MiB`) | **Closed.** A `FAILED` status (`QUARANTINE`, `REJECTED`, `UNKNOWN`, `EntityTooLarge`, ...) sends the user back to the upload page with a mapped error |
| Pensions scheme / pension administrator | Scheme association, status, flags | **Closed.** No details → `/unauthorised`. Other errors → `/service-unavailable` |

## Outbound payload validation

- No JSON Schema files are bundled or loaded. Outbound bodies are built from types that already passed form or CSV validation.
- **Size:** `PSRConnector.submitRequest` measures the UTF-8 JSON body size **before sending**. Above `etmpConfig.maxRequestSize` (512 000 bytes) it logs an error, sends a `PensionSchemeReturnFileUpload` audit event (asset submissions only) and fails with `EtmpRequestDataSizeExceedError`. A `413` from upstream is mapped to the same error.
- **Return identifier presence:** `createQueryParams*` require `fbNumber`, or both `periodStartDate` and `psrVersion`. Otherwise `EtmpServerError` is thrown synchronously.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| PSR create / PUT | Status `201`/`200`; body `.as[SippPsrJourneySubmissionEtmpResponse]` | Other status → `EtmpServerError` → `/service-unavailable`. Bad JSON → `InternalServerException` → same page |
| PSR GETs | `execute[T]` with Play JSON reads | `404` → `NotFoundException`; others → `EtmpServerError`/`InternalServerException` → `/service-unavailable` |
| Asset counts | `OptionalResponse` wrapper | Missing → `None` (no download links) |
| Scheme details | `Reads[SchemeDetails]` (requires name, pstr, a known `schemeStatus`, `schemeType.name`, `establishers` array) | Unknown status or missing field → exception → `/service-unavailable` |
| Minimal scheme details | Custom date regex `(\d{4})-(\d{1,2})-(\d{1,2})` | JsError → exception |
| Minimal details | `Json.reads[MinimalDetails]` | Exception, except for the mapped `404`/`403` messages |
| Upscan initiate | `PreparedUpload` reads | Exception → `/service-unavailable` |
| Stored CSV rows (object store) | Length-framed (≤256 000 bytes) JSON + decrypt | Exception → `ValidationException` / failed download |

## Inbound callbacks and opaque stores

- **Upscan callback**: see above.
- **Journey store (`user-answers`)**: typed `UserAnswers` JSON accessed through page `Gettable`s. Nothing is validated again on read.
- **Upload store**: the `UploadKey` string is parsed by splitting on `&&` and running the SRN regex (`UploadKey.uploadKeyReads`). A malformed key fails the read.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| Path binder (SRN) | `Left("Invalid scheme reference number")` | `400` |
| Form binding | `formWithErrors` | `400` (page re-rendered) |
| Missing session data / user answers | refiner `Left` | `303` `/there-is-a-problem` |
| Auth: no session | `NoActiveSession` | `303` login |
| Auth: other | `AuthorisationException` / missing enrolment | `303` MPS register / `/unauthorised` |
| Scheme access denied | `None` / bad status / flags / `403` | `303` `/unauthorised` or MPS page |
| PSR backend | `EtmpServerError`, `NotFoundException`, `InternalServerException` | `303` `/service-unavailable` |
| PSR backend / local size check | `EtmpRequestDataSizeExceedError` | `303` `/files-exceeds-limit` |
| Synchronous parse in action (`dateOfBirth`, session `taxYear`) | `DateTimeParseException` | `303` `/service-unavailable` |
| Accounting-period `index` outside 1–3 | `refineUnsafe` exception | `303` `/service-unavailable` |
| Missing `fbNumber` and `version`+`taxYear` for a connector call | `EtmpServerError` (thrown synchronously) | `303` `/service-unavailable` |
| Unknown template file | file not found | `303` `/service-unavailable` |
| Upscan failed status | `UploadStatus.Failed` | `303` upload page with an error |
| Background validation / submission | `ValidationException` / `SavingToEtmpException(url)` state | Loading page → `303` `/there-is-a-problem` or the ETMP error page |
| Callback body invalid | `JsError` | `400` |

## Recommendations

1. Turn a bad `dateOfBirth` query value into `400` rather than a server error.
2. Cap the number of stored CSV errors (e.g. `error-limit + 1`) instead of keeping them all in the `upload` document.
3. Reject an unknown `TemplateFileType` at binding time (`400`) rather than falling back to `Unknown` and failing with a server error.
4. Make the postcode regex case-insensitive or upper-case the input first, to match how NINO, UTR and CRN are handled.
5. Escape double quotes when echoing raw CSV values back in `DownloadCsvController.toCsvRow` and `getHeadersAndHelpersCombined`.
