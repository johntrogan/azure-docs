---
title: Understanding weaknesses data in firmware analysis
description: Learn what the weaknesses data are in the CVE view of the firmware analysis results.
author: karengu0
ms.author: karenguo
ms.topic: conceptual
ms.date: 02/19/2026
ms.service: azure
---

# Understanding weaknesses data in firmware analysis

Firmware analysis surfaces weaknesses detected in firmware components extracted during analysis. These signals help you understand potential security risks, but they should be interpreted carefully and in context.
This article explains the weakness related fields you may see in firmware analysis results and how they relate to each other.

> [!NOTE]
> The presence of a weakness or CVE in firmware analysis does not necessarily mean a device is vulnerable. Actual impact depends on how the affected component is used within the system.

# Weakness signals in firmware analysis

Firmware analysis may enrich findings with multiple industry standard signals. Each signal represents a different aspect of risk and should not be interpreted in isolation.

## Common Vulnerabilities and Exposures (CVE)

A CVE is a publicly disclosed identifier for a known security vulnerability.
Firmware analysis associates CVEs with extracted firmware components when a match is identified.
A single firmware component may be associated with multiple CVEs, and a single CVE may appear across multiple devices or components.

## CVSS scores and versions

Firmware analysis may display Common Vulnerability Scoring System (CVSS) data for a CVE.
Multiple CVSS versions can appear for the same CVE:
* CVSS v2 – legacy scoring used by older vulnerabilities
* CVSS v3 – widely adopted standard with improved metrics
* CVSS v4 – newer version that introduces additional dimensions
The presence of multiple versions reflects how vulnerability scoring evolves over time rather than multiple distinct vulnerabilities.

## CVSS vector

In addition to a numeric score, CVSS includes a vector string that describes the factors contributing to the score, such as:
* Required access level
* Attack complexity
* Impact on confidentiality, integrity, and availability
The vector provides additional context such as conditions and impact factors that contribute to a CVE’s severity rating.

## CISA Known Exploited Vulnerabilities (KEV)

Some CVEs may be marked as part of the CISA Known Exploited Vulnerabilities (KEV) catalog.
This designation indicates that the vulnerability is known to be actively exploited in real-world scenarios.

> [!NOTE]
> - KEV status reflects observed exploitation activity, not whether a specific device is affected.
> - KEV status is currently a static value, reflecting the state of the Firmware analysis CVE database at the time the scan was conducted. This value is not updated dynamically. To view the most up-to-date KEV status, please re-scan your firmware image.

## Exploit Prediction Scoring System (EPSS)

Firmware analysis may include EPSS data, which estimates the likelihood that a vulnerability will be exploited.
Two related values may appear:
* EPSS score – an estimated likelihood of exploitation based on observed trends across the vulnerability ecosystem 
* EPSS percentile – how that probability compares relative to other vulnerabilities
These values provide comparative risk context but do not guarantee exploitation.

> [!NOTE]
> - EPSS value is currently a static value, reflecting the state of the Firmware analysis CVE database at the time the scan was conducted. This value is not updated dynamically. To view the most up-to-date EPSS status, please re-scan your firmware image.

## Exploit maturity

Exploit maturity describes the current state of exploit availability for a vulnerability, such as whether publicly known exploit techniques or code exist.
When present, exploit maturity information is typically surfaced alongside CVSS v4 scoring.

## Using weakness data together

Each weakness signal represents a different perspective:
* CVE identifies what the issue is
* CVSS describes technical severity
* KEV indicates known exploitation
* EPSS estimates likelihood of exploitation
* Exploit maturity reflects availability of exploit techniques
Evaluating these signals together provides a more complete understanding of potential risk than relying on any single field.

## Important considerations

> [!NOTE]
> Firmware analysis identifies potential risks based on extracted firmware content. It does not determine whether a vulnerability is reachable, exploitable, or impactful in a specific deployment.

Always interpret weakness data alongside:
* Device role and exposure
* System configuration
* Firmware usage within the platform

## Next steps
To learn more about how firmware analysis extracts and presents component data, see [Interpreting extractor paths from SBOM view in firmware analysis](interpreting-extractor-paths.md).