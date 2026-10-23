# Pension Scheme Return SIPP Frontend – Summary

## Purpose

This service is the user-facing web journey for the **SIPP (Self-Invested Personal Pension) part of the annual Pension Scheme Return (PSR)**. Every registered pension scheme must file a yearly return. For SIPP-style schemes, the return describes the assets that each scheme member holds. The service lets a **Pension Scheme Administrator (PSA)** or **Pension Scheme Practitioner (PSP)** start a return for a scheme and tax year, confirm the return dates and accounting periods, and declare which asset types are held. It then lets them **bulk-upload member-level asset transactions as CSV files** (one file per asset type), have those files validated row by row, and submit the return with a declaration. They can also view earlier submitted versions, amend a return, and correct or remove individual scheme members.

It is a GOV.UK-styled frontend with a full UI (English and Welsh messages). Users reach it from the Manage Pension Schemes dashboard and the main (non-SIPP) Pension Scheme Return frontend, which share the same session. The service holds no master data of its own. Return data is read from and written to the PSR backend, which talks to the HMRC tax platform (ETMP). Scheme and administrator details come from the pensions scheme and pension administrator services. Uploaded files go through the platform's virus-scanning upload service.

## Main Capabilities

### 1. Access control and scheme context
- Only users enrolled as a PSA or PSP get in. Anyone else is sent to register or sign in.
- A user who holds **both** PSA and PSP enrolments must choose a role first. The choice is read from the pension administrator journey cache, and if none has been made the user is sent to the "administrator or practitioner" page.
- For every scheme page the service checks, through the pensions scheme service, that the user is associated with the scheme. The scheme must be **Open, Wound-up or Deregistered**.
- Administrator flags redirect the user elsewhere: deceased → contact HMRC; returned-letter (RLS) → update contact address; delimited PSA → cannot access a deregistered scheme.

### 2. Starting a return and basic details
- The start page looks for an existing return in ETMP. If there is none, it creates an **empty return** (status "Compiled") and stores the new **form bundle number** in the session. A "return started" audit event is sent.
- If the return already has member data or uploaded transaction files, the user goes straight to the task list or the "assets held" question.
- Tax year selection offers the current tax year and the six before it.
- The user confirms the return dates, or enters **up to 3 accounting periods**. Periods are validated against the tax year, and must not overlap or leave gaps. They can be removed or reset.
- The user answers whether the scheme held any member assets ("member transactions"). This answer is saved upstream.

### 3. Asset transaction uploads (six asset types)
The six asset types are: interest in land or property (connected party), arm's-length land or property, tangible moveable property, outstanding loans, unquoted shares, and assets other than land or property acquired from a connected party.
- The user can download an Excel **template** for each asset type.
- For each asset type the user first says whether any were held. A "No" is submitted upstream straight away as an empty declaration.
- The upload page starts a virus-scanned upload. The file must be **`.csv`**, at least 1 byte, and no bigger than the configured maximum (default 10 MB).
- After upload the user confirms the file name. The file is then **streamed and validated row by row** in the background. The first row is the header and the second is a helper row. A loading page polls the progress.
- Per-row validation covers member name and date of birth (including against the scheme wind-up date), NINO or a reason for no NINO, dates, money amounts, percentages, yes/no answers, addresses and postcodes, countries, company numbers (CRN, or a reason for having none) and free-text limits.
- If any row fails, the user sees an error summary when there are 25 errors or fewer (configurable), or a "too many errors" page otherwise. The user can **download the original file with an extra error column**.
- If every row passes, the transactions are submitted to the PSR backend for that asset type, together with file-upload audit context. A request bigger than **500 KB** is refused before it is sent, and the user sees a "files exceed limit" page.
- A file and its submitted data can be removed for an asset type, which deletes those assets upstream.

### 4. Viewing, amending and versions
- The service lists the earlier submitted **versions** of the return for the tax year.
- The user chooses "view" or "change" for a submitted return. View mode shows a read-only task list. Change mode runs the same journeys with the *Amend* flag.
- Submitted data for each asset type can be downloaded as CSV, for any form bundle number or tax-year-and-version pair.

### 5. Member management (amend)
- The service lists the scheme members, with paging (25 per page) and free-text search across name, date of birth and NINO.
- A member's first name, last name, date of birth, NINO, or reason for having no NINO can be changed. The current and updated details are sent upstream together.
- A member can be removed from the return.

### 6. Declaration and submission
- The declaration page shows links to the data that has been submitted for each asset type.
- A **PSA** declares directly. A **PSP** must enter the authorising PSA ID, which has to match the scheme's authorising PSA.
- Submitting sends the return with the PSA ID, the tax year and scheme name, and whether the return dates were confirmed. The upstream service reports whether a confirmation email was sent.

### 7. Session and journey state
- Journey answers are kept in a server-side store for each user and scheme, and expire after 1 hour of inactivity. The keep-alive call extends this.
- Upload metadata and validation state are kept per user, scheme and asset type. Validated rows are stored temporarily in object storage, encrypted when encryption is enabled.

## Cross-cutting Concerns

- **Authentication and authorisation**: Government Gateway session with a PSA or PSP enrolment. Each request is then authorised against the scheme through upstream association lookups.
- **Auditing**: three event types: return started, file upload (success, validation errors, size limit), and the platform's standard request auditing.
- **Validation and transformation**: form-level validation on all pages. Detailed per-cell CSV validation reuses the same form rules. NINOs are upper-cased with whitespace removed. Country names are converted to and from country codes.
- **Data protection**: the journey store, upload metadata and stored CSV rows go through a symmetric (AES-GCM) crypto layer, switched by `mongodb.encryption.enabled`. With the flag off (the local default) the layer does nothing. Object-store files are kept for 1 day. Mongo documents expire after 1 hour.
- **Error handling**: every unhandled server error, including upstream failures, sends the user to a "service unavailable / ETMP error" page. A request-size breach goes to a dedicated page.
- **Feature toggles**: `mongodb.encryption.enabled` (real or no-op crypto) and `create-internal-auth-token-on-start` (creates the object-store access token through the internal-auth test-only endpoint at start-up; meant for local and test environments).

## Dependencies (generic)

| Kind | Role |
|---|---|
| Identity / auth platform | Sign-in, enrolments (PSA / PSP IDs), internal and external user IDs |
| PSR backend (SIPP) | Creates, reads, updates and submits the SIPP return and its asset data; lists versions; member changes |
| Pensions scheme service | Scheme details and status, user-to-scheme association, list of the user's schemes |
| Pension administrator service | PSA/PSP minimal details and flags; role choice for dual-enrolled users |
| File upload (virus scanning) service | Hosted upload form, scanning, callback, temporary download URL |
| Object store | Temporary storage of the validated CSV rows |
| Internal auth | Token that grants this service access to the object store |
| Document database | Journey answers and upload metadata/state |
| Audit platform | Audit events |
| Related frontends | Manage Pension Schemes dashboard, main PSR frontend, PSA/PSP contact-details pages, contact/feedback |

## In One Sentence

A GOV.UK frontend that lets pension scheme administrators and practitioners prepare, upload (as validated CSV files), amend and submit the SIPP section of the annual Pension Scheme Return, by passing everything to the PSR backend and on to ETMP.
