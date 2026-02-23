---
title: Understanding UEFI firmware analysis capabilities and limitations
description: Learn what UEFI analysis capabilities and limitations exists.
author: karengu0
ms.author: karenguo
ms.topic: conceptual
ms.date: 02/19/2026
ms.service: azure
---

# UEFI firmware analysis capabilities

Firmware analysis can analyze UEFI firmware images and surface detected components, weaknesses, and selected binary attributes. This capability is currently provided in preview to gather customer feedback.
UEFI firmware differs from other firmware types in structure and content. As a result, some analysis results may vary across binaries or appear incomplete.
This article explains what firmware analysis currently supports for UEFI firmware and how to interpret the results.

> [!NOTE]
> Because UEFI analysis is in *preview*, not all firmware components, attributes, or weaknesses may be detected. Results should be interpreted as signals, not guarantees of vulnerability or protection.

# What is UEFI firmware?

UEFI (Unified Extensible Firmware Interface) firmware is system firmware used to initialize hardware and boot an operating system.
Many modern servers, PCs, and virtual machines use UEFI firmware instead of legacy BIOS.

# UEFI firmware contents

A single UEFI firmware image can contain:
* UEFI specific binaries
* Other executable formats embedded within the firmware
Because of this, firmware analysis results may include a mix of different executable types within the same analysis.

# UEFI SBOM detection

Firmware analysis can extract SBOM (Software Bill of Materials) information from UEFI firmware images.

## What is currently supported
* Detection of OpenSSL components embedded in UEFI firmware
* Detection of OpenSSL version information when available
* CVE association for OpenSSL when a version can be determined

> [!NOTE]
> SBOM coverage for UEFI firmware is currently limited
> Not all software components embedded in UEFI firmware can be detected
> Absence of a component does not mean it is not present

## Weakness detection for UEFI firmware
Weakness data for UEFI firmware is derived from detected SBOM components.
This means:
* CVEs may appear only for components that can be confidently identified
* Weakness results are signals, not verification of exploitability

## Binary hardening attributes for UEFI firmware
Binary hardening attributes reflect security properties detected from executable metadata.
### Supported today
* NX (NoExecute / DEP) is the supported binary hardening attribute for UEFI firmware (in Preview)
* Cryptographic keys and certificates remain fully supported and are in GA 
### Limitations
Other binary hardening attributes such as PIE, RELRO, or Stripped may appear in the results grid. These values may originate from:
* Non UEFI executables embedded in the firmware
* Generic executable analysis that does not apply uniformly to UEFI
These attributes are not considered reliable for UEFI firmware interpretation currently.

## Interpreting missing or partial data
UEFI firmware analysis relies on metadata that can be extracted from firmware binaries.
If metadata is unavailable or not applicable:
* Some fields may appear empty
* Some columns may not apply to all rows
Missing values should be interpreted as unknown, not as absence of a security feature.

# Summary
When reviewing UEFI firmware analysis results:
* Expect limited SBOM coverage
* Treat weakness results as indicative signals
* Rely on NX/DEP for UEFI binary hardening interpretation
* Interpret other binary attributes with caution
UEFI analysis capabilities may expand over time, and documentation will be updated accordingly.

