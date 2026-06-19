# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **SSIS (SQL Server Integration Services)** project that copies BCMS Gold data from a QA environment to Production. It uses the **Project Deployment Model** and is developed in Visual Studio with SSDT (SQL Server Data Tools) 17.0.1016.0.

- **Source DB:** `10.10.82.45` / `bcmsGold_DemoQA` (QA) — OLE DB via SQLOLEDB.1
- **Destination DB:** `10.9.57.8` / `bcmsGold` (Production) — OLE DB via SQLOLEDB.1
- **Third data source:** Linked server `[CINVSQL24.NYC.AMLAW.CORP]` — used in Prep SQL Task 4 only, to copy `PolicyModeledPremiums` directly via SQL (not a connection manager)
- **Auth:** SQL auth as `jdauser` on both OLE DB connections
- **Target server version:** SQL Server 2025
- **Protection level:** `DontSaveSensitive` — passwords are never persisted in any project file; they must be supplied at runtime through SSIS catalog environments or connection manager overrides.

## Key Files

| File | Role |
|---|---|
| `BCMSDataCycle_SSIS_MoveQAToProd.dtproj` | Project manifest — defines the package list, connection parameter metadata, protection level, and deployment configuration |
| `BCMSGoldQAToProd.dtsx` | The single SSIS package containing all tasks (see architecture below) |
| `Project.params` | Project-level parameters (currently empty; package-level parameters are declared inside the `.dtproj` manifest) |
| `BCMSDataCycle_SSIS_MoveQAToProd.slnx` | Solution file (new `.slnx` format); one configuration: `Development` |
| `obj/Development/BuildLog.xml` | Tracks last-known protection levels for the project and each package — SSDT uses this to detect consistency drift |

## Build & Deploy

**Build** (from Visual Studio / SSDT):
- Open `BCMSDataCycle_SSIS_MoveQAToProd.slnx` in Visual Studio.
- Build → Build Solution (`Ctrl+Shift+B`).
- Output `.ispac` lands in `bin/Development/`.

**Deploy** the `.ispac` to SSIS Catalog:
- Right-click project → Deploy, or use the Integration Services Deployment Wizard.
- After deployment, configure a catalog **Environment** to supply the `jdauser` password for both connection managers (`SourceConnectionOLEDB` and `DestinationConnectionOLEDB`).

**Command-line build** (MSBuild via Developer Command Prompt):
```
msbuild BCMSDataCycle_SSIS_MoveQAToProd.dtproj /p:Configuration=Development
```

## Package Architecture

`BCMSGoldQAToProd.dtsx` runs as a **linear chain** of 9 tasks. Each batch follows the pattern: truncate destination tables first (Preparation SQL Task), then bulk-copy from QA (Data Flow Task). The package fails on any single task error (`DTS:FailPackageOnFailure="True"` on every task; `DTS:MaxErrorCount="0"`).

```
Prep SQL 1 → DFT 1 → Prep SQL 2 → DFT 2 → Prep SQL 3 → DFT 3 → Prep SQL 4 → DFT 4 → Final Confirmation Email
```

### Preparation SQL Tasks (run against DestinationConnectionOLEDB)

| Task | Tables truncated |
|---|---|
| Prep SQL Task 1 | `BCMSSummaryBroker`, `BCMSSummaryCarrier`, `BCMSSummarySponsor`, `BCMSSummaryTotal`, `BCPrice` |
| Prep SQL Task 2 | `Broker`, `BrokerCountySt`, `Carrier`, `CarrierHistory`, `CMSA` |
| Prep SQL Task 3 | `CountyState`, `MSAPMSA`, `naics`, `PlanType`, `Policy` |
| Prep SQL Task 4 | `PolicyBroker`, `PolicyBrokerKabot`, `Sponsor`, `States`, `PolicyModeledPremiums` — **also** copies `PolicyModeledPremiums` directly from linked server `[CINVSQL24.NYC.AMLAW.CORP].[bcmsgold_demoQA]` via `SELECT * INTO TempPolicyModeledPremiums` then `INSERT INTO PolicyModeledPremiums` |

### Data Flow Tasks (OLE DB bulk copy, `TABLOCK,CHECK_CONSTRAINTS`, `KeepIdentity=true`)

| Task | Tables copied (QA → Prod) |
|---|---|
| DFT 1 | `BCMSSummaryBroker`, `BCMSSummaryCarrier`, `BCMSSummarySponsor`, `BCMSSummaryTotal`, `BCPrice` |
| DFT 2 | `Broker`, `BrokerCountySt`, `Carrier`, `CarrierHistory`, `CMSA` |
| DFT 3 | `CountyState`, `MSAPMSA`, `naics`, `PlanType`, `Policy` |
| DFT 4 | `PolicyBroker`, `PolicyBrokerKabot`, `Sponsor`, `States` |

All data flows are direct table-to-table copies with no transformations.

### Final Confirmation Email (Script Task)

Runs after DFT 4 succeeds. Sends a completion notification via Amazon SES (SMTP) using the package variables below.

## Package Variables (SMTP / Email)

These are `User` namespace variables stored in the `.dtsx` and used by the Final Confirmation Email Script Task. Update them directly in the DTSX if SMTP credentials or recipients change.

| Variable | Current value |
|---|---|
| `SMTPServer` | `email-smtp.us-east-1.amazonaws.com` |
| `SMTPPort` | `587` |
| `SMTPUsername` | `AKIATN243CRFNIAN7LHU` (AWS SES IAM key) |
| `SMTPPassword` | Encoded value in DTSX (update if SES credentials rotate) |
| `EmailFrom` | `DatabaseEmail@arc-network.com` |
| `EmailTo` | `admin@alm.com` |
| `EmailCC` | `Eric.ryles@arc-network.com`, `Ron.Lubke@arc-network.com`, `HarShah@synoptek.com`, `HVaghasiya@synoptek.com`, `MBhavsar@synoptek.com`, `bhushah@synoptek.com` |

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

## Source Control

- **Remote:** `https://github.com/Touchpoint-Markets/BCMSDataCycle_SSIS_MoveQAToProd_Upgrade.git`
- **Default branch:** `development`
- **Ignored:** `bin/`, `.vs/`, `*.dtproj.user` (see `.gitignore`). The `obj/Development/BuildLog.xml` is intentionally tracked — SSDT uses it for protection-level consistency checks.
