---
title: Understanding weaknesses data in firmware analysis
description: Learn what the weaknesses data are in the CVE view of the firmware analysis results.
author: karengu0
ms.author: karenguo
ms.topic: conceptual
ms.date: 02/25/2026
ms.service: azure
---

# Understanding weaknesses data in firmware analysis

Firmware analysis surfaces weaknesses detected in firmware components extracted during analysis. These signals help you understand potential security risks, but they should be interpreted carefully and in context.
This article explains the weakness related fields you may see in firmware analysis results and how they relate to each other.

> [!NOTE]
> The presence of a weakness or CVE in firmware analysis does not necessarily mean a device is vulnerable. Actual impact depends on how the affected component is used within the system.


## Weakness signals in firmware analysis

Firmware analysis may enrich findings with multiple industry standard signals. Each signal represents a different aspect of risk and should not be interpreted in isolation.


### Common Vulnerabilities and Exposures (CVE)

A CVE is a publicly disclosed identifier for a known security vulnerability.
Firmware analysis associates CVEs with extracted firmware components when a match is identified.
A single firmware component may be associated with multiple CVEs, and a single CVE may appear across multiple devices or components.

For more information about CVE identifiers and the CVE program, see the official [Common Vulnerabilities and Exposures documentation maintained by MITRE](https://www.cve.org).


### CVSS scores and versions

Firmware analysis may display Common Vulnerability Scoring System (CVSS) data for a CVE.
Multiple CVSS versions can appear for the same CVE:
* CVSS v2 – legacy scoring used by older vulnerabilities
* CVSS v3 – widely adopted standard with improved metrics
* CVSS v4 – newer version that introduces additional dimensions

The presence of multiple versions reflects how vulnerability scoring evolves over time rather than multiple distinct vulnerabilities.

For more information about CVSS scoring and version differences, see the official [Common Vulnerability Scoring System (CVSS) documentation maintained by FIRST](https://www.first.org/cvss/).


### CVSS vector

In addition to a numeric score, CVSS includes a vector string that describes the factors contributing to the score, such as:
* Required access level
* Attack complexity
* Impact on confidentiality, integrity, and availability

The vector provides additional context such as conditions and impact factors that contribute to a CVE’s severity rating.

For a full explanation of CVSS vector strings and metric meanings, see the [CVSS specification published by FIRST](https://www.first.org/cvss/specification-document).

For examples of how CVSS scores and vectors are published for CVEs, see the [NIST National Vulnerability Database (NVD)](https://nvd.nist.gov/vuln-metrics/cvss).


### CISA Known Exploited Vulnerabilities (KEV)

Some CVEs may be marked as part of the CISA Known Exploited Vulnerabilities (KEV) catalog.
This designation indicates that the vulnerability is known to be actively exploited in real-world scenarios.

> [!NOTE]
> - KEV status reflects observed exploitation activity, not whether a specific device is affected.
> - KEV status is currently a static value, reflecting the state of the Firmware analysis CVE database at the time the scan was conducted. This value is not updated dynamically. To view the most up-to-date KEV status, please re-scan your firmware image.

For authoritative KEV status and remediation guidance, see the [CISA Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog).


### Exploit Prediction Scoring System (EPSS)

Firmware analysis may include EPSS data, which estimates the likelihood that a vulnerability will be exploited.
Two related values may appear:
* EPSS score – an estimated likelihood of exploitation based on observed trends across the vulnerability ecosystem 
* EPSS percentile – how that probability compares relative to other vulnerabilities
These values provide comparative risk context but do not guarantee exploitation.

> [!NOTE]
> - EPSS value is currently a static value, reflecting the state of the Firmware analysis CVE database at the time the scan was conducted. This value is not updated dynamically. To view the most up-to-date EPSS status, please re-scan your firmware image.

For details on how EPSS scores and percentiles are calculated, see the [Exploit Prediction Scoring System documentation maintained by FIRST](https://www.first.org/epss/).


### Common Weakness Enumeration (CWE)

CWE represents the class of underlying weakness (for example, buffer overflow or improper input validation) that led to a vulnerability, rather than a specific vulnerability instance.
CWE identifiers provide additional context by describing why a vulnerability exists, not just where it occurs.

> [!NOTE]
> CWE data reflects standardized weakness classifications defined by the MITRE Common Weakness Enumeration project. CWE identifiers are informational and do not indicate exploitability or impact on a specific device or firmware image by themselves.

For more information about CWE definitions and classifications, see the official [MITRE CWE documentation](https://cwe.mitre.org/).


### Exploit maturity

Exploit maturity describes the current state of exploit availability for a vulnerability, such as whether publicly known exploit techniques or code exist.

When present, exploit maturity information is typically surfaced alongside CVSS v4 scoring, and described in the [CVSS specification maintained by FIRST](https://www.first.org/cvss/v4.0/specification-document).


## Using weakness data together

Each weakness signal represents a different perspective:
* CVE identifies what the issue is
* CVSS describes technical severity
* KEV indicates known exploitation
* EPSS estimates likelihood of exploitation
* Exploit maturity reflects availability of exploit techniques
Evaluating these signals together provides a more complete understanding of potential risk than relying on any single field.


## Important considerations

Always interpret weakness data alongside:
* Device role and exposure
* System configuration
* Firmware usage within the platform

> [!NOTE]
> Firmware analysis identifies potential risks based on extracted firmware content. It does not determine whether a vulnerability is reachable, exploitable, or impactful in a specific deployment.

## Next steps
To learn more about how firmware analysis extracts and presents component data, see [Interpreting extractor paths from SBOM view in firmware analysis](interpreting-extractor-paths.md).