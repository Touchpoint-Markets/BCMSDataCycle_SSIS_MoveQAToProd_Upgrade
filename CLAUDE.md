# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **SSIS (SQL Server Integration Services)** project that copies BCMS Gold data from a QA environment to Production. It uses the **Project Deployment Model** and is developed in Visual Studio with SSDT (SQL Server Data Tools) 17.0.1016.0.

- **Source DB:** `10.10.82.45` / `bcmsGold_DemoQA` (QA)
- **Destination DB:** `10.9.57.8` / `bcmsGold` (Production)
- **Auth:** SQL auth as `jdauser` via SQLNCLI11.1 (OLE DB)
- **Target server version:** SQL Server 2025
- **Protection level:** `DontSaveSensitive` — passwords are never persisted in any project file; they must be supplied at runtime through SSIS catalog environments or connection manager overrides.

## Key Files

| File | Role |
|---|---|
| `BCMSDataCycle_SSIS_MoveQAToProd.dtproj` | Project manifest — defines the package list, connection parameter metadata, protection level, and deployment configuration |
| `BCMSGoldQAToProd.dtsx` | The single SSIS package containing all 3 Data Flow Tasks |
| `Project.params` | Project-level parameters (currently empty; package-level parameters are declared inside the `.dtproj` manifest) |
| `BCMSDataCycle_SSIS_MoveQAToProd.slnx` | Solution file (new `.slnx` format); one configuration: `Development` |
| `obj/Development/BuildLog.xml` | Tracks last-known protection levels for the project and each package — SSDT uses this to detect consistency drift |

## Build & Deploy

**Build** (from Visual Studio / SSDT):
- Open `BCMSDataCycle_SSIS_MoveQAToProd.slnx` in Visual Studio.
- Build → Build Solution (or `Ctrl+Shift+B`).
- Output `.ispac` lands in `bin/Development/`.

**Deploy** the `.ispac` to SSIS Catalog:
- Right-click project → Deploy, or use the Integration Services Deployment Wizard.
- After deployment, configure a catalog **Environment** to supply the `jdauser` password for both connection managers (`SourceConnectionOLEDB` and `DestinationConnectionOLEDB`).

**Command-line build** (MSBuild via Developer Command Prompt):
```
msbuild BCMSDataCycle_SSIS_MoveQAToProd.dtproj /p:Configuration=Development
```

## Package Architecture

`BCMSGoldQAToProd.dtsx` contains **3 sequential Data Flow Tasks**, each bulk-copying a batch of tables from `bcmsGold_DemoQA` → `bcmsGold` using `TABLOCK, CHECK_CONSTRAINTS` and `KeepIdentity=true`.

**Data Flow Task 1** — Summary and pricing tables:
`BCMSSummaryBroker`, `BCMSSummaryCarrier`, `BCMSSummarySponsor`, `BCMSSummaryTotal`, `BCPrice`

**Data Flow Task 2** — Reference/lookup tables:
`Broker`, `BrokerCountySt`, `Carrier`, `CarrierHistory`, `CMSA`, `CountyState`, `MSAPMSA`, `naics`, `PlanType`, `Policy`

**Data Flow Task 3** — Policy and remaining reference tables:
`PolicyBroker`, `PolicyBrokerKabot`, `Sponsor`, `States`

All sources and destinations are OLE DB components. There are no transformations — each flow is a direct table-to-table copy.

## Protection Level & Consistency Check

The project enforces that the project manifest and every `.dtsx` package share the **same protection level**. A mismatch causes the build error:
> `Project consistency check failed`

Current state (must stay aligned):
- **Project** (`BCMSDataCycle_SSIS_MoveQAToProd.dtproj`, line 15): `SSIS:ProtectionLevel="DontSaveSensitive"`
- **Package** (`BCMSGoldQAToProd.dtsx`, line 15): `DTS:ProtectionLevel="0"` (0 = DontSaveSensitive)
- **BuildLog** (`obj/Development/BuildLog.xml`): both entries must show `DontSaveSensitive`

If you ever change the protection level, update all three locations atomically. Do **not** add a `PasswordVerifier` property to the project manifest unless the protection level is `EncryptSensitiveWithUserKey`.

## Connection Manager Parameters

The `.dtproj` manifest exposes connection manager properties as package parameters (prefixed `CM.SourceConnectionOLEDB.*` and `CM.DestinationConnectionOLEDB.*`). These are the authoritative override points for environment-specific deployment — do not hard-code connection strings inside the `.dtsx` file directly.
