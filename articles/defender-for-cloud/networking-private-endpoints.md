---
title: Configure private endpoints with Microsoft Defender Security Private Link
author: ami.hollander
ms.author: ami.hollander
description: Learn how to configure private endpoints for Microsoft Defender to secure access to your security resources from virtual networks.
ms.topic: how-to
ms.date: 11/16/2025
---

# Configure private endpoints with Microsoft Defender Security Private Link

This article shows you how to configure Security Private Link for Microsoft Defender using the Azure portal and Azure CLI. Security Private Link enables your workloads to securely communicate with Microsoft Defender services through private endpoints. For an overview, see the [use cases for private endpoints with Microsoft Defender](concepts-private-endpoints.md).

Security Private Link enables your workloads (such as AKS clusters, virtual machines, or container instances) to communicate with Microsoft Defender backend services entirely over private networks. This configuration ensures that all security-related traffic, including telemetry from agents and add-ons, doesn't traverse the public internet, meeting regulatory requirements for network isolation.

Security Private Link is supported in all public regions. It isn't supported in sovereign cloud regions, such as Microsoft Azure operated by 21Vianet (Azure in China) and Azure for US Government.

## Prerequisites

- A [virtual network and a subnet](../../virtual-network/quick-create-portal.md) where your workloads are deployed. This is where the private endpoints will be created.
- Workloads that need to communicate with Microsoft Defender services, such as:
  - Azure Kubernetes Service (AKS) clusters with Defender for Containers enabled
  - Virtual machines with Microsoft Defender for Servers
  - Container instances with security monitoring
- An Azure subscription with the appropriate Microsoft Defender plan enabled.
- The **Subscription Owner** role or **Contributor** role with the ability to create private endpoints.

> [!IMPORTANT]
> - Security Private Link supports communication from Defender agents, add-ons, and extensions to Defender backend services.
> - For network-isolated workloads, private endpoints replace the need for Azure Monitor Private Link Scope (AMPLS) and Azure Firewall egress rules.
> - Ensure your Defender components are using the latest versions that support private endpoint connectivity.
> - If your workloads interact with other Azure services (such as Azure Storage for diagnostics), configure private endpoints for those services as well.

## Set up private endpoint - Azure portal (recommended)

You can set up Security Private Link when deploying workloads with Defender enabled, or add private endpoints to an existing network-isolated environment.

### Create a private endpoint for Defender services

