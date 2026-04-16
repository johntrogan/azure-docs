---

title: What is Azure NAT Gateway?
titlesuffix: Azure NAT Gateway
description: Overview of Azure NAT Gateway features, resources, architecture, and implementation. Learn about what Azure NAT Gateway is and how to use it.

author: alittleton
ms.service: azure-nat-gateway
ms.topic: overview
ms.date: 04/14/2026
ms.author: alittleton
ms.customs: references_regions

#Customer intent: I want to understand what Azure NAT Gateway is and how to use it so that I can ensure scalable internet access for private resources without exposing them to unsolicited inbound connections.
---

# What is Azure NAT Gateway?

Azure NAT Gateway is a fully managed and highly resilient network address translation (NAT) service. Use Azure NAT Gateway to let all instances in a subnet connect outbound to the internet while remaining fully private. A NAT gateway doesn't permit unsolicited inbound connections from the internet. Only packets arriving as response packets to an outbound connection can pass through a NAT gateway.

Azure NAT Gateway dynamically allocates secure NAT (SNAT) ports to automatically scale outbound connectivity and minimize the risk of SNAT port exhaustion.

Azure NAT Gateway is available in two tiers:

* *Standard* is zonal (deployed to a single availability zone) and provides scalable outbound connectivity for subnets in a single virtual network.

* *StandardV2* is zone redundant with higher throughput than the Standard tier, IPv6 support, and flow log support.

## StandardV2 tier

The StandardV2 tier of Azure NAT Gateway provides all the same functionality of the Standard tier, such as dynamic SNAT port allocation and secure outbound connectivity for subnets within a virtual network. Additionally, StandardV2 is zone redundant, meaning that it provides outbound connectivity from all zones in a region instead of a single zone.  

:::image type="content" source="./media/nat-overview/zone-redundant-standard-2.png" alt-text="Diagram that shows a StandardV2 NAT gateway spanning multiple availability zones in a region." lightbox="./media/nat-overview/zone-redundant-standard-2.png":::

### Key capabilities of StandardV2

* **Zone redundancy**: Operates across all availability zones in a region to maintain connectivity during a single zone failure.
* **IPv6 support**: Supports both IPv4 and IPv6 public IP addresses and prefixes for outbound connectivity.
* **Higher throughput**: Provides up to 100 Gbps of data throughput per NAT gateway, compared to 50 Gbps for Standard NAT gateways.
* **Flow log support**: Provides IP-based traffic information to help monitor and analyze outbound traffic flows.

To learn more about how to deploy a StandardV2 NAT gateway, see [Create a StandardV2 NAT gateway](./quickstart-create-nat-gateway-v2.md).

### Key limitations of StandardV2

* The StandardV2 tier requires StandardV2 public IP addresses or prefixes. Standard public IPs aren't supported with StandardV2.
* The Standard tier can't be upgraded to the StandardV2 tier. You must create a StandardV2 NAT gateway and replace the Standard NAT gateway on your subnet.
* Terraform doesn't yet support attachment of IPv6 StandardV2 public IPs to a StandardV2 NAT gateway. IPv4 StandardV2 public IPs can be attached to a StandardV2 NAT gateway with Terraform. No other clients are affected.
* The following regions don't support StandardV2 NAT gateways:
  * Canada East
  * Chile Central
  * Indonesia Central
  * Israel Northwest
  * Malaysia West
  * Qatar Central
  * Sweden South
  * West Central US
  * West India
* StandardV2 NAT gateways don't support and can't be attached to delegated subnets for the following services:
  * Azure SQL Managed Instance
  * Azure Container Instances
  * Azure Database for PostgreSQL
  * Azure Database for MySQL  
  * Azure Data Factory (data movement)
  * Microsoft Power Platform
  * Azure Stream Analytics
  * Web Apps feature of Azure App Service
  * Azure Container Apps
  * Azure DNS Private Resolver

### Known issues of StandardV2

* IPv6 outbound traffic that uses load balancer outbound rules is disrupted when you associate a StandardV2 NAT gateway with a subnet. If you require both IPv4 and IPv6 outbound connectivity, use either:
  * Load balancer outbound rules for both IPv4 and IPv6 traffic
  * A Standard NAT gateway for IPv4 traffic and load balancer outbound rules for IPv6 traffic

