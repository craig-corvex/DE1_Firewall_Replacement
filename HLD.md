
# Executive Summary


The Firewall Replacement project is the final part of the larger Edge Network re-design project. In the Edge Network re-design, we introduced a pair of Juniper routers to perform ingress/egress routing between the Internet and the Delaware Ethernet (North-South) network fabric. In this project, we will replace the existing Netgate TNSR firewalls with Juniper SRX 4300 firewall appliances. We look to achieve the following with the new firewall layer: 1) increase overall network reliability and operational stability; 2) increase network resiliency via a true highly availabile implentation, especially as it relates to potential hardware failures; 3) increase network performance and capabilities, both in terms of number of available ethernet ports, network segmentation, and platform feature/functionality; and 4) adoption of the well-known and well-supported firewall platform to reduce the overall management burden, while supporting programmatic and automated platform configuration, monitoring and management. 


# Architecture

The Firewall layer sits above the Ethernet fabric layer and below the Routed Edge layer within the Data Center converged network. The layer segments the network into distinct security zones. Each security zone is defined by a set of network interfaces and a single cohesive security policy with controls for ingress and egress network traffic.  


<img width="241" height="460" alt="New_Firewall_Architecture drawio" src="https://github.com/user-attachments/assets/2083a47b-2741-4315-8449-025f967dec9a" />




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

In the design of the new Firewall layer, we adhere to the Juniper reference architecture for Multi-node High Availability (https://www.juniper.net/documentation/us/en/software/junos/high-availability/topics/topic-map/mnha-introduction.html), and determined that the best deployment model for our particular needs is the Hybrid deployment model. In this MNHA deployment model, the firewalls employ Layer 3 routing (specificly, the BGP protocol) to provide high availability via redundant routed path upstream to the Edge routing layer, and employs Layer 2 high availability for downstream hosts via floating Virtual IP addresses as the default gateways for the respective subnets.  

One advantage of this deployment model is that both Firewalls may act in an Active capacity -- that is, both Firewalls are active in the data plane simultaneously, and each can process traffic independant of the other. Contrast this with the Active/Standby model, where only one Firewall may process traffic unless there is a failure and subsequent failover. The hybrid model does have the caveat that the Firewall that owns the VIP is active for that subnet, since the VIP is the floating default gateway. However, there is no requirement that a single Firewall must be the owner of all the VIPs simultatneously -- rather, the active VIPs can be distributed between the two Firewalls. The ownership of the VIP is determined by configuration, measured health of the Firewall. or active path monitoring.

The second advantage of the MNHA hybrid model is that, for the Layer 3 routed connections, the preferred routed path simply determines which Firewall should recieve the network traffic. Traffic need not be "pinned" to a particular Firewall. This said, we implement routing policy to ensure symetric ingress and egress paths to ensure a consistent and predictable network behavior.

A schematic diagram is provided here:

<img width="390" height="717" alt="FW-New_Edge_hybrid-model drawio" src="https://github.com/user-attachments/assets/a390186b-7cfb-40bd-a96d-f582c1560174" />


 ## Default Gateway

The new Firewalls provide Layer 2 high availability to hosts via per-subnet floating IP default gateway functionality. With this functionality, the hosts are configured with a default gateway address that lives on the Firewall pair. The active Firewall responds as the default gateway unless the activeness verification check fails. In that case, the standyby Firewall will elevate itself to the Active state and will begin to respond as the default gateway. As was mentioned previously, there is no requirement that one Firewall perform as the active Firewall for all subnets simultaneously. Rather, one Firewall may be active for one set of subnets, while its peer may be active for a second set of subnets.

Activeness is determined in the following ways. 1) A priority value is configured on the preferred Firewall for the specific Security Group (subnet), which ensures that it becomes the active Firewall on boot; 2) an activeness probe is enabled to actively test and respond to upstream network failures, tiggering a reduction of the priorty value and a change of activeness; and 3) path monitoring is enabled via an injected signal route, which verifies that a specific remote route is learned and is present in the Firewall's routing table as test that the path to that route is valid. 


 ##  BGP Routing

The new Firewalls use BGP routing to provide high availability via redundant routed paths upstream to the Edge routing layer. Both Firewalls actively and independantly participate in BGP . The new Firewall-to-Edge router topology is different than our current topology. In our current topology, we connect each Firewall to each Edge router, in a so-called "bowtie" configuration. This ensures physical redundancy, and reduces the hop count for packets traversing across the Edge fabric (e.g. from Firewall11 to Edge router-2 vs Firewall-1 to Edge Router-1 to Edge Router-2). In the new design, Firewall-1 only has a physical connection to Edge Router-1, and Firewall-2 only has a connection to Edge Router-2). We chose this design to ensure ingress and egress path symmetry and simplify the BGP route policy design. Each new Firewall will only have a single routed path ingress and egress (two ports in link aggregation). In case of an upstream failure, the active Firewall will failover to the standby (for the subnets the active Firewall is primary). To reach the opposite-side ISP, the Edge Router-1 to Edge Router-2 interconnect link is used. 

Another design change we have made is to configure the new Firewalls in their own private Autonomous System (AS), vs having them in the same public AS the Edge Routers. The implication of this is that we will now use external BGP (eBGP) between the Edge Layer and the Firewall Layer vs using internal BGP (iBGP). This design change gives us much more flexibility in terms of routing policy because eBGP has more traffic control cababilties than iBGP, owning largely to the fact that iBGP mandates that an iBGP router not re-advertise routes it learns from other iBGP routers. Thus, iBGP requires a full mesh of iBGP speakers or a route reflector to ensure that each iBGP router have the same routing view as ther iBGP routers.  An additional benefit of having the Firewalls in a private, unique AS is that it follows an architecture where new Clusters may be added to the Edge fabric without impacting the existing Cluster. In this design, the Edge Routers act much like an ISP, where individual Compute clusters with unique AS zones are kept distinct and seperate from other Compute clusters, but are able to share the upstream Internet connections.

A illustrative diagram is provided below.

<img width="731" height="557" alt="FW-New_Edge_AS drawio" src="https://github.com/user-attachments/assets/d549e8eb-1578-4a42-8299-7bbd06835967" />



 ## Stateful firewall rules

The primary difference between the SRX Firewall implementation and the Netgate TNSR implementation is that the SRX introduces Security Zones as the point of security enforcement, rather than the individual interfaces. A Security Zone is a set of one or more interfaces representing one or more IP subnets. Security policy rules are applied to the Zone rather than the individual interface. One the one hand, this makes the security posture easier to enforce, but on the other hand we must be mindful that rules applied to the Zone apply to all interface members of that zone. In other to simplify this, we organize the Security zones based on function. Specifically, we configure the following:

We provide a diagram below.

<img width="241" height="380" alt="New_Firewall_Zones drawio" src="https://github.com/user-attachments/assets/4a4f6abb-f7cf-4478-864a-8824f33f796e" />




|Zone|Posture|Description|
|--------|-------|-----------|
|Edge|Untrust|All networks external to the DE1 fabric|
|Public|Trust|Public IP subnets for customer or internal use|
|Services|Trust|Central core services|
|Management|Trust|Tool and monitoring|








 ## NAT

 ## Wireguard VPN



