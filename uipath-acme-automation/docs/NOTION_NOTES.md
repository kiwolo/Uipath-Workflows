# ACMEEmployeeDataExtraction — Project Documentation

> **Studio Version:** UiPath Studio 26.2 | **Framework:** Windows | **Expression Language:** VB.NET | **Architecture:** RE-Framework State Machine

---

## 1. Project Overview

**ACMEEmployeeDataExtraction** automates the extraction of employee data from the ACME test web application and exports the results to a timestamped Excel file. It is built on the UiPath **Robotic Enterprise (RE) Framework**, which provides a battle-tested, production-grade state machine structure for attended and unattended robots.

### Key Features

- Credentials are fetched exclusively from **UiPath Orchestrator Credential Assets** — never stored in config files or logs
- Passwords are handled as `SecureString` throughout the entire lifecycle
- Structured error handling distinguishes **Business Exceptions** (expected, skip cleanly) from **System Exceptions** (unexpected, retry with backoff)
- Configurable via `Config.xlsx` for non-sensitive settings (URLs, timeouts, paths)
- Output is a clean, timestamped Excel file with 5 standardised employee columns

---

## 2. Setup & Prerequisites

### Environment Requirements

| Requirement | Details |
|---|---|
| UiPath Studio | 26.2 or later |
| Target Framework | Windows |
| Expression Language | Visual Basic .NET |
| Browser | Chrome or Edge (configurable) |
| Orchestrator | Required — credential asset must exist |

### NuGet Package Dependencies

| Package | Version |
|---|---|
| `UiPath.Excel.Activities` | 26.2.0 |
| `UiPath.System.Activities` | 26.2.4 |
| `UiPath.UIAutomation.Activities` | 26.2.0 |
| `UiPath.WebAPI.Activities` | 1.15.0 |

### Before First Run Checklist

- [ ] Robot is connected to Orchestrator with appropriate permissions
- [ ] Orchestrator Credential Asset is created and populated (see Section 3)
- [ ] `Config.xlsx` is populated with correct values (see Section 3)
- [ ] Chrome or Edge browser is installed and the corresponding UiPath extension is enabled
- [ ] Output folder path in `Config.xlsx` exists and is writable

---

## 3. Orchestrator Configuration

### Credential Asset Setup

Create a **Credential Asset** in Orchestrator with the following properties:

| Field | Value |
|---|---|
| Asset Name | (your chosen name, e.g. `ACME_Credentials`) |
| Asset Type | Credential |
| Username | ACME site username / email |
| Password | ACME site password |

> The asset name you choose must match the value of `OrchestratorCredentialAsset` in `Config.xlsx`.

### Config.xlsx Setup

Located in the project root. Contains **non-sensitive settings only**. Populate the `Settings` sheet with the following keys:

| Key | Description | Example Value |
|---|---|---|
| `AcmeSiteURL` | Full URL of the ACME web application | `https://acme-test.uipath.com` |
| `OrchestratorCredentialAsset` | Name of the Orchestrator credential asset | `ACME_Credentials` |
| `BrowserType` | Browser to use for automation | `Chrome` |
| `TimeoutMS` | Timeout in milliseconds for UI element waits | `30000` |
| `MaxRetries` | Number of retry attempts on system exceptions | `3` |
| `RetryIntervalSeconds` | Seconds to wait between retry attempts | `5` |
| `OutputFolderPath` | Folder where Excel output files are saved | `C:\RPA\Output\` |
| `OutputFileName` | Base filename (timestamp appended at runtime) | `EmployeeData` |

---

## 4. Workflow Architecture

### RE-Framework State Machine

The robot operates as a **4-state state machine** defined in `Main.xaml`. States transition sequentially under normal operation and route to `EndProcess` on terminal errors.

```
+------------------+       +--------------------+       +-----------+       +------------+
|  Initialization  | ----> | GetTransactionData | ----> |  Process  | ----> | EndProcess |
+------------------+       +--------------------+       +-----------+       +------------+
         |                          |                        |                     ^
         | (init failure)           | (no data)              | (system exception)  |
         +--------------------------|------------------------+---------------------+
                                    |                        (retry up to MaxRetries)
                                    +--- (business exception) ---> EndProcess
```

### State Descriptions

#### Initialization
**File:** `Framework/InitAllSettings.xaml`, `Framework/InitAllApplications.xaml`

- Loads `Config.xlsx` and builds the `Config` dictionary (non-sensitive settings only)
- Connects to Orchestrator and fetches the credential asset by name
- Stores username as `String`, password as `SecureString` — never logged
- Opens the configured browser and navigates to `AcmeSiteURL`
- Logs into the ACME site using the fetched credentials
- On failure: transitions directly to `EndProcess`

#### GetTransactionData
**File:** `Framework/GetTransactionData.xaml`

- Navigates to the Employees section of the ACME site
- Checks whether the employee data table is visible and populated
- Returns transaction data (table reference) if available
- If the table is empty or absent: throws a `BusinessRuleException` — no retry, clean skip

#### Process
**File:** `Framework/Process.xaml`

- Scrapes all rows from the employee table
- Extracts 5 columns per row: `EmployeeID`, `Name`, `Email`, `Department`, `Salary`
- Writes data to a timestamped Excel file at the configured output path
- On `BusinessRuleException`: logs warning, skips without retry
- On `SystemException`: increments retry counter; retries up to `MaxRetries` times with `RetryIntervalSeconds` delay

#### EndProcess
**File:** `Framework/EndProcess.xaml`

- Closes the browser gracefully
- Logs a run summary (records processed, exceptions encountered, output file path)
- Performs any final cleanup

---

## 5. Data Flow

The end-to-end data flow through the automation:

```
Config.xlsx
    |
    v
