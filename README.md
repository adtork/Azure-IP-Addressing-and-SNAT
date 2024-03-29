# Azure-IP-Addressing-and-SNAT
In this article, we are going to talk about Azure IP addressing behavior and SNAT options. I often get asked, if my Azure VM only has a private IP, what IP address will it use for outbound connections, and how can I choose that IP and whitelist it? We are going to go over a few topics concerning Azure IP addressing, SNAT, checking your public IP and some alternative solutoins for whitelisting Azure VIPs.

# Azure IP Addressing
```bash
As we know in networking, the forumula to calcuate the # of hosts in a given subnet is as follows:
# Subnet Hosts = [2^(# host bits)] -2 (-2 being your broadcast and default address, 0.0.0.0/255.255.255.255)

Azure adheres to this, but takes an additonal three IPs per subnet as well, so the above formula becomes:
# Subnet Hosts = [2 ^ (# host bits)] –2 – 3

# Some simple examples
If you have a [/27] network, you can fit [2^(32 – 27)] – 5 == 27 resources
If you have a [/28] network, you can fit [2^(32 – 28)] – 5 ==  11 resources
If you have a [/29] network, you can fit [2^(32 – 29)] – 5 == 3 resources
and so on...
```
We can use the above then to know in a given subnet, Azure will assign .4 and .5 as the first usable addresses. This also holds true for VPN or ExR GW for example, even though these are not visable inside the Azure Network allocated addresses visible in the portal.

# The Pseudo-VIP
Some background, any resource in Azure will have a public IP programmed for SNAT. If the VM is assinged a private IP (DIP), the Azure platform will still assign a platfrom generated public IP so the resource can communicate external to the VNET. Azure assigns this based on the given region the VM is in, and will assign a public IP address based on that region. Its important to note though, this behavior will be changing as th below articles mentions in 2025. At the time of this article, the azure platform will still assign a psuedo VIP. The easist way to check that VIP in both Windows and Linux, is to simply run curl ifconfig.me or culr ifconfig.io (in bash or cmd)
```bash

C:\Windows\system32>curl ifconfig.me                                            
40.76.244.175

C:\Windows\system32>ipconfig                                                    
                                                                                
Windows IP Configuration                                                        
                                                                                
                                                                                
Ethernet adapter Ethernet:                                                      
                                                                                
   Connection-specific DNS Suffix  . : vxtyv5ak24ietmdv15r0ycyipa.bx.internal.cl
oudapp.net                                                                      
   Link-local IPv6 Address . . . . . : fe80::fd13:9d2c:7a15:542d%6              
   IPv4 Address. . . . . . . . . . . : 10.2.0.5                                 
   Subnet Mask . . . . . . . . . . . : 255.255.255.0                            
   Default Gateway . . . . . . . . . : 10.2.0.1                                                                                                                 
```
We can see the platform assigned 40.76.244.175, even though this VM only as a DIP assigned

![image](https://user-images.githubusercontent.com/55964102/193902852-3f484eed-30b7-439d-98ce-1a9b1113f17a.png)

# Azure SNAT
From the above snippet, as explained the platform assigned that address to provide default outbound access. Azure will use this address for communcation external to the vnet. Customers often don't want public IPs for security reeasons, but they still need to know the IP the VM is using for outbound access. Technically the platform generated IP can be whitelisted, buf if the VM is deallocated its very unlikely it will get the same platform IP once its started back up, because its first come, first serve! There are three main options to provide outbound access to VMs in Azure:

Option 1:
<Br>
<Br>
Simply adding a public IP to the VM. This is discouraged due to inherint security risks of port scans and exposure if not properly locked down with NSGs or an Azure firewall/NVA. This also does not scale well based on the number of VMs in the vnet. Generally speaking, using public IPs should be kept to a minimum in production environments. The better option is to use Azure Bastion, serial console access or VPN to access virutal machines.
![image](https://github.com/adtork/Azure-IP-Addressing-and-SNAT/assets/55964102/4073d3a1-7c50-4d1f-9919-c9c31e228e23)
<Br>
<Br>
Option 2: 
<Br>
Creating an external load balancer (ELB) and front-end IP address and port. Once those two are done, Snat will be programmed via Azure Fabric for VMs residing in the back-end pool(s). Of course you can have more then one public IP, but the ELB still limits the number of Snat ports you can use, based on back-end VMs in the pool. SNAT translation will be random with more then one VIP assinged to the ELB front-end as well. The number of Snat ports is based on the number of VMs in the back-end pool. The ELB serves more roles then Nat-GW, which is purely focused on maximum SNAT port allocation.
![image](https://github.com/adtork/Azure-IP-Addressing-and-SNAT/assets/55964102/2831ab07-5fa1-4ce8-a299-eb4348c29395)
<Br>
<Br>
Option 3:
<Br>
The best option for pure snat port allocation is Nat-GW. This will use the full 64K port range and provide maximum number of snat ports for outbound access. A Nat-GW can be chained with an ELB per above, but the Nat-GW will take over the job of the ELB and use its front-end IPs for Snat. Nat-GWs are also used in conjunction with Azure Firewalls, which is sort of a 4th option to provide outbound access. Resources residing behind an AzFW will SNAT to the firewalls public IP(s), unless chained with Nat-GW.
![image](https://github.com/adtork/Azure-IP-Addressing-and-SNAT/assets/55964102/d1c0b4b1-e731-4020-8b48-5cf4a431f30b)


More Info on outbound access, SLB and Nat-GW in Azure:
<br>
[Azure SNAT Behavior](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)
<br>
[Nat Gateway](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-gateway-resource).
<br>
[AzFw Chained with NatGW](https://learn.microsoft.com/en-us/azure/firewall/integrate-with-nat-gateway).
<br>
[Software Load balancer Outbound Behavior](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-outbound-connections).

# Conclusion
From this article we can confirm basic concepts of Azure IP addressing, check a VMs platform provided SNAT addresses, and three alternatives to providing external access to virtual machines residing in virutal networks. 

