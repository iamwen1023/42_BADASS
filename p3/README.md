## P3

In P3, we add an EVPN to our network, which makes that the routers can automatically figure out where all the devices are. In our topology, we have three different kind of network devices:
* A spine, a FRRouting image, that will act as a route reflection server.
* Three leafs, the same FRRouting images, that will act as VTEPs, which means that they wrap and unwrap packages for the VXLAN.
* Three host images, the same simple Alpine image that we have been using since the beginning.

### EVPN Route Types

EVPN uses different route types to handle different types of traffic and MAC learning:

#### Type 2 Routes - MAC/IP Advertisement Route
**Purpose**: Advertise known MAC address and IP address bindings across VXLAN tunnels.

**What it does**:
- When a VTEP learns a MAC address from a connected host, it advertises this information to other VTEPs
- Contains MAC address, IP address, VNI, and VTEP location information
- Enables direct unicast communication between hosts on different VTEPs

**Configuration**: BGP EVPN address-family with `advertise-all-vni` and route-targets

**Detailed BGP Configuration for Leaf Routers**:

**Step 1: Basic BGP Setup**
```bash
router bgp 1                    # Start BGP process in AS 1
 bgp router-id 1.1.1.X          # Set unique router ID (use loopback IP)
 no bgp ebgp-requires-policy    # Disable eBGP policy requirement for lab
```

**Step 2: Configure BGP Neighbor (Spine)**
```bash
 neighbor 1.1.1.1 remote-as 1   # Add spine router (1.1.1.1) as BGP neighbor
 neighbor 1.1.1.1 update-source lo  # Use loopback interface for BGP sessions
```

**Step 3: Configure EVPN Address Family**
```bash
 address-family l2vpn evpn      # Enter EVPN address family
  neighbor 1.1.1.1 activate     # Activate EVPN with spine router
  advertise-all-vni             # Automatically advertise all VNIs (Type 2 routes)
  
  vni 10                        # Configure VNI 10 for our VXLAN
   rd 1.1.1.X:10               # Route Distinguisher (unique per leaf)
   route-target import 1:10     # Import routes with RT 1:10 (learn from other leafs)
   route-target export 1:10     # Export routes with RT 1:10 (advertise to other leafs)
 exit-address-family            # Exit EVPN address family
```

**Command Explanations**:
- `router bgp 1`: Start BGP process in Autonomous System 1
- `bgp router-id`: Unique identifier for this BGP router (must be unique)
- `neighbor remote-as`: Define BGP neighbor and its AS number
- `update-source lo`: Use loopback interface for BGP sessions (more stable)
- `address-family l2vpn evpn`: Enter EVPN address family for L2VPN services
- `advertise-all-vni`: Automatically advertise all locally configured VNIs
- `rd`: Route Distinguisher - unique identifier for this VNI on this router
- `route-target import/export`: Control which routes to learn and advertise

**How Type 2 Routes Work**:
1. Host sends packet → Leaf learns MAC address
2. Leaf advertises MAC via BGP EVPN Type 2 route to Spine
3. Spine reflects Type 2 route to all other Leafs
4. Other Leafs learn "this MAC is on this Leaf"
5. Future packets go directly to correct Leaf (no flooding needed)

#### Type 3 Routes - Inclusive Multicast Ethernet Tag Route
**Purpose**: Handle BUM traffic (Broadcast, Unknown unicast, Multicast) intelligently.

**What it does**:
- Controls how unknown destination MAC addresses are flooded
- Replaces traditional VXLAN flooding with intelligent multicast
- Uses underlay IP network for efficient BUM traffic delivery

**Configuration**: VXLAN interface with `nolearning` parameter

**Detailed VXLAN Configuration**:

**Step 1: Create VXLAN Interface**
```bash
/sbin/ip link add vxlan10 type vxlan id 10 dev eth0 local 1.1.1.X dstport 4789 nolearning
```

**Command Breakdown**:
- `vxlan10`: Name of the VXLAN interface
- `type vxlan`: Create VXLAN tunnel interface
- `id 10`: VXLAN Network Identifier (VNI) - must match on all VTEPs
- `dev eth0`: Use eth0 as the underlay transport interface
- `local 1.1.1.X`: Local VTEP IP address (loopback IP)
- `dstport 4789`: VXLAN destination port (standard port)
- `nolearning`: **Critical parameter** - disables MAC learning on VXLAN interface

**Why `nolearning`?**:
- Forces all MAC learning through BGP EVPN Type 2 routes
- Enables intelligent flooding via BGP EVPN Type 3 routes
- Without it: traditional VXLAN flooding (less efficient)
- With it: EVPN-controlled learning and flooding (more efficient)

**How Type 3 Routes Work**:
1. `nolearning` disables MAC learning on VXLAN interface
2. Forces all MAC learning through BGP EVPN Type 2 routes
3. When unknown MAC destination is needed:
   - BGP EVPN Type 3 routes determine flood targets
   - Uses underlay IP addresses for efficient delivery
   - Only floods to VTEPs that need the traffic

