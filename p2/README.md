## P2

For P2, we use the FRRouting and host docker images of P1 to make a more complex topology. We extend the network that we made with a VXLAN, to connect Layer-2 segments across a Layer-3 underlay:
* Host1 and Host2 are L2-attached to their routers’ bridges (`br0`) via `eth1`.
* `br0` on each router is a software switch joining `eth1` (access) and `vxlan10` (tunnel).
* `vxlan10` encapsulates L2 frames into UDP port 4789 over the underlay eth0 path between routers.

The traffic flow from Host1 to Host2 should be as follows:
1) Host1 sends ARP/ICMP to 20.1.1.20 out its NIC.
2) `eth1` on Router1 receives the frame and forwards it via `br0`.
3) If destination is remote, `br0` sends unknown/broadcast frames to `vxlan10`.
4) `vxlan10` encapsulates Layer-2 frame inside UDP/IPv4 and sends via `eth0` on Router1 (10.1.1.1) to `eth0` on Router2 (10.1.1.2).
5) Router2 `vxlan10` decapsulates, feeds the inner Layer-2 frame into `br0`.
6) `br0` forwards to `eth1` toward Host2.
7) Host2 receives ARP/ICMP and replies; reverse path is symmetric.

### Steps

The files in this folder describe which steps we need to take to configure the network emulation. There is one file per image. Create the image, connect the devices as shown in the subject, and execute the commands one by one. 