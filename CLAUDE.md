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
| `Project.params` | Project-level parameters: `SmtpServer`, `SmtpPort`, `AwsSecretName`, `AwsRegion` (see "Project Parameters" and "AWS Secrets Manager Credential Retrieval" below); package-level connection parameters are declared inside the `.dtproj` manifest |
| `BCMSDataCycle_SSIS_MoveQAToProd.slnx` | Solution file (new `.slnx` format); one configuration: `Development` |
| `obj/Development/BuildLog.xml` | Tracks last-known protection levels for the project and each package — SSDT uses this to detect consistency drift |
| `Imports/AWSSDK.Core.dll`, `Imports/AWSSDK.SecretsManager.dll` | AWS SDK DLLs (3.7.500, net45) used by the Final Confirmation Email Script Task to call AWS Secrets Manager at runtime — see "AWS Secrets Manager Credential Retrieval" below |

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

Runs after DFT 4 succeeds. Reads `$Project::SmtpServer`, `$Project::SmtpPort`, `$Project::AwsSecretName`, `$Project::AwsRegion`, `User::ImportsPath`, `User::EmailFrom`, `User::EmailTo`, `User::EmailCC`. SMTP credentials (`UserName`/`Password`) are retrieved from AWS Secrets Manager at runtime — see "AWS Secrets Manager Credential Retrieval" below — not stored in the project. Sends a completion notification via Amazon SES (SMTP). `<BinaryItem>` was removed when this task was converted — **needs an SSDT rebuild before deploying**.

## Package Variables (Email)

These are `User` namespace variables stored in the `.dtsx` and used by the Final Confirmation Email Script Task.

| Variable | Current value |
|---|---|
| `ImportsPath` | `I:\Git Solutions\BCMSDataCycle_SSIS_MoveQAToProd_Upgrade\Imports` — path to the `Imports\` folder containing the AWS SDK DLLs, read in `Main()` via the `_importsDir` static field before the AWS Secrets Manager call |
| `EmailFrom` | `DatabaseEmail@arc-network.com` |
| `EmailTo` | `admin@alm.com` |
| `EmailCC` | `Eric.ryles@arc-network.com`, `Ron.Lubke@arc-network.com`, `HarShah@synoptek.com`, `HVaghasiya@synoptek.com`, `MBhavsar@synoptek.com`, `bhushah@synoptek.com` |

`SMTPServer`, `SMTPPort`, `SMTPUsername`, and `SMTPPassword` package variables were **removed**. `SmtpServer`/`SmtpPort` moved to project parameters (see "Project Parameters" below); `SMTPUsername`/`SMTPPassword` (a plaintext AWS SES IAM key/password checked into git) were replaced entirely by an AWS Secrets Manager lookup — see "AWS Secrets Manager Credential Retrieval" below.

## Project Parameters

Project-level parameters are distinct from package variables — they are set once in `Project.params` (or overridden per SSIS Catalog environment) instead of being duplicated as package variables.

| Parameter | Default Value | Purpose |
|---|---|---|
| `SmtpServer` | `email-smtp.us-east-1.amazonaws.com` | AWS SES SMTP server, used by the Final Confirmation Email Script Task |
| `SmtpPort` | `587` (Int32) | AWS SES SMTP port |
| `AwsSecretName` | `JudyDiamond-SMTP` | Name of the AWS Secrets Manager secret holding the SMTP `UserName`/`Password` JSON — see "AWS Secrets Manager Credential Retrieval" |
| `AwsRegion` | `us-east-1` | AWS region of the `AwsSecretName` secret in Secrets Manager |

## AWS Secrets Manager Credential Retrieval

SMTP credentials are not stored anywhere in the project. The Final Confirmation Email Script Task fetches them at runtime from AWS Secrets Manager, named by `$Project::AwsSecretName` (default `JudyDiamond-SMTP`) in region `$Project::AwsRegion` (default `us-east-1`). The secret value is JSON: `{"Host":"...","Port":587,"UserName":"...","Password":"...","EnableSsl":true}` — only `UserName`/`Password` are used; `Host`/`Port`/`EnableSsl` are ignored (the project keeps `$Project::SmtpServer`/`$Project::SmtpPort` as-is).

The Script Task embeds two private static helpers:

```csharp
private static string GetSecretString(string secretName, string region)
{
    using (var client = new Amazon.SecretsManager.AmazonSecretsManagerClient(Amazon.RegionEndpoint.GetBySystemName(region)))
    {
        var response = client.GetSecretValue(new Amazon.SecretsManager.Model.GetSecretValueRequest { SecretId = secretName });
        return response.SecretString;
    }
}

private static string ExtractJsonStringField(string json, string fieldName)
{
    var match = System.Text.RegularExpressions.Regex.Match(json, "\"" + fieldName + "\"\\s*:\\s*\"((?:[^\"\\\\]|\\\\.)*)\"");
    if (!match.Success)
        throw new System.Exception("Field '" + fieldName + "' not found in secret.");
    return match.Groups[1].Value.Replace("\\\"", "\"").Replace("\\\\", "\\");
}
```

`ExtractJsonStringField` is a hand-rolled regex extractor, not a JSON library — avoids adding a JSON parser dependency for a single task.

**AWS credentials for calling Secrets Manager**: `AmazonSecretsManagerClient(RegionEndpoint)` uses the AWS SDK's default credential provider chain (environment variables, `~/.aws/credentials`, or — the expected case here — an IAM role/instance profile attached to the machine running the SSIS Catalog). No access key/secret is stored anywhere in the project; the server must have `secretsmanager:GetSecretValue` permission on the `JudyDiamond-SMTP` secret via its IAM role.

**Assembly loading**: `AWSSDK.Core.dll`/`AWSSDK.SecretsManager.dll` (3.7.500, net45 — self-contained, no further dependencies) live in `Imports\` and are loaded via an `AssemblyResolve` handler registered in `static ScriptMain()`, keyed off a `_importsDir` static field populated from `User::ImportsPath` at the top of `Main()` (before `sendMail()`/the AWS calls run) — the resolver probes `_importsDir`, then `%ProgramFiles%\Microsoft SQL Server\160\DTS\Tasks`, then the NuGet cache (`%NUGET_PACKAGES%` or `%USERPROFILE%\.nuget\packages\awssdk.core\3.7.500\lib\net45\...`) as a dev-machine fallback. The embedded `.csproj` references both DLLs via `HintPath` into `Imports\`.

**On a deployment server**, update the `User::ImportsPath` package variable to the path where the DLLs are placed on that server (or copy them to `C:\Program Files\Microsoft SQL Server\160\DTS\Tasks\`, which requires admin).

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
