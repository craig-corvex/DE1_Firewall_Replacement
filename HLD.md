
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

<img width="841" height="767" alt="FW-New_Edge drawio" src="https://github.com/user-attachments/assets/b5a119e5-d822-48a4-a62e-32831a848f7e" />





# Key Functions


 ## MNHA – Hybrid mode

 ## Default Gateway

 ##  BGP Routing

 ## Stateful firewall rules

 ## NAT

 ## Wireguard VPN



