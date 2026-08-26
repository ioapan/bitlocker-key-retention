# Solution Design Document — BitLocker Recovery Key Backup & Retrieval

**Reference Implementation — Intune / Endpoint Management**

| Field | Value |
|-------|-------|
| Document version | 4.0 |
| Date | v4.0 |
| Author | Ioannis Pantzis |
| Classification | Public — Reference Implementation |
| Status | Final |

---

## Change history

| Version | Date | Change summary |
|---------|------|----------------|
| 1.0 | iter. 1 | Initial draft (Azure Automation based). |
| 2.0 | iter. 2 | Architecture changed from Azure Automation to Azure Functions (Flex Consumption) with VNet integration and storage private endpoint. Added Graph API permission breakdown, network architecture, an HTTP-triggered retrieval function, Application Insights, and a cost estimate. |
| 3.0 | iter. 3 | **Major simplification.** Removed the HTTP retrieval function, EasyAuth, the dedicated app registration, the app role, and token validation — retrieval became a manual, PIM-gated operation. Reduced to a single Timer-triggered export function. Replaced the storage private endpoint + VNet integration with a storage firewall + Entra RBAC model (private endpoint made optional). Confirmed all data stored in the Storage Account (Key Vault holds only the CMK). Changed hosting from Flex Consumption to Elastic Premium. |
| 4.0 | iter. 4 | **Final — agreed direction.** Incorporates the Platform Infrastructure team decisions: hosting on an **App Service (Dedicated) Plan**; **Private Endpoint connectivity confirmed required** (back into the baseline); **CMK, Entra RBAC, shared-key/SAS disabled, diagnostics confirmed**; complete recovery record stored as a monthly JSON snapshot with Key Vault holding only the CMK; provisioning split agreed (the platform team provisions the infrastructure, the implementation team delivers code/config/retrieval script). No change to the core architecture. |

---

## Table of contents

1. Overview & objectives
2. Design simplifications (what changed from v2.0 and why)
3. Architecture diagram
4. Storage access model (networking)
5. Component — monthly export function
6. Authentication & identity
7. Graph API permissions
8. Data model
9. Storage account design
10. Retrieval workflow (manual, PIM-gated)
11. Security controls summary
12. Configuration reference
13. Azure RBAC & PIM matrix
14. Monitoring & alerting
15. Operational procedures
16. Cost estimate
17. Open items & customer decisions
- Appendix A — Azure resource summary

---

## 1. Overview & objectives

### Problem statement

BitLocker recovery keys escrowed to Microsoft Entra ID are stored **against the device object**. When that device object is deleted, **the keys are permanently destroyed**: Entra ID has **no recycle bin for device objects** — soft delete applies only to users, Microsoft 365 groups, cloud security groups, application registrations, service principals, administrative units, Conditional Access policies, and named locations. All other object types, devices included, are hard deleted and cannot be restored by an administrator or by Microsoft.

The exposure differs by join type:

| Device population | Native fallback | Exposure |
|---|---|---|
| **Hybrid-joined** (tied to on-prem AD DS) | Keys can additionally be escrowed to **AD DS**, where the **AD Recycle Bin** applies | Recoverable — restoring a deleted computer object also restores its BitLocker recovery information, within `msDS-DeletedObjectLifetime` (commonly 180 days, extendable) |
| **Entra-only** (e.g. Autopilot-provisioned) | **None** — no on-prem computer object exists; Entra ID is the sole native store | **No fallback, no grace period** — deletion is immediate and irreversible |

In large Autopilot-based fleets the Entra-only population can run to **tens of thousands of workstations** with no native retention path.

Note that **Intune's stale-device cleanup rule does not destroy keys** — it removes only the Intune management record, leaving the Entra ID device object and its recovery keys intact. The single action that permanently destroys a key is deletion of the **Entra ID device object** itself, typically a manual, RBAC-controlled step during device disposal. Disposal is therefore the critical control point.

Meanwhile **MBAM**, the on-premises product that historically provided enterprise key escrow and retention, reached end of extended support on **14 April 2026**, with no successor providing long-term, device-independent retention.

Legal-retention requirements mandate that recovery keys remain **retained and retrievable for a defined period even after device decommissioning**. Two obvious archive targets are ruled out: a **log/analytics workspace** is not an appropriate store for cryptographic secrets (and cannot guarantee record immutability), and **one vault secret per key** does not scale to a large fleet or preserve searchability. This design therefore establishes a durable, encrypted, access-controlled archive that is independent of the Entra device lifecycle.

### Objectives

