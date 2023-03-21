# Securing PHSM Connectivity

As explained in [our doc](https://learn.microsoft.com/en-us/azure/payment-hsm/overview), Payment HSM (Hardware Security Module) is a bare-metal service providing cryptographic key operations for real-time and critical payment transactions in the Azure cloud. 

In (very) short, Payment HSM is a variation of Dedicated HSM with more advanced cryptographic modules. The solution uses hardware from [Thales](https://cpl.thalesgroup.com/encryption/hardware-security-modules/payment-hsms/payshield-10k) as a vendor, provided in a single dedicated tenant to which the customer has full control and exclusive access. More details can be found [here](https://learn.microsoft.com/en-us/azure/payment-hsm/overview#customer-managed-hsm-in-azure).

Like Dedicated HSM, PHSM is a VNet-injected service in a delegated subnet, whih comes with some policy [restrictions](https://learn.microsoft.com/en-us/azure/payment-hsm/solution-design#constraints) on this subnet: Network Security Groups (NSGs) and User-Defined Routes (UDRs) are currently not supported.

The aim of this repo is to propose 2 solutions offering filtering and inspection capabilities to bypass these current constraints.

## 1. Firewall & SNAT 
This design is inspired from the [Dedicated HSM solution architecture](https://learn.microsoft.com/en-us/azure/dedicated-hsm/networking#solution-architecture).

The lack of UDRs is addressed by the Firewall doing SNAT on the client IP: when forwarding traffic to PHSM, the return traffic will automatically be directed back to the Firewall.

Filtering rules that would have been enforced using UDRs can be configured on the Firewall.

![image](https://user-images.githubusercontent.com/110976272/226541241-8b593ca9-341f-4327-adb9-e88db73b5c8e.png)

Both Spoke traffic and On-Prem traffic to the PHSM environment are secured.

Either an Azure Firewall or a 3rd party FW NVA can be used in this design.

## 2. Firewall & reverse-proxy

This design is a good option when performing SNAT on the Firewall is not approved by network security teams, requiring instead to keep the source and destination IPs unchanged for inbound ans outbound traffic.

This architecture leverages a reverse-proxy, deployed in a dedicated subnet in the PHSM VNet directly or in a peered VNet. 

Instead of sending traffic to the PHSM devices, the destination is set to the reverse-proxy IP, located in a subnet that does not have the restrictions of the PHSM delegated subnet: both NSGs and UDRs can be configured, and combined with a Firewall in the central hub:

![image](https://user-images.githubusercontent.com/110976272/226541198-40a74904-4713-4caa-a059-778727f423c7.png)

Like in the previous scenario, both Spoke traffic and On-Prem traffic are secured and either an Azure Firewall or a 3rd party FW NVA can be used.


fastpathenabled tag  is an AFEC (Azure Feature Exposure Control) flag, this flag will enable subscriptions to connect to Payment HSM. fastpathenabled tag  needs to be added/registered to all subscriptions that need to connect to PHSM. Enabling fastpathenabled tag  on the subscriptions with existing resources will have no impact on the existing resources.
