
# Executive Summary


The Firewall Replacement project is the final part of the larger Edge Network re-design project. In the Edge Network re-design, we introduced a pair of Juniper routers to perform ingress/egress routing between the Internet and the Delaware Ethernet (North-South) network fabric. In this project, we will replace the existing Netgate TNSR firewalls with Juniper SRX 4300 firewall appliances. We look to achieve the following with the new firewall layer: 1) increase overall network reliability and operational stability; 2) increase network resiliency via a true highly availabile implentation, especially as it relates to potential hardware failures; 3) increase network performance and capabilities, both in terms of number of available ethernet ports, network segmentation, and platform feature/functionality; and 4) adoption of the well-known and well-supported firewall platform to reduce the overall management burden, while supporting programmatic and automated platform configuration, monitoring and management. 


# Architecture

The Firewall layer sits above the Ethernet fabric layer and below the Routed Edge layer within the Data Center converged network. The layer segments the network into distinct security zones. Each security zone is defined by a set of network interfaces and a single cohesive security policy with controls for ingress and egress network traffic.  


<img width="241" height="460" alt="New_Firewall_Architecture drawio" src="https://github.com/user-attachments/assets/2083a47b-2741-4315-8449-025f967dec9a" />


At time of writing, we define four security zones: the Edge zone, the Public zone, the Services zone, and the Management zone. These function of these zones are described below:

|Zone|Posture|Description|
|--------|-------|-----------|
|Edge|Untrust|All networks external to the DE1 fabric|
|Public|Trust|Public IP subnets for customer or internal use|
|Services|Trust|Central core services|
|Management|Trust|Tool and monitoring|


# Physical Layer Design

The physical layer design of the new Firewall layer loosely adheres to the existing physical layer design and is shown in the figure below. Some notable exceptions are: 

(1) the SRX firewalls have sufficient port count to connect the firewalls directly to the edge routers, bypassing an underlying switching layer. This simplifies the physical layer design and implmentation, reduces configuration complexity, and makes troubleshooting and fault identification easier. Additionally, in the case of a link fault, the failover time is reduced because when a link is point-to-point vs through a switch, the link down signal on the routers is immediate; otherwise, the connection timer would need to expire before the router signaled the connection down.  

(2) the increased port count is that we have seperated key subnets onto different physical interfaces. Consider our current physical network design: we have our two primary "revenue" subnets -- 104.218.236.0/23 and 160.72.54.124/25, and our two primary Services subnets -- 10130.0.0/24 and 10.120.0.0/24, sharing a single physical 100G port. In the nbew design, the two revenue subnets share a single 100G, whereas our Services subnets are placed on a seperate physical 100G interface. 

(3) we have moved our OOB network subnets -- 172.20.0.0/24, 172.20.2.0/24, and 172.20.3.0/24 onto a 25G interface, upgrading from the existing 1G interface. 

(4) we have increased the uplink capacity from the Firewall layer to the Routed Edge layer from 100G to 2x 100G per SRX-J7024 pair. This configuration increases both bandwidth capacity and the resliliency of the topology. 

(5) we have left 2x 100G ports per SRX as spares, should we need to expand further. Note that the throughput of the SRX 4300 tops out at ~70 Gbps, so we shouldn't expect full line rate on all ports.

We reserve 2 10G ports to function as the InterChassis Link (ICL), which interconnects the two firewalls. The ICL is used to exchange firewall state information to ensure the packet processing state of the two firewalls in the Multi-node High Availability (MNHA) cluster remain synchronized. The ICL links are standard 10G ports, but are not used for transit traffic (e.g. as a packet forwarding/routing link between firewall chassis). The ICL is a logical link, though we choose to implement this logical link on two physical links, as provides significant reliability.


<img width="841" height="767" alt="FW-New_Edge drawio" src="https://github.com/user-attachments/assets/b5a119e5-d822-48a4-a62e-32831a848f7e" />



# Key Functions


## Mult-node High Availability

In the design of the new Firewall layer, we have adhered to the Juniper reference architecture for Multi-node High Availability (https://www.juniper.net/documentation/us/en/software/junos/high-availability/topics/topic-map/mnha-introduction.html), and determined that the best deployment model for our particular needs is the Hybrid deployment model. In this MNHA deployment model, the firewalls employ Layer 3 routing (specificly, the BGP protocol) to provide high availability via redundant routed path upstream to the Edge routing layer, and employs Layer 2 high availability for downstream hosts via floating Virtual IP addresses as the default gateways for the respecitive subnets.  

One advantage of this deployment model is that both Firewalls may act in an Active/Active capacity -- that is, both Firewalls are active simultaneous, and each can process traffic independant of the other. Contrast this with the Active/Standby model, where only one Firewall may process traffic unless there is a failure and subsequent failover. The hybrid model does have the caveat that the Firewall that owns the VIP is active for that subnet, since the VIP is the default gateway. However, there is no requirement that a single Firewall must be the owner of all the VIPs simultatneously -- rather, the active VIPs can be distributed between the two Firewalls. The ownership of the VIP is determined by configuration and by the measured health of the Firewall.

The second advantage of the MNHA hybrid model is that, for the Layer 3 routed connections, the preferred routed path simply determines which Firewall should recieve the network traffic. Traffic need not be "pinned" to a particular Firewall. This said, we implement routing policy to ensure symetric ingress and egress paths to ensure a consistent and predictable network behavior.

A schematic diagram is provided here:

<img width="390" height="717" alt="FW-New_Edge_hybrid-model drawio" src="https://github.com/user-attachments/assets/a390186b-7cfb-40bd-a96d-f582c1560174" />


 ## Default Gateway





 ##  BGP Routing

 ## Stateful firewall rules

 ## NAT

 ## Wireguard VPN