- Maintain a durable, encrypted, access-controlled archive of BitLocker recovery keys independent of the Entra device lifecycle.
- Support retrieval for legal/forensic/incident cases (~2–3 per month).
- Provide a full audit trail for every export and every retrieval.
- Avoid storing cryptographic secrets in Log Analytics (ruled out by Cloud Security) and avoid storing them as individual Key Vault secrets (see Section 9.3).
- Keep the solution as simple as possible: minimum components, minimum standing infrastructure, minimum operational overhead.

### Solution summary

A single Azure Function (App Service / Dedicated plan, PowerShell 7.4), Timer-triggered monthly, that:

1. Extracts all in-scope BitLocker recovery keys and device metadata via Microsoft Graph (read-only).
2. Writes one JSON snapshot per run to a dedicated, hardened Azure Blob Storage container.

Retrieval is performed on demand, manually, by an authorized operator who activates PIM for read access to the storage container and runs a small local PowerShell helper to search the latest snapshot. There is no standing retrieval API, no public HTTP endpoint, and no app registration to manage.

---

## 2. Design simplifications (what changed from v2.0 and why)

| v2.0 component | v3.0 / v4.0 decision | Rationale |
|----------------|----------------------|-----------|
| HTTP retrieval function (`Get-BitLockerKey`) | **Removed** | ~2–3 retrievals/month do not justify a standing authenticated API. |
| EasyAuth / App Service Authentication | **Removed** | No HTTP endpoint to protect anymore. |
| Dedicated app registration + app role (`KeyRetriever`) + token validation | **Removed** | Access is now controlled by Azure RBAC + PIM directly on the container. |
| Storage private endpoint + VNet integration + delegated subnet + private DNS zone | **Confirmed required** (the Platform Infrastructure team, the final design review) | Storage is provisioned behind a Private Endpoint per the customer's secure connectivity pattern (Section 4). |
| Two functions in one Function App | **One function** (export only) | Only the export function remains. |
| Application Insights as the retrieval audit plane | **Retained but reduced** (export telemetry only) | Retrieval audit now comes from PIM logs + storage diagnostics. |

**Net effect:** roughly half the Azure resources, no app registration lifecycle, and a single code artifact to maintain — with the same legal-retention outcome and an equal-or-stronger audit trail for retrieval (PIM + storage diagnostics + ITSM).

---

## 3. Architecture diagram

```
  MONTHLY EXPORT (Timer Trigger)                 MANUAL RETRIEVAL (on demand)
  ====================================           =============================

  Azure Function (App Service Plan)               Authorized Operator
  PowerShell 7.4 + VNet integration              (local PowerShell session)
         |                                              |
         | Timer: 0 0 2 1 * *                           | 1. Open ITSM ticket
         | (1st of month, 02:00 UTC)                    | 2. Activate PIM:
         | System Managed Identity                      |    Storage Blob Data
         |                                              |    Reader (container)
    +----+---------------------+                        | 3. Connect-AzAccount
    |  Microsoft Graph API     |                        |    (interactive)
    |  (read-only)             |                        v
    |  GET /devices            |                  +--------------------------+
    |  GET /informationProt... |                  |  Azure Blob Storage      |
    |     /bitlocker/recovery  |                  |  (Private Endpoint +     |
    |  GET /deviceManagement/  |                  |   firewall + RBAC)       |
    |     managedDevices       |                  |                          |
    |  GET /directory/deleted  | (optional)       |  Download latest         |
    |  GET /users              |                  |  snapshot JSON           |
    |  GET /groups/.../members | (optional)       |  Search by identifier    |
    +----+---------------------+                  |  (local helper script)   |
         |                                        +--------------------------+
         v                                              |
    +--------------------------+                  Recovery key released via
    |  Azure Blob Storage      |                  controlled channel; PIM
    |  (Private Endpoint +     |                  access expires automatically
    |   firewall + RBAC)       |
    |  Write snapshot JSON     |                  +--------------------------+
    |  - Public access locked  |                  |  Audit Trail             |
    |  - Entra RBAC only       |                  |  - ITSM ticket           |
    |  - CMK (Key Vault)       |                  |  - PIM activation log    |
    |  - Retention safeguards  |                  |  - Storage diagnostics   |
    +----+---------------------+                  +--------------------------+
         |
         v
    Storage Diagnostics ------> Log Analytics / Sentinel
```

**Data flow notes:**

- Graph API calls go to public Microsoft endpoints (read-only).
- Storage access (export write, retrieval read) is over a Private Endpoint and authenticated by Entra ID RBAC only; shared keys and SAS are disabled.
- The Function reaches the storage Private Endpoint via regional VNet integration on the App Service Plan.
- Application Insights captures export telemetry (counts, runId, errors).

---

## 4. Storage access model (networking)

### 4.1 Access model (agreed)

The storage account is protected by a defense-in-depth combination of private networking, identity, encryption, and retention safeguards. the Platform Infrastructure team confirmed (the final design review) that the storage account will be provisioned behind a Private Endpoint per their secure connectivity pattern, in addition to the identity and encryption controls below:

