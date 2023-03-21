# Securing Access to PHSM

Payment Hardware Security Module (Payment HSM or PHSM) is a [bare-metal service](https://learn.microsoft.com/en-us/azure/payment-hsm/overview) providing cryptographic key operations for real-time and critical payment transactions in the Azure cloud. 

Payment HSM devices are a variation of Dedicated HSM devices with more advanced cryptographic modules. A Payment HSM supports *PIN block translation*, i.e. never decrypts the PIN value in transit (making the actual PIN value much harder to discover by an attacker), while the general purpose HSMs are not capable of this end-to-end ¨PIN encryption.

The solution uses hardware from [Thales](https://cpl.thalesgroup.com/encryption/hardware-security-modules/payment-hsms/payshield-10k) as a vendor. The devices are provided in a single tenant, dedicated to the customer who has [full control and exclusive access](https://learn.microsoft.com/en-us/azure/payment-hsm/overview#customer-managed-hsm-in-azure).

The repo presents 2 designs providing Firewall filtering when accessing Payment HSM.

# 1. Payment HSM networking

Before using Azure Payment HSM, make sure to [register the Azure Payment HSM resource providers and resource provider features](https://learn.microsoft.com/en-us/azure/payment-hsm/register-payment-hsm-resource-providers?tabs=azure-cli).

This [tutorial](https://learn.microsoft.com/en-us/azure/payment-hsm/create-payment-hsm?tabs=azure-cli) provides the deployment steps of a Payment HSM device.

PHSM deployments include [High Availability and Distaster Recovery scenarios](https://learn.microsoft.com/en-us/azure/payment-hsm/deployment-scenarios).

## 1.1. PHSM delegated subnet

Like Dedicated HSM, Payment HSM is VNet-injected in a delegated subnet, integrated into a customer VNet. Each Payment HSM device is assigned 2 private IPs from this subnet, one dedicated for management.

The devices can be accessed from the peered Azure VNets as well as from the customer On-Prem network (via ER or VPN for example).


## 1.2. Networking restrictions

Payment HSM comes with some policy [restrictions](https://learn.microsoft.com/en-us/azure/payment-hsm/solution-design#constraints) on this subnet: Network Security Groups (NSGs) and User-Defined Routes (UDRs) are currently not supported.

PHSM is not compatible with vWAN topologies or cross region VNet peering, as listed in the [topology supported](https://learn.microsoft.com/en-us/azure/payment-hsm/solution-design#supported-topologies).

# 2. Solution Architecture

2 networking designs are available to secure the access to the Payment HSM devices offering filtering and inspection capabilities to bypass the current NSG and UDR constraints.

## 2.1. Option 1: Firewall & SNAT 

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
