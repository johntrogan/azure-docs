---
title: Use Rule Impact Analyzer in Traffic Analytics 
titleSuffix: Azure Network Watcher
description: Use Rule Impact Analysis to simulate and assess security admin rule effects in Azure Virtual Network Manager. Ensure compliance and prevent misconfigurations.
author: halkazwini
ms.author: halkazwini
ms.service: azure-network-watcher
ms.date: 04/06/2026
ms.topic: how-to
---

# Use rule impact analyzer in traffic analytics

In this article, you learn how to use the Rule Impact Analysis feature with network groups in Azure Virtual Network Manager. You can use the Azure portal to create a security admin configuration, add a security admin rule, and simulate the impact of your rule changes before deploying them.

The Rules Impact Analyzer enables you to preview the impact of security admin rules before applying them to your environment. This feature helps you validate rule behavior, identify potential conflicts, and ensure that connectivity requirements are met without disrupting live traffic. By understanding the impact of your proposed rules changes, you can confidently plan changes, maintain compliance, and reduce the risk of misconfiguration across your virtual networks.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/pricing/purchase-options/azure-account?cid=msft_learn).

- Traffic analytics enabled for your virtual network flow logs or network security group fow logs. For more information, see [Enable traffic analytics on virtual network flow logs](vnet-flow-logs-manage.md#enable-or-disable-traffic-analytics) or [Enable traffic analytics on network security group flow logs](nsg-flow-logs-manage.md#enable-or-disable-traffic-analytics).
 
- Required role-based access control (RBAC) permissions. For more information, see [Trafic analytics RBAC Permissions](required-rbac-permissions.md#traffic-analytics).

- A network group. If you don't have a network group, see [Create a network group](../virtual-network-manager/create-virtual-network-manager-portal#create-a-network-group).

## Workflow and Step-by-Step guide for Rules Impact Analysis

onfiguring network security across multiple environments requires careful planning. The Rule Impact Analyzer provides a safe way to validate security admin rules before they are applied, helping you understand their impact and avoid disruptions. By simulating changes first, you gain confidence that your rules will work as intended and maintain compliance without affecting live traffic.

## How does Rule Impact Analysis Work?

By using the Rule Impact Analyzer, you gain visibility and control over your network security posture, without the risk of making disruptive changes. Here is an overview of the workflow for rule simulation :

1.  :::image type="content" source="media/traffic-analytics-what-if/image1.png" alt-text="":::Enable Traffic Analytics for the relevant virtual networks, as simulation relies on traffic data collected through Traffic Analytics.

2.  To set the scope for simulation, choose the network groups that contain the resources you want to evaluate.

Add

3.  :::image type="content" source="media/traffic-analytics-what-if/image2.png" alt-text="":::the security admin rules you plan to deploy

4.  The simulation engine analyses how these rules would interact with your current configuration.

5.  The simulation engine calculates the effective outcome without applying changes to live traffic.

6.  Review the simulation report.

7.  Deploy the rules

## Step-by-Step Guide

### Step 1: Configure the simulation scope

1.  Open Azure portal \> Network Watcher \> Traffic Analytics \> Rule Impact Analyzer.

- Select **Configure simulation**.

- Choose the **Subscription** that contains the target resources.

- Select the **Type** (for example, **Network Security Group**) and then choose the target NSG/resource.

- Select the rule(s) to simulate (use **Filter items** or **Select all** as needed).

### Step 2: Select Virtual Networks & Run

Rule impact analysis is performed only on Virtual Networks with Traffic Analytics fully enabled. This ensures the simulation is based on complete and accurate traffic data. The following Virtual Networks are automatically excluded because they can result in incomplete or inaccurate simulation results:

- Virtual Networks with subnet‑ or NIC‑level flow logs (these override Virtual Network‑level logs)

- Virtual Networks with flow log filtering enabled

- AKS‑injected Virtual Networks

After selecting the rules to analyze, you must specify the scope of the evaluation by choosing the target virtual networks whose traffic data will be used. Only eligible Virtual Networks are considered to ensure the analysis reflects accurate, end‑to‑end traffic behaviour.

- On the **VNet Selection** page, review the list of Virtual Networks available for evaluation.

:::image type="content" source="media/traffic-analytics-what-if/image3.png" alt-text="":::

- Use the **search bar** or filters (Resource Group, Location, Traffic Analytics) to narrow down the list.

- Select one or more Virtual Networks (up to 500) that show Traffic Analytics: **Enabled**.

- Click **Apply**.

- The system analyses the rules against your current configuration.

### Step 3: Review results

:::image type="content" source="media/traffic-analytics-what-if/image4.png" alt-text="":::After running the simulation, you’ll see a detailed report that lists all traffic paths and how your rules impact them. This section helps you understand what the results mean and how to use them effectively.

#### Traffic Path and Table Overview

At the top of the report, you’ll find:

- Impacted: Paths where at least one simulated rule changes traffic behaviour.

- Not Impacted: Paths unaffected by the simulated rules.

- Indeterminate: Paths where the simulation couldn’t compute a result (for example, log analytics workspace doesn’t exist for traffic analytics, access to the workspace is denied, or required data/configuration is missing).

The table lists all target virtual networks analysed during the simulation and shows how your rules affect connectivity. Each row represents one target virtual network and includes the following fields:

| Column Name | Description | Example |
|----|----|----|
| VNet Name | Name of the virtual network included in the simulation scope. Each row represents the impact of the simulated rules on traffic within that Virtual Network. | sample-aks-vnet |
| Impacting Rule | The specific rule that causes an impact on the Virtual Network’s traffic. Shown as ‘-‘ when no rule impacts the traffic. | DenyAllOutbound |
| Priority | Priority value of the impacting rule. Displayed only when a rule is responsible for the impact; otherwise shown as ‘-‘. | 65500 |
| Impact | Result of the simulation for the VNet. Indicates whether traffic is affected by the simulated rules. Visible states include **Impacted**, **Not Impacted**, and **Indeterminate**. | Impacted |
| Flows Breaking | Number of traffic flows that would be disrupted if the impacting rule were deployed. A value of 0 indicates no traffic breakage. | 2,184,434 |
| Query | Action link to view the underlying query used to compute the simulation result for that VNet. | View Query |

For impacted virtual networks, the report identifies the **impacting rule**, its p**riority**, and the **number of** f**lows breaking**, helping you assess the severity of the change. Use **View Query** to inspect the underlying query and validate the result before deploying the rules.

# Related content


- [Create a security admin rule using network groups - Azure Virtual Network Manager \| Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-network-manager/how-to-create-security-admin-rule-network-group)

- [View configurations applied by Azure Virtual Network Manager \| Microsoft Learn](https://learn.microsoft.com/en-gb/azure/virtual-network-manager/how-to-view-applied-configurations)

