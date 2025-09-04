
Host1 and Host2 are L2-attached to their routers’ bridges (br0) via eth1.
br0 on each router is a software switch joining eth1 (access) and vxlan10 (tunnel).
vxlan10 encapsulates L2 frames into UDP/4789 over the underlay eth0 path between routers.
Traffic flow Host1 → Host2:
1) Host1 sends ARP/ICMP to 20.1.1.20 out its NIC.
2) Router1 eth1 receives the frame and forwards it via br0.
3) If destination is remote, br0 sends unknown/broadcast frames to vxlan10.
4) vxlan10 encapsulates L2 frame inside UDP/IPv4 and sends via Router1 eth0 (10.1.1.1) to Router2 eth0 (10.1.1.2).
5) Router2 vxlan10 decapsulates, feeds the inner L2 frame into br0.
6) br0 forwards to eth1 toward Host2.
7) Host2 receives ARP/ICMP and replies; reverse path is symmetric.
Key relationships:
eth1 ports: access-facing interfaces to hosts. No IPs here.
br0: L2 bridge domain; owns the subnet IPs (20.1.1.1/24 and 20.1.1.2/24).
vxlan10: tunnel port, part of the bridge. Carries bridged frames across the underlay.
eth0: underlay interface carrying UDP/4789 between routers (10.1.1.1 ↔ 10.1.1.2).
What must be true:
vxlan10 id/local/remote match on both routers.
br0 is UP and has the service IP; eth1/vxlan10 have no IPs.
Underlay eth0 interfaces can reach each other.
MTU aligned (or underlay increased) for non-fragmented traffic.
