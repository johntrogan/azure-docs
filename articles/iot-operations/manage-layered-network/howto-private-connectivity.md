---
title: Deploy Azure IoT Operations with private connectivity using Arc Gateway
description: Deploy Azure IoT Operations with private connectivity to Azure services using Arc Gateway, Azure Firewall Explicit Proxy, Private Link, and ExpressRoute.
author: david-emakenemi
ms.subservice: layered-network-management
ms.author: demakenemi
ms.topic: how-to
ms.date: 03/19/2026

#CustomerIntent: As an operator, I want to deploy Azure IoT Operations with private connectivity to Azure using Arc Gateway so that no endpoints are exposed to the public internet.
ms.service: azure-iot-operations
---

# Deploy Azure IoT Operations with private connectivity using Arc Gateway

This article describes how to deploy Azure IoT Operations with fully private connectivity to Azure services. The core of this approach is [Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking), which consolidates the ~200+ Azure endpoints that Arc agents and extensions require down to a small set of manageable FQDNs. Combined with Azure Firewall Explicit Proxy, ExpressRoute, and Azure Private Link, this architecture ensures that all traffic between your on-premises cluster and Azure stays on private networks with no public internet exposure.

This scenario is for environments with a single Arc-enabled Kubernetes cluster that needs private Azure connectivity. There is no Purdue-style network segmentation, no proxy chaining across layers, no CoreDNS multi-hop, and no Envoy deployment. If you have a layered network topology, see [Deploy Azure IoT Operations in a layered network with private connectivity](howto-layered-network-private-connectivity.md) instead.

In this article, you:

- Understand how Arc Gateway enables private connectivity for Azure IoT Operations
- Create Private Endpoints and Private DNS Zones for Azure services
- Connect your cluster to Azure Arc through Arc Gateway and Azure Firewall Explicit Proxy
- Assign RBAC roles required by Azure IoT Operations components
- Deploy Azure IoT Operations
- Validate end-to-end private connectivity
- Audit and verify the deployment

## How Arc Gateway enables private connectivity

Azure Arc Gateway is the foundation of this deployment. Without it, an Arc-enabled cluster must reach over 200 Azure endpoints directly — each of which you would need to allow through your firewall. Arc Gateway solves this by acting as a common front end for Azure Arc traffic:

1. **Arc Gateway (Azure resource):** You create an Arc Gateway resource in Azure, which provides a single gateway URL (`<gateway-name>.gw.arc.azure.com`). This resource acts as the cloud-side entry point for all Arc management traffic.

2. **Arc Proxy (in-cluster pod):** When you connect a cluster with `--gateway-resource-id`, the Arc agents deploy an Arc Proxy pod inside the cluster. This pod acts as a local forward proxy — all Arc agent and extension traffic routes through it.

3. **Azure Firewall Explicit Proxy:** The Azure Firewall sits in your private VNet (reachable over ExpressRoute) and acts as a forward proxy for outbound traffic. The Arc Proxy directs traffic to the firewall, which then forwards it to the Arc Gateway and other Azure services over the Microsoft backbone.

4. **Private Link and Private Endpoints:** For Azure services that support Private Link (Event Grid, Blob Storage, Key Vault), traffic bypasses the proxy chain entirely and routes directly to private IPs within your VNet. This provides the lowest latency and strongest isolation for data-plane traffic.

5. **ExpressRoute:** Provides the private physical link between your on-premises network and the Azure VNet where the firewall and Private Endpoints reside. No traffic touches the public internet.

The resulting traffic flow is:

- **Arc management traffic:** Arc Agents → Arc Proxy pod → Azure Firewall Explicit Proxy → Arc Gateway → Azure management services
- **Data-plane traffic:** Azure IoT Operations components → Azure Firewall Explicit Proxy → Private Endpoints (Event Grid, Storage, Key Vault) over ExpressRoute

With this architecture, you reduce your firewall allowlist to approximately 9 FQDNs, keep all traffic private, and maintain full Azure Arc management capabilities.

