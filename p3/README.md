## P3

In P3, we add an EVPN to our network, which makes that the routers can automatically figure out where all the devices are. In our topology, we have three different kind of network devices:
* A spine, a FRRouting image, that will act as a route reflection server.
* Three leafs, the same FRRouting images, that will act as VTEPs, which means that they wrap and unwrap packages for the VXLAN.
* Three host images, the same simple Alpine image that we have been using since the beginning.

### Steps

Open a new project in GNS3, and create a topology with 3 host images, 3 router images, and another router on top. Make sure that the connections have the same ethernet devices as seen on the picture in the subject. Then click play to turn all devices on. 

The three routers in the middle are the leaves, and the top router is the spine. 

Now open the auxiliary console of each image separately. For the host machines, you can execute the commands in the file with corresponding name one by one. For the leaves and spine you need to add the configurations manually as described below.

In the auxiliary console:
```
# start vtysh shell in config mode
vtysh
conf t
```

Then go to the config files and execute these parts:
* Zebra configuration
* OSPF configuration for underlay
* BGP configuration
* Logging (if mentioned)

Quit all the shells, so the configs will be sourced. Then re-open the auxiliary consoles of the leaf images and execute the rest of the commands one by one. 

Ping one host from the other:
```
ping <ip addr>
```
To see if everything is working as expected, click on one of the connections and open Wireshark to inspect the traffic.