# BitLocker Recovery Key — Long-Term Backup & Retrieval

Retain **BitLocker recovery keys beyond the Entra ID device lifecycle**, so they stay
retrievable for legal, forensic, and incident needs **after a device object has been
deleted**.

A single Azure Function exports recovery keys from Microsoft Entra ID to a hardened,
encrypted archive on a schedule; retrieval is a deliberate, access-controlled, on-demand
operation.

**This repository contains the design and the build workflow — not pre-built code.** The
solution design is written as a build-grade specification, and the generation guide walks you
through producing the infrastructure, the Function, and the retrieval script from it in your
own environment. See [Scope](#scope--what-is-and-is-not-here).

---

## Background — why this exists

Windows escrows BitLocker recovery keys to **Microsoft Entra ID**, where each key is stored
**against the device object**. That works well for day-to-day unlock support. It breaks down
when an organization has to *retain* those keys.

### Entra ID device deletion is a hard delete

When a device object is deleted from Microsoft Entra ID, **its BitLocker recovery keys are
permanently destroyed**. There is **no recycle bin for device objects** — Entra ID supports
soft delete only for users, Microsoft 365 groups, cloud security groups, application
registrations, service principals, administrative units, Conditional Access policies, and
named locations. Every other object type, devices included, is **hard deleted and cannot be
restored** — by an administrator or by Microsoft.

A detail worth knowing, because it is widely misunderstood: **Intune's stale-device cleanup
rule does not destroy keys.** It removes only the Intune management record, leaving the Entra
ID device object — and its recovery keys — intact. The single action that permanently
destroys a recovery key is **deletion of the Entra ID device object itself**, typically a
manual, RBAC-controlled step in a device disposal process. That makes disposal the critical
control point.

### The gap depends on how devices are joined

| Device population | Native fallback | Exposure |
|---|---|---|
| **Hybrid-joined** (still tied to on-prem AD DS) | Can additionally escrow keys to **AD DS**, where the **AD Recycle Bin** applies | Recoverable — a deleted computer object restored from the Recycle Bin brings its BitLocker recovery information with it, within `msDS-DeletedObjectLifetime` (commonly 180 days, extendable) |
| **Entra-only** (e.g. Autopilot-provisioned) | **None.** No on-prem computer object exists, so Entra ID is the sole native store | **No fallback and no grace period** — delete the device object and the keys are gone |

For fleets built on Autopilot, this second row can cover **tens of thousands of workstations**
with no native retention path at all.

### The historical tool is gone

**MBAM (Microsoft BitLocker Administration and Monitoring)** — the on-premises product that
provided enterprise key escrow and retention — reached **end of extended support on 14 April
2026** (mainstream support ended 9 July 2019). It has no successor offering long-term,
device-independent retention; current guidance points to Entra/Intune escrow, which is exactly
the device-lifetime-bound model described above.

### Why a second archive, and why not the obvious targets

Regulated organizations frequently must keep recovery keys **retained and retrievable for a
defined period even after a device is decommissioned** — a multi-year legal or audit
obligation. Deleting a laptop should not destroy evidence you are required to keep.

Two tempting shortcuts do not survive security review:

- **A log/analytics workspace** is not an appropriate store for cryptographic secrets, and is
  commonly rejected on that basis (it also cannot guarantee record immutability).
- **One vault secret per key** does not scale to a large fleet and loses searchability.

**In short:** organizations with a legal-retention requirement have no supported, native way
to guarantee a BitLocker recovery key survives the deletion of its device. This project closes
that gap with a small, auditable, low-standing-footprint pattern, validated through a formal
enterprise security design review before implementation.

Retrieval in practice is **infrequent** — on the order of a few cases per month for legal,
forensic, or incident response — which is why the design favors a scheduled export plus a
manual, gated retrieval rather than a standing service.

> **Complementary control, not a replacement.** If you have hybrid-joined devices, also enable
> **AD DS escrow** for them (Group Policy: *Computer Configuration → Administrative Templates →
> Windows Components → BitLocker Drive Encryption → Operating System Drives → "Choose how
> BitLocker-protected operating system drives can be recovered"*, with *Store BitLocker
> recovery information in AD DS*), and use `Backup-BitLockerKeyProtector` to back-fill
> already-encrypted devices. Also tighten the disposal SOP: confirm the drive is wiped or
> destroyed **before** the Entra object is deleted, and **disable** rather than delete while
> disposition is pending. This archive is what covers the Entra-only population, which those
> controls cannot reach.

*Technical references: Microsoft Learn — "Recover from deletions in Microsoft Entra ID" (object
types supporting soft delete); Microsoft Lifecycle — MBAM 2.5 support dates; Microsoft Graph
`bitlockerRecoveryKey` API.*

---

## What it does

**Export (automated).** A single **Azure Function** (PowerShell 7.4, App Service / Dedicated
plan), Timer-triggered monthly:

1. Reads all in-scope BitLocker recovery keys and device metadata via **Microsoft Graph**
   (read-only): `GET /informationProtection/bitlocker/recoveryKeys/{id}?$select=key`.
2. Writes one **JSON snapshot per run** to a dedicated, hardened Azure Blob Storage container
   (customer-managed-key encryption, private endpoint, shared-key/SAS disabled).

**Retrieval (manual, gated).** No standing API, no public endpoint, no app registration. An
authorized operator activates just-in-time (PIM) read access to the container and runs a
small local PowerShell helper that searches the latest snapshot by hostname, serial, device
ID, or UPN, and prints only the matching record. The key is handled **in memory** and never
written to disk.

**Audit.** Every export key-read generates an **Entra KeyManagement audit log** (triggered by
`$select=key`); every retrieval is covered by the PIM activation log, storage diagnostics, and
a mandatory ITSM ticket reference.

---

## Architecture at a glance

```
 Microsoft Graph  --read-->  Azure Function (Timer, monthly)  --write-->  Hardened Blob Storage
 (recovery keys)             PowerShell 7.4, Managed Identity              - CMK (Key Vault)
                                                                           - Private Endpoint
 Operator  --PIM activate-->  local Get-BitLockerRecoveryKey.ps1  --read-->  - RBAC read-only
 (on demand, ITSM ticket)     (in-memory search, no key on disk)             - diagnostics
```

Full detail — networking model, JSON data schema, RBAC/PIM matrix, retention options
(soft-delete vs WORM immutability), and cost estimate — is in
**[docs/01_Solution-Design.md](docs/01_Solution-Design.md)**.

---

## Deploying it

The design document is written as a **build-grade specification** — exact Graph call
sequence, output schema, permission set, and networking model — precise enough to generate
the code from directly.

1. Read **[docs/01_Solution-Design.md](docs/01_Solution-Design.md)** for the full design.
2. Follow **[GENERATING-THE-CODE.md](GENERATING-THE-CODE.md)** to generate the Bicep, the
   Azure Function, and the retrieval script from the spec with GitHub Copilot, then review
   the output against the named design sections.
3. Provision the platform and deploy per the ownership split in
   **[docs/03_Responsibilities.md](docs/03_Responsibilities.md)**.
4. **Test in a non-production tenant first.** Read **[DISCLAIMER.md](DISCLAIMER.md)** before
   handling real recovery keys.

---

## Repository contents

| Path | What it is |
|------|-----------|
| `README.md` | This overview |
| `docs/01_Solution-Design.md` | Full build-grade solution design |
| `docs/03_Responsibilities.md` | Provisioning / ownership split (RACI) |
| `GENERATING-THE-CODE.md` | Generate the code from the design with GitHub Copilot |
| `DISCLAIMER.md` | Scope, ownership, and safe-use notes |
| `.gitignore` | Keeps generated code and any real snapshots out of version control |

### Scope — what is and is not here

**In this repository:** the solution design, the responsibilities split, the code-generation
guide, and safe-use notes.

**Not in this repository:** the Bicep templates, the Azure Function source, the deployment
scripts, and the `Get-BitLockerRecoveryKey.ps1` retrieval helper. Those are **generated from
the design** by following [GENERATING-THE-CODE.md](GENERATING-THE-CODE.md), and land in a
local `code/` folder that is deliberately git-ignored.

This is intentional. The generated code is shaped by your tenant, naming, networking, and
security baseline, and should be reviewed by your own engineers before deployment — a
pre-built drop would invite copy-paste into production. Documents referenced in the design
(deployment steps, operational procedures) are covered by the design's own sections rather
than shipped as separate files.

---

## Design decisions worth knowing

- **Manual, PIM-gated retrieval instead of a standing API.** With only a few retrievals per
  month, a permanent authenticated endpoint would add attack surface and lifecycle cost for
  no benefit. Access is granted just-in-time and expires automatically.
- **Storage + CMK + RBAC instead of a secrets vault.** Storing each key as an individual
  vault secret does not scale and loses searchability; a CMK-encrypted, access-controlled
  container with disabled shared keys/SAS provides equivalent protection while keeping keys
  searchable and retainable long-term.
- **Retention is separated from immutability.** Soft-delete retention is the default; WORM
  immutability is an opt-in flag for environments that require tamper-proof, legally
  defensible retention (litigation hold), locked only by a deliberate manual step.
- **Minimal standing footprint.** One function, one container, no app registration to
  manage — the same retention outcome with roughly half the moving parts.