For full details on Arc Gateway architecture and supported endpoints, see [Simplify network configuration requirements with Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking).

## Decision guidance: private-only vs. layered network vs. sovereign cloud

Before you begin, determine which networking approach fits your scenario:

- **Private-only with Arc Gateway (this article):** If you have a single cluster that needs private connectivity to Azure without network segmentation between layers, use this approach. Arc Gateway consolidates Azure endpoints, Azure Firewall Explicit Proxy keeps traffic on private networks, and Private Link eliminates public endpoint exposure for data-plane services.
- **Layered network:** If you have a Purdue/ISA-95 segmented topology with multiple network layers (L2/L3/L4) and adjacent-only communication, use a layered network deployment. This approach adds Envoy proxy chaining, CoreDNS at each layer, and multi-cluster Azure IoT Operations deployments across layers. **If you have a layered topology, the layered approach is recommended.** See [Deploy Azure IoT Operations in a layered network with private connectivity](howto-layered-network-private-connectivity.md).
- **Sovereign cloud:** If you operate in a sovereign cloud (Azure Government, Azure China, Azure Germany), the private connectivity patterns may differ due to regional service availability and network architecture. Use this approach only if your cluster is in a sovereign cloud and you require private connectivity. Refer to your sovereign cloud documentation for specific guidance.

## Prerequisites

Before you begin, make sure the following requirements are met.

### Azure access and permissions

