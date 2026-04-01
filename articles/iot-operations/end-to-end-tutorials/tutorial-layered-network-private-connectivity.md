---
title: "Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity"
description: "Tutorial: Deploy Azure IoT Operations in a Purdue/ISA-95 layered network with private Azure connectivity using ExpressRoute, Private Link, and explicit proxy routing."
author: david-emakenemi
ms.author: demakenemi
ms.topic: tutorial
ms.date: 03/24/2026

#CustomerIntent: As an operator in an industrial environment with Purdue-style network segmentation, I want to deploy Azure IoT Operations with private Azure connectivity so that no endpoints are exposed to the public internet.
ms.service: azure-iot-operations
---

# Tutorial: Deploy Azure IoT Operations in a layered network with private connectivity

This tutorial describes how to deploy Azure IoT Operations in a physically layered network topology with explicit proxy routing and private connectivity to Azure services via ExpressRoute. This deployment uses Private Link to reach services like Event Grid, and exposes no public endpoints at any network layer.

This scenario was validated using physical machines in a Purdue/ISA-95 segmented network spanning Levels 2 through 4. The Azure Firewall Explicit Proxy is deployed in an Azure VNet, with connectivity provided via ExpressRoute.

In this article, you:

- Create Azure resources and prepare a layered network environment
- Prepare Kubernetes clusters at each network layer
- Configure private endpoints and DNS for Azure services
- Arc-enable clusters with Arc Gateway and explicit proxy routing
- Deploy Azure IoT Operations on Arc-enabled clusters
- Assign RBAC roles required by Azure IoT Operations components
- Deploy CoreDNS, Envoy Proxy, MQTT broker, and data flows across network layers
- Validate end-to-end telemetry flow from OPC UA sources to Azure Event Grid
- Audit and verify network isolation, private connectivity, and RBAC assignments

## Prerequisites