| Setting | Value |
|---------|-------|
| Shared key access | Disabled |
| SAS token generation | Disabled |
| Auth method | Entra ID RBAC only |
| Public network access | Disabled |
| Private endpoint | Enabled — blob sub-resource. The Function reaches it via regional VNet integration on the App Service Plan. |
| VNet integration | Enabled — Function App regional VNet integration via a delegated subnet (`Microsoft.Web/serverFarms`, `/26` minimum) |
| Private DNS zone | `privatelink.blob.core.windows.net`, linked to the VNet |
| App setting | `WEBSITE_VNET_ROUTE_ALL = 1` |
| Storage firewall | Enabled — backs the private endpoint; no public network paths are permitted |

**Why this is sufficient:** traffic to the storage account never traverses a public endpoint; the data is encrypted at rest with a customer-managed key (CMK); retention safeguards protect against deletion (Section 9.4); all access is gated by Entra ID RBAC (no anonymous or key-based access is possible); and every access is logged to storage diagnostics.

### 4.2 Provisioning responsibility

the Platform Infrastructure team provisions the Private Endpoint, VNet, delegated subnet, Private DNS zone, the storage account, the CMK / Key Vault, and the App Service Plan, per their secure connectivity pattern. The implementation team provides the Function code, the Function App configuration (including VNet integration and `WEBSITE_VNET_ROUTE_ALL`), and the retrieval helper script. The Bicep/IaC generated from this design is reference material for that build.

### 4.3 Provenance note — private endpoint

The storage private endpoint originated as a design recommendation (initial recommendation email and v1.0 design). It was made optional in v3.0 pending confirmation of a the customer private-connectivity policy. the Platform Infrastructure team has since confirmed (the final design review) that the resources will be provisioned behind a Private Endpoint, so private networking is part of the agreed baseline in this version.

---

## 5. Component — monthly export function (`Export-BitLockerKeys`)

### 5.1 Purpose

Perform a full monthly snapshot of all in-scope BitLocker recovery keys and associated device/user metadata, and upload the result to Azure Blob Storage.

### 5.2 Execution model

- **Platform:** Azure Functions on an App Service (Dedicated) Plan. the Platform Infrastructure team confirmed (the final design review) that the App Service Plan is the qualified hosting option. The plan must support regional VNet integration (Basic/B1 and higher) so the Function can reach the storage Private Endpoint, and **"Always On" MUST be enabled** so the Functions runtime stays warm and the timer trigger fires reliably. `functionTimeout` can be set to unbounded (`-1`) on a Dedicated plan, so there is no execution-time limit. the customer will work to qualify Flex Consumption / Elastic Premium separately; either can be adopted later with no code change (Flex would add scale-to-zero cost savings).
- **Runtime:** PowerShell 7.4
- **Trigger:** Timer trigger (CRON `0 0 2 1 * *` = 1st of month, 02:00 UTC)
- **Identity:** Function App system-assigned Managed Identity
- **Dependencies:** declared in `requirements.psd1` (managed dependencies): `Microsoft.Graph.Authentication`, `Az.Accounts`, `Az.Storage`. All Graph calls are made through `Invoke-MgGraphRequest` against the REST API, so the heavy per-workload Graph SDK sub-modules are not required — this keeps cold starts fast.

**Note on `requirements.psd1`:** this is the managed-dependencies file for Azure Functions PowerShell projects. It declares which modules and version ranges the Function App needs; Azure Functions downloads and installs them at deployment time. No manual module import is required.

### 5.3 Processing flow

1. **Authenticate** — `Connect-MgGraph -Identity`; `Connect-AzAccount -Identity`.
2. **Enumerate devices** — if `SCOPE_GROUP_ID` is configured, `GET /groups/{id}/members` (devices); else `GET /devices` (all Entra device objects, paginated). If `LOCATION_FILTER` is configured (e.g. `US`), resolve the registered owner and filter on `usageLocation`.
3. **Enrich device data** (per device, with try/catch):
   - Entra ID: `deviceId`, `displayName`, `operatingSystem`, `trustType`.
   - Intune: `managedDeviceId`, `serialNumber`, `userPrincipalName` via `GET /deviceManagement/managedDevices?$filter=azureADDeviceId eq '{deviceId}'`.
   - BitLocker: all recovery keys via `GET /informationProtection/bitlocker/recoveryKeys?$filter=deviceId eq '{deviceId}'`, then `GET .../recoveryKeys/{keyId}?$select=key`. Captures `keyId`, `volumeType`, `createdDateTime`, `recoveryKey`.
   - Owner: `usageLocation` (region), cached from step 2.
   Per-device errors are caught and logged; the device is still included with partial data and an entry in the errors array. Processing continues.