[InitAllSettings] --> Config{} dictionary (non-sensitive)
                                |
                                v
                  [Orchestrator Credential Asset]
                                |
                                v
                  [InitAllApplications] --> Browser open --> ACME Login
                                                                  |
                                                                  v
                                                     [GetTransactionData]
                                                       Employees table check
                                                                  |
                                                                  v
                                                           [Process.xaml]
                                                         Scrape table rows
                                                         Extract 5 columns
                                                                  |
                                                                  v
                                                       Timestamped Excel file
                                                       (OutputFolderPath\OutputFileName_YYYYMMDD_HHMMSS.xlsx)
```

---

## 6. Security Notes

### What IS protected

- **ACME credentials are never stored in `Config.xlsx`** or any file on disk
- Credentials are fetched at runtime exclusively via the **Orchestrator Credential Asset API**
- Passwords are typed directly from `SecureString` using UiPath's `TypeSecureText` activity — never converted to plain string
- No credential values appear in Robot logs, Orchestrator logs, or exception messages

### What Config.xlsx contains (safe to version-control)

- URLs, timeout values, file paths, retry counts
- The **name** of the credential asset (not the credential itself)

### Recommendations

- Restrict Orchestrator Credential Asset access to this robot's folder/tenant only
- Do not commit `Config.xlsx` if it contains environment-specific URLs that reveal internal infrastructure
- Rotate ACME credentials in Orchestrator without touching any workflow file

---

## 7. Error Handling

### Exception Types

| Exception Type | Trigger | Robot Behaviour |
|---|---|---|
| `BusinessRuleException` | Employee table is empty or not found | Log warning, skip transaction, no retry, continue to EndProcess |
| `SystemException` | Unexpected UI/network/application error | Log error, increment retry counter, wait `RetryIntervalSeconds`, retry up to `MaxRetries` |
| Init failure | Cannot load config or open browser | Immediately transition to EndProcess, log critical error |

### Retry Logic

1. `SystemException` caught in `Process` state
2. Retry count checked against `Config("MaxRetries")`
3. If retries remain: wait `RetryIntervalSeconds`, return to `GetTransactionData`
4. If retries exhausted: log final failure, transition to `EndProcess`

---

## 8. Output

### File Details

| Property | Details |
|---|---|
| Format | `.xlsx` (Excel workbook) |
| Location | `Config("OutputFolderPath")` |
| Filename pattern | `{OutputFileName}_{YYYYMMDD}_{HHmmss}.xlsx` |
| Sheet name | `Employees` |

### Column Definitions

| Column | Description |
|---|---|
| `EmployeeID` | Unique identifier for the employee record |
| `Name` | Full name of the employee |
| `Email` | Corporate email address |
| `Department` | Department or business unit |
| `Salary` | Reported salary value (as scraped from ACME) |

---

## 9. Skills Required

To understand, modify, or extend this project, the following UiPath and related skills are needed:

- **`uipath:uipath-rpa-workflows`** — Reading and editing `.xaml` workflow files in UiPath Studio
- **`uipath:re-framework`** — Understanding the RE-Framework state machine pattern, transaction handling, and config loading
- **`uipath:orchestrator`** — Managing credential assets, robot configurations, and deployment
- **`uipath:ui-automation`** — Web scraping, browser interaction, selector management in Chrome/Edge
- **`uipath:excel-activities`** — Writing structured data to Excel with `UiPath.Excel.Activities`
- **`vb-net:basics`** — Reading and writing VB.NET expressions used throughout the workflow arguments and conditions
- **`uipath:exception-handling`** — Business vs System exception patterns, try/catch blocks, retry logic

---

## 10. TODO / Next Steps

Suggested improvements for future iterations:

- [ ] **Add Notion API export** — After generating the Excel file, push a summary row (run date, record count, status) to a Notion database via the Notion REST API using `UiPath.WebAPI.Activities`
- [ ] **Screenshot on error** — Capture a screenshot when a `SystemException` is thrown and attach it to the Orchestrator job log for easier debugging
- [ ] **Parameterise selectors via Config** — Move hard-coded UI selectors (e.g., table CSS selectors) into `Config.xlsx` so they can be updated without opening Studio
- [ ] **Email notification** — Send a summary email on job completion using `UiPath.Mail.Activities`
- [ ] **Input validation** — Add a preflight check in `InitAllSettings` to validate all required Config keys are present before proceeding
- [ ] **Multi-page scraping** — Extend `GetTransactionData` and `Process` to handle paginated employee tables
- [ ] **Unit test workflows** — Add test cases using UiPath Test Suite to cover happy path, empty table, and credential failure scenarios
- [ ] **Orchestrator queue** — Refactor to push each employee row as a Queue Item for parallel processing across multiple robots

---

## 11. Notion Setup Note

This file was created as a local markdown document because **no Notion API credentials are currently available** in the environment.

To push this documentation to a Notion page:

1. Create a Notion integration at [https://www.notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Share the target Notion page with your integration
3. Set the following environment variable:
   ```bash
   export NOTION_API_KEY=secret_xxxxxxxxxxxxxxxxxxxx
   ```
4. Note the **Page ID** from the Notion page URL (the 32-character hex string)
5. Use the Notion API (`POST /v1/blocks/{page_id}/children`) or an MCP Notion tool to append this content

Until then, the content of this file can be **copy-pasted directly into a Notion page** — the markdown headings, tables, and code blocks render correctly in Notion's paste handler.

---

*Last updated: 2026-04-07 | Project: ACMEEmployeeDataExtraction | Studio: 26.2*
