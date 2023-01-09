# VxLAN
Cisco Live:
- BRKDCN-2450 - VXLAN EVPN Day-2 operation  
- BRKDCT-3378 - Building Data Center Networks with VXLAN BGP-EVPN - done
- BRKDCT-2404 - VXLAN Deployment Models - A practical perspective  
- BRKDCN-2304 - L4-L7 Service Integration in Multi-Tenant VXLAN EVPN Data Center Fabrics
- LTRDCN-2223 - Implementing VXLAN In a Data Center
- DGTL-BRKDCN-1645 - Introduction to VXLAN - The future path of your datacenter  
  
Juniper:
- Data Center EVPN-VXLAN Fabric Architecture Guide
- DAY ONE: DAtA CENtEr FUNDAMENtALS  

- RFC 7348
- Virtual Extensible LAN, MAC in UDP encapsulation, port 4789, 50 bytes overhead
- MTU should be large
- Only transport requirement is unicast IP
- VLAN numbers can be different on all switches, but VNI number should be the same 
- Default gateway can be any device in a fabric
- VxLAN does not encapsulate CDP, STP....

## Components
- VNID - Vxlan Network Identifier - defines broadcast segment - transfered in VxLAN header - 3 bytes
- NVE - Network Virtualization Edge - interface where VXLAN ends - only 1 NVE interface is allowed on the switch.
- VTEP - VXLAN Tunnel Endpoint - Nexus - Leaf

## BUM traffic forwarding: data plane
Can be processed in 2 ways:
- Multicast replication - multicast is enabled on Underlay - BUM is not packed inside VxlAN - it is sent via Underlay - replication point is on rendezvous point - spine - it is difficult to configure multicast on Underlay - not used today
- Ingress replication - packed to VxLAN - and sent to all in VNI - configured statically or with BGP EVPN

## Operations
### Without BGP, ingress replication of BUM traffic
Host A in VLAN 1 SVI 1 sends ARP request for Host B in VLAN 2 svi 1. VTEP 1 encapsulates this ARP request to VXLAN and sends it to peer VTEPs for this SVI. VTEP 2 replies, and then all other traffic goes between these 2 VTEPs

### With BGP, ingress replication traffic
- Host A is online, Leaf-1 sees its MAC and forwards Route Type 2 to all other VTEPS, and now all VTEPs know that Host A is behind Leaf-1
- Host A in VLAN 1 SVI 1 sends ARP request for Host B in VLAN 2 svi 1
- Leaf 1 sends this ARP request in VXLAN to all VTEPs from which it got Route Type 3 - Inclusive Multicast Ethernet Tag with SNI 1
- All VTEPs which got ARP, forward it to all physical ports in this SVI
- Host B replies and Leaf 2 sends reply in VXLAN to Leaf 1

### BUM traffic via Multicast
- Host sends broadcast
- Leaf forwards it without VXLAN as a multicast to 225.2.2.1
- We configure a match between VNI and multicast address - only one multicast packet - Spines will propogate it to all subscribers - load is much less then ingress replication

## Host mobility
- VM moves from one VTEP to another and continue
- VTEP-2 creates route type 2 with higher Sequence Number
- All other VTEPs get a new route
- Route with higher sequence number wins
- Problems may appear if MAC is configured staticly on port

## Service models
- VLAN based service: one VLAN - one MAC VRF (RT, RD) - one bridge table, Ethernet Tag = 0 - one VNI - forwarding based on VNI - most popular - good for multi vendor
- VLAN bundle service - one bridge table, MAC VRF with RT RD for all VLANs - frames are sent with 802.1Q - Ethernet Tag = 0, MACs in different VLANs should not match
- VLAN aware bundle service - not supported by all vendors - all VLANs have one VRF(RT, RD) - every VLAN has its own bridge table with VNI and Ethernet tag = VNI

## Load balancing
The Layer 3 routes that form VXLAN tunnels use per-packet load balancing by default, which means that load balancing is implemented if there are ECMP paths to the remote VTEP. This is different from normal routing behavior in which per-packet load balancing is not used by default. (Normal routing uses per-prefix load balancing by default.)  
  
The source port field in the UDP header is used to enable ECMP load balancing of the VXLAN traffic in the Layer 3 network. This field is set to a hash of the inner packet fields, which results in a variable that ECMP can use to distinguish between tunnels (flows).  
  
None of the other fields that flow-based ECMP normally uses are suitable for use with VXLANs. All tunnels between the same two VTEPs have the same outer source and destination IP addresses, and the UDP destination port is set to port 4789 by definition. Therefore, none of these fields provide a sufficient way for ECMP to differentiate flows.    
That is why UDP is used, not IP for example.  
All VxLAN packets from one Leaf to another have different UDP source ports for load balancing purposes.

## Static configuration
It is called ingress-replication. No control plane is used. First packet (ARP) and broadcast are sent to all peers, configured for this VNI. All next packets are sent only to particular VTEP. VxLAN packets between leafs are load balanced between spines.

