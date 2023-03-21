# Azure Payment HSM - Inspect traffic 

Intent of this article is to explain how to inspect traffic to Azure Payment HSM.

## Azure Payment HSM

Payment Hardware Security Module (Payment HSM or PHSM) is a [bare-metal service](https://learn.microsoft.com/en-us/azure/payment-hsm/overview) providing cryptographic key operations for real-time and critical payment transactions in the Azure cloud. 

Payment HSM devices are a variation of Dedicated HSM devices with more advanced cryptographic modules and features: a Payment HSM never decrypts the PIN value in transit for example. 

The Azure Payment HSM solution uses hardware from [Thales](https://cpl.thalesgroup.com/encryption/hardware-security-modules/payment-hsms/payshield-10k) as a vendor. Customers have [full control and exclusive access](https://learn.microsoft.com/en-us/azure/payment-hsm/overview#customer-managed-hsm-in-azure) to the Payment HSM.

## Azure Payment HSM - Networking

When Payment HSM is deployed, it comes with a host network interface and a management network interface. There are several deployment scenarios:
1. [With host and management ports in same VNet](https://learn.microsoft.com/en-us/azure/payment-hsm/create-payment-hsm?tabs=azure-cli)
2. [With host and management ports in different VNets](https://learn.microsoft.com/en-us/azure/payment-hsm/create-different-vnet?tabs=azure-cli)
3. [With host and management port with IP addresses in different VNets](https://learn.microsoft.com/en-us/azure/payment-hsm/create-different-ip-addresses?tabs=azure-cli)

In all of the above scenarios, Payment HSM is a VNet-injected service in a delegated subnet: `hsmSubnet` and `managementHsmSubnet` must be delegated to `Microsoft.HardwareSecurityModules/dedicatedHSMs` service.

## Azure Payment HSM - Networking limitation

Payment HSM comes with some policy [restrictions](https://learn.microsoft.com/en-us/azure/payment-hsm/solution-design#constraints) on these subnets: **Network Security Groups (NSGs) and User-Defined Routes (UDRs) are currently not supported**.

> Note: PHSM is not compatible with vWAN topologies or cross region VNet peering, as listed in the [topology supported](https://learn.microsoft.com/en-us/azure/payment-hsm/solution-design#supported-topologies).

This article present two ways to inspect traffic destined to a Payment HSM:
* Solution #1 - Firewall with SNAT
* Solution #2 - Firewall & Reverse proxy

# Solution #1 - Firewall with SNAT

This design is inspired by the [Dedicated HSM solution architecture](https://learn.microsoft.com/en-us/azure/dedicated-hsm/networking#solution-architecture).

![image](https://user-images.githubusercontent.com/110976272/226541241-8b593ca9-341f-4327-adb9-e88db73b5c8e.png)

1. On-Prem to PHSM: a Route Table containing a UDR for the Payment HSM VNet range and pointing to the central hub Firewall is applied to the GatewaySubnet.

2. Spoke VNet to PHSM: a Route Table containing a UDR for the Payment HSM subnet and pointing to the central hub Firewall is applied to the Spoke VNet subnets. 

In both use-cases, the Firewall SNATs the client IP address before forwarding traffic to the PHSM NIC, guaranteeing that the return traffic will automatically be directed back to the Firewall.

:arrow_right: The lack of UDRs is addressed by the Firewall doing SNAT on the client IP: when forwarding traffic to PHSM, the return traffic will automatically be directed back to the Firewall.

:arrow_right: Filtering rules that would have been enforced using UDRs can be configured on the Firewall.

Both Spoke traffic and On-Prem traffic to the PHSM environment are secured.

Either an Azure Firewall or a 3rd party FW NVA can be used in this design.

## 2.2. Option 2: Firewall & reverse-proxy

This design is a good option when performing SNAT on the Firewall is not approved by network security teams, requiring instead to keep the source and destination IPs unchanged for traffic crossing the Firewall.

This architecture leverages a reverse-proxy, deployed in a dedicated subnet in the PHSM VNet directly or in a peered VNet. 

:arrow_right: Instead of sending traffic to the PHSM devices, the destination is set to the reverse-proxy IP, located in a subnet that does not have the restrictions of the PHSM delegated subnet: both NSGs and UDRs can be configured, and combined with a Firewall in the central hub:

![image](https://user-images.githubusercontent.com/110976272/226541198-40a74904-4713-4caa-a059-778727f423c7.png)

1. On-Prem to PHSM: a Route Table containing a UDR for the Payment HSM VNet range and pointing to the central hub Firewall is applied to the GatewaySubnet.

2. Spoke VNet to PHSM a Route Table containing a UDR for the reverse-proxy IP and pointing to the central hub Firewall is applied to the Spoke VNet subnets. 

In both use cases *Gateway Route propagation* is disabled on the reverse-proxy subnet, so that a 0/0 UDR is enough to force the return traffic via the Firewall 

Like in the previous scenario, both Spoke traffic and On-Prem traffic are secured and either an Azure Firewall or a 3rd party FW NVA can be used.


fastpathenabled tag  is an AFEC (Azure Feature Exposure Control) flag, this flag will enable subscriptions to connect to Payment HSM. fastpathenabled tag  needs to be added/registered to all subscriptions that need to connect to PHSM. Enabling fastpathenabled tag  on the subscriptions with existing resources will have no impact on the existing resources.
