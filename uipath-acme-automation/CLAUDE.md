# CLAUDE.md — ACME Employee Data Extraction

## Project Overview

RE-Framework automation that logs into the ACME website, extracts employee data, and exports results to Excel. Built on the UiPath RE-Framework State Machine pattern for robust error handling, retry logic, and Orchestrator integration.

Credentials (username/password) are **always fetched from Orchestrator Assets** at runtime. No sensitive data is hardcoded or stored in this project.

## UiPath Skills

Use these Claude Code skills when working on this project:

- `uipath:uipath-rpa-workflows` — creating and editing XAML RPA workflows
- `uipath:uipath-coded-workflows` — coded workflow variants (C# or VB.NET)
- `uipath:uipath-project-discovery-agent` — generating project context for AI assistance
- `uipath:uipath-agents` — agent-based automation tasks

## Project Conventions

| Setting | Value |
|---|---|
| Expression Language | VisualBasic |
| Target Framework | Windows |
| Studio Version | 26.x (26.2.0.0) |
| Architecture | RE-Framework State Machine |
| Credential Source | Orchestrator Assets only |
| Schema Version | 4.0 |

## File Structure

```
uipath-acme-automation/
├── Main.xaml                          # Entry point — RE-Framework state machine
├── project.json                       # Project config (Studio 26.x schema)
├── Framework/
│   ├── InitAllSettings.xaml           # Loads config from Config/ and Orchestrator
│   ├── InitAllApplications.xaml       # Opens applications (ACME login)
│   ├── GetTransactionData.xaml        # Fetches next work item
│   ├── Process.xaml                   # Business logic per transaction
│   └── CloseAllApplications.xaml     # Graceful teardown
├── Config/
│   └── Config.xlsx                    # Settings and constants (no credentials)
└── Data/                              # Temp data folder (gitignored at runtime)
```

## Validation

The `uip` CLI is used for workflow validation and publishing. It is **not currently installed** — ask the user to install it before running validation commands:

```bash
# Install via: https://docs.uipath.com/studio/standalone/current/user-guide/uipath-cli
uip package pack . --output ./output
uip package analyze .
```
