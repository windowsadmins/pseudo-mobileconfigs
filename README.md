# pseudo-mobileconfigs

Sample Windows configuration profiles authored as **"pseudo-mobileconfigs"** —
YAML files, modelled on the macOS `.mobileconfig` idea, that describe Intune
configuration as code. A pipeline reads each file and pushes it to Microsoft
Intune via the Microsoft Graph API as an **OMA-URI custom profile**, a
**Settings Catalog profile**, or a **Microsoft Store app**, then assigns the
result to Entra ID groups.

These are examples to learn from and adapt. Identifiers use the
`com.example.winadmins.*` namespace and hosts use `example.com`, so copy a
profile, point it at your own environment, and fill in the `REPLACE_WITH_…`
fields (secrets, certificates, tenant IDs) with your own values before
deploying.

## Layout

```
.
├── Assigned/        # profiles for individually-assigned (1:1) devices
├── Shared/          # profiles for shared / lab / multi-user devices
├── System/          # system-wide security, Defender, certificates, OS policy
├── Apps/            # app-specific preference profiles (Office, Teams, …)
└── api-examples/
    ├── azdevops/    # Azure DevOps pipeline that pushes profiles + apps to Intune
    └── github/      # GitHub Actions workflow that does the same
```

| Folder | Profiles | What they target |
|---|---|---|
| `Assigned/` | 19 | Per-user assigned devices — browser prefs, Windows Update rings, Hello, VPN, power |
| `Shared/` | 33 | Shared/lab devices — kiosk, profile retention, drive mapping, signage, Wi-Fi |
| `System/` | 19 | Defender, ASR rules, certificates, time zone, storage, Secure Boot |
| `Apps/` | 6 | Office, Outlook, Teams, Cimian, ReportMate client config |

## Profile YAML shape

Each profile is a single YAML document. Two shapes appear:

**OMA-URI custom profile** — a list of `omaSettings`, each mapping a CSP
OMA-URI to a typed value:

```yaml
displayName: "CimianPrefs"
identifier: "com.example.winadmins.CimianPrefs"
description: "Sets Cimian fallback CSP OMA-URI values."
version: "2025.08.18"
applicability:
  osEditions: ["Windows 11 Enterprise"]
  minVersion: "21H2"
omaSettings:
  - name: "Cimian Software Repository URL"
    dataType: "String"
    omaUri: "./Device/Vendor/MSFT/Policy/Config/Software/Cimian/Config/SoftwareRepoURL"
    value: "https://cimian.example.com/deployment"
```

**Settings Catalog profile** — a list of `settingsCatalog` entries keyed by the
Graph `definitionId`:

```yaml
displayName: "SecureBootCertUpdate"
identifier: "com.example.winadmins.SecureBootCertUpdate"
platform: "windows10"
settingsCatalog:
  - kind: "choice"
    definitionId: "device_vendor_msft_policy_config_secureboot_enablesecurebootcertificateupdates"
    value: "device_vendor_msft_policy_config_secureboot_enablesecurebootcertificateupdates_22852"
```

## Pipelines (`api-examples/`)

Both pipelines do the same thing — read the profile YAML, push it to Intune over
Microsoft Graph, and assign it to Entra groups — using **Workload Identity
Federation** (no stored client secret).

- `api-examples/azdevops/microsoft-graph-intune-entra.yml` — a full Azure DevOps
  pipeline with stages for OMA-URI profiles, Settings Catalog profiles, Windows
  Update profiles, Microsoft Store apps, group-membership sync, and assignments.
  It authenticates through an Azure service connection (`IntuneGraphConnection`)
  and reads its secrets from a variable group (`intune-pipeline-secrets`).
- `api-examples/github/microsoft-graph-intune-entra.yml` — a GitHub Actions
  workflow mirroring the same stages, authenticating with `azure/login` via an
  OIDC federated credential.

### Required Microsoft Graph application permissions

Grant these to the app/identity the pipeline runs as, and admin-consent them:

- `DeviceManagementConfiguration.ReadWrite.All` — manage configuration profiles
- `DeviceManagementApps.ReadWrite.All` — manage Microsoft Store apps
- `DeviceManagementServiceConfig.ReadWrite.All` — Settings Catalog operations
- `Group.Read.All` — resolve target groups for assignments

> The Azure DevOps sample is a complete, real-world pipeline (~3,900 lines of
> embedded PowerShell/Python). The GitHub Actions sample is a runnable skeleton
> that implements the OMA-URI push end-to-end and points to the AzDevOps sample
> for the full body construction of the remaining stages.

## Using these

1. Copy a profile, change `identifier` to your own reverse-DNS namespace.
2. Fill in every `REPLACE_WITH_…` field and swap `example.com` for your hosts.
3. Validate the CSP OMA-URI or Settings Catalog `definitionId` against the
   [Microsoft MDM / Policy CSP reference](https://learn.microsoft.com/en-us/windows/client-management/mdm/).
4. Wire the folder into one of the `api-examples/` pipelines.

## License

MIT — see [LICENSE](LICENSE).
