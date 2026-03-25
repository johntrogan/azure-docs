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

This article describes how to configure private connectivity for Azure IoT Operations. It covers four scenarios that you can use independently or combine:

| Scenario | When to apply | What it does |
|----------|--------------|-------------|
| [Connect your cluster via Arc Gateway](#connect-your-cluster-via-arc-gateway) | During Arc onboarding | Reduce the ~200+ Azure endpoints your cluster needs to reach down to ~9 FQDNs |
| [Use Arc Gateway with explicit proxy for private connection](#use-arc-gateway-with-explicit-proxy-for-private-connection) | During Arc onboarding | Combine Arc Gateway with Azure Firewall Explicit Proxy so all Azure traffic stays on private networks |
| [Use a private storage account](#use-a-private-storage-account) | After AIO deployment | Restrict the storage account used by Schema Registry so it isn't exposed to the public internet |
| [Configure data flow destinations with private endpoints](#configure-data-flow-destinations-with-private-endpoints) | After AIO deployment | Route data flow traffic to cloud destinations like Event Hubs and Azure Data Explorer through Private Link |

These scenarios apply to environments with a single Arc-enabled Kubernetes cluster. There's no Purdue-style network segmentation, no proxy chaining across layers, and no Envoy deployment. If you have a layered network topology, see [Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity](../end-to-end-tutorials/tutorial-layered-network-private-connectivity.md) instead.

## Prerequisites

- An [Azure subscription](/azure/cost-management-billing/manage/create-subscription). If you don't have one, [create a free account](https://azure.microsoft.com/free/) before you begin.
- Sufficient permissions to create Private Endpoints, Private DNS Zones, and role assignments in your subscription (typically **Owner** or **Contributor** + **User Access Administrator**).
- A Kubernetes cluster deployed and ready to Arc-enable. See [Prepare your cluster](/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster) for supported configurations and setup steps.
- An Azure VNet where you create Private Endpoints and Private DNS Zones.
- Network connectivity from your on-premises cluster to the Azure VNet ([ExpressRoute](/azure/expressroute/expressroute-introduction), [VPN Gateway](/azure/vpn-gateway/vpn-gateway-about-vpngateways), VNet peering, or other private routing).
- [Azure CLI](/cli/azure/install-azure-cli) and [kubectl](https://kubernetes.io/docs/tasks/tools/) installed on your admin or jump machine.


## Connect your cluster via Arc Gateway

[Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) consolidates the ~200+ Azure endpoints that Arc agents and extensions require into a single gateway URL. This significantly simplifies your firewall allowlist — instead of allowing 200+ individual FQDNs, you allow approximately 9.

Arc Gateway works by deploying two components:

1. **Arc Gateway (Azure resource):** A cloud-side resource that provides a single gateway URL (`<gateway-name>.gw.arc.azure.com`). This URL acts as the entry point for all Arc management traffic.
2. **Arc Proxy (in-cluster pod):** When you connect a cluster with `--gateway-resource-id`, the Arc agents deploy an Arc Proxy pod. All Arc agent and extension traffic routes through this pod to the gateway.

### Step 1: Create an Arc Gateway resource

If you don't already have an Arc Gateway resource, create one. For creation steps, see [Create the Arc Gateway resource](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#create-the-arc-gateway-resource).

> [!NOTE]
> A maximum of five Arc Gateway resources are supported per subscription.

### Step 2: Retrieve the custom locations Object ID

The `--custom-locations-oid` parameter requires the Object ID (OID) of the Azure Arc Custom Locations service principal.

To find it:

1. Go to **Microsoft Entra ID** in the Azure portal.
1. Select **Enterprise applications**.
1. Search for **Azure Arc Kubernetes Custom Locations**.
1. Open the application, go to **Properties**, and copy the **Object ID**.

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

This command registers the cluster with Azure Arc, deploys Arc agents and the Arc Proxy pod, and routes all Arc traffic through the gateway.

For the list of FQDNs that must be allowed through your firewall when using Arc Gateway, see [Allowed endpoints with Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking#allowed-endpoints-with-arc-gateway).

### Step 4: Verify Arc Gateway connectivity

1. Confirm the Arc agents and Arc Proxy pod are running:

   ```bash
   kubectl get pods -n azure-arc
   ```

1. Check the Arc Proxy pod logs to confirm traffic routes through the gateway:

   ```bash
   kubectl logs -n azure-arc -l app.kubernetes.io/component=arc-proxy
   ```

1. Verify the cluster appears as **Connected** in the Azure portal under **Azure Arc > Kubernetes clusters**.

## Use Arc Gateway with explicit proxy for private connection

To achieve fully private connectivity where no traffic leaves your private network for the public internet, combine Arc Gateway with [Azure Firewall Explicit Proxy](/azure/azure-arc/azure-firewall-explicit-proxy). The explicit proxy acts as a forward proxy for all outbound traffic, routing it through your private network to Azure services.

This scenario requires:

- **Arc Gateway** to consolidate Azure endpoints (as described in the [previous section](#connect-your-cluster-via-arc-gateway))
- **Azure Firewall Explicit Proxy** deployed in your Azure VNet, reachable from your cluster over a private connection
- **Private Endpoints** for Azure services that support Private Link (Event Grid, Storage, Key Vault)
- **Network connectivity** from your cluster to the Azure Firewall's private IP, this can be [ExpressRoute](/azure/expressroute/expressroute-introduction), [VPN Gateway](/azure/vpn-gateway/vpn-gateway-about-vpngateways), or any private routing that provides reachability

The resulting traffic flow is:

- **Arc management traffic:** Arc agents → Arc Proxy pod → Azure Firewall Explicit Proxy → Arc Gateway → Azure management services
- **Data-plane traffic:** Azure IoT Operations components → Azure Firewall Explicit Proxy → Private Endpoints (Event Grid, Storage, Key Vault)

With this architecture, you reduce your firewall allowlist to approximately 9 FQDNs and keep all traffic on private networks.

### Step 1: Create Private Endpoints for core Azure services

Create Private Endpoints for Event Grid, Storage, and Key Vault as described in [Use a private storage account](#step-1-create-a-private-endpoint-for-the-storage-account). Additionally, create endpoints for Event Grid and Key Vault:

#### Event Grid namespace

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

Create Private DNS Zones so Azure service FQDNs resolve to Private Endpoint IPs. Link each zone to your VNet:

| Service | Private DNS Zone |
|---------|-----------------|
| Event Grid | `privatelink.ts.eventgrid.azure.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |

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

### Step 5: Verify private connectivity

1. Confirm the Arc Proxy pod is running:

   ```bash
   kubectl get pods -n azure-arc -l app.kubernetes.io/component=arc-proxy
   ```

1. Verify DNS resolves to private IPs:

   ```bash
   nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
   nslookup <storage-account>.blob.core.windows.net
   nslookup <keyvault-name>.vault.azure.net
   ```

   Each result should return an IP in your private address range, not a public IP.

1. Verify the cluster appears as **Connected** in the Azure portal under **Azure Arc > Kubernetes clusters**.

> [!IMPORTANT]
> If any FQDN resolves to a public IP, check your Private DNS Zone linkage and VNet configuration before proceeding.
> 

## Deploy Azure IoT Operations
After configuring private connectivity for the scenarios that apply to your environment, deploy Azure IoT Operations to your Arc-enabled cluster.

For deployment instructions, see [Deploy Azure IoT Operations](../deploy-iot-ops/howto-deploy-iot-ops.md). During deployment, the Arc agents will route through the Azure Firewall Explicit Proxy to reach Azure services, and data-plane traffic will route through Private Endpoints, ensuring a fully private connection.

## Use a private storage account

Azure IoT Operations uses an Azure Storage account with [hierarchical namespace enabled](/azure/storage/blobs/data-lake-storage-namespace) (Data Lake Storage Gen2) to store schemas for the [Schema Registry](/azure/iot-operations/connect-to-cloud/concept-schema-registry). By default, this storage account is publicly accessible. To restrict access, you can use Private Link and configure the trusted service bypass so Schema Registry can still reach the storage account.

### Step 1: Create a Private Endpoint for the storage account

Create a Private Endpoint for the storage account's blob service:

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

### Step 2: Create a Private DNS Zone for Blob Storage

Create a Private DNS Zone and link it to your VNet so the storage FQDN resolves to the Private Endpoint IP:

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
```

### Step 3: Disable public access and enable trusted service bypass

Disable public network access on the storage account while keeping the trusted Azure services bypass. This allows Schema Registry (`Microsoft.DeviceRegistry/schemaRegistries`) to access the storage account through the trusted service path:

```azurecli
az storage account update \
  --name <storage-account> \
  --resource-group <resource-group> \
  --public-network-access Disabled \
  --bypass AzureServices
```

<!-- TODO: Validate that Schema Registry functions correctly when storage has publicNetworkAccess=Disabled + bypass=AzureServices + Private Endpoint. Some discussions and customer evidence (Husqvarna) indicate this works at the ARM/platform level, but it needs explicit testing on a demo cluster. If validated, also update concept-production-guidelines.md lines 90-91 which currently say public access must be enabled. -->

> [!IMPORTANT]
> The Schema Registry may require public access during initial creation of the storage account. After the Schema Registry is created, you can disable public access as shown above. Use the `--skip-ra` flag when creating the Schema Registry to avoid requiring Owner-level permissions.

### Step 4: Assign RBAC roles for Schema Registry

The Schema Registry's managed identity needs access to the storage account:

```azurecli
az role assignment create \
  --assignee <identity-client-id> \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-resource-id>
```

### Step 5: Verify private connectivity

From your cluster node, confirm the storage FQDN resolves to a private IP:

```bash
nslookup <storage-account>.blob.core.windows.net
```

The result should return an IP in your private address range (for example, `10.x.x.x`), not a public IP.

## Configure data flow destinations with private endpoints

Azure IoT Operations [data flows](/azure/iot-operations/connect-to-cloud/overview-dataflow) send telemetry to cloud destinations like Azure Event Hubs, Azure Data Explorer, Data Lake Storage Gen2, and Microsoft Fabric OneLake. By default, data flows connect to these services over their public endpoints. To keep traffic private, create Private Endpoints for each destination and ensure DNS resolves to the private IPs.

<!-- TODO: Validate end-to-end data flow connectivity through Private Endpoints for each destination type on a demo cluster. The Azure networking steps below follow standard PE patterns and are expected to be correct. What needs validation is whether the Data Flow runtime honors PE DNS resolution and whether managed identity token exchange works when the destination has no public endpoint. -->

The following table shows supported data flow destinations and the Private DNS Zone, group ID, and port for each:

| Destination | Private DNS Zone | Group ID | Port |
|-------------|-----------------|----------|------|
| Azure Event Hubs | `privatelink.servicebus.windows.net` | `namespace` | 9093 (Kafka) |
| Azure Data Explorer | `privatelink.<region>.kusto.windows.net` | `cluster` | 443 |
| Data Lake Storage Gen2 | `privatelink.blob.core.windows.net` or `privatelink.dfs.core.windows.net` | `blob` or `dfs` | 443 |
| Microsoft Fabric OneLake | `privatelink.dfs.fabric.microsoft.com` | `onelake` | 443 |

The steps below use **Azure Event Hubs** as the example. The same pattern applies to every destination — substitute the values from the table.

### Step 1: Create an Event Hubs namespace and event hub

If you don't already have one, create a Kafka-enabled Event Hubs namespace and an event hub (topic):

```azurecli
az eventhubs namespace create \
  --name <namespace> \
  --resource-group <resource-group> \
  --location <region> \
  --sku Standard \
  --enable-kafka true

az eventhubs eventhub create \
  --name <eventhub-name> \
  --namespace-name <namespace> \
  --resource-group <resource-group> \
  --partition-count 2
```

### Step 2: Assign RBAC for Event Hubs

Grant the Azure IoT Operations managed identity permission to send data:

```azurecli
az role assignment create \
  --assignee <aio-managed-identity-client-id> \
  --role "Azure Event Hubs Data Sender" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventHub/namespaces/<namespace>"
```

If you also need the Data Flow to read from Event Hubs, add `Azure Event Hubs Data Receiver`.

### Step 3: Create a Private Endpoint for the Event Hubs namespace

```azurecli
az network private-endpoint create \
  --name pe-eventhubs \
  --resource-group <resource-group> \
  --location <region-of-vnet> \
  --subnet "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>/subnets/<subnet>" \
  --private-connection-resource-id "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.EventHub/namespaces/<namespace>" \
  --group-id namespace \
  --connection-name pe-conn-eventhubs
```

### Step 4: Create the Private DNS Zone and link it to your VNet

```azurecli
az network private-dns zone create \
  --resource-group <resource-group> \
  --name privatelink.servicebus.windows.net

az network private-dns link vnet create \
  --resource-group <resource-group> \
  --zone-name privatelink.servicebus.windows.net \
  --name eventhubs-dns-link \
  --virtual-network "/subscriptions/<subscription-id>/resourceGroups/<rg-vnet>/providers/Microsoft.Network/virtualNetworks/<vnet>" \
  --registration-enabled false

az network private-endpoint dns-zone-group create \
  --resource-group <resource-group> \
  --endpoint-name pe-eventhubs \
  --name eventhubs-zone-group \
  --private-dns-zone "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/privateDnsZones/privatelink.servicebus.windows.net" \
  --zone-name servicebus
```

### Step 5: Disable public access on the Event Hubs namespace

```azurecli
az eventhubs namespace update \
  --name <namespace> \
  --resource-group <resource-group> \
  --public-network-access Disabled
```

### Step 6: Verify DNS resolves to a private IP

From your cluster node (or a VM in the same VNet), confirm the FQDN resolves to the Private Endpoint IP:

```bash
nslookup <namespace>.servicebus.windows.net
```

The result should return an IP in your private address range (for example, `10.x.x.x`), not a public IP. If it returns a public IP, check your Private DNS Zone linkage.

### Step 7: Create the data flow endpoint for Event Hubs

In the [Azure IoT Operations portal](https://iotoperations.azure.com), create an Event Hubs data flow endpoint. Or use Azure CLI:

```azurecli
az iot ops dataflow endpoint create eventhub \
  --resource-group <resource-group> \
  --instance <aio-instance-name> \
  --name eventhubs-private-endpoint \
  --eventhub-namespace <namespace>
```

This creates an endpoint using system-assigned managed identity authentication. The `host` is set to `<namespace>.servicebus.windows.net:9093` (Kafka protocol). No special configuration is needed for Private Link — the Data Flow resolves the FQDN through DNS, which returns the Private Endpoint IP if your DNS zones are configured correctly.

### Step 8: Create a data flow to test

Create a data flow that routes MQTT broker messages to the Event Hubs destination:

1. In the [Azure IoT Operations portal](https://iotoperations.azure.com), select **Data flows** > **Create data flow**.
1. Set the **source** to the default MQTT broker endpoint.
1. Set the **destination** to the `eventhubs-private-endpoint` you created.
1. Set the destination topic to `<eventhub-name>` (the event hub you created in Step 1).
1. Apply the data flow.

### Step 9: Validate telemetry arrives at Event Hubs

Publish a test message to the MQTT broker:

```bash
mqttui --broker mqtt://<cluster-host-ip>:1883
```

Then confirm the message arrives in Event Hubs using the Azure portal:

1. Navigate to your Event Hubs namespace > **Event Hubs** > `<eventhub-name>`.
1. Check **Overview** for incoming message count.
1. Optionally use [Stream Analytics](/azure/stream-analytics/stream-analytics-introduction) or the built-in data explorer to inspect the messages.

If messages arrive, the Data Flow is successfully routing through the Private Endpoint with managed identity auth. If messages don't arrive, check:

- The data flow pod logs: `kubectl logs -n azure-iot-operations -l app=dataflow`
- DNS resolution from within the cluster
- The Private Endpoint connection status in the Azure portal (should show **Approved**)
- RBAC assignments (the managed identity needs `Azure Event Hubs Data Sender`)

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
