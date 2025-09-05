Big picture
Underlay: OSPF provides IP reachability between routers (spine/RR and leaves).
Overlay: BGP EVPN carries MAC/IP reachability for VXLAN VNI 10, so VTEPs learn/advertise MACs dynamically.
Data plane: On each leaf, br0 bridges host-facing eth1 with vxlan10. EVPN programs vxlan10 with remote VTEP next-hops (no static remotes).
What Router1 (spine/RR) config does
interface eth0/eth1/eth2 with /30s, and interface lo 1.1.1.1/32: build the underlay.
router ospf on all links (area 0): distributes reachability (including loopback).
router bgp 1:
neighbor ibgp peer-group with update-source lo: iBGP over loopbacks.
bgp listen range 1.1.1.0/29 peer-group ibgp: auto-accept iBGP sessions from leaves’ loopbacks in that range.
address-family l2vpn evpn + neighbor ibgp route-reflector-client: spine acts as route reflector for EVPN; it reflects EVPN routes between leaves. Spine itself isn’t a VTEP.
Concept: Spine learns loopbacks via OSPF, forms iBGP EVPN with leaves, and reflects EVPN MAC/IP routes. It doesn’t bridge or run VXLAN.
What Router2 (leaf/VTEP) config does
Linux:
ip link add br0 type bridge → software switch for the tenant L2.
ip link add vxlan10 type vxlan id 10 dstport 4789 → VXLAN tunnel port (no static remote; EVPN will populate FDB).
brctl addif br0 vxlan10 and brctl addif br0 eth1 → plug host port and VXLAN port into the same L2.
FRR:
interface eth0 + OSPF: underlay adjacency to spine.
interface lo 1.1.1.2/32: stable VTEP/BGP ID.
router bgp 1 neighbor to spine 1.1.1.1 using update-source lo.
address-family l2vpn evpn + advertise-all-vni: advertise and learn MACs for VNI 10 via EVPN.
Concept: Leaf bridges local hosts with br0, encapsulates traffic into VXLAN on vxlan10, and uses EVPN to learn and program remote MACs and their VTEP IPs.
How EVPN + VXLAN ties together
Each leaf’s FRR:
Learns local MACs seen on br0 and advertises them as EVPN Type-2 routes (MAC/IP) with VNI 10 and its VTEP IP.
Receives remote MACs from other leaves via the spine RR, and programs Linux bridge FDB entries with dst <remote VTEP IP> on vxlan10.
Data flow hostA→hostB:
Frame enters leafA eth1 → br0 → hits FDB entry pointing to vxlan10 with remote VTEP IP → encapsulated (UDP/4789) out eth0.
LeafB vxlan10 decapsulates → br0 → eth1 to hostB.
No static remote on vxlan10 is needed because EVPN provides the unicast replication list (ingress replication).
Two important implementation details
On Linux leaves, also set in FRR:
interface vxlan10
vxlan local-tunnelip 1.1.1.2 (use the leaf’s loopback as VTEP IP)
bridge access 10
When creating vxlan10 in Linux, specify the underlay device for clarity:
ip link add vxlan10 type vxlan id 10 dev eth0 dstport 4789
Don’t assign an IP to vxlan10 or eth1; if you need an IP in the tenant subnet, put it on br0.
Minimal test plan
Underlay:
show ip ospf neighbor on all; pings between loopbacks 1.1.1.1 ↔ 1.1.1.2 should succeed.
EVPN:
show bgp l2vpn evpn summary on spine and leaves → Established.
show evpn vni and show evpn mac vni 10 on leaves → see local and remote MACs.
Data plane:
Put hosts behind each leaf on the same subnet (e.g., 20.1.1.10/24, 20.1.1.20/24, 20.1.1.30/24).
Ping across hosts; on leaves, tcpdump -ni eth0 udp port 4789 shows encapsulated traffic.
In short: OSPF builds reachability for loopbacks; iBGP EVPN (with a spine RR) distributes MAC/IP for VNI 10; FRR programs the Linux bridge/VXLAN so your leaves bridge L2 across the IP underlay without static VXLAN peers.
what is OSPF