#### Spine Router (Route Reflector) Configuration
**Purpose**: Acts as Route Reflector to reduce full-mesh BGP requirements between leafs.

**Detailed BGP Configuration for Spine Router**:

**Step 1: Basic BGP Setup**
```bash
router bgp 1                    # Start BGP process in AS 1
 bgp router-id 1.1.1.1          # Set unique router ID (spine loopback IP)
 no bgp ebgp-requires-policy    # Disable eBGP policy requirement for lab
```

**Step 2: Create iBGP Peer Group**
```bash
 neighbor ibgp peer-group       # Create peer group for all leaf routers
 neighbor ibgp remote-as 1      # All leafs are in same AS (1) - internal BGP
 neighbor ibgp update-source lo # Use loopback interface for BGP sessions
```

**Step 3: Dynamic Neighbor Discovery**
```bash
 bgp listen range 1.1.1.0/29 peer-group ibgp  # Listen for BGP connections
 # This covers IPs 1.1.1.1-1.1.1.6 (spine + 3 leafs + 2 spare)
 # Alternative: manually add each leaf as "neighbor 1.1.1.X peer-group ibgp"
```

**Step 4: Configure EVPN Route Reflection**
```bash
 address-family l2vpn evpn      # Enter EVPN address family
  neighbor ibgp activate        # Activate EVPN with all iBGP peers
  neighbor ibgp route-reflector-client  # Enable route reflection for all peers
 exit-address-family            # Exit EVPN address family
```

**Command Explanations**:
- `neighbor ibgp peer-group`: Create a group for all leaf routers
- `neighbor ibgp remote-as 1`: All leafs are in same AS (internal BGP)
- `bgp listen range`: Automatically accept BGP connections from IP range
- `route-reflector-client`: This spine will reflect routes between leafs
- **Why Route Reflector?**: Without it, each leaf would need BGP peering with every other leaf (full-mesh)

**How Route Reflection Works**:
1. Leaf1 advertises MAC via Type 2 route to Spine
2. Spine reflects Type 2 route to Leaf2 and Leaf3
3. Leaf2 and Leaf3 learn MAC location without direct BGP peering
4. Reduces full-mesh requirement: N leafs need only N connections to spine

#### Underlay IP Network
**Purpose**: Provides connectivity between VTEPs for BGP sessions and VXLAN tunnels.

**Configuration**: OSPF underlay network with point-to-point links between spine and leafs

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

Quit all the shells, so the configs will be sourced. Then re-open the auxiliary consoles of the leaf images and execute the rest of the commands one by one. 

Ping one host from the other:
```
ping <ip addr>
```

### Verification Commands

To verify that EVPN Type 2 and Type 3 routes are working correctly:

#### Check Type 2 Routes (MAC Address Advertisement):
```bash
# On any leaf router
vtysh -c "show bgp l2vpn evpn route type mac-ip"
vtysh -c "show bgp l2vpn evpn summary"

# Check MAC address table
brctl showmacs br10
bridge fdb show dev vxlan10
```

#### Check Type 3 Routes (Intelligent Multicast Flooding):
```bash
# On any leaf router
vtysh -c "show bgp l2vpn evpn route type multicast"

# Check VXLAN interface configuration
ip -d link show vxlan10
```

#### Check BGP Neighbor Status:
```bash
# On spine router
vtysh -c "show bgp summary"
vtysh -c "show bgp l2vpn evpn summary"
vtysh -c "show bgp peer-group"

# On leaf routers
vtysh -c "show bgp summary"
```

#### Spine-Specific Verification Commands:
```bash
# Check BGP neighbor status (should show all leafs connected)
vtysh -c "show bgp summary"

# Check EVPN summary (should show active EVPN sessions)
vtysh -c "show bgp l2vpn evpn summary"

# Check all EVPN routes (Type 2 and Type 3)
vtysh -c "show bgp l2vpn evpn route"

# Check Type 2 routes (MAC/IP advertisements from leafs)
vtysh -c "show bgp l2vpn evpn route type mac-ip"

# Check Type 3 routes (multicast flood routes)
vtysh -c "show bgp l2vpn evpn route type multicast"

# Check BGP peer group status
vtysh -c "show bgp peer-group"

# Check OSPF underlay connectivity
vtysh -c "show ip ospf neighbor"

# Check routing table
vtysh -c "show ip route"
```

**What to Look For**:
- **BGP Summary**: Should show 3 leaf neighbors in "Established" state
- **EVPN Routes**: Should see Type 2 routes from each leaf with MAC addresses
- **Peer Group**: Should show "ibgp" peer group with all leafs
- **OSPF**: Should show 3 OSPF neighbors (one for each leaf)
To see if everything is working as expected, click on one of the connections and open Wireshark to inspect the traffic.