- A valid Azure subscription and tenant ID.
- An [Arc Gateway resource](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) already created in your subscription. This article assumes the gateway exists; for creation steps, see [Create the Arc Gateway resource](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#create-the-arc-gateway-resource).
- Required role assignments: In validated scenarios, role assignments were created manually by admins with elevated privileges (Owner), since Contributor alone was insufficient. The following custom roles may be required:
  - **ACX–Secrets Store Extension Owner** — For registering/managing the Secrets Store CSI driver, configuring Azure Key Vault secret provider classes, and managing user-assigned managed identities.
  - **AdaptiveCloud_AIO–Contributors** — For managing federated identity credentials and role assignments for user-assigned managed identities within the resource group.

> [!TIP]
> For production deployments, use Azure Policy automation to pre-create these RBAC assignments. This eliminates the need for manual Owner intervention and allows OT teams to deploy with Contributor only.

For custom role definitions, see [Deploy Azure IoT Operations in a layered network with private connectivity — Appendix](howto-layered-network-private-connectivity.md#appendix).

### Network and infrastructure requirements

- A K3s cluster (or equivalent Kubernetes cluster) deployed and ready to Arc-enable.
- [ExpressRoute](/azure/expressroute/expressroute-introduction) or equivalent private routing between your on-premises network and your Azure VNet.
- [Azure Firewall Explicit Proxy](/azure/azure-arc/azure-firewall-explicit-proxy) deployed in your Azure VNet, reachable from your cluster over ExpressRoute. Note the firewall's private IP and port (for example, `10.254.x.x:8443`).
- Network connectivity from your cluster to the Azure Firewall's private IP over ExpressRoute.

### Tools

- Access to the [Azure IoT Operations portal](https://iotoperations.azure.com).
- Azure CLI installed on your admin or jump machine.
- kubectl installed on your admin or jump machine.

## Create private endpoints for Azure services

Create Private Endpoints to give Azure services private IPs within your VNet. For services that support Private Link, data-plane traffic routes directly to these private IPs over ExpressRoute — bypassing the proxy chain entirely.

Create Private Endpoints for the following services:

### Event Grid namespace

```azurecli
az network private-endpoint create \
  --name pe-eventgrid \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventGrid/namespaces/<namespace>" \
  --group-id topicspaces \
  --connection-name pe-conn-eventgrid
```

### Azure Storage account

```azurecli
az network private-endpoint create \
  --name pe-storage-blob \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<account>" \
  --group-id blob \
  --connection-name pe-conn-storage-blob
```

### Azure Key Vault

```azurecli
az network private-endpoint create \
  --name pe-keyvault \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.KeyVault/vaults/<keyvault-name>" \
  --group-id vault \
  --connection-name pe-conn-keyvault
```

### Verify Private Endpoints

In the Azure portal, navigate to each Private Endpoint and confirm the connection status shows **Approved**. Confirm each endpoint has a private IP assigned in your VNet's address space.

## Configure Private DNS Zones

Create Private DNS Zones so that DNS queries for Azure services resolve to the Private Endpoint IPs instead of public IPs. This is essential — without correct DNS resolution, traffic falls back to public endpoints.

### Create DNS zones

| Service | Private DNS Zone |
|---------|-----------------|
| Event Grid | `privatelink.ts.eventgrid.azure.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |

Link each Private DNS Zone to the VNet where your Azure Firewall and Private Endpoints reside. This linkage enables the Azure DNS resolver (`168.63.129.16`) to return private IPs for service FQDNs when queried from within the VNet.

For the full list of private DNS zone names, see [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns).

### Verify DNS resolution

From your cluster node, confirm that Azure service FQDNs resolve to private IPs:

```bash
nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
nslookup <storage-account>.blob.core.windows.net
nslookup <keyvault-name>.vault.azure.net
```

Each result should return an IP in your private address range (for example, `10.254.x.x`), not a public IP.

> [!IMPORTANT]
> If any FQDN resolves to a public IP, check your Private DNS Zone linkage and VNet configuration before proceeding. Incorrect DNS resolution means traffic will route over the public internet.

## Connect your cluster through Arc Gateway

This is the central step in the deployment. You connect your Kubernetes cluster to Azure Arc, routing all Arc traffic through the Azure Firewall Explicit Proxy and associating the cluster with your Arc Gateway resource.

### Set proxy environment variables

On the machine where you run the `az connectedk8s connect` command, set the proxy environment variables to point to your Azure Firewall Explicit Proxy:

```bash
export HTTPS_PROXY=http://<firewall-private-ip>:<port>
export HTTP_PROXY=http://<firewall-private-ip>:<port>
export NO_PROXY=localhost,127.0.0.1,.svc,.local,<cluster-subnet-cidr>
```

> [!NOTE]
> The `HTTPS_PROXY` and `HTTP_PROXY` values point to the Azure Firewall's private IP and explicit proxy port (for example, `http://10.254.0.68:8443`). Adjust `NO_PROXY` to include your cluster's internal CIDRs and any local domains that should bypass the proxy.

### Retrieve the custom locations Object ID

The `--custom-locations-oid` parameter requires the Object ID (OID) of the Azure Arc Custom Locations service principal.

To find it:

1. Go to **Microsoft Entra ID** in the Azure portal.
1. Select **Enterprise applications**.
1. Search for **Azure Arc Kubernetes Custom Locations**.
1. Open the application, go to **Properties**, and copy the **Object ID**.

### Connect the cluster with Arc Gateway

Connect the cluster and associate it with your Arc Gateway resource. The `--gateway-resource-id` parameter is what routes Arc traffic through the Arc Gateway, consolidating the endpoints your firewall must allow:

```azurecli
az connectedk8s connect \
  --name <cluster-name> \
  --resource-group <resource-group> \
  --location <region> \
  --custom-locations-oid <OID> \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --disable-auto-upgrade \
  --proxy-https $HTTPS_PROXY \
  --proxy-http $HTTP_PROXY \
  --proxy-skip-range $NO_PROXY \
  --gateway-resource-id <gateway-resource-id>
```

This command:

- Registers the cluster with Azure Arc
- Deploys Arc agents and the Arc Proxy pod into the cluster
- Configures all Arc traffic to route through the Azure Firewall Explicit Proxy
- Associates the cluster with the Arc Gateway, consolidating ~200+ endpoints to ~9 allowed FQDNs

For the list of FQDNs that must be allowed through your firewall when using Arc Gateway, see [Allowed endpoints with Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway).

### Verify Arc connectivity

1. Confirm the Arc agents and Arc Proxy pod are running:

   ```bash
   kubectl get pods -n azure-arc
   ```

1. Check the Arc Proxy pod logs to confirm it reaches Azure through the gateway:

   ```bash
   kubectl logs -n azure-arc -l app.kubernetes.io/component=arc-proxy
   ```

1. Verify the cluster appears as **Connected** in the Azure portal under **Azure Arc > Kubernetes clusters**.

## Assign RBAC roles

Azure IoT Operations requires specific RBAC assignments to allow its components to interact with Azure services.

### Required role assignments

| Identity | Role | Scope | Notes |
|----------|------|-------|-------|
| Azure IoT Operations system-assigned managed identity | Storage Blob Contributor | Storage account containing schema files | For Schema Registry |
| Azure IoT Operations system-assigned managed identity | EventGrid TopicSpaces Publisher | Event Grid namespace | Enables publish to TopicSpaces |
| Azure IoT Operations system-assigned managed identity | EventGrid TopicSpaces Subscriber | Event Grid namespace | Enables subscribe to TopicSpaces |

Assign each role using Azure CLI:

```azurecli
# Storage Blob Contributor (for Schema Registry)
az role assignment create \
  --assignee <identity-client-id> \
  --role "Storage Blob Contributor" \
  --scope <storage-account-resource-id>

# Event Grid Publisher
az role assignment create \
  --assignee <identity-client-id> \
  --role "EventGrid TopicSpaces Publisher" \
  --scope <event-grid-namespace-resource-id>

# Event Grid Subscriber
az role assignment create \
  --assignee <identity-client-id> \
  --role "EventGrid TopicSpaces Subscriber" \
  --scope <event-grid-namespace-resource-id>
```

> [!IMPORTANT]
> When creating the Schema Registry, use the `--skip-ra` flag. This prevents the CLI from attempting manual role assignments that require Owner rights. Instead, use Azure Policy automation to handle RBAC configuration.

## Deploy Azure IoT Operations

Deploy Azure IoT Operations to your Arc-enabled cluster. Since your cluster is already connected to Azure Arc through the Arc Gateway and proxy, Azure IoT Operations components inherit the private connectivity path — all Azure communication routes through the Arc Proxy pod, Azure Firewall Explicit Proxy, and Private Endpoints.

For deployment instructions, see [Deploy Azure IoT Operations](/azure/iot-operations/deploy-iot-ops/overview-deploy).

### Verify deployment

Confirm that Azure IoT Operations components are running:

```bash
kubectl get pods -n azure-iot-operations
```

Verify MQTT Broker listeners are active:

```bash
kubectl get service -n azure-iot-operations
```

## Validate connectivity

### Verify Arc Gateway is routing traffic

Confirm the Arc Proxy pod is healthy and actively routing traffic:

```bash
kubectl get pods -n azure-arc -l app.kubernetes.io/component=arc-proxy
kubectl logs -n azure-arc -l app.kubernetes.io/component=arc-proxy --tail=50
```

The logs should show successful connections to Azure services through the gateway.

### Verify DNS resolution

From the cluster node, confirm that all Azure service FQDNs resolve to private IPs:

```bash
nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
nslookup <storage-account>.blob.core.windows.net
nslookup <keyvault-name>.vault.azure.net
```

Each result should return an IP in your private address range, not a public IP.

### Verify telemetry flow

Confirm telemetry is flowing from Azure IoT Operations to Event Grid:

```bash
mqttui --broker mqtt://<cluster-host-ip>:1883
```

Check Event Grid Topic Spaces in the Azure portal or CLI to confirm messages are being received.

### Verify private routing

Confirm that traffic reaches Azure services through Private Endpoints, not public endpoints:

```bash
curl -v https://<eventgrid-namespace>.ts.eventgrid.azure.net
```

The resolved IP address should be in your private range.

## Audit and post-deployment verification

<!-- TODO: This section was authored based on deployment patterns and requires SME review by Qiang/Grishma. It was not sourced from the validated PDF. -->

> [!IMPORTANT]
> This section provides recommended audit procedures. Have your network and security team review these steps before using them in production.

After deployment, verify that private connectivity, Arc Gateway routing, and RBAC assignments are correctly configured.

### Confirm Arc Gateway is the active routing path

Verify that Arc management traffic routes through the Arc Gateway, not directly to public Azure endpoints:

1. Check the Arc Proxy pod is running and healthy:

   ```bash
   kubectl get pods -n azure-arc -l app.kubernetes.io/component=arc-proxy
   ```

1. Review Azure Firewall logs to confirm outbound traffic from your cluster routes to the Arc Gateway URL (`<gateway-name>.gw.arc.azure.com`) and Private Endpoint IPs — not to public Azure service IPs.

### Confirm traffic routes through Private Endpoints

Verify that data-plane traffic to Azure services routes through Private Endpoints:

1. From your cluster node, resolve Azure service FQDNs and confirm they return private IPs:

   ```bash
   nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
   nslookup <storage-account>.blob.core.windows.net
   nslookup <keyvault-name>.vault.azure.net
   ```

   Each result should return an IP in your private address range (for example, `10.254.x.x`), not a public IP.

1. Check the Azure portal for each Private Endpoint to confirm the connection status shows **Approved**.

1. In the Azure Firewall logs, verify that outbound traffic routes to private endpoint IPs only — no traffic should target public Azure service endpoints.

### Validate RBAC assignments

Confirm that the required role assignments are in place:

```azurecli
# Check Storage Blob Contributor
az role assignment list \
  --scope <storage-account-resource-id> \
  --role "Storage Blob Contributor" \
  --output table

# Check Event Grid Publisher
az role assignment list \
  --scope <event-grid-namespace-resource-id> \
  --role "EventGrid TopicSpaces Publisher" \
  --output table

# Check Event Grid Subscriber
az role assignment list \
  --scope <event-grid-namespace-resource-id> \
  --role "EventGrid TopicSpaces Subscriber" \
  --output table
```

Verify that the Azure IoT Operations system-assigned managed identity is listed as the assignee for each role.

### Verify DNS resolves to private IPs only

From the cluster node, confirm that Azure service names resolve only to private IPs:

```bash
dig <eventgrid-namespace>.ts.eventgrid.azure.net
dig <storage-account>.blob.core.windows.net
dig <keyvault-name>.vault.azure.net
```

If any query returns a public IP, check your Private DNS Zone linkage to the VNet.

## Known limitations

- **Platform validation:** The private connectivity patterns described here are based on validated K3s on Ubuntu Server 24.04 scenarios. Other Kubernetes distributions or operating systems haven't been independently validated for this configuration.
- **Schema Registry public access:** Schema Registry may require public access enabled at creation time. After creation, you can disable public access. Use the `--skip-ra` flag when creating the Schema Registry to avoid requiring Owner-level permissions.
- **TLS inspection:** Arc Gateway does not support TLS termination or inspection. If your firewall performs TLS inspection, you must exclude the Arc Gateway endpoint from inspection. See [Arc Gateway and TLS inspection](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#azure-arc-gateway-and-tls-inspection).
- **Arc Gateway limits:** A maximum of five Arc Gateway resources are supported per subscription.

## Related content

- [Simplify network configuration requirements with Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking)
- [Access Azure services over Azure Firewall Explicit Proxy](/azure/azure-arc/azure-firewall-explicit-proxy)
- [Deploy Azure IoT Operations in a layered network with private connectivity](howto-layered-network-private-connectivity.md)
- [How does Azure IoT Operations work in layered network?](concept-iot-operations-in-layered-network.md)
- [Azure IoT Operations networking](overview-layered-network.md)
- [Deploy Azure IoT Operations](/azure/iot-operations/deploy-iot-ops/overview-deploy)
- [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns)