OSPF in a nutshell
OSPF (Open Shortest Path First): a link-state IGP that builds IP reachability inside one organization (intra-domain).
How it works:
Routers form adjacencies, flood LSAs (link-state advertisements), build a full topology map.
Each router runs SPF (Dijkstra) to compute shortest paths and installs routes.
Key features:
Fast convergence, hierarchical design with areas (e.g., area 0 backbone).
Auth support, equal-cost multipath (ECMP), uses cost (usually based on bandwidth).
Operates directly over IP (protocol 89), no TCP/UDP.
Why used in your setup:
Provides underlay IP reachability (e.g., between loopbacks and underlay interfaces) so BGP EVPN sessions can form and VXLAN VTEPs can reach each other.
do you think the cmd are good? 

Short answer: Close, but fix a few key points.
What to change:
Don’t put an IP on vxlan10 or eth1. If you need a gateway, put it on br0. Underlay IPs stay on eth0; BGP uses a loopback.
For EVPN, don’t set remote/group on vxlan10. FRR will program peers dynamically. Create it with dev eth0 only.
Prefer iproute2 bridging (works everywhere): use ip link set dev <if> master br0 instead of brctl addif.
OSPF “network” statements must match your actual underlay/loopback subnets (your /30s and the /32 loopbacks).
On leaves, tell FRR the VTEP source and VNI binding on the vxlan10 interface.
Minimal corrected leaf (Linux):
Leaf (FRR):
Spine/RR (FRR):
Why these changes:
EVPN programs the VXLAN FDB; static remote/multicast isn’t needed.
IP on br0 keeps the bridge as the L2 domain; vxlan10 is just a port.
OSPF underlay gives reachability to loopbacks so iBGP EVPN can form.
vxlan local-tunnelip tells FRR which source IP to advertise as the VTEP.
Quick tests:
Underlay: show ip ospf neighbor; ping spine/leaf loopbacks.
EVPN: show bgp l2vpn evpn summary; show evpn vni; show evpn mac vni 10.
Linux: ip -d link show vxlan10 (no remote, has master br0); bridge link | grep master br0.
Data: host behind each leaf on 20.1.1.0/24 → ping across; tcpdump -ni eth0 udp port 4789 shows encapsulation.
how to test if those two routers contact to each other?

Quick checks (underlay → control plane → overlay)
Underlay IP reachability (eth0):
OSPF adjacency (if using OSPF):
Loopback reachability (used for BGP sessions if configured):
BGP EVPN session state (if using EVPN):
VXLAN/bridge wiring on each router:
Overlay data test (host-to-host or router br0 IPs):
Observe encapsulation on underlay during overlay ping:
What “working” looks like:
Underlay pings succeed both ways.
OSPF neighbors are Full (if used).
BGP EVPN Established (if used).
vxlan10 is UP, in br0; eth1 also in br0.
Overlay pings succeed; tcpdump shows UDP/4789.
Don’t assign an IP to vxlan10 or eth1; if you need an IP in the tenant subnet, put it on br0.
Minimal test plan
Underlay:
show ip ospf neighbor on all; pings between loopbacks 1.1.1.1 ↔ 1.1.1.2 should succeed.
EVPN:
show bgp l2vpn evpn summary on spine and leaves → Established.
show evpn vni and show evpn mac vni 10 on leaves → see local and remote MACs.
Data plane:
Put hosts behind each leaf on the same subnet (e.g., 20.1.1.10/24, 20.1.1.20/24, 20.1.1.30/24).
Ping across hosts; on leaves, tcpdump -ni eth0 udp port 4789 shows encapsulated traffic.
In short: OSPF builds reachability for loopbacks; iBGP EVPN (with a spine RR) distributes MAC/IP for VNI 10; FRR programs the Linux bridge/VXLAN so your leaves bridge L2 across the IP underlay without static VXLAN peers.





vtysh -c "show bgp l2vpn evpn"
vtysh -c "show ip ospf neighbor"