* Attaching a StandardV2 NAT gateway to an empty subnet created before April 2025 without any virtual machines (VMs) might cause the virtual network to go into a failed state. To return the virtual network to a successful state, remove the StandardV2 NAT gateway, create and add a VM to the subnet, and then reattach the StandardV2 NAT gateway.

* Outbound connections that use a load balancer, Azure Firewall, or VM instance-level public IPs might be interrupted when you add a StandardV2 NAT gateway to a subnet. All net new outbound connections use the StandardV2 NAT gateway.

For more information about known issues and limitations of the StandardV2 tier of Azure NAT Gateway, see [Known limitations](./nat-sku.md#known-limitations).

## Standard tier

You can associate a Standard NAT gateway with subnets in the same virtual network to provide outbound connectivity to the internet. A Standard NAT gateway operates out of a single availability zone.

:::image type="content" source="./media/nat-overview/zonal-standard-1.png" alt-text="Diagram that shows a Standard NAT gateway in a single availability zone." lightbox="./media/nat-overview/zonal-standard-1.png":::

## Azure NAT Gateway benefits

### Simple setup

Deployments with Azure NAT Gateway are intentionally simple. Attach a NAT gateway to a subnet and public IP address, and start connecting outbound to the internet right away. No maintenance or routing configurations are required. You can add more public IPs or subnets later without affecting your existing configuration.

The following steps show an example of how to set up a NAT gateway:

1. Create a non-zonal or zonal NAT gateway.

1. Assign a public IP address or public IP prefix.

1. Configure a subnet to use the NAT gateway.

If necessary, modify Transmission Control Protocol (TCP) idle timeout (optional). Review [timers](/azure/nat-gateway/nat-gateway-resource#idle-timeout-timers) before you change the default.

### Security

Azure NAT Gateway is built on the Zero Trust network security model. When you use Azure NAT Gateway, private instances within a subnet don't need public IP addresses to reach the internet. Private resources can reach external sources outside the virtual network by using SNAT for static public IP addresses or prefixes in Azure NAT Gateway.

You can provide a contiguous set of IPs for outbound connectivity by using a public IP prefix. You can configure destination firewall rules based on this predictable IP list.

### Resiliency

Azure NAT Gateway is a fully managed and distributed service. It doesn't depend on individual compute instances such as virtual machines or a single physical gateway device. A NAT gateway always has multiple fault domains and can sustain multiple failures without service outages. Software-defined networking makes a NAT gateway highly resilient.

### Scalability

A NAT gateway is scaled out from creation. No ramp-up or scale-out operation is required. Azure manages the operation of a NAT gateway for you.

Attach a NAT gateway to a subnet to provide outbound connectivity for all private resources in that subnet. All subnets in a virtual network can use the same NAT gateway resource. You can scale out outbound connectivity by assigning up to 16 public IP addresses to a NAT gateway. When you associate a NAT gateway with a public IP prefix, it automatically scales to the number of IP addresses needed for outbound.

### Performance

Azure NAT Gateway is a software-defined networking service. Each NAT gateway can process up to 50 Gbps of data for both outbound and return traffic.

A NAT gateway doesn't affect the network bandwidth of your compute resources. For more information, see [Performance](nat-gateway-resource.md#performance).

## Azure NAT Gateway basics

Azure NAT Gateway provides secure, scalable outbound connectivity for resources in a virtual network.

### Outbound connectivity

* Azure NAT Gateway is the method that we recommend for outbound connectivity.

  To migrate outbound access to a NAT gateway from default outbound access or load balancer outbound rules, see [Migrate outbound access to Azure NAT Gateway](./tutorial-migrate-outbound-nat.md).

  > [!NOTE]
  > As of March 31, 2026, new virtual networks default to using private subnets. [Default outbound access](/azure/virtual-network/ip-services/default-outbound-access#when-is-default-outbound-access-provided) isn't provided by default. Use an explicit form of outbound connectivity instead, like Azure NAT Gateway.

  The service:

  * Provides outbound connectivity at a subnet level. It replaces the default internet destination of a subnet to provide outbound connectivity.

  * Doesn't require any routing configurations on a subnet route table. After you attach a NAT gateway to a subnet, it provides outbound connectivity right away.

  * Allows flows to be created from the virtual network to the services outside your virtual network. Return traffic from the internet is only allowed in response to an active flow. Services outside your virtual network can't initiate an inbound connection through a NAT gateway.

  * Takes precedence over other outbound connectivity methods, including a load balancer, instance-level public IP addresses, and Azure Firewall.

  * Takes priority over other explicit outbound methods configured in a virtual network for all new connections. There are no drops in traffic flow for existing connections that use other explicit methods of outbound connectivity.
  
  * Doesn't have the same limitations of SNAT port exhaustion as [default outbound access](../virtual-network/ip-services/default-outbound-access.md) and [outbound rules of a load balancer](../load-balancer/outbound-rules.md).

  * Supports TCP and User Datagram Protocol (UDP) protocols only. Internet Control Message Protocol (ICMP) isn't supported.

    Azure NAT Gateway supports [Azure App Service instances](/azure/app-service/networking/nat-gateway-integration) (web applications, REST APIs, and mobile backends) through [virtual network integration](/azure/app-service/overview-vnet-integration).

* The subnet has a [system default route](/azure/virtual-network/virtual-networks-udr-overview#default) that routes traffic with destination 0.0.0.0/0 to the internet automatically. After you configure a NAT gateway to the subnet, virtual machines in the subnet communicate with the internet by using the public IP of the NAT gateway.

* When you create a user-defined route (UDR) in your subnet route table for 0.0.0.0/0 traffic, you override the default internet path for this traffic. A UDR that sends 0.0.0.0/0 traffic to a virtual appliance or a virtual network gateway (Azure VPN Gateway and Azure ExpressRoute) as the next hop type instead overrides NAT gateway connectivity to the internet.

  Here's the flow:

  UDR to next hop virtual appliance or virtual network gateway >> NAT gateway >> Instance-level public IP address on a virtual machine >> load balancer outbound rules >> default system route to the internet.

### NAT gateway configurations

* Multiple subnets within the same virtual network can either use different NAT gateways or the same NAT gateway.

* You can't attach multiple NAT gateways to a single subnet.

* A NAT gateway can't span multiple virtual networks. However, you can use a NAT gateway to provide outbound connectivity in a hub and spoke model. For more information, see the [Azure NAT Gateway hub and spoke tutorial](/azure/nat-gateway/tutorial-hub-spoke-route-nat).

* A Standard NAT gateway resource can use up to 16 IPv4 public IP addresses. A StandardV2 NAT gateway resource can use up to 16 IPv4 and 16 IPv6 public IP addresses.

* You can't deploy a NAT gateway in a [gateway subnet](../vpn-gateway/vpn-gateway-about-vpn-gateway-settings.md#gwsub) or subnet containing [SQL Managed Instances](/azure/azure-sql/managed-instance/connectivity-architecture-overview#networking-constraints).

* Azure NAT Gateway works with any VM network interface or IP configuration. A NAT gateway can SNAT multiple IP configurations on a network interface.

* You can associate a NAT gateway with an Azure Firewall subnet in a hub virtual network and provide outbound connectivity from spoke virtual networks peered to the hub. To learn more, see [Azure Firewall integration with Azure NAT Gateway](../firewall/integrate-with-nat-gateway.md).

### Availability zones

* You can create a Standard NAT gateway in a specific availability zone or place it in **no zone**.

* You can isolate a Standard NAT gateway in a specific zone when you create a [zonal NAT gateway](/azure/reliability/reliability-nat-gateway). After you deploy the NAT gateway, you can't change the zone selection.

* By default, a Standard NAT gateway is placed in **no zone**. Azure places a [nonzonal NAT gateway](/azure/reliability/reliability-nat-gateway) in a zone for you.

* A StandardV2 NAT gateway is zone-redundant and operates across all availability zones in a region to maintain connectivity during a single zone failure.

### Default outbound access

* To provide secure outbound connectivity to the internet, [enable private subnet](/azure/virtual-network/ip-services/default-outbound-access#how-can-i-transition-to-an-explicit-method-of-public-connectivity-and-disable-default-outbound-access) to prevent the creation of default outbound IPs and instead use an explicit method of outbound connectivity like a NAT gateway.

* Certain services don't function on a virtual machine in a private subnet without an explicit method of outbound connectivity, such as Windows Activation and Windows Updates. To activate or update VM operating systems, such as Windows, an explicit method of outbound connectivity is required, like a NAT gateway.

* To migrate outbound access to a NAT gateway from default outbound access or load balancer outbound rules, see [Migrate outbound access to Azure NAT Gateway](./tutorial-migrate-outbound-nat.md).

> [!NOTE]
> On March 31, 2026, new virtual networks default to using private subnets, meaning that [default outbound access](/azure/virtual-network/ip-services/default-outbound-access#when-is-default-outbound-access-provided) is no longer provided by default, and you must enable an explicit outbound method to reach public endpoints on the Internet and within Microsoft. Use an explicit form of outbound connectivity instead, like a NAT gateway.

### Azure NAT Gateway and basic resources

* A Standard NAT gateway works with Standard public IP addresses or public IP prefixes. A StandardV2 NAT gateway works only with StandardV2 public IP addresses or public IP prefixes.

* You can't use Azure NAT Gateway with subnets that have basic resources. Basic-tier resources, such as a basic load balancer or basic public IPs, don't work with Azure NAT Gateway. You can upgrade a basic load balancer and basic public IP to standard to work with a NAT gateway.
  
  * For more information about upgrading a boad balancer from basic to standard, see [Upgrade a basic load balancer](/azure/load-balancer/upgrade-basic-standard-with-powershell).

  * For more information about upgrading a public IP from basic to standard, see [Upgrade a public IP address](../virtual-network/ip-services/public-ip-upgrade-portal.md).

  * For more information about upgrading a basic public IP attached to a virtual machine from basic to standard, see [Upgrade a basic public IP attached to a virtual machine](/azure/virtual-network/ip-services/public-ip-upgrade-vm).  

### Connection timeouts and timers

* Azure NAT Gateway sends a TCP Reset (RST) packet for any connection flow that it doesn't recognize as an existing connection. The connection flow no longer exists if the Azure NAT Gateway idle timeout is reached or the connection was closed earlier.

* When the sender of traffic on the nonexisting connection flow receives the Azure NAT Gateway TCP RST packet, the connection is no longer usable.

* SNAT ports aren't readily available for reuse to the same destination endpoint after a connection closes. Azure NAT Gateway places SNAT ports in a cooldown state before they can be reused to connect to the same destination endpoint.

* SNAT port reuse (cooldown) timer durations vary for TCP traffic depending on how the connection closes. To learn more, see [Port Reuse Timers](./nat-gateway-resource.md#port-reuse-timers).

* The Azure NAT Gateway TCP idle timeout timer defaults to 4 minutes but can be increased up to 120 minutes. Any activity on a flow can reset the idle timer, including TCP keepalives. To learn more, see [Idle Timeout Timers](./nat-gateway-resource.md#idle-timeout-timers).

* UDP traffic has an idle timeout timer of 4 minutes that you can't change.

* UDP traffic has a port reuse timer of 65 seconds for which a port is in hold down before it's available for reuse to the same destination endpoint.

## Pricing and service level agreement (SLA)

Standard and StandardV2 NAT gateways are the same price. For more information, see [Aure NAT Gateway pricing](https://azure.microsoft.com/pricing/details/azure-nat-gateway/).

For information on the SLA, see [SLA for Azure NAT Gateway](https://azure.microsoft.com/support/legal/sla/virtual-network-nat/v1_0/).

## Related content

* For more information about creating and validating a NAT gateway, see [Quickstart: Create a NAT gateway by using the Azure portal](quickstart-create-nat-gateway-portal.md).

* To view a video on more information about Azure NAT Gateway, see [How to get better outbound connectivity using Azure NAT Gateway](https://www.youtube.com/watch?v=2Ng_uM0ZaB4).

* For more information about the NAT gateway resource, see [NAT gateway resource](./nat-gateway-resource.md).

* Learn more about Azure NAT Gateway in [Learn module: Introduction to Azure NAT Gateway](/training/modules/intro-to-azure-virtual-network-nat).

* For more information about architecture options for Azure NAT Gateway, see [Azure Well-Architected Framework review of an Azure NAT Gateway](/azure/architecture/networking/guide/well-architected-network-address-translation-gateway).
