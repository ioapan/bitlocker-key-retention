# How to Generate the Code with GitHub Copilot

This guide turns the **Solution Design v4.0** into working code, generated in your own
environment. The design document ([`docs/01_Solution-Design.md`](docs/01_Solution-Design.md))
is written as a build-grade specification — it contains the exact Microsoft Graph call
sequence, the JSON output schema, the permission set, and the networking model, so Copilot
has everything it needs to reproduce the solution faithfully.

> **The single most important step:** put the design document **into Copilot's context**
> before you run a prompt. Either open `docs/01_Solution-Design.md` in the editor and
> attach it in Copilot Chat, or reference it with `#file`. Do **not** rely on Copilot's
> general knowledge alone — the design is what makes the output match the agreed solution.

---

## Prerequisites

- **VS Code** with the **GitHub Copilot** + **Copilot Chat** extensions, and an active
  Copilot seat (sign in with a GitHub account to activate the seat — no code is hosted on
  GitHub).
- **Recommended model: Claude Opus 4.8 or higher.** Select it in the Copilot Chat model
  picker before running the prompt. A high-capability model produces markedly more faithful
  infrastructure and PowerShell than smaller/older models; lower-tier models are more likely
  to miss details (e.g. the two-step key read or the two-storage-account split) and need more
  iterations.
- This package available as a **local folder opened in VS Code** (a GitHub repository is
  optional — use one only if you want version control on the generated code).
- Reviewers available for a second technical look at the generated infrastructure code.

## Recommended workflow

1. Open `docs/01_Solution-Design.md` and keep it in context.
2. Run **the prompt** below to generate the whole solution in one pass — the infrastructure
   (Bicep + deploy scripts), the Azure Function, and the retrieval script. Review the output
   against Section 4 (networking), Appendix A (resource summary), and Sections 5, 7, and 8 of
   the design.
3. Iterate: if something looks off, ask Copilot to refine, quoting the relevant design
   section. **Expect a few iterations** — this is normal and expected.
4. **Test in a non-production environment.** Validate that the export produces a snapshot
   with real recovery-key values (see the "known pitfalls" note below), then validate
   retrieval.

---

## Generation only — this will not touch Azure

The prompt asks Copilot to **write files only**. It does **not** sign in to Azure, create or
deploy any resources, or publish anything — running it produces a local `code/` folder and
nothing else. Deployment happens later and deliberately, by an engineer following the
deployment steps in design Sections 4, 12, and Appendix A.

To keep it that way:

- Prefer Copilot Chat **Ask** mode for generation — it can only return code, it cannot run
  commands or reach Azure. If you use **Agent** mode, **do not approve any terminal command**
  (e.g. `az login`, `az deployment…`, `func … publish`) it may propose during generation.
- The prompt explicitly tells Copilot not to run commands, sign in, or deploy, and to treat
  resource `apiVersions` as a written note to verify manually rather than checking them
  against a live subscription.

> **Note on the generated scripts.** `Deploy-Solution.ps1` and `Grant-GraphPermissions.ps1`
> *do* contain Azure/Graph sign-in calls — but those run only when **you** later execute them
> at deployment time. Generating the files does not connect to anything.

### Expected output (the `code/` folder you will generate)

```
code/
  src/
    host.json
    profile.ps1
    requirements.psd1
    Export-BitLockerKeys/
      function.json
      run.ps1
  deploy/
    main.bicep
    main.parameters.json
    Deploy-Solution.ps1
    Grant-GraphPermissions.ps1
  Get-BitLockerRecoveryKey.ps1
```

The prompt produces all of these in one pass: **Part A** writes the `deploy/` files, **Part B**
writes the `src/` files and `Get-BitLockerRecoveryKey.ps1`.

> **None of these files ship with this repository.** This repo contains the specification and
> the generation workflow; the `code/` folder above is what *you* produce by running the
> prompt below in your own environment. That is deliberate — the code is generated against
> your tenant, your naming, and your networking, then reviewed by your engineers before it
> ever touches a subscription.

---

## The prompt — generate the full solution

