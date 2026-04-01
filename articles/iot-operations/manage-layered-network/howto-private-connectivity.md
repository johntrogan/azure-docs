---
title: Deploy Azure IoT Operations with private connectivity
description: Configure private connectivity for Azure IoT Operations using Private Link, Arc Gateway, and Azure Firewall Explicit Proxy.
author: david-emakenemi
ms.subservice: layered-network-management
ms.author: demakenemi
ms.topic: how-to
ms.date: 03/25/2026

#CustomerIntent: As an operator, I want to deploy Azure IoT Operations with private connectivity to Azure so that no endpoints are exposed to the public internet.
ms.service: azure-iot-operations
---

# Deploy Azure IoT Operations with private connectivity

This article describes how to configure private connectivity for Azure IoT Operations. Follow the sections in order:

| Step | Section | What it does |
|------|---------|-------------|
| 1 | [Connect your cluster via Arc Gateway](#connect-your-cluster-via-arc-gateway) | Create the Arc Gateway resource and retrieve the custom locations OID |
| 2 | [Create private endpoints and connect the cluster](#create-private-endpoints-and-connect-the-cluster) | Create Private Endpoints, DNS zones, and Arc-enable the cluster. Choose between **Arc Gateway only** or **Arc Gateway + Explicit Proxy** tabs |
| 3 | [Deploy Azure IoT Operations](#deploy-azure-iot-operations) | Deploy AIO. Traffic already routes privately via DNS |
| 4 | [Disable public access on storage and Key Vault](#disable-public-access-on-storage-and-key-vault) | Lock down the storage account and Key Vault after AIO is healthy |
| 5 | [Configure data flow destinations with private endpoints](#configure-data-flow-destinations-with-private-endpoints) | Route data flow traffic to cloud destinations like Event Grid through Private Link |

These scenarios apply to environments with a single Arc-enabled Kubernetes cluster. There's no Purdue-style network segmentation, no proxy chaining across layers, and no Envoy deployment. If you have a layered network topology, see [Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity](../end-to-end-tutorials/tutorial-layered-network-private-connectivity.md) instead.

## Prerequisites

- An [Azure subscription](/azure/cost-management-billing/manage/create-subscription). If you don't have one, [create a free account](https://azure.microsoft.com/free/) before you begin.
- Sufficient permissions to create Private Endpoints, Private DNS Zones, and role assignments in your subscription (typically **Owner** or **Contributor** + **User Access Administrator**).
- A Kubernetes cluster deployed and ready to Arc-enable. See [Prepare your cluster](/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster) for supported configurations and setup steps.
- An Azure VNet where you create Private Endpoints and Private DNS Zones.
- Network connectivity from your on-premises cluster to the Azure VNet ([ExpressRoute](/azure/expressroute/expressroute-introduction), [VPN Gateway](/azure/vpn-gateway/vpn-gateway-about-vpngateways), VNet peering, or other private routing).

  > [!NOTE]
  > If your cluster runs on Azure VMs within the same VNet (or a peered VNet), this connectivity is already in place. This prerequisite applies primarily to on-premises or edge clusters that need a private network path to the Azure VNet.

- [Azure CLI](/cli/azure/install-azure-cli) and [kubectl](https://kubernetes.io/docs/tasks/tools/) installed on your admin or jump machine.

## Connect your cluster via Arc Gateway

[Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) consolidates the ~200+ Azure endpoints that Arc agents and extensions require into a single gateway URL. This significantly simplifies your firewall allowlist, instead of allowing 200+ individual FQDNs, you allow approximately 9.

### Step 1: Create an Arc Gateway resource

If you don't already have an Arc Gateway resource, create one. You need the gateway resource ID when you connect the cluster in the next section. For creation steps, see [Create the Arc Gateway resource](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#create-the-arc-gateway-resource).

> [!NOTE]
> A maximum of five Arc Gateway resources are supported per subscription.

### Step 2: Retrieve the custom locations Object ID

The `--custom-locations-oid` parameter used when connecting the cluster requires the Object ID (OID) of the Azure Arc Custom Locations service principal.

To find it:

1. Go to **[Microsoft Entra ID](https://entra.microsoft.com)** in the Azure portal.
1. Select **Enterprise applications**.
1. Search for **Azure Arc Kubernetes Custom Locations**.
1. Open the application, go to **Properties**, and copy the **Object ID**.

For the list of FQDNs that must be allowed through your firewall when using Arc Gateway, see [Allowed endpoints with Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway).

## Create private endpoints and connect the cluster

Before deploying Azure IoT Operations, set up Private Endpoints and DNS zones so traffic to Azure services routes privately from the start. Choose the tab that matches your connectivity approach:

- **Arc Gateway only** — Create Private Endpoints for Storage and Key Vault. The cluster connects through Arc Gateway with a simplified firewall allowlist (~9 FQDNs), but outbound traffic still uses public internet paths.
- **Arc Gateway + Explicit Proxy** — Create Private Endpoints for Storage, Key Vault, and Event Grid. All outbound traffic routes through [Azure Firewall Explicit Proxy](/azure/azure-arc/azure-firewall-explicit-proxy) over your private network, with no public internet exposure.

> [!NOTE]
> Both tabs build on [Connect your cluster via Arc Gateway](#connect-your-cluster-via-arc-gateway). Complete that section first to create the Arc Gateway resource and retrieve the custom locations OID.

> [!NOTE]
> Some Azure subscriptions enforce policies that require storage accounts to disable shared key access. If you encounter a **RequestDisallowedByPolicy** error referencing "Storage accounts should prevent shared key access" during creation, add `--allow-shared-key-access false` to the `az storage account create` command or uncheck **Allow storage account key access** on the **Advanced** tab in the portal.

# [Arc Gateway only](#tab/arc-gateway-only)

### Step 1: Create Private Endpoints

Create Private Endpoints for the storage account and Key Vault.

#### Azure Blob Storage

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

#### Azure Key Vault

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

### Step 2: Configure Private DNS Zones

Create Private DNS Zones so Azure service FQDNs resolve to Private Endpoint IPs. Link each zone to your VNet and create DNS zone groups so the Private Endpoint A records are registered automatically.

#### Azure Blob Storage

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.blob.core.windows.net \
  --name storage-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-storage-blob \
  --name storage-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.blob.core.windows.net" \
  --zone-name blob
```

#### Azure Key Vault

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.vaultcore.azure.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.vaultcore.azure.net \
  --name keyvault-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-keyvault \
  --name keyvault-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net" \
  --zone-name vault
```

For the full list of private DNS zone names, see [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns).

### Step 3: Connect the cluster with Arc Gateway

Connect your cluster and associate it with the Arc Gateway:

```azurecli
az connectedk8s connect \
  --name <cluster-name> \
  --resource-group <resource-group> \
  --location <region> \
  --custom-locations-oid <OID> \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --disable-auto-upgrade \
  --gateway-resource-id <gateway-resource-id>
```

### Step 4: Verify connectivity

1. Confirm the Arc agents and Arc Proxy pod are running:

   ```bash
   kubectl get pods -n azure-arc
   ```

1. Verify DNS resolves to private IPs:

   ```bash
   nslookup <storage-account>.blob.core.windows.net
   nslookup <keyvault-name>.vault.azure.net
   ```

   Both should return IPs in your private address range (for example, `10.x.x.x`), not public IPs.

1. Verify the cluster appears as **Connected** in the Azure portal under **Azure Arc > Kubernetes clusters**.

> [!IMPORTANT]
> If any FQDN resolves to a public IP, check your Private DNS Zone linkage and VNet configuration before proceeding.

# [Arc Gateway + Explicit Proxy](#tab/arc-gateway-proxy)

This tab covers fully private connectivity with no public internet exposure. It requires an Azure Firewall with explicit proxy enabled in your VNet, reachable from your cluster over [ExpressRoute](/azure/expressroute/expressroute-introduction) or [VPN Gateway](/azure/vpn-gateway/vpn-gateway-about-vpngateways).

### Step 1: Create Private Endpoints

Create Private Endpoints for the storage account, Key Vault, and Event Grid so all traffic to these services routes privately through your VNet.

#### Azure Blob Storage

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

#### Azure Key Vault

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

#### Event Grid namespace

```azurecli
az network private-endpoint create \
  --name pe-eventgrid \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventGrid/namespaces/<namespace>" \
  --group-id topicspace \
  --connection-name pe-conn-eventgrid
```

### Step 2: Configure Private DNS Zones

Create Private DNS Zones so Azure service FQDNs resolve to Private Endpoint IPs. Link each zone to your VNet and create DNS zone groups so the Private Endpoint A records are registered automatically.

#### Azure Blob Storage

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.blob.core.windows.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.blob.core.windows.net \
  --name storage-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-storage-blob \
  --name storage-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.blob.core.windows.net" \
  --zone-name blob
```

#### Azure Key Vault

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.vaultcore.azure.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.vaultcore.azure.net \
  --name keyvault-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-keyvault \
  --name keyvault-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net" \
  --zone-name vault
```

#### Event Grid

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.ts.eventgrid.azure.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.ts.eventgrid.azure.net \
  --name eventgrid-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-eventgrid \
  --name eventgrid-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.ts.eventgrid.azure.net" \
  --zone-name eventgrid
```

For the full list of private DNS zone names, see [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns).

### Step 3: Set proxy environment variables

On the machine where you run the `az connectedk8s connect` command, set the proxy environment variables to point to the Azure Firewall Explicit Proxy:

```bash
export HTTPS_PROXY=http://<firewall-private-ip>:<port>
export HTTP_PROXY=http://<firewall-private-ip>:<port>
export NO_PROXY=localhost,127.0.0.1,.svc,.local,<cluster-subnet-cidr>
```

> [!NOTE]
> The `HTTPS_PROXY` and `HTTP_PROXY` values point to the Azure Firewall's private IP and explicit proxy port (for example, `http://10.254.0.68:8443`). Adjust `NO_PROXY` to include your cluster's internal CIDRs and any local domains that should bypass the proxy.

### Step 4: Connect the cluster with Arc Gateway and proxy

Connect the cluster, associating it with both the Arc Gateway and the explicit proxy:

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

This command configures all Arc traffic to route through the Azure Firewall Explicit Proxy and the Arc Gateway, consolidating ~200+ endpoints to ~9 allowed FQDNs with no public internet exposure.

### Step 5: Verify connectivity

1. Confirm the Arc Proxy pod is running:

   ```bash
   kubectl get pods -n azure-arc -l app.kubernetes.io/component=arc-proxy
   ```

1. Verify DNS resolves to private IPs:

   ```bash
   nslookup <storage-account>.blob.core.windows.net
   nslookup <keyvault-name>.vault.azure.net
   nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
   ```

   Each result should return an IP in your private address range, not a public IP.

1. Verify the cluster appears as **Connected** in the Azure portal under **Azure Arc > Kubernetes clusters**.

> [!IMPORTANT]
> If any FQDN resolves to a public IP, check your Private DNS Zone linkage and VNet configuration before proceeding.

---

## Deploy Azure IoT Operations

After creating Private Endpoints for storage and Key Vault, deploy Azure IoT Operations to your Arc-enabled cluster.

For deployment instructions, see [Deploy Azure IoT Operations](../deploy-iot-ops/howto-deploy-iot-operations.md). During deployment, Arc agent traffic routes through the connectivity options you configured (Arc Gateway, Explicit Proxy, or both). Storage and Key Vault traffic routes privately because DNS already resolves to the Private Endpoint IPs.

> [!NOTE]
> The storage account and Key Vault must have public access enabled during deployment. Schema Registry requires public access at creation time, and the initial secret sync needs to reach Key Vault. You disable public access in the next section after confirming AIO is healthy.

## Disable public access on storage and Key Vault

After AIO is deployed and healthy, disable public access on the storage account and Key Vault to complete the lockdown.

### Prerequisites

Before disabling public access, confirm the following:

- **Azure IoT Operations is deployed and healthy.** Run `az iot ops check` and verify all pods in the `azure-iot-operations` namespace are running. See [Deploy Azure IoT Operations](../deploy-iot-ops/howto-deploy-iot-operations.md).
- **Secret sync is configured and working.** Verify that SecretSync and SecretProviderClass resources exist and that secrets are syncing from Azure Key Vault. See [Manage secrets for your Azure IoT Operations deployment](../secure-iot-ops/howto-manage-secrets.md).
- **Schema Registry is functional.** Confirm the schema registry pods (`adr-schema-registry-*`) are running and can reach the storage account.
- **Cluster nodes can resolve Azure DNS.** If your cluster uses custom DNS, configure DNS forwarding to Azure DNS (`168.63.129.16`) so that Private DNS Zone records resolve correctly.

### Step 1: Disable public access on the storage account

Disable public network access on the storage account while keeping the trusted Azure services bypass. This allows Schema Registry (`Microsoft.DeviceRegistry/schemaRegistries`) to access the storage account through the trusted service path:

```azurecli
az storage account update \
  --name <storage-account> \
  --resource-group <resource-group> \
  --public-network-access Disabled \
  --bypass AzureServices
```

> [!NOTE]
> Schema Registry continues to function correctly through the trusted service bypass (`AzureServices`). Use the `--skip-ra` flag during Schema Registry creation if you don't have Owner-level permissions.

### Step 2: Assign RBAC roles for Schema Registry

The Schema Registry's managed identity needs access to the storage account:

```azurecli
az role assignment create \
  --assignee <identity-client-id> \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-resource-id>
```

### Step 3: Disable public access on Key Vault

```azurecli
az keyvault update \
  --name <keyvault-name> \
  --resource-group <resource-group> \
  --public-network-access Disabled
```

### Step 4: Verify private connectivity

1. From your cluster node, confirm the storage FQDN resolves to a private IP:

   ```bash
   nslookup <storage-account>.blob.core.windows.net
   ```

1. Confirm the Key Vault FQDN resolves to a private IP:

   ```bash
   nslookup <keyvault-name>.vault.azure.net
   ```

   Both should return IPs in your private address range (for example, `10.x.x.x`), not public IPs.

1. Verify that secret sync is still working after disabling public access:

   ```bash
   kubectl get secretsync -n azure-iot-operations
   ```

   All SecretSync resources should show a status of `Synced`. If any show errors, see [Troubleshooting](#troubleshooting).

## Configure data flow destinations with private endpoints

Azure IoT Operations [data flows](/azure/iot-operations/connect-to-cloud/overview-dataflow) send telemetry to cloud destinations like Azure Event Grid, Azure Event Hubs, Azure Data Explorer, Data Lake Storage Gen2, and Microsoft Fabric OneLake. By default, data flows connect to these services over their public endpoints. To keep traffic private, create Private Endpoints for each destination and ensure DNS resolves to the private IPs.

The following table shows supported data flow destinations and the Private DNS Zone, group ID, and port for each:

| Destination | Private DNS Zone | Group ID | Port |
|-------------|-----------------|----------|------|
| Azure Event Grid (MQTT) | `privatelink.ts.eventgrid.azure.net` | `topicspace` | 8883 |
| Azure Event Hubs | `privatelink.servicebus.windows.net` | `namespace` | 9093 (Kafka) |
| Azure Data Explorer | `privatelink.<region>.kusto.windows.net` | `cluster` | 443 |
| Data Lake Storage Gen2 | `privatelink.blob.core.windows.net` or `privatelink.dfs.core.windows.net` | `blob` or `dfs` | 443 |
| Microsoft Fabric OneLake | `privatelink.dfs.fabric.microsoft.com` | `onelake` | 443 |

The steps below use **Azure Event Grid** as the example. The same pattern applies to every destination — substitute the values from the table.

### Step 1: Create an Event Grid namespace

If you don't already have one, create an Event Grid namespace with MQTT (topic spaces) enabled:

```azurecli
az eventgrid namespace create \
  --name <namespace> \
  --resource-group <resource-group> \
  --location <region> \
  --topic-spaces-configuration state=Enabled \
  --sku name=Standard capacity=1
```

Then create a topic space. For testing, you can use the wildcard `#` as the topic template:

```azurecli
az eventgrid namespace topic-space create \
  --name <topic-space-name> \
  --resource-group <resource-group> \
  --namespace-name <namespace> \
  --topic-templates "#"
```

> [!NOTE]
> In the Event Grid namespace, set **Maximum client sessions per authentication name** to **3** or more so data flows can scale up. See [Event Grid MQTT multi-session support](/azure/event-grid/mqtt-establishing-multiple-sessions-per-client).

### Step 2: Assign RBAC for Event Grid

Grant the Azure IoT Operations managed identity permission to publish messages to the Event Grid MQTT broker:

```azurecli
az role assignment create \
  --assignee <aio-managed-identity-client-id> \
  --role "EventGrid TopicSpaces Publisher" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventGrid/namespaces/<namespace>"
```

If you also need the Data Flow to subscribe to messages, add `EventGrid TopicSpaces Subscriber`.

### Step 3: Create a Private Endpoint for the Event Grid namespace

If you used the **Arc Gateway + Explicit Proxy** tab and already created an Event Grid Private Endpoint, skip to [Step 5](#step-5-disable-public-access-on-the-event-grid-namespace).

```azurecli
az network private-endpoint create \
  --name pe-eventgrid \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventGrid/namespaces/<namespace>" \
  --group-id topicspace \
  --connection-name pe-conn-eventgrid
```

### Step 4: Create the Private DNS Zone and link it to your VNet

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.ts.eventgrid.azure.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.ts.eventgrid.azure.net \
  --name eventgrid-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-eventgrid \
  --name eventgrid-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.ts.eventgrid.azure.net" \
  --zone-name eventgrid
```

### Step 5: Disable public access on the Event Grid namespace

```azurecli
az eventgrid namespace update \
  --name <namespace> \
  --resource-group <resource-group> \
  --public-network-access Disabled
```

### Step 6: Verify DNS resolves to a private IP

From your cluster node (or a VM in the same VNet), confirm the FQDN resolves to the Private Endpoint IP:

```bash
nslookup <namespace>.<region>-1.ts.eventgrid.azure.net
```

The result should return an IP in your private address range (for example, `10.x.x.x`), not a public IP. If it returns a public IP, check your Private DNS Zone linkage.

### Step 7: Create the data flow endpoint for Event Grid

In the [Azure IoT Operations portal](https://iotoperations.azure.com), create an Event Grid MQTT data flow endpoint. Or use Azure CLI:

```azurecli
az iot ops dataflow endpoint create eventgrid \
  --resource-group <resource-group> \
  --instance <aio-instance-name> \
  --name eventgrid-private-endpoint \
  --host <namespace>.<region>-1.ts.eventgrid.azure.net
```

This creates an endpoint using system-assigned managed identity authentication. The host uses the Event Grid namespace's MQTT hostname on port 8883. No special configuration is needed for Private Link — the Data Flow resolves the FQDN through DNS, which returns the Private Endpoint IP if your DNS zones are configured correctly. For more information, see [Configure MQTT data flow endpoints for Event Grid](/azure/iot-operations/connect-to-cloud/howto-configure-mqtt-endpoint#azure-event-grid).

### Step 8: Create a data flow to test

Create a data flow that routes MQTT broker messages to the Event Grid destination.

In the [Azure IoT Operations portal](https://iotoperations.azure.com):

1. Select **Data flows** > **Create data flow**.
1. Set the **source** to the default MQTT broker endpoint.
1. Set the **destination** to the `eventgrid-private-endpoint` you created.
1. Set the destination topic to a topic that matches your topic space template.
1. Apply the data flow.

Or use Azure CLI with a JSON configuration file:

```azurecli
az iot ops dataflow apply \
  --resource-group <resource-group> \
  --instance <aio-instance-name> \
  --profile default \
  --name <dataflow-name> \
  --config-file <config-file-path>
```

The configuration file defines the source and destination for the data flow. For example, to route messages from the `test/eventgrid` topic on the local MQTT broker to the Event Grid endpoint:

```json
{
  "mode": "Enabled",
  "operations": [
    {
      "operationType": "Source",
      "sourceSettings": {
        "endpointRef": "default",
        "dataSources": [
          "test/eventgrid"
        ]
      }
    },
    {
      "operationType": "Destination",
      "destinationSettings": {
        "endpointRef": "eventgrid-private-endpoint",
        "dataDestination": "test/eventgrid"
      }
    }
  ]
}
```

### Step 9: Validate telemetry arrives at Event Grid

Publish a test message to the MQTT broker:

```bash
mqttui --broker mqtt://<cluster-host-ip>:1883
```

Then check the data flow is working:

1. Navigate to your Event Grid namespace in the Azure portal.
1. Check **Metrics** for incoming MQTT messages.
1. Verify the data flow pod logs show successful message delivery:

   ```bash
   kubectl logs -n azure-iot-operations -l app=dataflow --tail=50
   ```

If messages are flowing, the Data Flow is successfully routing through the Private Endpoint with managed identity auth. If messages don't arrive, check:

- The data flow pod logs: `kubectl logs -n azure-iot-operations -l app=dataflow`
- DNS resolution from within the cluster
- The Private Endpoint connection status in the Azure portal (should show **Approved**)
- RBAC assignments (the managed identity needs `EventGrid TopicSpaces Publisher` on the namespace)

## Verify Azure IoT Operations health after lockdown

After disabling public access on any Azure resource (Storage, Key Vault, Event Grid), verify that Azure IoT Operations is still healthy:

1. **Run the AIO health check:**

   ```azurecli
   az iot ops check
   ```

   All checks should pass. A warning about missing data flows is expected if you haven't created any yet.

1. **Verify all pods are running:**

   ```bash
   kubectl get pods -n azure-iot-operations
   ```

   All pods should be in `Running` or `Completed` state. Pay special attention to the schema registry pods (`adr-schema-registry-*`) and the secret store pods.

1. **Verify secret sync:**

   ```bash
   kubectl get secretsync -n azure-iot-operations
   ```

   All resources should show `Synced` status.

1. **Verify schema registry connectivity:**

   ```bash
   kubectl logs -n azure-iot-operations -l app=adr-schema-registry --tail=50
   ```

   Look for successful storage connections. Errors like `AuthorizationFailure` or `connection refused` indicate DNS or Private Endpoint misconfiguration.

## Troubleshooting

### DNS resolves to a public IP instead of a private IP

**Symptom:** `nslookup <service>.vault.azure.net` (or `.blob.core.windows.net`) returns a public IP.

**Cause:** The Private DNS Zone isn't linked to your VNet, or the DNS zone group wasn't created for the Private Endpoint.

**Fix:**
- Verify the Private DNS Zone exists: `az network private-dns zone list --resource-group <resource-group>`
- Verify the VNet link exists: `az network private-dns link vnet list --resource-group <resource-group> --zone-name <zone-name>`
- Verify the DNS zone group exists: `az network private-endpoint dns-zone-group list --resource-group <resource-group> --endpoint-name <pe-name>`
- If your cluster uses custom DNS (not Azure DNS), configure a DNS forwarder to `168.63.129.16` for the `privatelink.*` zones.

### SecretSync shows errors after disabling Key Vault public access

**Symptom:** `kubectl get secretsync -n azure-iot-operations` shows sync failures.

**Cause:** The Secret Store extension can't reach Key Vault because DNS doesn't resolve to the Private Endpoint IP, or the Private Endpoint connection isn't approved.

**Fix:**
- Verify DNS from inside the cluster: `kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup <keyvault-name>.vault.azure.net`
- Check the Private Endpoint connection status in the Azure portal — it should show **Approved**.
- Verify the user-assigned managed identity still has the `Key Vault Secrets User` role on the Key Vault.

### Schema Registry can't reach storage after disabling public access

**Symptom:** Schema registry pods log `AuthorizationFailure` or connectivity errors. Schema operations fail.

**Cause:** The storage account's trusted service bypass isn't configured, the Private Endpoint DNS isn't resolving, or the managed identity lacks the `Storage Blob Data Contributor` role.

**Fix:**
- Verify the bypass is set: `az storage account show --name <account> --query networkRuleSet`
- Verify DNS resolves to a private IP from the cluster.
- Restart the schema registry pods after fixing: `kubectl delete pods -n azure-iot-operations -l app=adr-schema-registry`

### Data flow can't reach Event Grid after disabling public access

**Symptom:** Data flow pod logs show connection refused or timeout errors to the Event Grid namespace.

**Cause:** DNS doesn't resolve the Event Grid FQDN to the Private Endpoint IP, or the Private Endpoint connection isn't approved.

**Fix:**
- Verify DNS from the cluster: `kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup <namespace>.<region>-1.ts.eventgrid.azure.net`
- Check the Private Endpoint connection status in the Azure portal.
- Verify RBAC: the AIO managed identity needs `EventGrid TopicSpaces Publisher` on the namespace.
- Check data flow pod logs: `kubectl logs -n azure-iot-operations -l app=dataflow`

## Known limitations

- **Platform validation:** The private connectivity patterns described here are based on validated K3s on Ubuntu Server 24.04 scenarios. Other Kubernetes distributions or operating systems haven't been independently validated.
- **Schema Registry creation:** Schema Registry may require public access enabled at creation time. After creation, you can disable public access and rely on Private Endpoints plus trusted service bypass. Use the `--skip-ra` flag when creating the Schema Registry to avoid requiring Owner-level permissions.
- **TLS inspection:** Arc Gateway doesn't support TLS termination or inspection. If your firewall performs TLS inspection, you must exclude the Arc Gateway endpoint from inspection. See [Arc Gateway and TLS inspection](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#azure-arc-gateway-and-tls-inspection).
- **Arc Gateway limits:** A maximum of five Arc Gateway resources are supported per subscription.
- **Explicit Proxy:** Only Azure Firewall Explicit Proxy has been validated. Third-party proxies (for example, Palo Alto) or transparent proxies aren't supported in validated scenarios. Azure IoT Operations doesn't support proxy servers that require a trusted certificate.

## Related content

- [Simplify network configuration requirements with Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking)
- [Access Azure services over Azure Firewall Explicit Proxy](/azure/azure-arc/azure-firewall-explicit-proxy)
- [Configure a data flow endpoint](/azure/iot-operations/connect-to-cloud/howto-configure-dataflow-endpoint)
- [Schema Registry](/azure/iot-operations/connect-to-cloud/concept-schema-registry)
- [Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity](../end-to-end-tutorials/tutorial-layered-network-private-connectivity.md)
- [Azure IoT Operations networking](overview-layered-network.md)
- [Deploy Azure IoT Operations](/azure/iot-operations/deploy-iot-ops/overview-deploy)
- [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns)
