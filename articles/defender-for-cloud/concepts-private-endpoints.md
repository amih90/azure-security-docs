---
title: Private endpoints with Microsoft Defender
author: ami.hollander
ms.author: ami.hollander
description: Learn about using private endpoints with Microsoft Defender for Cloud to ensure secure and private connectivity in your virtual network.
ms.topic: concept-article
ms.date: 11/16/2025
---

# Private endpoints with Microsoft Defender (Preview)

This article provides an overview of using private endpoints with Microsoft Security Private Links to ensure secure and private connectivity in your virtual network.

> [!NOTE]
> For a complete understanding of private endpoints and private links, see [What is a private endpoint?](/azure/private-link/private-endpoint-overview).

## Use cases

You can use private endpoints with Microsoft Security Private Link to allow workloads on a virtual network to securely access Microsoft Defender services. The private endpoint uses an IP address from the virtual network address space to access Microsoft Defender services. Network traffic between your workloads and Microsoft Defender services traverses the virtual network and a private link on the Microsoft backbone network, eliminating exposure to the public internet while enabling security protection for workloads in network-isolated environments.

Using private endpoints with Microsoft Security Private Link enables you to:

- **Enable network-isolated workloads**: Allow workloads in completely isolated networks to be protected by Microsoft Defender without requiring public internet access, meeting strict regulatory compliance requirements.
- **Securely connect from on-premises networks**: Connect on-premises networks and hybrid environments to Defender services using [VPN](/azure/vpn-gateway/vpn-gateway-about-vpngateways) or [ExpressRoutes](/azure/expressroute/expressroute-locations) with private peering.

> [!IMPORTANT]
> - For network-isolated workloads, Microsoft Security Private Link replaces the need for Azure Monitor Private Link Scope (AMPLS) and Azure Firewall egress rules.

:::image type="content" source="media/active-user/recommended-owner.png" alt-text="Screenshot of a conceptual diagram showing Security Private Link with customer's.":::


## Conceptual overview

A private endpoint is a special network interface for an Azure service in your virtual network. When you create a private endpoint for your Security Private Link, it provides secure connectivity between workloads on your virtual network and Microsoft Defender services. The private endpoint is assigned an IP address from the IP address range of your virtual network. The connection between the private endpoint and the Microsoft Defender service uses a secure private link.

Workloads in the virtual network can connect to the Microsoft Defender service over the private endpoint seamlessly. They can communicate with Defender using agents, add-ons, and extensions with the fully qualified domain name (FQDN) of Microsoft Defender services. The connection uses the same authorization.

When you create a private endpoint connection for a Security Private Link in your virtual network, a consent request is sent for approval to the Security Administrator. If the user requesting the creation of the private endpoint is also an owner of the Security Private Link, this consent request is automatically approved.

Workload owners and security administrators can manage consent requests and the private endpoints through the **Private endpoint connections** tab for the Security Private Link in the Azure portal.

## DNS changes for private endpoints

> [!NOTE]
> For details about how to configure your DNS settings for private endpoints, see [Azure Private Endpoint DNS integration](../../private-link/private-endpoint-dns-integration.md).

When you create a private endpoint, by default, a [private DNS zone](/azure/dns/private-dns-overview) is provisioned that corresponds to the Microsoft Defender private link subdomain `*.defender.microsoft.com`.

When your workloads make connections to Defender service endpoints from within the virtual network with private endpoints configured, the FQDN is resolved to the private IP address of the endpoint. Connections from outside the virtual network (if public access is still enabled) resolve to the public endpoint.

Security Private Link enables your workload to communicate with different Microsoft Defender services. Each service requires specific domain endpoints to be configured. For example, the DNS resource records for Microsoft Defender services, when resolved from outside the virtual network hosting the private endpoint, would be:

| Service | Name | Type | Value | Port |
|---------|------|------|-------|------|
| Defender for Cloud | `*.cloud.defender.microsoft.com` | CNAME | `*.privatelink.cloud.defender.microsoft.com` | 443 |
| Defender for Cloud | `*.privatelink.cloud.defender.microsoft.com` | CNAME | `<Defender public endpoint>` | 443 |

As previously mentioned, you can deny or control access for clients outside the virtual network through the public endpoint using network security controls.

The DNS resource records when resolved by a client in the virtual network hosting the private endpoint would be:

| Service | Name | Type | Value | Port |
|---------|------|------|-------|------|
| Defender for Cloud | `*.cloud.defender.microsoft.com` | CNAME | `*.privatelink.cloud.defender.microsoft.com` | 443 |
| Defender for Cloud | `*.privatelink.cloud.defender.microsoft.com` | A | 10.0.0.5 | 443 |

This approach enables workloads on the virtual network hosting the private endpoints and workloads outside the virtual network to access Microsoft Defender services.

If you're using a custom DNS server on your network, clients must be able to resolve the FQDN for the Microsoft Defender service endpoints to the private endpoint IP address. You should configure your DNS server to delegate your private link subdomain to the private DNS zone for the virtual network, or configure the A records with the private endpoint IP address.

## Next steps

- [Configure private endpoints with Microsoft Defender Security Private Link](networking-private-endpoints.md)
- To learn more about Private Link, see the [Azure Private Link](/azure/private-link) documentation.
