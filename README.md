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

**Concept**: The firewall **SNATs the client IP address** before forwarding traffic to the PHSM NIC, guaranteeing that the return traffic will automatically be directed back to the Firewall. Either an Azure Firewall or a 3rd party FW NVA can be used in this design.

**Architecture diagram**:

![image](https://user-images.githubusercontent.com/110976272/226541241-8b593ca9-341f-4327-adb9-e88db73b5c8e.png)

**Route tables required**:
1. On-Prem to PHSM: a Route Table containing a UDR for the Payment HSM VNet range and pointing to the central hub Firewall is applied to the GatewaySubnet.
2. Spoke VNet(s) to PHSM: a Route Table containing the usual default route pointing to the central hub Firewall is applied to the Spoke VNet(s) subnets. 

**Results**:
* The lack of UDRs is addressed by the Firewall doing SNAT on the client IP: when forwarding traffic to PHSM, the return traffic will automatically be directed back to the Firewall.
* Filtering rules that would have been enforced using NSGs can be configured on the Firewall.
* Both Spoke traffic and On-Prem traffic to the PHSM environment are secured.

# Solution 2: Firewall & reverse-proxy

This design is a good option when performing SNAT on the Firewall is not approved by network security teams, requiring instead to keep the source and destination IPs unchanged for traffic crossing the Firewall.

**Concept**: This architecture leverages a reverse-proxy, deployed in a dedicated subnet in the PHSM VNet directly or in a peered VNet:
* Instead of sending traffic to the PHSM devices, the destination is set to the reverse-proxy IP, located in a subnet that does not have the restrictions of the PHSM delegated subnet: both NSGs and UDRs can be configured, and combined with a Firewall in the central hub.

**Architecture diagram**:

![image](https://user-images.githubusercontent.com/110976272/226541198-40a74904-4713-4caa-a059-778727f423c7.png)
**Infrastructure required**:

**Route tables required**:
1. On-Prem to PHSM: a Route Table containing a UDR for the Payment HSM VNet range and pointing to the central hub Firewall is applied to the GatewaySubnet.
2. Spoke VNet(s) to PHSM: a Route Table containing the usual default route pointing to the central hub Firewall is applied to the Spoke VNet(s) subnets. 

> ***Gateway Route propagation*** must be disabled on the reverse-proxy subnet, so that a 0/0 UDR is enough to force the return traffic via the Firewall 

**Results**:
* UDRs not supported on the PHSM subnet is addressed by the reverse proxy (doing SNAT on the client IP): when forwarding traffic to PHSM, the return traffic will automatically be directed back to the reverse proxy.
* TO COMPLETE CYNTHIA
* Filtering rules that cannot be enforced using NSGs on dedicated PHSM subnet can be configured on the Firewall and/or on the reverse proxy subnet using NSG.
* Both Spoke traffic and On-Prem traffic to the PHSM environment are secured.

