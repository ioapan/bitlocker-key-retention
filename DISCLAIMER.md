# Disclaimer & Safe Use

## Scope

This repository is a **reference implementation** of a pattern for retaining BitLocker
recovery keys beyond the Entra ID device lifecycle. It is design documentation, an
AI-assisted code-generation workflow, and reference infrastructure — **not** an official,
supported product. Generate the code, review it, and adapt it to your own tenant, networking,
and security baseline before use.

## Handling recovery keys — read this

BitLocker recovery keys are **high-value secrets**. If you build on this pattern:

- **Test in a non-production tenant first**, with test devices only.
- **Never write recovery keys to logs, Log Analytics, or disk.** The retrieval helper
  searches snapshots **in memory** by design.
- **Keep the archive private** — customer-managed-key encryption, private endpoint, and
  shared-key/SAS disabled on the storage account.
- **Gate retrieval behind just-in-time (PIM) access** with a mandatory ticket reference, and
  grant operators **read-only** storage access.
- **Grant the Function's managed identity read-only Graph permissions** only
  (`BitlockerKey.Read.All`).

## Ownership

Retention duration, WORM/immutability, and litigation hold are decisions your Legal and
Security teams must own. Nothing here is legal advice.

## Use and distribution

Shared for internal review and reuse. Provided "as is", without warranty of any kind. You are
responsible for reviewing and securing anything you deploy from it.
