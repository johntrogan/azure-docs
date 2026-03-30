---
title: Networking endpoint requirements
description: Consolidated outbound endpoint requirements for deploying and operating Azure IoT Operations, including firewall rules, proxy configuration, and private connectivity guidance.
author: david-emakenemi
ms.subservice: layered-network-management
ms.author: demakenemi
ms.topic: concept-article
ms.date: 03/30/2026

#CustomerIntent: As a network administrator, I want a single reference for all outbound endpoints required by Azure IoT Operations so that I can configure firewall rules and proxy allowlists without cross-referencing multiple pages.
ms.service: azure-iot-operations
---

# Networking endpoint requirements

This page provides a consolidated reference for configuring firewall rules, proxy allowlists, and private connectivity for Azure IoT Operations deployments. It's designed so a network administrator can identify exactly which outbound endpoints to allow for their specific deployment scenario — without cross-referencing multiple Microsoft Learn pages.

**Design principle:** This page **links to** existing endpoint documentation owned by partner teams (Azure Arc, AKS Edge Essentials, Azure CLI, data platform services) rather than duplicating their lists. This approach ensures the referenced content stays current as partner teams update their endpoints. Azure IoT Operations owns only the endpoints listed in [Azure IoT Operations specific network requirements](#azure-iot-operations-specific-network-requirements) and the scenario-to-source mapping in [Endpoint decision matrix](#endpoint-decision-matrix).

**How to use this page:**

1. Start with the [Endpoint decision matrix](#endpoint-decision-matrix) to identify which combination of endpoint sources applies to your deployment.
1. Review [Azure IoT Operations specific network requirements](#azure-iot-operations-specific-network-requirements) for endpoints unique to Azure IoT Operations that aren't covered by any linked page.
1. Review the remaining sections for firewall formatting guidance, proxy bypass requirements, port reference, and deployment model guidance.

## Endpoint decision matrix

Every Azure IoT Operations deployment requires a base set of Azure Arc-enabled Kubernetes endpoints. Your specific scenario determines which additional endpoint sources you need to allowlist. **Scenarios are additive** — combine all rows that match your deployment.

For example: Azure IoT Operations on AKS Edge Essentials with secure settings and data flows to Event Hubs requires the base Arc endpoints + AKS-EE endpoints + Key Vault + Event Hubs.

| Your scenario | Endpoint source to allowlist | Phase | Notes |
|---|---|---|---|
| **Any Azure IoT Operations deployment (base)** | [Arc-enabled Kubernetes endpoints](/azure/azure-arc/network-requirements-consolidated#azure-arc-enabled-kubernetes-endpoints) | Install + Runtime + Update | Always required. Use Arc Gateway reduced set if using Arc Gateway. |
| + Azure CLI (from jump box) | [Azure CLI endpoints](/cli/azure/azure-cli-endpoints) | Install + Update | Only needed on the machine running `az` CLI commands. |
| + Azure CLI (from cluster node) | [Azure CLI endpoints](/cli/azure/azure-cli-endpoints) + all image registries | Install + Update | Broader surface — CLI + image pulls must be reachable from the node itself. See [Deployment model guidance](#deployment-model-guidance). |
| + AKS Edge Essentials | [AKS-EE networking guidance](/azure/aks/hybrid/aks-edge-concept-networking) | Install + Runtime + Update | Includes `docker.io` for local-path-provisioner and AKS-EE system images. See [Docker Hub dependency (AKS-EE only)](#docker-hub-dependency-aks-ee-only). |
| + K3s | [K3s system requirements](https://docs.k3s.io/installation/requirements) | Install + Runtime + Update | Includes k3s.io and any upstream image registries K3s uses. |
| + Secure settings (Key Vault) | `*.vault.azure.net` — also listed in [Azure CLI endpoints](/cli/azure/azure-cli-endpoints) | Runtime | Required for secret sync when deploying with secure settings. |
| + Data flows to Event Hubs | `*.servicebus.windows.net` (port 9093 for Kafka) — see [Event Hubs connectivity](/azure/event-hubs/troubleshooting-guide) | Runtime | Azure IoT Operations data flows use Kafka protocol by default. Verify whether your configuration uses AMQP (5671/5672) or Kafka (9093). |
| + Data flows to Event Grid MQTT | `<namespace>.<region>-1.ts.eventgrid.azure.net` (port 8883) — see [Event Grid connectivity](/azure/event-grid/troubleshoot-network-connectivity) | Runtime | See [Azure IoT Operations specific network requirements](#azure-iot-operations-specific-network-requirements) for this endpoint. |
| + Data flows to Fabric OneLake | [Fabric URL allowlist](/fabric/security/fabric-allow-list-urls) | Runtime | See Fabric networking docs for full URL list. |
| + Data flows to ADLS Gen2 | [Storage account endpoints](/azure/storage/common/storage-account-overview#standard-endpoints) | Runtime | `<account>.dfs.core.windows.net` |
| + Schema registry (always required) | See [Azure IoT Operations specific network requirements](#azure-iot-operations-specific-network-requirements) | Runtime | `<account>.blob.core.windows.net` — Azure IoT Operations specific. |
| + Arc Gateway | [Arc Gateway allowed endpoints](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway) (replaces base Arc list) | Install + Runtime + Update | Reduces outbound endpoint count significantly. |
| + Arc GW + Explicit Proxy | Arc GW endpoints + Azure Firewall Explicit Proxy config | Install + Runtime + Update | Preview. Achieves private connectivity without direct internet. |
| + Private connectivity (Private Link) | Private endpoints + Private DNS zones per service | Runtime | See [Deploy Azure IoT Operations with private connectivity](howto-private-connectivity.md). Validated on K3s / Ubuntu Server 24.04 only. |
| + Proxy environment | Add `NO_PROXY` bypass entries — see [Proxy bypass requirements](#proxy-bypass-requirements-no_proxy) | Install + Runtime | IMDS, cluster CIDRs, `.svc`, `.local` must bypass proxy. |

> [!IMPORTANT]
> **Private connectivity** (Arc Gateway + Explicit Proxy, Private Link) has been validated on K3s / Ubuntu Server 24.04 only. Other distributions have not been independently validated for private connectivity scenarios.

## Azure IoT Operations specific network requirements

The following endpoints are specific to Azure IoT Operations and aren't covered by any linked page in the decision matrix. These are the only entries the Azure IoT Operations team directly owns and maintains on this page.

| Endpoint / Pattern | Purpose | Port(s) | Notes |
|---|---|---|---|
| `<account>.blob.core.windows.net` | Schema registry storage | 443 | Always required. Container must expose a public endpoint or designate `Microsoft.DeviceRegistry/schemaRegistries` as a trusted Azure service. |
| `<namespace>.<region>-1.ts.eventgrid.azure.net` | Event Grid MQTT broker | 8883 / 9092 | Required when using Event Grid as MQTT bridge or data flow destination. This FQDN pattern isn't documented on the general Event Grid connectivity page. |
| `169.254.169.254` | Azure IMDS (Instance Metadata Service) | — | **Not a firewall allowlist entry.** Must be added to `NO_PROXY` / `--proxy-skip-range` in proxy environments. Required for Azure Device Registry and managed identity. Omitting causes cryptic auth failures. See [Proxy bypass requirements](#proxy-bypass-requirements-no_proxy). |

<!-- TODO: Verify Azure Device Registry API endpoint. If Device Registry has a distinct runtime FQDN called by Azure IoT Operations components at runtime, add it to this table. Capture actual cluster traffic to confirm. -->

> [!NOTE]
> The schema registry storage container must either expose a public endpoint or designate `Microsoft.DeviceRegistry/schemaRegistries` as a [trusted Azure service](/azure/storage/common/storage-network-security-trusted-azure-services#trusted-access-based-on-a-managed-identity) (`publicNetworkAccess=Disabled` with `bypass=AzureServices`). This is an Azure-side configuration and doesn't require additional firewall rules at the edge.

## Firewall implementation notes

Endpoint lists specify FQDNs, but real-world firewall rules require additional consideration. This section addresses common issues encountered during Azure IoT Operations deployments in locked-down environments.

### Wildcard and path-based rule behavior

Different firewall vendors interpret wildcard rules differently. A rule for `*.example.com` may or may not match `example.com` itself, or URLs with path segments like `example.com/path/resource`.

- **Palo Alto:** Allowing `docker.io` alone doesn't permit `docker.io/rancher/local-path-provisioner`. You must add both `docker.io` and `docker.io/*` as separate rules.
- **`*.servicebus.windows.net`** may not match URLs with deeper path segments on some firewalls. Add both the wildcard pattern and explicit path-based rules if your firewall treats these separately.
- **Azure Firewall** with FQDN tags generally handles wildcard matching more broadly. If using Azure Firewall, prefer FQDN tags over manual wildcard rules where available.
- **General guidance:** Test with the full image pull URLs (including path and tag) rather than just the base domain.

### TLS inspection bypass

Arc Gateway doesn't support TLS termination or inspection. The following traffic must flow end-to-end TLS without interception or header rewrites:

- **Arc control plane:** `*.connectedk8s.azure.com`, `*.kubernetesconfiguration.azure.com`, `*.guestconfiguration.azure.com`
- **Identity:** `*.microsoftonline.com`, `*.msftauth.net`, `*.msauth.net`
- **Control channel:** `*.servicebus.windows.net` (SNI must be preserved)
- **Image downloads:** `mcr.microsoft.com/*`, `*.blob.core.windows.net`

For more information, see [Arc Gateway and TLS inspection](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#azure-arc-gateway-and-tls-inspection).

> [!IMPORTANT]
> Third-party proxies (for example, Palo Alto, Zscaler) and transparent proxies aren't supported in validated Azure IoT Operations private connectivity scenarios. Only Azure Firewall Explicit Proxy has been validated.

### Docker Hub dependency (AKS-EE only)

AKS Edge Essentials pulls the local-path-provisioner image from Docker Hub (`docker.io/rancher/local-path-provisioner:master-head`). This is an **AKS-EE dependency**, not an Azure IoT Operations dependency, but it causes deployment failures if blocked.

- Allowlist both `docker.io` and `registry-1.docker.io` (Docker Hub's actual registry backend).
- Also allowlist `production.cloudflare.docker.com` (Docker Hub CDN).
- This dependency is documented in the [AKS-EE Local Path Provisioner documentation](/azure/aks/hybrid/aks-edge-howto-use-storage-local-path), not in Azure IoT Operations docs.

### OIDC issuer endpoint

Arc-enablement requires the `--enable-oidc-issuer` flag. The OIDC issuer URL exposed by the cluster must be reachable by Microsoft Entra ID for token validation. Ensure the OIDC issuer URL isn't blocked by outbound firewall rules. This URL is cluster-specific and generated during Arc onboarding.

## Proxy bypass requirements (NO_PROXY)

If your cluster connects through a proxy, the following addresses must be added to `NO_PROXY` / `--proxy-skip-range`. Failing to bypass these causes authentication failures, internal routing issues, and deployment errors that are difficult to diagnose.

| NO_PROXY entry | Purpose | Notes |
|---|---|---|
| `169.254.169.254` | Azure IMDS (Instance Metadata Service) | Required for Azure Device Registry and managed identity token acquisition. Omitting causes cryptic auth failures. |
| Cluster pod/service CIDRs | Internal cluster networking | Typically `10.42.0.0/16` and `10.43.0.0/16` for K3s. Check your distro defaults. |
| `.svc` | Kubernetes service DNS | Internal service-to-service communication. |
| `.local` | Local DNS resolution | mDNS and local hostname resolution. |
| `localhost` / `127.0.0.1` | Loopback | Local services including MQTT broker on port 1883. |

Set these values during Arc onboarding using `--proxy-skip-range` or via environment variables (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`) before Azure IoT Operations deployment.

## Port reference

Most Azure service endpoints use TCP 443 (HTTPS). The following table lists all non-standard ports referenced across Azure IoT Operations deployment and runtime.

| Port | Protocol | Used by | Notes |
|---|---|---|---|
| 443 (TCP) | HTTPS | Most Azure service endpoints, Arc control plane, ACR, Key Vault, ARM | Default for all endpoints unless noted otherwise. |
| 8883 (TCP) | MQTTS | Event Grid MQTT broker | Secure MQTT. Required for MQTT bridge data flows. |
| 9092 (TCP) | MQTT/Kafka | Event Grid alternate port | Alternative to 8883 for some Event Grid configurations. |
| 9093 (TCP) | Kafka TLS | Event Hubs Kafka protocol | Required when using Kafka protocol to Event Hubs. Azure IoT Operations data flows use Kafka by default — verify your configuration. |
| 80 (TCP) | HTTP | Certificate updates (`www.microsoft.com/pkiops`) | Some certificate and package endpoints use HTTP. |
| 123 (UDP) | NTP | `time.windows.com` | Time sync for Arc Resource Bridge. |
| 1883 (TCP) | MQTT (local) | MQTT broker local access | Local/internal only. Not routed through firewall. |

<!-- TODO: Verify whether Azure IoT Operations data flows use only Kafka (9093) to Event Hubs or also AMQP (5671/5672). Current evidence from howto-private-connectivity.md suggests Kafka only. If confirmed, remove AMQP references. -->

## Deployment model guidance

The network surface required for Azure IoT Operations deployment varies depending on **where you run the deployment commands**. The Azure IoT Operations documentation assumes a jump box model, but customers may deploy differently. See the corresponding rows in the [Endpoint decision matrix](#endpoint-decision-matrix) ("Azure CLI from jump box" and "Azure CLI from cluster node") for the specific endpoint sources each model requires.

### Jump box deployment (recommended)

A dedicated machine with `kubectl` access runs `az login`, `az connectedk8s connect`, and `az iot ops create`. In this model:

- The **jump box** needs outbound access to Azure CLI endpoints, ARM, and Microsoft Entra ID for authentication.
- The **cluster nodes** need outbound access to Arc endpoints, container registries (MCR, ACR), and Azure IoT Operations runtime endpoints.
- CLI extension endpoints (for example, `github.com` for the `azure-iot-ops` extension) are only needed on the jump box.

### Direct deployment from cluster node

If you run deployment commands directly on an AKS-EE VM or K3s node (not recommended for production):

- The node needs the **full combined surface**: CLI endpoints + image registries + Arc endpoints + Azure IoT Operations runtime endpoints.
- This is a significantly broader allowlist than the jump box model.
- AKS-EE VMs may have separate network interfaces from the Kubernetes cluster network, further complicating firewall rules.

## Known limitations

- Private connectivity (Arc Gateway + Explicit Proxy) has been validated on **K3s / Ubuntu Server 24.04 only**. AKS Edge Essentials and other distributions haven't been independently validated.
- Third-party proxies (Palo Alto, Zscaler, transparent proxies) aren't supported in validated private connectivity scenarios.
- Azure Firewall Explicit Proxy integration with Arc Gateway is in **preview**.
- Private Arc Gateway (Private Link native to Arc Gateway) is in **private preview**. See [Arc Gateway documentation](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) for current availability timeline.
- Schema registry private storage (`publicNetworkAccess=Disabled` with `bypass=AzureServices`) and data flow private endpoint DNS resolution require additional end-to-end validation.
- The Arc Gateway reduced endpoint set for Azure IoT Operations hasn't been explicitly published — customers must reference the general [Arc Gateway allowed endpoints](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway) documentation.

<!-- TODO: Verify Azure Device Registry API endpoint FQDN pattern against actual cluster traffic. -->
<!-- TODO: Confirm Event Hubs AMQP vs Kafka protocol usage in Azure IoT Operations data flows. -->

## Reference documentation

The following external documentation pages are referenced throughout this document. These are owned by partner teams and are the authoritative source for their respective endpoint lists.

| Source | Link |
|---|---|
| Arc-enabled Kubernetes endpoints | [Azure Arc network requirements](/azure/azure-arc/network-requirements-consolidated#azure-arc-enabled-kubernetes-endpoints) |
| Arc Gateway allowed endpoints | [Arc Gateway simplified networking](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway) |
| Azure CLI endpoints | [Azure CLI endpoints reference](/cli/azure/azure-cli-endpoints) |
| AKS-EE networking guidance | [AKS Edge Essentials networking](/azure/aks/hybrid/aks-edge-concept-networking) |
| AKS-EE Local Path Provisioner | [AKS-EE storage local path](/azure/aks/hybrid/aks-edge-howto-use-storage-local-path) |
| K3s system requirements | [K3s installation requirements](https://docs.k3s.io/installation/requirements) |
| Event Hubs connectivity | [Event Hubs troubleshooting guide](/azure/event-hubs/troubleshooting-guide) |
| Event Grid connectivity | [Event Grid troubleshooting guide](/azure/event-grid/troubleshoot-network-connectivity) |
| Fabric URL allowlist | [Fabric URLs](/fabric/security/fabric-allow-list-urls) |
| Azure IoT Operations private connectivity guide | [Deploy with private connectivity](howto-private-connectivity.md) |
| Storage account endpoints | [Storage account overview](/azure/storage/common/storage-account-overview#standard-endpoints) |

## Related content

- [Azure IoT Operations networking](overview-layered-network.md)
- [Deploy Azure IoT Operations with private connectivity](howto-private-connectivity.md)
- [Deployment details](../deploy-iot-ops/overview-deploy.md)
- [Production deployment guidelines](../deploy-iot-ops/concept-production-guidelines.md)
