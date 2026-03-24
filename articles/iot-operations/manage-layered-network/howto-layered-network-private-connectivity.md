---
title: Deploy Azure IoT Operations in a layered network with private connectivity
description: Deploy Azure IoT Operations in a Purdue/ISA-95 layered network with private Azure connectivity using ExpressRoute, Private Link, and explicit proxy routing.
author: david-emakenemi
ms.subservice: layered-network-management
ms.author: demakenemi
ms.topic: how-to
ms.date: 03/19/2026

#CustomerIntent: As an operator in an industrial environment with Purdue-style network segmentation, I want to deploy Azure IoT Operations with private Azure connectivity so that no endpoints are exposed to the public internet.
ms.service: azure-iot-operations
---

# Deploy Azure IoT Operations in a layered network with private connectivity

This article describes how to deploy Azure IoT Operations (AIO) in a physically layered network topology with explicit proxy routing and private connectivity to Azure services via ExpressRoute. This deployment uses Private Link to reach services like Event Grid, and exposes no public endpoints at any network layer.

This scenario was validated using physical machines in a Purdue/ISA-95 segmented network spanning Levels 2 through 4. The Azure Firewall Explicit Proxy is deployed in an Azure VNet, with connectivity provided via ExpressRoute.

In this article, you:

- Prepare a layered network environment with static IPs and adjacent-only communication
- Create private endpoints and private DNS zones for Azure services
- Configure [Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) with explicit proxy routing
- Assign RBAC roles required by Azure IoT Operations components
- Deploy CoreDNS, Envoy Proxy, MQTT broker, and data flows across network layers
- Validate end-to-end telemetry flow from OPC UA sources to Azure Event Grid
- Audit and verify network isolation, private connectivity, and RBAC assignments

## Decision guidance: Layered network vs. sovereign vs. private-only

Before you begin, determine which networking approach fits your scenario:

- **Layered network (this article):** If you have a Purdue/ISA-95 segmented topology, use a layered network deployment. This approach connects your management plane to your Azure Resource Manager estate, lets Azure manage identity tokens, and provides validated resilience for extended disconnected operation. If you have a layered topology, this is the recommended approach.
- **Private-only (non-layered):** If you need private connectivity to Azure but don't have network segmentation between layers, use the simpler non-layered topology. See [Deploy Azure IoT Operations with private connectivity](howto-private-connectivity.md).
- **Sovereign:** If your organization operates under government regulations, tariffs, military security requirements, or similar constraints that mandate a sovereign cloud, a sovereign deployment path is available. The overhead of managing a sovereign cloud applies.

## Prerequisites

Before you begin, make sure the following requirements are met.

### Azure access and permissions

