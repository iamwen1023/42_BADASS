## P2

For P2, we use the FRRouting and host docker images of P1 to make a more complex topology. We extend the network that we made with a VXLAN, to connect Layer-2 segments across a Layer-3 underlay:
* Host1 and Host2 are Layer-2-attached to their routers’ bridges (`br0`) via `eth1`.
* `br0` on each router is a software switch joining `eth1` (access) and `vxlan10` (tunnel).
* `vxlan10` encapsulates Layer-2 frames into UDP port 4789 over the underlay `eth0` path between routers.

The traffic flow from Host1 to Host2 should be as follows:
1) Host1 sends ARP/ICMP to 20.1.1.20 out its NIC.
2) `eth1` on Router1 receives the frame and forwards it via `br0`.
3) If destination is remote, `br0` sends unknown/broadcast frames to `vxlan10`.
4) `vxlan10` encapsulates Layer-2 frame inside UDP/IPv4 and sends via `eth0` on Router1 (10.1.1.1) to `eth0` on Router2 (10.1.1.2).
5) Router2 `vxlan10` decapsulates, feeds the inner Layer-2 frame into `br0`.
6) `br0` forwards to `eth1` toward Host2.
7) Host2 receives ARP/ICMP and replies; reverse path is symmetric.

### Steps

Open a new project in GNS3, and create a topology with 2 host images, 2 routers, and a basic Ethernet Switch. Make sure that the connections have the same ethernet devices as seen on the picture in the subject. Click play to turn on all devices.

Open the auxiliary console of each device, open the file in this folder that corresponds to the image, and execute the commands one by one. For the routers, use the normal files first, test, then use the file suffixed by `_multi` to test the multicast setup.

To test, ping between the two routers, and between the two hosts:
```
ping <dest ip addr>
```
To see if everything is working as expected, click on one of the connections and open Wireshark (start capture) to inspect the traffic.