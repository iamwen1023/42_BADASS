# 42_BADASS

BADASS is a networking project that builds progressively from simple host-router communication to dynamic VXLAN overlays with BGP EVPN. The project is divided into three parts (P1, P2, P3), each building on the previous.

# Notions

## Tools

### GNS3
GNS3 is a network emulator that lets you build and test complete network topologies on your computer. Instead of wiring up physical routers, switches, and hosts, you can drag virtual devices onto a canvas and connect them with links. Each device runs its own operating system or image (like FRRouting for routers, or Alpine Linux for hosts), so you can configure them exactly like real hardware.

### FRRouting
An open source image that provides basic routing software, so that your container can act as a router. These routers can look at the destination address, consult their routing table, and forward the packet to the next hop. They follow packet routing protocols like OSPF, BGP and RIP. FRRouting is an example of packet routing software. \
The FRRouting image includes:
* **Busybox**, a single binary with a collection of networking commands, like ping. 
* Routing protocol **daemons** (need to be activated with a config file), that will handle the routing protocol they are designed for.
* **Zebra**, a daemon that manages the routing table. It takes the routing tables that it learned from the protocol daemons, and installs the correct routes.

## Protocols

### BGP
BGP stands for Border Gateway Protocol, and this is basically how the internet works. BGP doesn’t forward packets itself, but it advertises routes, so that separate private networks can build routing tables. The private networks are mostly big service providers, like telecom providers, and government agencies. BGP is based on policy and rules, which is why traffic doesn’t necessarily take the shortest route. It’s extremely flexible, because changes advertise themselves.

### OSPF
OSPF stands for Open Shortest Path First. This is what will be used within companies to efficiently move packets. Routers advertise direct links, so they all have a complete map of the entire network, and can choose the shortest path.

### ISIS
ISIS stands for Intermediate System to Intermediate System, and is similar to OSPF, but older. Here also all routers are aware of the complete network. The big difference is that ISIS runs over layer 2, it does not encapsulate layer 3 information. It’s used within the backbone of large service providers, because it’s robust, and scalable.

### ARP
ARP stands for Address Resolution Protocol. It is used to find the MAC address that corresponds to an IP address on a local network. A device will send a message asking where IP x.x.x.x is, that device replies with its MAC address, so that the frame can be sent through layer 2. 

## General concepts

### Layer 2
On Layer 2, devices communicate using ethernet frames. Each frame has a source and destination MAC address, which identifies the physical or virtual network interface. Traffic is being directed by switches and bridges. They learn which MACs live behind which ports and forward frames accordingly. Layer 2 is about direct delivery inside the same local network segment (or subnet).

### Layer 3
On Layer 3, devices communicate using packets. Each packet has a source and destination IP address, which is independent of the physical interface. Traffic is directed by routers. They look at the destination IP, consult their routing table, and forward the packet to the next hop. Layer 3 is about getting data from one network segment to another. As soon as it reaches the destination subnet, it will be stripped of the layer 3 information, and brought to its destination over layer 2.

### Broadcast vs multicast
Broadcasting is delivering traffic to each connected port on the network. This is for example used for ARP (Address Resolution Protocol), to learn routes for unknown addresses. This is also how routing messages with routing information are spread. Multicast works the other way. A frame is also sent to a group of devices, but only the ones that have indicated that they are interested in this specific data. You can see this as a group of subscribers.

### Route Reflector
A concept used to reduce the number of BGP peerings in a large network. Historically, in an internal BGP network, all routers know the complete network map. Route Reflector works with reflectors, and clients. The reflectors are main nodes that advertise network information to their clients. 

### Static vs dynamic
When you configure a VXLAN network completely by hand, this is called static setup. You manually create bridges and interfaces, and map the IP-routes. Whenever a new host is connected to the VXLAN, you need to reconfigure the VTEPs. Adding BGP EVPN makes the network dynamic, which mean the VXLAN can now use BGP to advertise the location of IP-addresses and MAC-addresses. They learn about other routes from the control plane. In a dynamic network, flooding frames is not necessary, because reachability is being advertised with BGP updates.

## Devices

### Switch
A switch is a layer 2 device that forwards ethernet frames. They are usually hardware appliances, but virtual switches exist too now. A switch learns which MAC addresses are reachable through which port, and sends the data there. The learning process happens in two ways:
* MAC addresses of received packages are linked to the port it comes from.
* When the destination MAC address is unknown, the switch will flood the frame to all ports, and link the port to the address when it gets a response that it has been received successfully.

### Bridge
Connects multiple layer 2 network segments. They used to be physical boxes, but now if we talk about a bridge, it’s mostly software. Like a switch, it learns the MAC addresses on the interfaces on all ports, and only forwards frames where they need to go. Or broadcasts frames if they don’t know the destination yet. A bridge is often a bit more basic than a switch.

### VLAN
VLAN stands for Virtual Local Area Network. With a VLAN, one big ethernet network is being split into different logical networks, so they can be managed separately. These small networks are connected with each other through routers. A VLAN works with a 4-byte header on the data frame, of which 12 bits reserved for the VLAN ID. This means that a complete network can have a maximum of 4096 VLANs.

### VXLAN
The X in VXLAN stands for Extended. A VXLAN has the same purpose as a VLAN, it splits one big physical network into smaller virtual networks. The difference is that it wraps the ethernet frame in a UDP header, with a 24-bit Virtual Network Identifier (**VNI**), which means it can be sent across IP-networks. Where a VLAN can have only 4096 networks, a VXLAN can connect 16 million. VXLAN traffic gets deliverd in two steps:
* With the **VNI** in the UDP header it gets sent to the bridge of its local subnet (layer 3).
* With the VLAN ID in the header of the unwrapped ethernet frame, it gets forwarded to the right destination (layer 2).

### VTEP
VTEP stands for VXLAN Tunnel End Point. This device is responsible for wrapping ethernet frames with a UDP header, so it can be sent to through the VXLAN tunnel. It also strips received frames from their UDP header, so they can be treated by the local bridge to find their destination. Each VTEP has at least one underlay IP address, used as the source/destination of VXLAN tunnels.

### EVPN
EVPN stands for Ethernet Virtual Private Network. And is an extension of BGP that is used to distribute layer 2 and layer 3 reachability information over a layer 3 network. It doesn’t only advertise IP-addresses, like BGP, but also tells you which MAC addresses or VLANs are accessible across the network. 