- An [Azure subscription](/azure/cost-management-billing/manage/create-subscription). If you don't have one, [create a free account](https://azure.microsoft.com/free/) before you begin.
- Your [tenant ID](/azure/azure-portal/get-subscription-tenant-id).
- Sufficient permissions to create Private Endpoints, Private DNS Zones, and role assignments (typically **Owner** or **Contributor** + **User Access Administrator**). This tutorial uses custom roles defined in the [Appendix](#appendix): **ACX–Secrets Store Extension Owner** and **AdaptiveCloud_AIO–Contributors**.
- A [Kubernetes cluster](/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster) deployed at each network layer (Level 2, Level 3, and Level 4), with devices or VMs assigned static IPs.
- Network segmentation between layers (for example, firewalls allowing only L2 ↔ L3 ↔ L4 communication) and DNS resolution across layers using CoreDNS.
- [Azure Private Endpoints](/azure/private-link/private-endpoint-overview) for Event Grid, Storage (for Schema Registry), and Key Vault, assigned private IPs and accessible via [ExpressRoute](/azure/expressroute/expressroute-introduction) or equivalent private routing.
- [Azure Firewall Explicit Proxy](/azure/firewall/explicit-proxy) at Level 4 (ports 8080/8443), reachable from Level 4 over ExpressRoute. All outbound HTTP/HTTPS traffic from Level 4 flows through this proxy.
- [Azure CLI](/cli/azure/install-azure-cli), [kubectl](https://kubernetes.io/docs/tasks/tools/), and [Docker](https://docs.docker.com/get-docker/) installed on your admin or jump machine.

> [!NOTE]
> In the validated telemetry flow, only HTTPS (port 8443) was used. In customer environments, Level 4 may route through your own enterprise proxy instead.

## Architecture summary

This deployment aligns with the Purdue Model, implementing a physically segmented, multi-level architecture spanning Levels 2 through 4.

Each level is separated by network firewalls that restrict communication to adjacent layers only (for example, L2 ↔ L3 ↔ L4), ensuring tight segmentation. Outbound traffic to Azure is proxied through an Arc Gateway and routed to Azure Event Grid over Private Link, ensuring no internet-exposed endpoints are used at any layer.

:::image type="content" source="media/layered-network-private-connectivity-architecture.png" alt-text="Diagram showing Layered Network Guidance for Azure IoT Operations in segmented industrial-style network environments, with a Purdue model pyramid spanning Levels 2 through 5 on the left and an Azure Arc architecture on the right showing CoreDNS, Envoy, and Azure IoT Operations deployed across Levels 3 and 4.":::

### Validated components

| Layer | Components | Purpose |
| ----- | ---------- | ------- |
| L2 | CoreDNS, Azure IoT Operations Dataflows, Azure IoT Operations MQTT Broker | Ingests telemetry from OPC UA sources, applies initial enrichment, and forwards data upward |
| L3 | CoreDNS, Envoy Proxy, Azure IoT Operations Dataflows, Azure IoT Operations MQTT Broker | Aggregates and transforms data, resolves DNS to reach L4, and securely forwards telemetry |
| L4 | Envoy Proxy | Forwards enriched telemetry to Event Grid via Azure Firewall Explicit Proxy and Private Endpoint over ExpressRoute |

## Prepare your layered network environment

Each layer of the network (Level 2, 3, and 4) uses static IPs and strict firewall rules to enforce isolation. Each layer only communicates with its adjacent layer (for example, L2 ↔ L3 ↔ L4), implementing the Purdue model zones.

### Step 1: Create Azure resources

Before deploying to the edge, create the following Azure resources:

- [Resource group(s)](/azure/azure-resource-manager/management/manage-resource-groups-portal)
- [Azure Blob Storage](/azure/storage/blobs/storage-quickstart-blobs-portal) account with containers for schemas
- [Azure Key Vault](/azure/key-vault/general/quick-create-portal)
- [Event Grid Namespace and Topic Space](/azure/event-grid/create-view-manage-namespaces)
- [Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) resource

After creating these resources, disable public network access where allowed.

> [!IMPORTANT]
> Schema Registry has a known limitation with disabling public access at creation time. See [Known limitations](#known-limitations) for details.

### Step 2: Assign static IPs for each layer

Deploy one machine (physical or virtual) per network layer, with the following requirements:

- Assign static IPs within a shared address space.
- All devices run Ubuntu Server 24.04 with K3s clusters preinstalled.
- Devices on L2 and L3 are Arc-enabled and host their respective Azure IoT Operations instances.

> [!NOTE]
> IPs shown here are examples from the validation lab and are not internet accessible. Replace with IPs appropriate to your own network.

| Layer | Purpose | Example Hostname | Example IP | Notes |
| ----- | ------- | ---------------- | ---------- | ----- |
| L2 | OPC UA simulator, Azure IoT Operations (MQTT Broker, Dataflows), Arc, CoreDNS | p3tiny-01 | 172.22.232.X | Arc-enabled, Azure IoT Operations deployed |
| L3 | Azure IoT Operations (MQTT Broker, Dataflows), CoreDNS, Arc | p3tiny-02 | 172.22.232.Y | Arc-enabled, Azure IoT Operations deployed |
| L4 | Envoy Proxy (egress only, outbound access) | p3tiny-03 | 172.22.232.Z | Not Arc-enabled, handles egress via Azure Firewall Explicit Proxy over ExpressRoute |

### Step 3: Enforce network isolation between layers

Use firewalls or host-level policies to enforce adjacent-only communication.

| Communication Path | Access |
| ------------------ | ------ |
| L2 ↔ L3 | Allow |
| L3 ↔ L4 | Allow |
| L2 ↔ L4 | Block |
| L2/L3 → Internet | Block |
| L4 → Azure | Allow via Azure Firewall Explicit Proxy over ExpressRoute |

### Step 4: Route Azure-bound traffic through L4 only

Only the Level 4 node may initiate outbound traffic, forwarding it to the Azure Firewall Explicit Proxy over ExpressRoute, which then routes it to Azure services via Private Link.

1. Deploy Envoy Proxy on the L4 machine.
1. Forward HTTPS traffic to the Azure Firewall Explicit Proxy:
   - IP: `10.254.0.68`
   - Ports: `8080` (HTTP), `8443` (HTTPS)
1. Route outbound Azure traffic through Private Link.

> [!NOTE]
> This validation used HTTP(S) traffic through the proxy. If your proxy supports MQTT or other non-HTTP protocols, those can also be used in a similar configuration.

## Prepare Kubernetes clusters

Deploy a K3s cluster on each machine at Levels 2, 3, and 4. For instructions, see [Prepare your Azure Arc-enabled Kubernetes cluster](/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster).

After installation, verify each cluster is ready:

```bash
kubectl get nodes
```

> [!NOTE]
> Level 4 only runs Envoy Proxy and doesn't require Azure IoT Operations or Arc. A lightweight K3s install is sufficient.

## Configure Private Link and DNS

Configure Azure Private Link to connect securely to Event Grid and Azure Storage, using Private Endpoints and CoreDNS-based name resolution. All traffic to these services remains on private IPs, with no internet exposure.

### Step 1: Create Private Endpoints for Event Grid and Azure Storage

Create a private endpoint for the Event Grid namespace:

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

Create a private endpoint for the Azure Storage account:

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

### Step 2: Create Private DNS Zones

For each service, create the appropriate Azure Private DNS Zone. Only the Level 4 virtual network is linked to these Private DNS Zones. CoreDNS at L3 (and optionally L2) forwards requests to Azure's internal DNS resolver (`168.63.129.16`), which resolves names based on the L4 zone's DNS zone linkage.

| Service | Private DNS Zone |
| ------- | ---------------- |
| Event Grid | `privatelink.ts.eventgrid.azure.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |

For the full list of private DNS zone names, see [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns).

### Step 3: Enable DNS resolution for Azure Private Endpoints

Deploy CoreDNS on L2 and L3. Forward private Azure domain queries to `168.63.129.16`, which is Azure's internal DNS resolver used for resolving Private Endpoint domains.

Example CoreDNS forwarding rules:

```yaml
eventgrid.azure.net {
  forward . 168.63.129.16
}
blob.core.windows.net {
  forward . 168.63.129.16
}
```

> [!NOTE]
> This configuration ensures that Event Grid and Storage resolve to Private Endpoint IPs only.

### Step 4: Verify DNS resolution and connectivity

From L4, confirm DNS resolution:

```bash
nslookup <eventgrid-namespace>.privatelink.eventgrid.azure.net
nslookup <account>.privatelink.blob.core.windows.net
```

Confirm traffic flows via Envoy and Private Link, not the public internet:

```bash
curl -v https://<eventgrid-namespace>.ts.eventgrid.azure.net
```

Verify that:

- FQDNs resolve to private IPs.
- Traffic flows through Envoy and Private Link, not public endpoints.

> [!NOTE]
> Arc requires working DNS resolution (via CoreDNS) to complete onboarding.

## Arc-enable clusters with Arc Gateway

With DNS resolution and private connectivity in place, Arc-enable your Kubernetes clusters behind the explicit proxy. This step connects each cluster to Azure Arc and associates it with the Arc Gateway resource created in [Step 1](#step-1-create-azure-resources).

1. Configure the connected cluster so all outbound Arc traffic routes through the proxy.
1. Pass the Arc Gateway resource ID to the `az connectedk8s connect` or `az connectedk8s update` command.

### Step 1: Set proxy environment variables

On the Arc Gateway VM hosting the Azure Arc services, set the proxy environment variables. The `HTTPS_PROXY` variable must point to your network's firewall explicit proxy:

```bash
export HTTPS_PROXY=http://<proxy-server>:<port>
export NO_PROXY=localhost,127.0.0.1,.svc,.local,<your-private-DNS-zone>
```

### Step 2: Retrieve the service principal Object ID

The `--custom-locations-oid` parameter requires the Object ID (OID) of the Azure Arc Custom Locations service principal.

To find it in the Azure portal:

1. Go to **Microsoft Entra ID**.
1. Select **Enterprise applications**.
1. Search for **Azure Arc Kubernetes Custom Locations**.
1. Open the application, go to **Properties**, and copy the **Object ID**.

### Step 3: Connect the cluster with Arc Gateway

Connect the cluster behind the proxy and associate it with the Arc Gateway:

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

> [!NOTE]
> Omit `--gateway-resource-id` if you aren't using Arc Gateway (for example, if you use ExpressRoute with Private Endpoints only).

### Step 4: Verify Arc connectivity

1. Run `kubectl logs` on the Arc gateway pod to confirm it reaches Azure.
1. Verify that DNS resolution and TLS handshake are successful through the proxy.

## Deploy Azure IoT Operations

With Arc-enabled clusters at L2 and L3, deploy Azure IoT Operations on each. This creates the system-assigned managed identity needed for RBAC assignments in the next section.

For full instructions, see [Deploy Azure IoT Operations](/azure/iot-operations/deploy-iot-ops/howto-deploy-iot-operations).

> [!IMPORTANT]
> Use the `--skip-ra` flag when creating the Schema Registry. This prevents the CLI from attempting role assignments that require Owner rights. See [Known limitations](#known-limitations) for details.

After deployment, verify that all pods are running:

```bash
kubectl get pods -n azure-iot-operations
```

## Assign RBAC roles

Azure IoT Operations requires specific role-based access control (RBAC) assignments to allow components to interact with Azure services like Blob Storage and Event Grid.

> [!NOTE]
> These permissions were manually assigned in the lab by admins with elevated privileges. For production, use Azure Policy automation.

### Required role assignments

| Identity | Role | Scope | Notes |
| -------- | ---- | ----- | ----- |
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

### Layered enrichment context

Each Azure IoT Operations network layer contributes business metadata to outbound telemetry:

| Layer | Adds Metadata Field |
| ----- | ------------------- |
| L2 | `product` |
| L3 | `line-config` |
| L4 | `factory-code` |

This layered enrichment allows downstream systems to understand manufacturing context without burdening edge devices with full metadata.

## Deploy components by layer

Deploy the following components to each network layer:

- **CoreDNS (L2, L3):** Deploy on Levels 2 and 3 as described in [Configure infrastructure](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/configure-infrastructure.md).
- **Envoy Proxy (L4):** Deploy on Level 4 as described in [Configure infrastructure](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/configure-infrastructure.md). This proxy handles all outbound traffic to Azure via the Azure Firewall Explicit Proxy and Private Link.
- **Azure IoT Operations MQTT Broker and Dataflows (L2, L3):** Deploy on Levels 2 and 3 as described in [Asset telemetry](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/asset-telemetry.md).

### Verify deployment

**CoreDNS:** Run `dig` to confirm resolution to the appropriate Envoy IP at the adjacent layer:

```bash
kubectl config use-context level3
kubectl get pods -l app=<envoy-proxy>
kubectl config use-context level4
kubectl get pods -l app=<envoy-proxy>
```

**Envoy Proxy:** Check pods are running:

```bash
kubectl config use-context <level>
kubectl get pods -l app=<envoy-proxy-pod>
```

**Azure IoT Operations MQTT Broker:** Verify listeners are active:

```bash
kubectl get service <k3s-service-name> -n azure-iot-operations
```

**Azure IoT Operations Dataflows:** Check telemetry with `mqttui` and confirm metadata in Event Grid Topic Spaces:

```bash
mqttui --broker mqtt://<level-host-ip>:1883
```

Confirm that enrichment metadata (for example, `product: flakes`, `line-config: cereal`, `factory-code: 1032`) appears in Event Grid Topic Spaces using the Azure portal or CLI.

## Validate telemetry flow

### Connectivity path

The end-to-end telemetry flow follows this path:

1. **Telemetry ingestion (L2):** OPC UA connectors at L2 publish telemetry to Azure IoT Operations Dataflows, which forward it to the MQTT Broker.
1. **L2 to L3:** MQTT Broker at L2 publishes messages to L3 using MQTT.
1. **L3 to L4:** MQTT Broker at L3 publishes messages upstream using MQTT over WebSocket.
1. **DNS resolution (L3):** Azure IoT Operations Dataflows and CoreDNS at L3 resolve private service names to reach L4.
1. **Proxy forwarding (L3 to L4):** Envoy Proxy on L3 forwards MQTT traffic to Envoy Proxy on L4.
1. **Egress (L4):** Envoy Proxy on L4 sends traffic to the Azure Firewall Explicit Proxy on port 8443 over ExpressRoute.
1. **Private routing:** The proxy routes requests to Azure services via Private Endpoints.
1. **Cloud integration:** Services such as Event Grid Topic Spaces, Azure Storage, and Azure Key Vault are accessed privately using Azure Private Link. Public network access is disabled for all Azure services in the deployment.

### Event Grid topic spaces

Data is published to Event Grid via MQTT over WebSocket (`/mqtt` path suffix). Outbound traffic from Level 4 is routed through the Azure Firewall Explicit Proxy (port 443), then reaches the private endpoint for Event Grid over ExpressRoute. Level 4's managed identity has both EventGrid TopicSpaces Publisher and Subscriber roles to authenticate and push events to the namespace.

Telemetry is validated against schemas defined in Blob Storage, enforced by the Azure IoT Operations Dataflows running at L2 and L3.

### Sample output

```json
{
  "Temperature": {
    "Value": 98.2,
    "SourceTimestamp": "2025-08-05T13:45:22Z"
  },
  "EnergyUse": {
    "Value": 212,
    "SourceTimestamp": "2025-08-05T13:45:22Z"
  },
  "Weight": {
    "Value": 229,
    "SourceTimestamp": "2025-08-05T13:45:22Z"
  },
  "product": "flakes",
  "line-config": "cereal",
  "factory-code": "1032"
}
```

## Audit and post-deployment verification

<!-- TODO: This section was authored based on deployment patterns and requires SME review by Qiang/Grishma. It was not sourced from the validated PDF. -->

> [!IMPORTANT]
> This section provides recommended audit procedures. Have your network and security team review these steps before using them in production.

After deployment, verify that network isolation, private connectivity, and RBAC assignments are correctly configured.

### Step 1: Verify network isolation between layers

Confirm that no traffic leaks between non-adjacent layers (for example, L2 should not reach L4 directly):

1. From an L2 host, attempt to reach the L4 host IP directly. The connection should time out or be refused:

   ```bash
   curl -v --connect-timeout 5 https://<L4-host-ip>:8443
   ```

1. From an L2 host, confirm connectivity to L3 is working:

   ```bash
   curl -v --connect-timeout 5 https://<L3-host-ip>:<port>
   ```

1. Review firewall logs to confirm no unexpected cross-layer traffic.

### Step 2: Confirm traffic routes through private endpoints

Verify that all Azure-bound traffic routes through private endpoints and not the public internet:

1. From L4, resolve Azure service FQDNs and confirm they return private IPs:

   ```bash
   nslookup <eventgrid-namespace>.ts.eventgrid.azure.net
   nslookup <storage-account>.blob.core.windows.net
   nslookup <keyvault-name>.vault.azure.net
   ```

   Each result should return an IP in your private address range (for example, `10.254.x.x`), not a public IP.

1. Check the Azure portal for each Private Endpoint to confirm the connection status shows **Approved**.

1. In the Azure Firewall logs, verify that outbound traffic from L4 is routed to private endpoint IPs only.

### Step 3: Validate RBAC assignments

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

### Step 4: Verify DNS resolves to private IPs only

From each layer with CoreDNS deployed, confirm that Azure service names resolve to private IPs:

```bash
# From L2
dig <eventgrid-namespace>.ts.eventgrid.azure.net
dig <storage-account>.blob.core.windows.net

# From L3
dig <eventgrid-namespace>.ts.eventgrid.azure.net
dig <storage-account>.blob.core.windows.net
```

If any query returns a public IP, check your CoreDNS forwarding rules and Private DNS Zone linkage.

## Known limitations

- **Platform validation:** This scenario was validated on K3s running on Ubuntu Server 24.04 only. Other Kubernetes distributions or operating systems haven't been validated.
- **Proxy support:** Only Azure Firewall Explicit Proxy was validated. Third-party proxies (for example, Palo Alto) or transparent proxies aren't supported in this validated scenario.
- **Schema Registry public access:** Schema Registry may require public access enabled at creation time. After creation, you can disable public access. Use the `--skip-ra` flag when creating the Schema Registry to avoid requiring Owner-level permissions.
- **Level 1:** The L1 device layer is unused in this deployment flow.
- **Level 4 Arc:** Level 4 is not Arc-enabled; only Envoy Proxy is deployed at this layer.
- **Out-of-scope configurations:** Scenarios involving Azure VNets with external firewalls, transparent proxies, or cloud-only VNet deployments haven't been validated and are outside the support scope.

## Appendix

### Key Azure resource reference

```text
Subscription: <subscription-id>
Tenant: <tenant-id>

Arc Clusters:
- Level2: <L2-cluster-name>
- Level3: <L3-cluster-name>

Storage Containers (schemas):
- L2: <L2-schema-container-name>
- L3: <L3-schema-container-name>

Event Grid Namespace: <eventgrid-namespace-name>
Private Endpoint (Event Grid Topic Spaces): <pe-eventgrid-name> (IP: <private-ip-eventgrid>)

Schema Registries:
- L2: <L2-schema-registry-name>
- L3: <L3-schema-registry-name>

Azure IoT Operations Instances:
- L2: <L2-instance-name>
- L3: <L3-instance-name>
- L4: <L4-instance-name>
```

### Custom role definitions

**ACX–Secrets Store Extension Owner:** Grants permissions to register and manage the Secrets Store CSI driver, configure Azure Key Vault secret provider classes, and manage user-assigned managed identities.

```json
{
  "roleName": "ACX-Secrets Store Extension Owner",
  "assignableScopes": ["/subscriptions/<subscription-id>"],
  "permissions": [
    {
      "actions": [
        "Microsoft.SecretSyncController/register/action",
        "Microsoft.SecretSyncController/unregister/action",
        "Microsoft.SecretSyncController/azureKeyVaultSecretProviderClasses/read",
        "Microsoft.SecretSyncController/azureKeyVaultSecretProviderClasses/write",
        "Microsoft.SecretSyncController/azureKeyVaultSecretProviderClasses/delete",
        "Microsoft.Insights/alertRules/*",
        "Microsoft.Resources/deployments/*",
        "Microsoft.ManagedIdentity/userAssignedIdentities/assign/action",
        "Microsoft.ManagedIdentity/userAssignedIdentities/write",
        "Microsoft.ManagedIdentity/userAssignedIdentities/revokeTokens/action"
      ]
    }
  ]
}
```

**AdaptiveCloud_AIO–Contributors:** Grants permissions to manage role assignments and federated identity credentials for user-assigned managed identities within the resource group.

```json
{
  "roleName": "AdaptiveCloud_AIO-Contributors",
  "assignableScopes": ["/subscriptions/<subscription-id>/resourceGroups/<resource-group>"],
  "permissions": [
    {
      "actions": [
        "Microsoft.Authorization/roleAssignments/read",
        "Microsoft.Authorization/roleAssignments/write",
        "Microsoft.Authorization/roleAssignments/delete",
        "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/read",
        "Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/write"
      ]
    }
  ]
}
```

> [!NOTE]
> In environments using Azure Policy automation, these manual role definitions may not be required for OT teams. Policies can pre-assign Contributor or Storage Blob Data Contributor roles as needed.

## Related content

- [Azure IoT Operations networking](../manage-layered-network/overview-layered-network.md)
- [Deploy Azure IoT Operations with private connectivity](../manage-layered-network/howto-private-connectivity.md)
- [Configure infrastructure](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/configure-infrastructure.md)
- [Deploy Azure IoT Operations](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/deploy-aio.md)
- [Arc enable the K3s clusters](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/arc-enable-clusters.md)
- [Asset telemetry](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/asset-telemetry.md)
- [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns)