## Firewall injection
- Connected to border leaf via TRUNK interface
- All required subinterfaces are configured on firewall
- On all required clients we configure firewall as a default gateway
- All traffic between VLANs goes via firewall
- Bottle neck for the entire network
- High load on border leaf
- ePBR can be also used

## MAC VRF
- RD:MAC-PREFIX:ETI
- RD:IP-PREFIX:ETI
- MAC-PREFIX = MAC/48
- Ethernet Tag ID = 0 - in most cases - defines bridge table inside VRF - several bridge tables can be in one vrf - in most cases we add 1 vlan to one MAC VRF, this is required only when several VLANs are in 1 MAC VRF
- We add VLAN to MAC VRF instead of interface in IP VRF

## Configuration of static peers on Nexus 9000
We create special Loopback for overlay, announce it to BGP.  
For every VLAN where clients are connected we configure VNI.  
Next we configure nve interface, where we configure peers for every VNI.  
Static confogiration:
```
feature vn-segment-vlan-based
feature nv overlay

vlan 100
  name PROD
  vn-segment 100
  
interface Ethernet1/3
  switchport access vlan 100

interface nve1
  no shutdown
  source-interface loopback1
  member vni 100
    ingress-replication protocol static
      peer-ip 10.10.2.4
      peer-ip 10.10.2.5
  member vni 200
    ingress-replication protocol static
      peer-ip 10.10.2.4
      peer-ip 10.10.2.5
      
```

## Verification
Show all NVE neighboors, not very reliable data, because connections are connectionless :)  
Peer is up just because it exists in routing table. It even mau not accept VxLAN packets.

```
show nve peers
Interface Peer-IP                                 State LearnType Uptime   Router-Mac
--------- --------------------------------------  ----- --------- -------- -----------------
nve1      10.10.2.4                               Up    DP        01:08:14 n/a
nve1      10.10.2.5                               Down  DP        0.000000 n/a
```
Show info about nve interface
```
leaf-1(config)# show int nve1
nve1 is up
admin state is up,  Hardware: NVE
  MTU 9216 bytes
  Encapsulation VXLAN
  Auto-mdix is turned off
  RX
    ucast: 71 pkts, 6890 bytes - mcast: 15 pkts, 960 bytes
  TX
    ucast: 92 pkts, 12834 bytes - mcast: 0 pkts, 0 bytes
```

## EVPN
- Ethernet VPN (EVPN) is a technology for carrying layer 2 Ethernet traffic as a virtual private network using wide area network protocols. EVPN technologies include Ethernet over MPLS and Ethernet over VXLAN
- AFI 25, SAFI 70
- EVPN - is an address family used in MP BGP, MP BGP is transport for it
- RFC 7432 - EVPN over MPLS
- RFC 8365 - EVPN over VXLAN
- RFC 7623 - Provider Backbone Bridges
- Dynamic discover of new VTEPs
- Dynamic learning of MAC and IP information
- Decrease amount of BUM traffic
- Providing host mobility
- Providing multi-homing for active active connections - one host to several switches without any VPC

## Inside EVPN packet
- Type: Update message (2)
- Path attributes:
  - Origin: IGP 
  - AS_PATH
  - MP_REACH_NLRI
    - Address Family Identifier (AFI) - Layer-2 VPN - 25
    - Subsequent Address Family Identifier (SAFI) - EVPN - 70
    - Nexthop - lookback IP
    - Network Layer Reachability Information (NLRI)
      - EVPN NLRI
        - Route type: MAC advertisment route (2)
        - Route Distinguisher
        - Ethernet TAG ID - 0
        - MAC address
        - VNI - it can be MPLS label here
- Extended communities
  - Route target
  - Encapsulation - VXLAN - because it can work via MPLS as well
  - Sequence number - the more the better, used when host moves from one VTEP to another, new VTEP increases this number and sends to everyone

## EVPN route types
VTEP gets all BGP updates and stores them in global table, but installs them to VRF only if there is required VNI
- L2 operations
  - Type 2 - Host Advertisment - advertising MACs - Generated when new MAC is discovered, is sento all VTEPs withis this VNI, VTEPs import it to required MAC VRF according to RT
  - Type 3 - Inclusive Multicast Ethernet Tag  - for BUM - Ingress Replication - VTEP with particular VNI and VLAN sends this update to inform everyone that it is ready to accept BUM traffic for this particular VNI. Attribute PMSI_TUNNEL_ATTRIBUTE is added to BGP update, it contains VNI and replication type: Ingress Replication (or Multicast). This route update is generated for every VNI configured.
- L3 operations
  - Type 4 - Ethernet Segment Route - one LAN connected to two VTEPs - only one of them should process BUM
  - Type 1 - Ethernet Auto Discovery Route
- External connections
  - Type 5 - IP prefix route advertisment
- Multicast
  - Type 6,7,8 - PIM, IGMP Leave/Join
