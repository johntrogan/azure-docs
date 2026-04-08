|  |  |
|----|----|
| IssueType | Description |
| **AgentStopped** | The Network Watcher agent on the source VM has stopped or is unresponsive. |
| **GuestFirewall** | Traffic is being blocked by the guest OS firewall on the source or destination VM. |
| **DNSResolution** | The DNS lookup for the destination hostname failed on the source agent. |
| **SocketError** | The source agent failed to bind or listen on the required local socket (e.g., **SocketBindFailed** or **ListenFailed**). |
| **NetworkSecurityRule** | An NSG rule is denying inbound or outbound traffic between the source and destination. |
| **UserDefinedRoute** | A UserDefinedRoute was found that routes traffic to a ‘None’ next hop, creating a blackhole routing. |
| **Platform** | An Azure platform-level issue is affecting connectivity. |
| **NetworkError** | A generic network failure occurred (e.g., connection timed out, connect failed, no response, or send/receive failure). |
| **CPU** | CPU usage on the source or destination VM exceeded threshold. |
| **Memory** | Memory usage on the source or destination VM exceeded threshold. |
| **ARPMissing** | The ARP table on the Microsoft Edge (ExpressRoute) hop is missing or has an incomplete entry for the customer/Microsoft edge IP. |
| **RouteMissing** | Raised when no valid route to the destination can be found at a hop. |
| **VMRebooting** | The source or destination VM is currently in a rebooting state. |
| **VMNotAllocated** | VM is not allocated (deallocated/stopped). |
| **NoListenerOnDestination** | The destination connectivity check confirmed that no process is listening on the specified port. |
| **DIPProbeDown** | The SLB health probe reports the backend DIP (destination IP) as "Down". |
| **NoRouteLearned** | The SLB or Virtual Hub found no effective route to the destination. |
| **PeeringInfoNotFound** | The peering information between two VNets could not be retrieved. |
| **VMStarting** | The destination VM is in a starting state and is not yet ready to accept traffic. |
| **VMStopped** | The destination VM is stopped (but still allocated), so it cannot accept network traffic. |
| **VMStopping** | The destination VM is in the process of stopping and is not reliably accepting traffic. |
| **VMDeallocating** | The destination VM is being deallocated and is in the process of releasing its resources, making it temporarily unreachable. |
| **VMDeallocated** | The destination VM has been fully deallocated. |
| **SystemError** | An internal system or infrastructure error occurred. |
| **UDRLoop** | User Defined Route found. This results in a routing loop, as the next hop IP matches the current hop IP. |
| **IPForwardingNotEnabled** | The NVA (virtual appliance) VM that traffic is routed through does not have IP forwarding enabled on its NIC. |
| **VnetAccessNotAllowed** | The VNet peering link has <u>AllowVNetAccess</u> set to **false**, blocking traffic from crossing the peering boundary. |
| **AllowGatewayTransitNotEnabled** | The peering on the hub/gateway side does not have <u>AllowGatewayTransit</u> enabled. |
| **MultiNICsInSameSubnet** | Multiple NICs on the VM are in the same subnet, which can cause asymmetric routing and unpredictable traffic behavior. |
| **StandardILBOutboundInternetNotAllowed** | Raised when a VM in the backend pool of a Standard Internal Load Balancer attempts to reach the internet — Standard ILB backends have no default outbound internet access, unlike Basic ILB. |
| **MultiNICsInSameSubnetWithWeakHostSendEnabled** | Multiple NICs are in the same subnet and weak host send is enabled, which can cause traffic to egress from an unexpected interface. |
| **MultiNICsInSameSubnetWithWeakHostEnabled** | Multiple NICs are in the same subnet and weak host (send/receive) is enabled on the VM, which may route packets through unintended interfaces. |
| **SourcePortInUse** | The source port selected by the agent is already in **TIME_WAIT** state (a lingering TCP socket), preventing a new connection from being established from that port. |
| **InvalidResponseFromServer** | A DNS probe queried the server but received no matching records. |
| **DNSResponseValidationFailed** | The DNS probe response failed a configured validation rule (e.g., wrong record count, wrong recursion support, wrong RCode, or wrong authority flag). |
| **UnsupportedSystem** | The agent is running on an OS or system configuration that does not support the requested probe type. |
| **IncompleteTopology** | Service could not build a complete hop path to the destination. |
| **DestinationUnreachable** | Agent on the source machine was unable to reach the destination. |
| **TraceRouteUnavailable** | The agent did not return a traceroute result (no paths), so cannot determine the connectivity status between source and destination. |
| **DestinationPartiallyReachable** | Some but not all traceroute paths from the agent successfully reached the destination. |
| **GatewayNotProvisioned** | The VPN/ExpressRoute gateway returned a **GatewayNotProvisioned** error. |
| **ResourceHealthUnavailable** | Azure Resource Health reports the hop resource (VM, gateway, firewall, etc.) as **Unavailable**. |
| **ResourceHealthDegraded** | Azure Resource Health reports the hop resource as **Degraded**. |
| **VirtualHubNotProvisioned** | The Virtual WAN Hub associated with the path is not in a <u>Succeeded</u> provisioning state. |
| **StatusCodeValidationFailed** | The HTTP probe received a response but the returned HTTP status code did not match the expected value. |
| **HeaderValidationFailed** | The HTTP probe received a response but one or more expected HTTP response headers were missing or did not match. |
| **ContentValidationFailed** | The HTTP probe received a response but the response body content did not match the expected value. |
| **NoConnectionConfigured** | No connection is configured between the source and destination endpoints in the connection monitor settings. |
| **ConnectionStateDisconnected** | The monitored connection is in a disconnected state, indicating a break in the logical connection path. |
| **BasicILBNotSupportedWithGlobalPeering** | A Basic Internal Load Balancer does not support Global VNet Peering. |
| **BGPRoutePropogationDisabled** | BGP route propagation is disabled on the route table associated with the source subnet. |
| **UseRemoteGatewaysNotEnabled** | The peering on the spoke side does not have UseRemoteGateways enabled. |
| **UnexpectedVirtualNetworkGatewayConnection** | A virtual network gateway connection was found on the path that was not expected. |