4. **Scan deleted devices** (optional, `INCLUDE_DELETED_DEVICES`) — `GET /directory/deletedItems/microsoft.graph.device`; apply the same location filter; capture `deletedDateTime`; attempt key retrieval (may 404 if purged); mark `isDeleted = true`.
5. **Build output** — generate a unique `runId` (GUID); wrap records in the run envelope (Section 8); `ConvertTo-Json`.
6. **Upload to blob storage** — blob name `BitLockerBackup_YYYY-MM_Full_<RunId>.json`; authenticate via Managed Identity (no shared keys, no SAS); upload via `Set-AzStorageBlobContent`.
7. **Summary & logging** — write processed/captured/error counts and duration; emit structured telemetry to Application Insights; throw on critical failure (auth, storage upload) so the run is marked failed.

### 5.4 Performance considerations

- Uses `[System.Collections.Generic.List[object]]` (not `+=` array concat) for O(1) append at scale.
- Devices with no keys are included with an empty `keys` array.
- Graph calls use pagination (`@odata.nextLink`).
- An App Service (Dedicated) plan with "Always On" imposes no hard execution-time limit (`functionTimeout` is configurable; `-1` = unbounded), so large tenants are not constrained by a timeout. For very large tenants, consider batching to bound memory.

---

## 6. Authentication & identity

### 6.1 Function App Managed Identity

The export function uses the Function App's system-assigned Managed Identity. This eliminates client secrets, certificates, and shared keys/SAS tokens. The Managed Identity is granted:

- Microsoft Graph application permissions (read-only — Section 7), via app role assignment to the Managed Identity's service principal.
- Storage Blob Data Contributor on the target container (to write snapshots).

### 6.2 No interactive accounts in the export path

The export runs entirely non-interactively via the Timer trigger. The only interactive authentication in the solution is the operator's PIM-gated, on-demand retrieval (Section 10).

---

## 7. Graph API permissions

All permissions are **APPLICATION** type (not delegated), require admin consent, and are granted via app role assignment to the Function App Managed Identity's service principal. No write permissions are requested.

| Permission | Conditional? | Purpose |
|------------|--------------|---------|
| `BitLockerKey.Read.All` | No | Read recovery keys (incl. key value). Every key read is auto-audited in Entra. |
| `Device.Read.All` | No | Enumerate Entra devices and resolve owners. |
| `DeviceManagementManagedDevices.Read.All` | No | Intune enrichment (serial, UPN). |
| `User.Read.All` | No | Owner `usageLocation` (region field). |
| `GroupMember.Read.All` | Yes (group) | Only if device scoping via Entra group is used. |
| `Directory.Read.All` | Yes (deleted) | Only if deleted-device scanning is enabled. |

**Least-privilege notes:**

- `BitLockerKey.ReadBasic.All` omits the key value; `.Read.All` is required because the backup must contain the actual recovery password.
- The two conditional permissions can be omitted if their features are not used, reducing the consented surface to four read-only permissions.

---

## 8. Data model

### 8.1 Export file structure (JSON)

```json
{
  "runId"          : "a3f7c2e1-...",
  "runTimestamp"   : "2026-06-01T02:00:00Z",
  "runType"        : "Full",
  "scriptVersion"  : "4.0",
  "tenantId"       : "<tenant-id>",
  "scopeGroupId"   : null,
  "locationFilter" : "US",
  "deviceCount"    : 12450,
  "keyCount"       : 14230,
  "errorCount"     : 3,
  "devices"        : [
    {
      "entraDeviceId"      : "d1a2b3c4-...",
      "intuneDeviceId"     : "e5f6a7b8-...",
      "hostname"           : "US-PC-12345",
      "serialNumber"       : "ABC123DEF",
      "primaryUPN"         : "jane.doe@contoso.com",
      "region"             : "US",
      "operatingSystem"    : "Windows",
      "trustType"          : "AzureAd",
      "isDeleted"          : false,
      "deletedDateTime"    : null,
      "keys"               : [
        {
          "keyId"            : "k9l0m1n2-...",
          "volumeType"       : "osDrive",
          "createdDateTime"  : "2024-06-15T10:30:00Z",
          "recoveryKey"      : "123456-789012-345678-901234-567890-123456-789012-345678"
        }
      ]
    }
  ],
  "errors"         : [
    {
      "entraDeviceId"  : "x1y2z3...",
      "hostname"       : "US-PC-99999",
      "errorMessage"   : "BitLocker key retrieval returned 404",
      "timestamp"      : "2026-06-01T02:15:30Z"
    }
  ]
}
```

### 8.2 Field descriptions

