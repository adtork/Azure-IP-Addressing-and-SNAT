# Azure-IP-Addressing-and-SNAT
In this article, we are going to talk about Azure IP addressing behavior and SNAT options. I often get asked, if my Azure VM only has a private IP, what IP address will it use for outbound connections and how can I choose that IP and whitelist it. We are going to go over a few topics concerning Azure IP addressing, SNAT, checking your IP and some alternative solutoins for whitelisting Azure VIPs.

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
```
We can use the above then to know in a given subnet, Azure will assign .4 and .5 as the first addresses. This also holds true for VPN or ExR GW, which we don't expose those addresses in the UI. Either .4 or .5 will be pingable, and both should be if the GW is setup for A/A for VPN.

# The Pseudo-VIP
Some background, any resource in Azure will have a public IP programmed for SNAT. Even if the VM is assinged a private IP (DIP), the Azure platform will still assign a platfrom generated public IP so the resource can communicate outside the VNET. Azure assigns this based on the given region the VM is in, and will assign a public IP prefix based on that region. The easist way to check that VIP in both Windows and Linux, is to simply run curl ifconfig.me or culr ifconfig.io (in bash or cmd)
```bash
C:\Windows\system32>ipconfig                                                    
                                                                                
Windows IP Configuration                                                        
                                                                                
                                                                                
Ethernet adapter Ethernet:                                                      
                                                                                
   Connection-specific DNS Suffix  . : vxtyv5ak24ietmdv15r0ycyipa.bx.internal.cl
oudapp.net                                                                      
   Link-local IPv6 Address . . . . . : fe80::fd13:9d2c:7a15:542d%6              
   IPv4 Address. . . . . . . . . . . : 10.2.0.5                                 
   Subnet Mask . . . . . . . . . . . : 255.255.255.0                            
   Default Gateway . . . . . . . . . : 10.2.0.1                                 
                                                                                
C:\Windows\system32>curl ifconfig.me                                            
40.76.244.175
```
We can see the platform assigned 40.76.244.175, even though this VM only as a DIP assigned

![image](https://user-images.githubusercontent.com/55964102/193902852-3f484eed-30b7-439d-98ce-1a9b1113f17a.png)

# Azure SNAT
From the above snippet, as explained the platform assigned that address to provide default outbound connections. Azure will use this address for SNAT connections leaving the VNET. Often customers will have many VMs only with a DIPs, and they don't want to whitelist many VIP addresses say for application requirements and security. Technically these addresses will never change, as long as the VM instance(s) are not deallocated. There are two easy workarounds to provide a group of VMs a single outbound addresses for SNAT for whitelisting purposes:

Create an Azure external SLB in portal, a back-end pool and dummy LB rule. By default, the SLB will not program SNAT until you create a LB rule. Customers often don't like this solution, because they don't want to expose a LB rule from the internet to their back-end VMs. As long as a dummy high number port is created and nothing is listening on that port, that will be sufficeint to program SNAT and all VMs in the back-end pool will use that Front-End IP for outbound connections:

![image](https://user-images.githubusercontent.com/55964102/193906089-e61fcfa9-181f-4dc2-a56d-2bdfbdbfc149.png)

The second option is NAT-GW. This will serve the same function as the SLB, and all VMs in the given subnet will SNAT to the given NAT-GW IP(s) address for outbound connections.NAT-GW is also the better choice in terms of SNAT exahaustion as it provides the full range of ports and its not based on the number of VMs in the back-end pool. The advantage is that you don't need to create a dummy LB rule to program SNAT for NAT-GW. It also important to note, this can be combined with SLB above, but NAT-GW will take precendance over SLB for SNAT, even with outbound rules:

![image](https://user-images.githubusercontent.com/55964102/193910886-234b697f-5950-4e09-8c66-e9e9da08bec8.png)


The third option is to simply use a VM or NVA with a public IP, not really recommended for security reasons, or use a VM without a public IP and SNAT will be providedw with the pseduo VIP as explained above. If you don't want those options, SLB or Azure NAT Gateway are the bettter options! 

Public docs on defualt outbound access in Azure:
[SNAT Behavior](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)

# Conclusion
From this article we can confirm basic concepts of Azure IP addressing, check a VMs platform provided SNAT addresses, and two alternatives to providing a single outbound address for SNAT in order to prevent whitelisting of many IPs that may be needed for an application. 