Attach `docs/01_Solution-Design.md`, then paste this in one go. It produces the entire
`code/` folder above — infrastructure, deploy scripts, the Function, and the retrieval script.

> Using the attached **BitLocker Recovery Key Backup — Solution Design** as the
> authoritative spec, generate the **complete solution** as a `code/` folder with a `deploy/`
> subfolder and a `src/` subfolder. **Only write the files — do not run any commands, do not
> sign in to Azure or Graph, and do not deploy or publish.**
>
> ### Part A — Infrastructure (`deploy/`), following design Section 4 and Appendix A
>
> Generate **`main.bicep`** that deploys:
> - A **hardened backup Storage Account** (Standard_GRS, `allowSharedKeyAccess=false`,
>   `allowBlobPublicAccess=false`, TLS 1.2, `networkAcls.defaultAction=Deny`), with blob and
>   container **soft-delete retention** and an **optional WORM immutability** policy
>   (parameterised, default off).
> - A **separate runtime Storage Account** for `AzureWebJobsStorage` / content share where
>   shared-key access **is** allowed (Functions plumbing only — keep this distinct from the
>   hardened backup account).
> - An **App Service (Dedicated) plan, B1, Always On**, and a **PowerShell 7.4 Function App**
>   with a **system-assigned managed identity**, ready for regional VNet integration
>   (`WEBSITE_VNET_ROUTE_ALL=1`) to reach the backup storage over a Private Endpoint.
> - **Application Insights** and a **Log Analytics** workspace, with a **diagnostic setting**
>   on the backup account's blob service sending the `audit` log category to Log Analytics.
> - A **role assignment** granting the Function's managed identity **Storage Blob Data
>   Contributor** on the backup account only.
> - Parameters for: `namePrefix`, `tenantId`, `backupContainerName`,
>   `softDeleteRetentionDays`, `enableWormImmutability`, `immutabilityDays`, `scopeGroupId`,
>   `locationFilter`, `locationAttribute`, `includeDeletedDevices`, `allowedIpRules`,
>   `functionPlanSku` (allowed: `B1`, `S1`, `P1v3`).
> - Bicep **outputs**: `functionAppName`, `functionPrincipalId` (the managed identity object
>   ID), `backupStorageName`, and `backupContainerName`.
>
> Do **not** create the Private Endpoint, VNet/subnet, Private DNS zone, or CMK — those are
> provisioned separately by the the customer platform team (see design Section 4.2). For resource
> `apiVersions`, use recent stable versions and add a code comment listing them so we can
> verify them against our subscription manually.
>
> Also generate in `deploy/`:
> - A **`main.parameters.json`** parameters file listing every parameter above with sensible
>   placeholder/default values, ready to edit.
> - A **`Grant-GraphPermissions.ps1`** script that assigns the read-only Graph **application**
>   permissions to the Function's managed identity as app-role assignments on the Microsoft
>   Graph service principal, and is **idempotent** (skips already-assigned roles):
>   `BitLockerKey.Read.All`, `Device.Read.All`, `DeviceManagementManagedDevices.Read.All`,
>   `User.Read.All` always, plus `GroupMember.Read.All` only when a `-ScopeGroup` switch is
>   passed and `Directory.Read.All` only when `-IncludeDeleted` is passed. Parameters:
>   `-PrincipalId`, `-ScopeGroup`, `-IncludeDeleted`; use `Connect-MgGraph` and
>   `New-MgServicePrincipalAppRoleAssignment`.
> - A **`Deploy-Solution.ps1`** orchestrator that sets the subscription, creates the resource
>   group, deploys `main.bicep` with `main.parameters.json`, calls `Grant-GraphPermissions.ps1`
>   with the switches implied by the parameters, publishes the Function code from `../src`
>   using `func azure functionapp publish`, and prints the remaining manual tenant actions
>   (CMK, PIM, immutability lock, firewall). Parameters: `-SubscriptionId`,
>   `-ResourceGroupName`, `-Location`, `-ParametersFile`, `-SkipGraphGrant`, `-SkipCodePublish`.
>
> ### Part B — Function + retrieval script (`src/` + `Get-BitLockerRecoveryKey.ps1`), following design Sections 5, 6, 7, and the JSON envelope in Section 8
>
> 1. A **timer-triggered Azure Function** named `Export-BitLockerKeys` (CRON `0 0 2 1 * *`)
>    in `src/Export-BitLockerKeys/`, that authenticates with the **system-assigned managed
>    identity for both Microsoft Graph and Azure Storage** (`Connect-MgGraph -Identity`,
>    `Connect-AzAccount -Identity`, storage via `New-AzStorageContext -UseConnectedAccount` —
>    **no secrets, keys, or SAS**). It must:
>    - Enumerate devices — all Entra devices, or the members of `SCOPE_GROUP_ID` when set —
>      paging on `@odata.nextLink`.
>    - For each device, enrich from Intune via
>      `GET /deviceManagement/managedDevices?$filter=azureADDeviceId eq '{deviceId}'`, and read
>      BitLocker keys via
>      `GET /informationProtection/bitlocker/recoveryKeys?$filter=deviceId eq '{deviceId}'`
>      **then a second `GET .../recoveryKeys/{keyId}?$select=key`** to retrieve the actual key
>      material.
>    - Optionally scan `GET /directory/deletedItems/microsoft.graph.device` when
>      `INCLUDE_DELETED_DEVICES` is true, marking `isDeleted` and capturing `deletedDateTime`.
>    - Apply an optional `LOCATION_FILTER` on the registered owner's `usageLocation`.
>    - Catch per-device errors, record them in an `errors` array, and continue.
>    - Wrap results in the **exact run envelope schema from Section 8** (`runId`,
>      `runTimestamp`, `runType`, `scriptVersion`, `tenantId`, `scopeGroupId`,
>      `locationFilter`, `deviceCount`, `keyCount`, `errorCount`, `devices[]`, `errors[]`) and
>      upload one JSON blob named `BitLockerBackup_<YYYY-MM>_Full_<ShortRunId>.json` via
>      `Set-AzStorageBlobContent`.
>    - Read configuration from App Settings: `TENANT_ID`, `STORAGE_ACCOUNT_NAME`,
>      `CONTAINER_NAME`, `SCOPE_GROUP_ID`, `LOCATION_FILTER`, `LOCATION_ATTRIBUTE`,
>      `INCLUDE_DELETED_DEVICES`.
>    - Emit a telemetry summary line (`deviceCount`, `keyCount`, `errorCount`, `runId`) for
>      Application Insights.
>    - Include `src/host.json`, `src/requirements.psd1` (declaring
>      `Microsoft.Graph.Authentication`, `Az.Accounts`, `Az.Storage`), `src/profile.ps1`, and
>      the `function.json` timer binding.
> 2. A **PIM-gated operator retrieval script** (`Get-BitLockerRecoveryKey.ps1`, at the `code/`
>    root) that downloads the latest snapshot from the backup container and searches it by
>    **hostname, serial number, or deviceId**, returning the recovery key(s) for the matched
>    device.

---

## Known pitfalls to check during testing

These are the things a model most commonly gets wrong. The design already specifies the
correct behavior — verify the generated code matches:

- **Two-step key read.** BitLocker key values are **not** returned by the list call. The code
  must list keys by `deviceId`, then do a **second GET per key** with `?$select=key`. If your
  snapshot shows `null` recovery keys, this is why (design Section 5.3).
- **Two storage accounts.** The *runtime* account (Functions plumbing) keeps shared-key
  access **on**; the *backup* account keeps it **off**. Models often collapse these into one.
- **Same managed identity for Graph and Storage.** No client secret should appear anywhere.
- **Graph permissions are application-type, read-only** (design Section 7): `BitLockerKey.Read.All`
  (the `.Read.All`, not `.ReadBasic.All`), `Device.Read.All`,
  `DeviceManagementManagedDevices.Read.All`, `User.Read.All`, and conditionally
  `GroupMember.Read.All` / `Directory.Read.All`. These must be granted to the managed
  identity's service principal with admin consent.
- **Bicep `apiVersions`** — confirm they are valid in your subscription.

## Need help?

Open an issue on the repository if a design section is unclear or the generated code
diverges from it — that feedback also improves the specification for the next person.