1. Sign in to the [Azure portal](https://portal.azure.com/).

1. Navigate to the **Private Link Center**.

1. Select **Create private endpoint**.

   :::image type="content" source="media/networking-private-endpoints/private-link-center.png" alt-text="Screenshot of the Private Link Center with Create private endpoint highlighted.":::

1. In the **Basics** tab, provide the following information:

   | Setting | Value |
   |---------|-------|
   | **Project details** | |
   | Subscription | Select your subscription. |
   | Resource group | Enter the name of an existing group or create a new one. |
   | **Instance details** | |
   | Name | Enter a unique name for the private endpoint. |
   | Region | Select a region. |

1. Select **Next: Resource**.

1. In the **Resource** tab, enter or select the following information:

   | Setting | Value |
   |---------|-------|
   | Connection method | Select **Connect to an Azure resource in my directory**. |
   | Subscription | Select your subscription. |
   | Resource type | Select **Microsoft.Security/securityConnectors** or the appropriate Defender resource type for your workload. |
   | Resource | Select the Security Private Link resource for Defender services. |
   | Target sub-resource | Select the appropriate sub-resource for the Defender service endpoints your workload needs to access. |

1. Select **Next: Virtual Network**.

1. In the **Virtual Network** tab, enter or select the information:

   | Setting | Value |
   |---------|-------|
   | **Networking** | |
   | Virtual network | Select the virtual network for the private endpoint. |
   | Subnet | Select the subnet for the private endpoint. |
   | **Private IP configuration** | |
   | Allocate private IP address dynamically | Select this option to automatically assign an IP address. |
   | **Application security group** | |
   | Application security group | Optionally select an application security group. |

1. Select **Next: DNS**.

1. In the **DNS** tab, configure DNS settings:

   | Setting | Value |
   |---------|-------|
   | **Private DNS integration** | |
   | Integrate with private DNS zone | Select **Yes**. |
   | Private DNS Zone | The field autopopulates with `privatelink.azure.com` or the appropriate zone for your resource. |

1. Select **Next: Tags** and optionally add tags.

1. Select **Review + create**.

1. After validation passes, select **Create**.

1. Wait for the deployment to complete.

### Confirm endpoint configuration

After the private endpoint is created, verify the DNS settings.

1. In the Azure portal, navigate to your private endpoint.

1. Select **DNS configuration**.

1. Review the DNS settings and verify that the FQDN resolves to the private IP address assigned to the endpoint.

   :::image type="content" source="media/networking-private-endpoints/endpoint-dns-settings.png" alt-text="Screenshot of the endpoint DNS configuration showing the private IP mapping.":::

## Set up private endpoint - Azure CLI

The following examples use Azure CLI to create Security Private Link for Microsoft Defender. These private endpoints enable your workloads to communicate with Defender services without traversing the public internet.

Set the following environment variables appropriate for your environment:

```azurecli
SUBSCRIPTION_ID=<your-subscription-id>
RESOURCE_GROUP=<resource-group-name>
LOCATION=<azure-region>
VNET_NAME=<virtual-network-name>
SUBNET_NAME=<subnet-name>
PRIVATE_ENDPOINT_NAME=<private-endpoint-name>
```

### Disable network policies in subnet

[Disable network policies](../../private-link/disable-private-endpoint-network-policy.md) such as network security groups in the subnet for the private endpoint. Update your subnet configuration with [az network vnet subnet update](/cli/azure/network/vnet/subnet#az-network-vnet-subnet-update):

```azurecli
az network vnet subnet update \
  --name $SUBNET_NAME \
  --vnet-name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --disable-private-endpoint-network-policies true
```

### Configure the private DNS zone

Create a [private Azure DNS zone](../../dns/private-dns-privatednszone.md) for the private Microsoft Defender service endpoints. In later steps, you create DNS records for your endpoint in this DNS zone. This ensures that your workloads can resolve Defender service FQDNs to the private IP addresses.

To use a private zone to override the default DNS resolution, the zone must be named `privatelink.azure.com`. Run the following [az network private-dns zone create](/cli/azure/network/private-dns/zone#az-network-private-dns-zone-create) command to create the private zone:

```azurecli
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name "privatelink.azure.com"
```

### Create an association link

Run [az network private-dns link vnet create](/cli/azure/network/private-dns/link/vnet#az-network-private-dns-link-vnet-create) to associate your private zone with the virtual network. This example creates a link called **MyDNSLink**.

```azurecli
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name "privatelink.azure.com" \
  --name MyDNSLink \
  --virtual-network $VNET_NAME \
  --registration-enabled false
```

### Create a private endpoint

Create the Security Private Link endpoint for Microsoft Defender services using [az network private-endpoint create](/cli/azure/network/private-endpoint#az-network-private-endpoint-create). This endpoint enables your workloads to communicate securely with Defender backend services.

```azurecli
az network private-endpoint create \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Security/securityConnectors/<connector-name>" \
  --group-id <target-sub-resource> \
  --connection-name MyConnection \
  --location $LOCATION
```

### Get endpoint IP configuration

After creating the private endpoint, retrieve the private IP address configuration. Run [az network private-endpoint show](/cli/azure/network/private-endpoint#az-network-private-endpoint-show) to query the private endpoint:

```azurecli
NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name $PRIVATE_ENDPOINT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query 'networkInterfaces[0].id' \
  --output tsv)
```

Get the private IP address:

```azurecli
PRIVATE_IP=$(az network nic show \
  --ids $NETWORK_INTERFACE_ID \
  --query "ipConfigurations[0].privateIPAddress" \
  --output tsv)

echo "Private IP: $PRIVATE_IP"
```

### Create DNS records in the private zone

Create DNS records in the private zone for the endpoint. First, create an empty A-record:

```azurecli
az network private-dns record-set a create \
  --name management \
  --zone-name privatelink.azure.com \
  --resource-group $RESOURCE_GROUP
```

Add the A-record with the private IP address:

```azurecli
az network private-dns record-set a add-record \
  --record-set-name management \
  --zone-name privatelink.azure.com \
  --resource-group $RESOURCE_GROUP \
  --ipv4-address $PRIVATE_IP
```

## Configure network isolation

For network-isolated environments and regulated workloads, you should disable public network access to ensure all communication with Defender services occurs through private endpoints only. This configuration is essential for meeting compliance requirements for network isolation.

### Disable public access - Azure portal

1. In the Azure portal, navigate to your subscription.

1. Under **Settings**, select **Defender for Cloud**.

1. Select **Settings**, then **Environment settings**.

1. Select your subscription.

1. Navigate to **Settings & monitoring**.

1. Under **Network**, select **Disabled** for public network access.

1. Select **Save**.

### Disable public access - Azure CLI

To disable public access using the Azure CLI, you need to configure network settings at the subscription level. This can be managed through Azure Policy or custom configurations based on your organizational requirements.

## Validate private endpoint connection

After setting up Security Private Link, validate that your workloads can communicate with Defender services through the private endpoints.

### Test DNS resolution

From a workload (VM, AKS node, or container) within the virtual network, verify that Defender service endpoints resolve to private IP addresses:

```bash
nslookup <defender-service-endpoint>
```

The output should show that the Defender service FQDN resolves to the private IP address assigned to your private endpoint, not a public IP.

Example output:

```
Server:    168.63.129.16
Address:   168.63.129.16#53

Non-authoritative answer:
defender.endpoint.security.azure.net    canonical name = defender.privatelink.security.azure.net.
Name:   defender.privatelink.security.azure.net
Address: 10.1.1.5
```

### Test workload connectivity

Verify that Defender agents and add-ons running on your workloads can successfully communicate with Defender services:

1. **For AKS clusters**: Check that the Defender for Containers add-on is successfully sending data:

   ```bash
   kubectl logs -n kube-system -l app=microsoft-defender
   ```

   Look for successful connection messages and absence of public endpoint connection attempts.

2. **For VMs**: Check that the Microsoft Defender for Endpoint agent is connected:

   ```bash
   # On Linux
   mdatp connectivity test
   
   # On Windows (PowerShell)
   Get-MpComputerStatus
   ```

3. **Verify in Azure portal**: Navigate to Microsoft Defender for Cloud and confirm that your workloads are reporting security data and recommendations.

## Manage private endpoint connections

You can manage Security Private Link connections for Microsoft Defender using the Azure portal or Azure CLI. This includes approving connections, monitoring status, and troubleshooting connectivity issues.

### List private endpoint connections

To list the private endpoint connections:

```azurecli
az network private-endpoint-connection list \
  --resource-group $RESOURCE_GROUP \
  --name <resource-name> \
  --type Microsoft.Security/securityConnectors
```

### Approve or reject connections

If manual approval is required, you can approve or reject private endpoint connections:

```azurecli
az network private-endpoint-connection approve \
  --resource-group $RESOURCE_GROUP \
  --name <connection-name> \
  --resource-name <resource-name> \
  --type Microsoft.Security/securityConnectors \
  --description "Approved by security team"
```

## DNS configuration options

The private endpoint integrates with a private DNS zone associated with a basic virtual network. This setup uses the Azure-provided DNS service directly to resolve the public FQDN to its private IP address in the virtual network.

Private Link supports additional DNS configuration scenarios that use the private zone, including with custom DNS solutions. For example, you might have a custom DNS solution deployed in the virtual network, or on-premises in a network you connect to the virtual network using a VPN gateway or Azure ExpressRoute.

To resolve the public FQDN to the private IP address in these scenarios, you need to configure a server-level forwarder to the Azure DNS service (168.63.129.16). Exact configuration options and steps depend on your existing networks and DNS. For examples, see [Azure Private Endpoint DNS configuration](../../private-link/private-endpoint-dns.md).

### Manually configure DNS records

For some scenarios, you may need to manually configure DNS records in a private zone instead of using the Azure-provided private zone. Ensure that you create records for all necessary endpoints.

## Clean up resources

To clean up your resources in the portal, navigate to your resource group and select **Delete resource group** to remove the resource group and all associated resources.

If you created all Azure resources in the same resource group and no longer need them, you can delete them using a single command:

```azurecli
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## Next steps

- To learn more about Private Link, see the [Azure Private Link](../../private-link/private-link-overview.md) documentation.
- For information about network security in Defender for Cloud, see [Planning network requirements for Microsoft Defender for Cloud](networking-requirements.md).
- To learn about other networking configurations, see [Configure network settings for Microsoft Defender for Cloud](networking-requirements.md).
