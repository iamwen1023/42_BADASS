## P3

In P3, EVPN dynamically programs VXLAN FDBs so hosts can reach each other without static remote configuration.
* Spine/RR router runs OSPF underlay and iBGP route-reflector for EVPN.
* Leaf/VTEP routers bridge `eth1` and `vxlan10` via `br0`.
* EVPN advertises local MACs and learns remote MACs for VNI 10.

### Steps

The files in this folder describe which steps we need to take to configure the network emulation. There is one file per image. Create the image, connect the devices as shown in the subject, and execute the commands one by one. 