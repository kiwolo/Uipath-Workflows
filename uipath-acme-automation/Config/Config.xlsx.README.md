# Config.xlsx Setup Guide

Create `Config/Config.xlsx` with a sheet named **Settings** and these two columns: `Key` | `Value`

## Required Rows

| Key | Value | Notes |
|-----|-------|-------|
| AcmeSiteURL | https://www.acme-site.com/login | Update to actual ACME URL |
| OrchestratorCredentialAsset | ACME_Credentials | **Name of the Orchestrator Credential Asset** — credentials are NEVER stored here |
| OutputFolderPath | C:\RPA_Output\ | Local output folder (created automatically) |
| OutputFileName | EmployeeData_placeholder.xlsx | Overwritten at runtime with timestamp |
| BrowserType | Chrome | Chrome or Edge |
| TimeoutMS | 30000 | Milliseconds (30 seconds) |
| MaxRetries | 3 | Max retry attempts on system exceptions |
| RetryIntervalSeconds | 5 | Seconds between retries |

## Security Rules

1. **DO NOT** add `Username` or `Password` rows to this file.
2. Credentials must live in a **UiPath Orchestrator Credential Asset** named `ACME_Credentials`
   (or whatever name you set in `OrchestratorCredentialAsset`).
3. This file is safe to commit to version control — it contains no sensitive data.

## Creating the Orchestrator Credential Asset

In UiPath Orchestrator:
1. Go to **Tenant → Assets → Add Asset**
2. Type: **Credential**
3. Name: `ACME_Credentials` (must match `OrchestratorCredentialAsset` value)
4. Enter the ACME site username and password
5. Assign to the robot that will run this process