- **Run-level:** `runId`, `runTimestamp`, `runType`, `scriptVersion`, `tenantId`, `scopeGroupId`, `locationFilter`, `deviceCount`, `keyCount`, `errorCount`.
- **Device:** `entraDeviceId`, `intuneDeviceId`, `hostname`, `serialNumber`, `primaryUPN`, `region`, `operatingSystem`, `trustType`, `isDeleted`, `deletedDateTime`.
- **Key:** `keyId`, `volumeType` (`osDrive` | `fixedDataDrive` | `unknownFutureValue`), `createdDateTime`, `recoveryKey` (48-digit).
- **Errors:** devices that could not be fully processed, with message and timestamp.

### 8.3 File naming convention

```
BitLockerBackup_<YYYY-MM>_<RunType>_<ShortRunId>.json
Example: BitLockerBackup_2026-06_Full_a3f7c2e1.json
```

One file per execution; `ShortRunId` prevents collisions for ad-hoc runs in the same month.

---

## 9. Storage account design

### 9.1 Dedicated storage account

A standalone Azure Storage Account used exclusively for BitLocker recovery key backups. No other workloads share this account. A single blob container holds all snapshot files.

### 9.2 Protection summary

| Setting | Value |
|---------|-------|
| Shared key access | Disabled |
| SAS token generation | Disabled |
| Auth method | Entra ID RBAC only |
| Public network access | Disabled — storage account fronted by a Private Endpoint (Section 4.1) |
| Encryption type | Customer-managed key (CMK) in Azure Key Vault |
| Key rotation | Managed by Cloud Security (existing policy) |
| Infrastructure encryption | Recommended (double encryption at rest) |
| Retention safeguard | Blob + container soft-delete (default; see 9.4), retention window aligned to the legal SLA |
| Immutability (WORM) | Optional, off by default (see 9.4) — enable only if Legal mandates tamper-proof hold |
| Diagnostics | Enabled → Log Analytics / Sentinel (metadata only — blob name, caller, timestamp; no key material in logs) |
| Sensitivity label | Container-level MIP label — **name to be confirmed** (e.g. "Highly Confidential — Encryption Keys — Legal Retention") |

### 9.3 Data architecture decision — all data in storage (not Key Vault)

**Decision:** the complete BitLocker recovery record (recovery password AND device/user metadata) is stored together in the Storage Account snapshot. Key Vault is used ONLY as the holder of the CMK that encrypts the storage account — it does NOT hold the recovery records themselves.

This addresses the storage-mechanism question raised in review. The complete-record-in-Key-Vault and the split (password-in-KV / metadata-in-storage) options were both evaluated and not selected, for the following reasons:

- **No metadata search in Key Vault.** Key Vault secrets are retrievable only by exact secret name; you cannot query "find the key for hostname X" or "for serial Y". Legal/incident retrieval is identifier-driven, so the searchable JSON snapshot in storage is the right structure. A split model would still require the storage snapshot for lookup, then a second hop to Key Vault — more moving parts for no benefit.
- **Weaker retention controls in Key Vault.** Key Vault offers soft-delete and purge protection (max 90 days), but not multi-year retention or optional time-based / legal-hold immutability. Storage supports both a long soft-delete retention window and, if Legal requires it, a WORM policy spanning years (Section 9.4).
- **Throttling and object-count at scale.** Tens of thousands of devices, each with one or more keys, mapped to individual Key Vault secrets would create a very large secret population and risk Key Vault transaction throttling on bulk operations. A single JSON snapshot per month avoids both.
- **Equivalent confidentiality.** Storage with CMK + Entra RBAC + disabled shared keys/SAS + retention safeguards + container MIP label provides protection equivalent to a vault for this data, while preserving searchability and long-term retention.

Cloud Security's original concern was specifically about Log Analytics being unsuitable for secrets of this sensitivity; that concern is fully satisfied by the hardened Storage Account above. If Cloud Security insists that the secret material itself must live in a vault, a fallback hybrid model (recovery password as a Key Vault secret keyed by `keyId`, metadata index in storage) can be added — but it is not recommended due to the search, retention, and throttling tradeoffs above.

### 9.4 Retention & immutability model

the customer's stated requirement is that BitLocker recovery keys remain **retained and retrievable** for a defined period after device deletion (a US legal-audit requirement). That requirement does not, by itself, mandate WORM immutability; WORM is one way to satisfy it and is the strongest (and least reversible) option. This design therefore separates the two:

**Default (recommended) — non-WORM retention:**

- Blob soft-delete and container soft-delete enabled, with a retention window aligned to the legal retention SLA (configurable; the IaC default is 365 days, the platform maximum for soft-delete).
- Entra RBAC grants operators READ only (Storage Blob Data Reader via PIM); no standing identity holds delete permission on the container.
- All write/delete operations are logged to Log Analytics / Sentinel.
- Backed by an operational retention SLA / procedure owned by the customer.

