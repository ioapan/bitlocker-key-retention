# Provisioning & Responsibilities (RACI) — BitLocker Recovery Key Backup

**Solution Design v4.0 (Agreed Direction)** · Reference Implementation — Intune / Endpoint Management

This document defines who provisions, owns, and operates each part of the solution. It reflects a provisioning split in which **the platform team provisions the infrastructure, and the implementation team delivers the Function code, configuration, and retrieval script.** The Bicep generated from the solution design serves as reference IaC for the platform build. Adjust the roles below to your own organization.

Legend: **R** = Responsible (does the work) · **A** = Accountable (owns the outcome) · **C** = Consulted · **I** = Informed.

---

## 1. High-level split

| Area | Customer | This project |
|------|----------|-----------|
| Platform resources (storage, Private Endpoint, VNet/subnet, Private DNS, CMK/Key Vault, App Service Plan) | **Provision & own** | Consult / reference IaC |
| Function code, Function App configuration, retrieval script | Deploy in tenant | **Deliver** |
| Classification (MIP) label, retention model & duration | **Own** | Recommend |
| Operations group, PIM policy | **Own** | Recommend |
| Graph permission approval & admin consent | **Approve** | Provide grant script |
| Ongoing operations (monthly checks, retrieval) | **Own & run** | — |

---

## 2. Detailed RACI

| # | Task / artifact | Platform Infrastructure | Cloud Services | Security | Legal | This project |
|---|-----------------|------------------------------|-------------------------|-------------------------|----------------|-----------|
| 1 | Confirm target subscription & resource group | A/R | C | I | I | C |
| 2 | Provision dedicated Storage Account (hardened) | A/R | C | C | I | C |
| 3 | Provision Private Endpoint + VNet + delegated subnet + Private DNS zone | A/R | C | C | I | C |
| 4 | Provision / assign CMK Key Vault + key | A/R | C | C | I | C |
| 5 | Provision App Service (Dedicated) Plan (Always On) | A/R | C | I | I | C |
| 6 | Choose App Service Plan SKU (B1/S1/P1v3) | A/R | C | I | I | C |
| 7 | Deploy Function App + config (VNet integration, App Settings) | C | A/R | I | I | R |
| 8 | Deliver Function code + retrieval script + reference Bicep | I | C | I | I | A/R |
| 9 | Approve Graph API permissions + admin consent | I | C | A/R | I | C |
| 10 | Grant Graph app roles to the Managed Identity | I | R | A | I | R (script) |
| 11 | Define MIP classification label + apply at container | I | C | A/R | C | C |
| 12 | Decide retention model (soft-delete vs WORM) & duration | I | C | C | A/R | C |
| 13 | Define BitLocker operations group + membership | I | C | A/R | C | I |
| 14 | Configure PIM policy for Storage Blob Data Reader | I | A/R | C | I | C |
| 15 | Enable CMK on the storage account (post-deploy) | R | A/R | C | I | C |
| 16 | Configure monitoring alerts (Azure Monitor / Sentinel) | C | A/R | C | I | C |
| 17 | Operate monthly export health checks | I | A/R | I | I | I |
| 18 | Perform key retrieval (per ticket) | I | C | A/R | C | I |
| 19 | Rotate any legacy PoC secrets | I | A/R | C | I | C |

> Team names above map to the roles referenced in the Solution Design Section 17. Adjust to the customer's actual org unit names as needed.

---

## 3. Provisioning sequence (recommended)

```
  1. the Cloud Services team confirm subscription + RG + naming/region.       (item 1)
  2. the Platform Infrastructure team provision:                                 (items 2–6)
        - Storage Account (hardened)
        - Private Endpoint + VNet + delegated subnet + Private DNS zone
        - CMK / Key Vault + key
        - App Service (Dedicated) Plan (Always On), chosen SKU
  3. Deploy Function App + config (project code, customer tenant).          (items 7–8)
  4. Grant Graph app roles to the MI + admin consent.                         (items 9–10)
  5. Enable CMK on the storage account.                                       (item 15)
  6. Apply MIP label; set retention model/duration.                          (items 11–12)
  7. Configure operations group + PIM policy.                                (items 13–14)
  8. Configure monitoring/alerts.                                            (item 16)
  9. Validate end-to-end (export + retrieval).
 10. Hand over to operations.                                               (items 17–18)
```

---

## 4. Implementation artifacts

The implementation side of the split consists of the following, all **generated from the
solution design** using the generation guide — they are not shipped as pre-built files:

- **Function code:** `code/src/` (`Export-BitLockerKeys` timer function, host/profile/requirements).
- **Retrieval script:** `code/Get-BitLockerRecoveryKey.ps1`.
- **Reference IaC:** `code/deploy/main.bicep` + `main.parameters.json`.
- **Deployment automation:** `code/deploy/Deploy-Solution.ps1`, `code/deploy/Grant-GraphPermissions.ps1`.

This repository provides the **design and the generation workflow**. The result is an
**organization-owned custom pattern**, not a packaged product: once generated and deployed,
the resources and their operation are owned by the adopting organization.

---

## 5. Ownership after handover

| Function | Owner |
|----------|-------|
| Azure resources (cost, lifecycle) | the Cloud Services team / Cloud Infrastructure |
| Security controls & classification | the Security team |
| Retention policy & legal hold | the Legal team |
| Day-to-day export operations | the Intune admin team |
| Key retrieval | the BitLocker operations group |
