---
title: Load balancing strategies in Application Gateway for Containers
description: Learn about different load balancing strategies to help build resilient and performant workloads.
services: application-gateway
author: JackStromberg
ms.service: azure-appgw-for-containers
ms.topic: concept-article
ms.date: 4/22/2026
ms.author: jstrom
# Customer intent: "As a cloud administrator, I need to use different load balancing algorithms to efficiently scale and optimize traffic placement to backend services."
---

# Load balancing strategies in Application Gateway for Containers

In today's dynamic and high-demand environments, efficient load balancing is crucial for maintaining the performance and reliability of applications. Application Gateway for Containers offers multiple load balancing algorithms to cater to different scenarios and requirements. Understanding and choosing the right load balancing algorithm can significantly impact the overall user experience, resource utilization, and system resilience.

### Why care about changing the load balancing algorithm

Different load balancing algorithms provide unique benefits and are suited to various use cases. By having multiple options to choose from, you can:

1. **Optimize Performance:** Different algorithms can help distribute traffic more effectively based on the current load, server capacity, and other metrics. This ensures that applications run smoothly and efficiently, even under varying loads.

2. **Enhance Reliability:** Algorithms like Circuit Breaker and Slow Start can improve the resilience of the system by preventing overloads and managing server recovery. This helps maintain service availability and reduces the risk of downtime.

3. **Improve Resource Utilization:** Weighted Round Robin and Load Aware Routing can ensure that more powerful servers handle a larger share of the traffic, optimizing resource utilization and reducing costs.

4. **Adapt to Changing Conditions:** In dynamic environments, the load on backend servers can vary significantly. Load Aware Routing and Priority-based algorithms allow the system to adapt to these changes in real-time, ensuring optimal performance and reliability.

5. **Tailor to Specific Needs:** Different applications and services may have unique requirements. By offering multiple load balancing options, Application Gateway for Containers allows customers to tailor the load balancing strategy to their specific needs, whether it's prioritizing speed, reliability, or resource efficiency.

By leveraging these load balancing options, customers can ensure that their applications are robust, responsive, and capable of handling varying loads and conditions effectively.

---

Below are the descriptions and scenarios for each load balancing option:

## Round Robin

**Description:**
Round Robin is a simple load balancing method where each incoming request is distributed sequentially across the available backend servers. Once the last server in the list is reached, the process starts over from the first server.

**Scenario:**
Round Robin is suitable for scenarios where backend servers have similar capabilities and workloads. It ensures an even distribution of traffic without considering the current load or performance of each server.

**Configuration:**

This is the default option used by Application Gateway for Containers to distribute traffic to pods, no specific configuration is needed to enable this load balancing algorithm.

## Weighted Round Robin

**Description:**
Weighted Round Robin assigns a weight to each backend server based on its capacity or performance. Servers with higher weights receive more requests compared to those with lower weights. The requests are distributed in a round-robin fashion, but the frequency of requests to each server is proportional to its weight.

**Scenario:**
Weighted Round Robin is ideal for environments where backend servers have different capacities or performance levels. It ensures that more powerful servers handle a larger share of the traffic, optimizing resource utilization.

**Configuration:**

A weight can be applied between services within the HTTPRoute resource in Gateway API.  Details on how a weight is defined can be defined [here](https://gateway-api.sigs.k8s.io/reference/spec/#httpbackendref)

In this example, approximately 75 of a 100 requests would be preferred to backend-v1, while the remaining 25 would go to backend-v2.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: traffic-split-route
  namespace: test-infra
spec:
  parentRefs:
  - name: gateway-01
  rules:
  - backendRefs:
    - name: backend-v1
      port: 8080
      weight: 75
    - name: backend-v2
      port: 8080
      weight: 25
```

## Slow Start

**Description:**
Slow Start gradually increases the amount of traffic sent to a new or recently recovered backend server. This allows the server to warm up and handle the load more effectively before being fully utilized.

**Scenario:**
Slow Start is useful when adding new servers to the pool or when recovering servers from failure. It helps prevent sudden spikes in load that could overwhelm the server and ensures a smooth transition to full capacity.

**Configuration:**

There are three parameters, in addition to two load balancing strategies that can be paired with slow start.

| Parameter Name | Measurement | Default value | Description |
| -------------- | ----------- | ------------- | ----------- |
| window | duration | 0 seconds | Represents the duration of slow start window. |
| aggression | decimal | 1.0 | Controls the speed of traffic increase over the slow start window. The input value must be greater than 0. |
| startWeightPercent| percent | 10% | Configures the minimum percentage of origin weight that avoids too small new weight, which may cause endpoints in slow start mode receive no traffic in slow start window. |

Load balancing strategies

- Round Robin
- Least Request

```yaml
version: alb.networking.azure.io/v1
kind: BackendLoadBalancingPolicy
metadata:
  name: lbpolicy-example
  namespace: test-namespace
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: threshold-test
    ports:
    - port: 443
  loadBalancing:
    strategy: "round-robin" # "round-robin", "least-request"
    slowStart: # slow start allowed for only loadBalancing strategy "round-robin" and "least-request"
      window: 0s
      aggression: "1.0"
      startWeightPercent: 10
```

## Load Aware Routing

**Description:**
Load Aware Routing distributes traffic based on the current load and availability of backend servers. The load balancer collects metrics such as the number of active requests, CPU utilization, and memory usage to make informed routing decisions.

**Scenario:**
Load Aware Routing is beneficial in dynamic environments where the load on backend servers can vary significantly. It ensures that requests are directed to servers with the most available capacity, improving overall performance and reliability.

By leveraging these load balancing options, Application Gateway for Containers can efficiently manage traffic distribution, improve performance, and enhance the reliability of your applications.