- A valid Azure subscription and tenant ID.
- Required role assignments: In this validated scenario, role assignments were created manually by admins with elevated privileges (Owner), since Contributor alone was insufficient. The following custom roles (definitions in [Appendix](#appendix)) are required:
  - **ACX–Secrets Store Extension Owner** — For registering/managing the Secrets Store CSI driver, configuring Azure Key Vault secret provider classes, and managing user-assigned managed identities.
  - **AdaptiveCloud_AIO–Contributors** — For managing federated identity credentials and role assignments for user-assigned managed identities within the resource group.

> [!TIP]
> For production deployments, use Azure Policy automation to pre-create these RBAC assignments. This eliminates the need for manual Owner intervention and allows OT teams to deploy with Contributor only, while still ensuring the correct permissions are in place.

### Cluster and network requirements

- A K3s cluster deployed at each network layer (Level 2, Level 3, and Level 4).
- Devices or VMs assigned static IPs.
- Network segmentation between layers (for example, firewalls allowing only L2 ↔ L3 ↔ L4 communication).
- DNS resolution across layers using CoreDNS (deployed at L2 and L3).

### Azure connectivity

- At least one Azure Private Endpoint deployed (for example, for Event Grid), assigned a private IP, and accessible via ExpressRoute or equivalent private routing.
- Event Grid, Storage (for Schema Registry), and Key Vault all require private endpoints.
- Azure Firewall Explicit Proxy at Level 4:
  - Ports: 8080 (HTTP), 8443 (HTTPS)
  - Reachable from Level 4 over ExpressRoute
  - All outbound HTTP/HTTPS traffic from Level 4 flows through this proxy

> [!NOTE]
> In the validated telemetry flow, only HTTPS (port 8443) was used. In customer environments, Level 4 may route through your own enterprise proxy instead.

### Tools

- Access to the [Azure IoT Operations portal](https://iotoperations.azure.com).
- Azure CLI installed on your admin or jump machine.
- Docker and kubectl installed (Helm optional).

## Architecture summary

This deployment aligns with the Purdue Model, implementing a physically segmented, multi-level architecture spanning Levels 2 through 4.

Each level is separated by network firewalls that restrict communication to adjacent layers only (for example, L2 ↔ L3 ↔ L4), ensuring tight segmentation. Outbound traffic to Azure is proxied through an Arc Gateway and routed to Azure Event Grid over Private Link, ensuring no internet-exposed endpoints are used at any layer.

### Validated components

| Layer | Components | Purpose |
| ----- | ---------- | ------- |
| L2 | CoreDNS, AIO Dataflows, AIO MQTT Broker | Ingests telemetry from OPC UA sources, applies initial enrichment, and forwards data upward |
| L3 | CoreDNS, Envoy Proxy, AIO Dataflows, AIO MQTT Broker | Aggregates and transforms data, resolves DNS to reach L4, and securely forwards telemetry |
| L4 | Envoy Proxy | Forwards enriched telemetry to Event Grid via Azure Firewall Explicit Proxy and Private Endpoint over ExpressRoute |

## Prepare your layered network environment

Each layer of the network (Level 2, 3, and 4) uses static IPs and strict firewall rules to enforce isolation. Each layer only communicates with its adjacent layer (for example, L2 ↔ L3 ↔ L4), implementing the Purdue model zones.

### Create Azure resources

Before deploying to the edge, create the following Azure resources:

1. Create resource group(s).
1. Create an Azure Blob Storage account and containers for schemas.
1. Create an Azure Key Vault.
1. Create an Event Grid Namespace and Topic Space.
1. Create Private Endpoints for:
   - Storage (blob)
   - Event Grid (Topic Spaces)
   - Key Vault
1. Create Private DNS Zones:
   - `privatelink.blob.core.windows.net`
   - `privatelink.ts.eventgrid.azure.net`
   - `privatelink.vaultcore.azure.net`
1. Assign RBAC assignments (see [Assign RBAC roles](#assign-rbac-roles)).
1. Disable public network access where allowed.

> [!IMPORTANT]
> Schema Registry has a known limitation with disabling public access at creation time. See [Known limitations](#known-limitations) for details.

### Assign static IPs for each layer

Deploy one machine (physical or virtual) per network layer. Assign static IPs within a shared address space. All devices run Ubuntu Server 24.04 with K3s clusters preinstalled. Devices on L2 and L3 are Arc-enabled and host their respective AIO instances.

> [!NOTE]
> IPs shown here are examples from the validation lab and are not internet accessible. Replace with IPs appropriate to your own network.

| Layer | Purpose | Example Hostname | Example IP | Notes |
| ----- | ------- | ---------------- | ---------- | ----- |
| L2 | OPC UA simulator, AIO (MQTT Broker, Dataflows), Arc, CoreDNS | p3tiny-01 | 172.22.232.X | Arc-enabled, AIO deployed |
| L3 | AIO (MQTT Broker, Dataflows), CoreDNS, Arc | p3tiny-02 | 172.22.232.Y | Arc-enabled, AIO deployed |
| L4 | Envoy Proxy (egress only, outbound access) | p3tiny-03 | 172.22.232.Z | Not Arc-enabled, handles egress via Azure Firewall Explicit Proxy over ExpressRoute |

### Enforce network isolation between layers

Use firewalls or host-level policies to enforce adjacent-only communication.

| Communication Path | Access |
| ------------------ | ------ |
| L2 ↔ L3 | Allow |
| L3 ↔ L4 | Allow |
| L2 ↔ L4 | Block |
| L2/L3 → Internet | Block |
| L4 → Azure | Allow via Azure Firewall Explicit Proxy over ExpressRoute |

### Route Azure-bound traffic through L4 only

Only the Level 4 node may initiate outbound traffic, forwarding it to the Azure Firewall Explicit Proxy over ExpressRoute, which then routes it to Azure services via Private Link.

1. Deploy Envoy Proxy on the L4 machine.
1. Forward HTTPS traffic to the Azure Firewall Explicit Proxy:
   - IP: `10.254.0.68`
   - Ports: `8080` (HTTP), `8443` (HTTPS)
1. Route outbound Azure traffic through Private Link.

> [!NOTE]
> This validation used HTTP(S) traffic through the proxy. If your proxy supports MQTT or other non-HTTP protocols, those can also be used in a similar configuration.

## Configure Private Link and DNS

Configure Azure Private Link to connect securely to Event Grid and Azure Storage, using Private Endpoints and CoreDNS-based name resolution. All traffic to these services remains on private IPs, with no internet exposure.

### Create Private Endpoints for Event Grid and Azure Storage

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

### Create Private DNS Zones

For each service, create the appropriate Azure Private DNS Zone. Only the Level 4 virtual network is linked to these Private DNS Zones. CoreDNS at L3 (and optionally L2) forwards requests to Azure's internal DNS resolver (`168.63.129.16`), which resolves names based on the L4 zone's DNS zone linkage.

| Service | Private DNS Zone |
| ------- | ---------------- |
| Event Grid | `privatelink.ts.eventgrid.azure.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |

For the full list of private DNS zone names, see [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns).

### Enable DNS resolution for Azure Private Endpoints

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

### Verify DNS resolution and connectivity

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

## Configure Arc Gateway for explicit proxy

If your network uses an explicit proxy and you plan to deploy Azure Arc–enabled Kubernetes with Arc Gateway, you must:

1. Deploy an [Azure Arc Gateway](/azure/azure-arc/kubernetes/arc-gateway-simplify-networking) resource in Azure.
1. Configure the connected cluster so all outbound Arc traffic routes through the proxy.
1. Pass the Arc Gateway resource ID to the `az connectedk8s connect` or `az connectedk8s update` command.

This ensures Arc agents can reach Azure services while honoring your proxy and private connectivity rules.

### Set proxy environment variables

On the Arc Gateway VM hosting the Azure Arc services, set the proxy environment variables. The `HTTPS_PROXY` variable must point to your network's firewall explicit proxy:

```bash
export HTTPS_PROXY=http://<proxy-server>:<port>
export NO_PROXY=localhost,127.0.0.1,.svc,.local,<your-private-DNS-zone>
```

### Retrieve the service principal Object ID

The `--custom-locations-oid` parameter requires the Object ID (OID) of the Azure Arc Custom Locations service principal.

To find it in the Azure portal:

1. Go to **Microsoft Entra ID**.
1. Select **Enterprise applications**.
1. Search for **Azure Arc Kubernetes Custom Locations**.
1. Open the application, go to **Properties**, and copy the **Object ID**.

### Connect the cluster with Arc Gateway

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

### Verify Arc connectivity

1. Run `kubectl logs` on the Arc gateway pod to confirm it reaches Azure.
1. Verify that DNS resolution and TLS handshake are successful through the proxy.

## Assign RBAC roles

Azure IoT Operations requires specific role-based access control (RBAC) assignments to allow components to interact with Azure services like Blob Storage and Event Grid.

> [!NOTE]
> These permissions were manually assigned in the lab by admins with elevated privileges. For production, use Azure Policy automation.

### Required role assignments

| Identity | Role | Scope | Notes |
| -------- | ---- | ----- | ----- |
| AIO system-assigned managed identity | Storage Blob Contributor | Storage account containing schema files | For Schema Registry |
| AIO system-assigned managed identity | EventGrid TopicSpaces Publisher | Event Grid namespace | Enables publish to TopicSpaces |
| AIO system-assigned managed identity | EventGrid TopicSpaces Subscriber | Event Grid namespace | Enables subscribe to TopicSpaces |

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

### Layered enrichment context

Each AIO network layer contributes business metadata to outbound telemetry:

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
- **AIO MQTT Broker and Dataflows (L2, L3):** Deploy on Levels 2 and 3 as described in [Asset telemetry](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/asset-telemetry.md).

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

**AIO MQTT Broker:** Verify listeners are active:

```bash
kubectl get service <k3s-service-name> -n azure-iot-operations
```

**AIO Dataflows:** Check telemetry with `mqttui` and confirm metadata in Event Grid Topic Spaces:

```bash
mqttui --broker mqtt://<level-host-ip>:1883
```

Confirm that enrichment metadata (for example, `product: flakes`, `line-config: cereal`, `factory-code: 1032`) appears in Event Grid Topic Spaces using the Azure portal or CLI.

## Validate telemetry flow

### Connectivity path

The end-to-end telemetry flow follows this path:

1. **Telemetry ingestion (L2):** OPC UA connectors at L2 publish telemetry to AIO Dataflows, which forward it to the MQTT Broker.
1. **L2 to L3:** MQTT Broker at L2 publishes messages to L3 using MQTT.
1. **L3 to L4:** MQTT Broker at L3 publishes messages upstream using MQTT over WebSocket.
1. **DNS resolution (L3):** AIO Dataflows and CoreDNS at L3 resolve private service names to reach L4.
1. **Proxy forwarding (L3 to L4):** Envoy Proxy on L3 forwards MQTT traffic to Envoy Proxy on L4.
1. **Egress (L4):** Envoy Proxy on L4 sends traffic to the Azure Firewall Explicit Proxy on port 8443 over ExpressRoute.
1. **Private routing:** The proxy routes requests to Azure services via Private Endpoints.
1. **Cloud integration:** Services such as Event Grid Topic Spaces, Azure Storage, and Azure Key Vault are accessed privately using Azure Private Link. Public network access is disabled for all Azure services in the deployment.

### Event Grid topic spaces

Data is published to Event Grid via MQTT over WebSocket (`/mqtt` path suffix). Outbound traffic from Level 4 is routed through the Azure Firewall Explicit Proxy (port 443), then reaches the private endpoint for Event Grid over ExpressRoute. Level 4's managed identity has both EventGrid TopicSpaces Publisher and Subscriber roles to authenticate and push events to the namespace.

Telemetry is validated against schemas defined in Blob Storage, enforced by the AIO Dataflows running at L2 and L3.

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

### Verify network isolation between layers

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

### Confirm traffic routes through private endpoints

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

Verify that the AIO system-assigned managed identity is listed as the assignee for each role.

### Verify DNS resolves to private IPs only

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

AIO Instances:
- L2: <L2-aio-instance-name>
- L3: <L3-aio-instance-name>
- L4: <L4-aio-instance-name>
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

- [How does Azure IoT Operations work in layered network?](concept-iot-operations-in-layered-network.md)
- [Azure IoT Operations networking](overview-layered-network.md)
- [Deploy Azure IoT Operations with private connectivity](howto-private-connectivity.md)
- [Configure infrastructure](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/configure-infrastructure.md)
- [Deploy Azure IoT Operations](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/deploy-aio.md)
- [Arc enable the K3s clusters](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/arc-enable-clusters.md)
- [Asset telemetry](https://github.com/Azure-Samples/explore-iot-operations/blob/main/samples/layered-networking/asset-telemetry.md)
- [Azure Private DNS Zone values](/azure/private-link/private-endpoint-dns)