This meets the retention requirement while remaining fully reversible and operationally simple.

**Optional — WORM (write-once-read-many) immutability:**

- A time-based immutability policy on the container makes blobs undeletable and unmodifiable for the retention period.
- Enabled via the deployment parameter `enableWormImmutability` (default `false`). When enabled, the policy is created **unlocked**; locking it (which makes it irreversible for the full period, even by an administrator) is a deliberate, manual the Legal team step.
- Recommended ONLY if the Legal team requires tamper-proof, legally defensible immutability (e.g. for litigation hold). Note that soft-delete retention beyond the platform maximum, or guaranteed multi-year tamper-proofing, requires this WORM option.

**Provenance note:** WORM/immutability was a design recommendation, not an explicit customer requirement. the customer asked for retention and rejected Log Analytics partly for its "inability to overwrite records"; the specific mechanism (soft-delete vs WORM) and the retention duration are open items for the Legal team to confirm (Section 17).

---

## 10. Retrieval workflow (manual, PIM-gated)

Expected frequency: ~2–3 cases per month. There is no standing retrieval API; retrieval is a deliberate, audited, manual operation.

1. **Request initiation** — a legal/forensic/security incident ticket is opened in the ITSM tool, referencing a specific device, user, or case ID.
2. **Access elevation (PIM)** — an authorized member of the BitLocker operations group activates PIM for "Storage Blob Data Reader" scoped to the backup container. PIM requires justification (the ITSM ticket number) and may require approval per the customer policy. Access is time-limited and expires automatically.
3. **Retrieve and search** — the operator runs the retrieval script, `Get-BitLockerRecoveryKey.ps1` (generated from this design — see the generation guide). The script authenticates interactively with the operator's own PIM-elevated identity (`Connect-AzAccount`, `-UseConnectedAccount`; no keys, no SAS), lists snapshot blobs and selects the latest (or a specified month, or all months), downloads and parses the JSON snapshot **in memory** (never written to disk), searches `devices[]` by hostname, serial, deviceId, or UPN (case-insensitive), prints only the matching device record (including recovery key(s)), and requires a mandatory ITSM ticket reference (`-TicketId`).

   ```powershell
   .\Get-BitLockerRecoveryKey.ps1 `
       -StorageAccountName "<account>" `
       -ContainerName     "<container>" `
       -SearchBy          Hostname `
       -SearchValue       "US-PC-12345" `
       -TicketId          "INC0012345"
   ```

   Parameters: `-SearchBy` (Hostname | Serial | DeviceId | UPN), `-SearchValue` (case-insensitive), `-TicketId` (mandatory ITSM/legal reference), `-BackupMonth` (optional: `YYYY-MM`, `*` for all snapshots newest-first, or omit for latest only), `-TenantId` (optional). Prerequisites: PowerShell 7.x; `Az.Accounts` and `Az.Storage`; PIM activated before running.

4. **Key release** — the recovery key is provided to legal/forensics through a controlled channel. The key is NEVER pasted into ITSM comments, emailed, shared in chat, or stored in any secondary location.
5. **Closure & audit** — PIM access expires automatically. The audit trail consists of the ITSM ticket (justification, approval), the PIM activation log (who elevated, when, why), storage diagnostics (which identity read which blob, when), and the Entra audit log (from the original export key reads).

---

## 11. Security controls summary

| Control area | Implementation |
|--------------|----------------|
| Authentication | Managed Identity for export (no secrets) |
| Storage auth | Entra ID RBAC only (shared keys & SAS disabled) |
| Network (storage) | Private Endpoint + storage firewall + RBAC; no public network access (Section 4.1) |
| Encryption at rest | CMK via Key Vault (+ optional infrastructure encryption) |
| Encryption in transit | TLS 1.2+ (enforced by Azure) |
| Classification | MIP label at container level |
| Access control | Named Entra operations group only |
| Elevation | PIM with justification, time-limited |
| Audit (export keys) | Entra audit log — every Graph BitLocker key read |
| Audit (storage) | Storage diagnostics → Log Analytics / Sentinel |
| Audit (retrieval) | PIM activation log + storage diagnostics + ITSM ticket |
| Retention | Blob/container soft-delete (default) aligned to legal SLA; optional WORM immutability if Legal mandates it (Section 9.4) |
| Data minimization | Operator extracts only the matching device record |
| No standing API | No public HTTP endpoint, no app registration to manage |
| No secrets in code | All configuration via App Settings |

---

## 12. Configuration reference

All configuration is stored in the Function App's Application Settings. No secrets or keys are embedded in code.

| App setting | Type | Default | Description |
|-------------|------|---------|-------------|
| `TENANT_ID` | String | Required | Entra tenant ID |
| `STORAGE_ACCOUNT_NAME` | String | Required | Target storage account name |
| `CONTAINER_NAME` | String | Required | Target blob container name |
| `SCOPE_GROUP_ID` | String | (empty) | Entra group ID for device scoping. Empty = all devices |
| `LOCATION_FILTER` | String | (empty) | ISO country code, e.g. `US`. Empty = no location filter |
| `LOCATION_ATTRIBUTE` | String | `usageLocation` | Entra user attribute for location resolution |
| `INCLUDE_DELETED_DEVICES` | Boolean | `true` | Scan Entra recycle bin for recently deleted devices |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | String | Required | Application Insights connection (auto-set when AI linked) |

**Note:** `STORAGE_ACCOUNT_NAME` needs no connection string — the function authenticates to storage via Managed Identity. The operations-group ID is not an App Setting (retrieval auth is handled by Azure RBAC + PIM on the container, not by the function).

---

## 13. Azure RBAC & PIM matrix

### 13.1 Graph API application permissions (on Function App Managed Identity)

See Section 7. Read-only; four core permissions plus two conditional.

### 13.2 Azure RBAC on Function App Managed Identity

| Role | Scope |
|------|-------|
| Storage Blob Data Contributor | Target blob container only (write snapshots) |

### 13.3 Azure RBAC for the operations group (via PIM)

| Role | Scope | Activation |
|------|-------|-----------|
| Storage Blob Data Reader | Target blob container | PIM (time-bound, with justification) |

This single role replaces v2.0's app-role / EasyAuth mechanism. Operators elevate to read the container directly for retrieval.

### 13.4 Azure RBAC for platform admins

| Role | Scope |
|------|-------|
| Owner / Contributor | Storage account and/or Function App (admin operations only — separated from day-to-day key retrieval) |

---

## 14. Monitoring & alerting

### 14.1 Application Insights (export telemetry)

Captured automatically: function name, invocation ID, trigger type, start time, duration, success/failure, exceptions. Captured via custom telemetry: `deviceCount`, `keyCount`, `errorCount`, `runId`.

### 14.2 Recommended alerts (Azure Monitor)

| Alert | Condition | Severity |
|-------|-----------|----------|
| Export function failed | Result = failure on `Export-BitLockerKeys` | Critical |
| Export did not run | No successful invocation in the last 35 days | Warning |
| Export error count high | `errorCount` > threshold (e.g. 50) | Warning |
| Unexpected storage access | Sentinel rule: blob read by an identity other than the Function MI or an expected PIM-elevated user | High |

### 14.3 Log retention

Application Insights default retention is 90 days. For compliance, extend it or export to Log Analytics for long-term retention alongside storage diagnostics.

---

## 15. Operational procedures

### 15.1 Monthly export

Runs automatically via the Timer trigger on the 1st of each month at 02:00 UTC. The Intune admin team verifies success via the Function App Monitor blade or Application Insights, and reviews `deviceCount`, `keyCount`, and `errorCount`. Investigate if `errorCount` > 0 or `deviceCount` deviates significantly from prior months.

### 15.2 Ad-hoc export

Trigger on demand via the Azure Portal (Function App → Functions → `Export-BitLockerKeys` → Code + Test → Run) before a large device cleanup, after a major migration, or to re-run a failed scheduled export. The `RunId` in the filename prevents collisions with the scheduled run.

### 15.3 Retrieval

Follow Section 10. Always use a valid ITSM ticket ID, activate PIM, start with the latest snapshot, search older snapshots only if needed, and never share the recovery key via uncontrolled channels.

### 15.4 Module / dependency updates

Update version ranges in `requirements.psd1`, redeploy, and verify the export function runs correctly.

### 15.5 Function App maintenance

The App Service (Dedicated) plan is platform-managed (no OS/infrastructure patching). Monitor PowerShell runtime deprecation notices and plan runtime upgrades proactively. "Always On" must remain enabled so the timer trigger fires reliably; the plan therefore bills continuously rather than scaling to zero (see Section 16).

---

## 16. Cost estimate

| Component | Pricing model | Estimated monthly cost |
|-----------|---------------|------------------------|
| Azure Function (App Service Plan, Always On) | App Service / Dedicated (billed hourly per tier) | ~$13/month (B1); S1 ~$70; P1v3 ~$120+ |
| Storage Private Endpoint + Private DNS zone | Per endpoint-hour + DNS | ~$7/month + ~$0.50/month |
| Azure Storage Account (Blob) | Storage + transactions | ~$1–5/month (12 JSON files/year) |
| Application Insights | Per GB ingested (first 5 GB free) | ~$0–1/month |
| Azure Key Vault (CMK) | Per key + operations | ~$1–3/month (existing KV may apply) |
| **Estimated total (B1 plan)** | | **~$23–30/month** |

**Note on plan cost:** the App Service (Dedicated) plan bills continuously because "Always On" keeps the runtime warm (it does NOT scale to zero). B1 is the smallest tier that supports both regional VNet integration and Always On, and is sufficient for a once-monthly export; production may prefer S1 or P1v3 for deployment slots and a higher SLA. The plan tier is the main cost lever. Flex Consumption (once qualified) would scale to zero and reduce cost — the preferred future option. The legacy Consumption (Y1) plan is NOT suitable (no VNet, hard 10-minute execution limit).

---

## 17. Open items & customer decisions

The table below is the authoritative list of open decisions to confirm before deployment. Owners are shown as generic roles — map them to your own teams.

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Storage access model. **Agreed (final review):** Private Endpoint + firewall + Entra RBAC, no public network access (Section 4). | the Platform Infrastructure team | CLOSED |
| 2 | Confirm MIP classification label name and retention duration for the container | the Security team | OPEN |
| 3 | Confirm BitLocker operations group name, group ID, and membership (for PIM read access) | the Security team | OPEN |
| 4 | Provision or confirm storage account (subscription, resource group, name, region) | the Cloud Services team | OPEN |
| 5 | Approve Graph API permissions on Function App Managed Identity (Section 7) | the Security team / Cloud Services | OPEN |
| 6 | Confirm whether deleted-device scanning should be enabled by default | the Legal team | OPEN |
| 7 | Confirm any additional export fields needed beyond Section 8 | the Legal team / ISC | OPEN |
| 8 | Confirm PIM activation policy for Storage Blob Data Reader (approval, max duration) | the Cloud Services team | OPEN |
| 9 | Confirm Key Vault for CMK — existing or new, which subscription/resource group | the Cloud Services team | OPEN |
| 10 | Confirm retention model & duration: non-WORM soft-delete (default) vs WORM immutability, and the retention period (WORM was a Microsoft recommendation, not an explicit customer requirement — see 9.4) | the Legal team | OPEN |
| 11 | Confirm storage tier preference (Hot or Cool) | the Cloud Services team | OPEN |
| 12 | Confirm Azure subscription for the Function App deployment | the Cloud Services team | OPEN |
| 13 | Function App hosting plan. **Agreed (final review):** App Service (Dedicated) Plan is the qualified option; Flex Consumption / Elastic Premium to be qualified later (either can be adopted with no code change). See Section 5.2. | the Platform Infrastructure team | CLOSED |
| 14 | Confirm the App Service Plan SKU/tier. Design assumes B1 (smallest tier supporting VNet integration + Always On). Production may prefer S1 or P1v3 for slots / higher SLA. | the Platform Infrastructure team | OPEN |
| 15 | Confirm "Always On" will be enabled on the plan (required for the timer trigger to fire reliably). | the Platform Infrastructure team | OPEN |
| 16 | Confirm provisioning split: the platform team provisions storage, Private Endpoint, VNet/subnet, CMK/Key Vault, and the App Service Plan; the implementation team delivers the Function code, configuration, and retrieval script. | the Platform Infrastructure team | OPEN |

---

## Appendix A — Azure resource summary

| Resource | Purpose |
|----------|---------|
| Azure Function App (App Service / Dedicated plan, Always On enabled) | Hosts the single monthly export function |
| Function App Managed Identity (system-assigned) | Authenticates to Graph API and Storage |
| Azure Storage Account | Stores encrypted BitLocker backup snapshots |
| Blob Container (soft-delete retention) | Single container for all snapshot files; retention via blob/container soft-delete (WORM immutability optional — see 9.4) |
| Azure Key Vault | Hosts CMK for storage encryption |
| Private Endpoint (Storage) | Private network access to blob storage |
| Virtual Network | Hosts Function integration subnet + PE |
| Delegated Subnet | Function App outbound VNet integration |
| Private DNS Zone (`privatelink.blob.core.windows.net`) | Resolves storage FQDN to private IP |
| Application Insights | Export monitoring, logging, alerting |
| Log Analytics Workspace | Storage diagnostics, long-term audit logs |
| `Get-BitLockerRecoveryKey.ps1` (generated) | Operator-run retrieval script (local, PIM-gated). Replaces the v2.0 HTTP retrieval function. See Section 10. |

**Optional (only if Legal mandates tamper-proof legal hold — Section 9.4):**

| Resource | Purpose |
|----------|---------|
| Container WORM Immutability Policy | Time-based immutability policy on the backup container (`enableWormImmutability`) |

**Removed in v3.0 (vs v2.0):** HTTP retrieval function (`Get-BitLockerKey`); App Registration / EasyAuth identity provider; App role (`KeyRetriever`) and operations-group token validation.

---

*End of document.*
