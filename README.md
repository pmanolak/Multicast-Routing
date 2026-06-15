# Multicast 
This project is a basic GNS3 topology to demonstrate multicast communication using PIM Sparse Mode.

## What is Multicast?
Multicast is a way of sending a single stream of data to multiple devices at once without sending multiple separate copies. It’s often used in streaming, video conferencing, and real-time data feeds. 
Unlike broadcast, multicast traffic only goes to devices that are part of the multicast group.



The following topology will be used:

![Topology](/Topology/Multicast.PNG)


Multicast uses the following protocols:
-  IGMP (layer 2 multicast)
-  PIM (layer 3 multicast)

## Activating Multicast 
By default, Multiccast routing is diabled. Therefore for multicast to function, it is activated by the following command:

```bash
R5#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R5(config)#ip multicast-routing
R5(config)#
R5(config)#end
```

## Internet Group Management Protocol (IGMP)
IGMP is the protocol used by mutlicast capable receivers to tell the local router that they want to join a multicast group.

For example, if a host wants to receive data sent to 239.1.1.10, it sends an IGMP join request to the router or switch it is connected to.

IGMP runs on the LAN facing interface on the same subnet as the receivers.

IGMP is usually configured on the Last Hop Router, that is the router on the router interface directly connected to the multcast receiver.

In the topology above, R5 is the LHR.

Therefore enabling IGMP on 172.19.1.0/24 subnet is crucial.

```bash
R5(config)#int e0/1
R5(config-if)#ip igmp version 2
R5(config-if)#ip igmp join-group 239.1.1.10
R5(config-if)#end
R5#
```

**NOTE:** IGMPv2 usually sends a *leave-group message* so that when the last host on the multicast group leaves, it sends a leave message to 224.0.0.2 so that the routers stop fowarding the multicast stream/traffic



To verify IGMP:

``` bash

R5#sh ip igmp interface e0/1
Ethernet0/0 is up, line protocol is up
  ! Output omitted for brevity
  Multicast designated router (DR) is 172.19.1.1 (this system)
  IGMP querying router is 172.19.1.1 (this system)
  Multicast groups joined by this system (number of users):
      239.1.1.10(1)  224.0.1.40(1)

```

``` bash
R5#sh ip igmp groups
IGMP Connected Group Membership
Group Address    Interface                Uptime    Expires   Last Reporter   Group Accounted
239.1.1.10       Ethernet0/1              00:08:51  00:02:11  172.19.1.1

```



## PIM Sparse Mode (Protocol Independent Multicast)

PIM is the routing protocol used to build multicast distribution trees between sources and receivers.


In sparse mode, multicast traffic is not forwarded by default. It’s only sent when receivers explicitly request to join a group through IGMP ("pull method").

All routers whose interfaces are participating in Multicast must be configured in PIM-Sparse mode.

The following is a sample configuration of this on R5:

```bash
interface Ethernet0/0
 description LINK TO R3
 ip address 10.0.0.18 255.255.255.252
 duplex auto
!
interface Ethernet0/1
 description LINK TO MULTICAST RECEIVER
 ip address 172.19.1.1 255.255.255.0
 ip igmp join-group 239.1.1.10
 duplex auto
!
R5#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R5(config)#ip multicast-routing
R5(config)#int range e0/0-1
R5(config-if-range)#ip pim sparse-mode

```
### Rendezvous point:

A central point called the Rendezvous Point (RP) is manually configured in this lab. It acts as a meeting place for sources and receivers.

Once routers learn about active receivers, they build a multicast tree from the source to the receivers via the RP.

All routers (including the RP router) must be configured with the RP address.

```bash

R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#ip pim rp-address 192.168.0.1

```

### Shared Path tree:

PIM-SM is used by routers to build multicast distribution trees.

When a source starts sending multicast traffic, it is forwarded to the Rendezvous Point (RP) first — this forms the shared tree (RPT).

Receivers also join via the RP, and traffic flows from source → RP → receiver.



### Switching to a Better Path (Shortest-Path Tree)

After initial traffic is received via the RP, routers closer to the receivers can switch to a better path directly to the source (using the configured dynamic routing).

This is called moving from the Shared Tree (via RP) to the Shortest-Path Tree (SPT).

When this happens, the router sends a prune message upstream toward the RP, telling it to stop forwarding traffic for that group down the shared tree.

This reduces unnecessary hops and improves efficiency



To verify the PIM RP configured:

``` bash
R5#sh ip pim rp
Group: 239.1.1.10, RP: 192.168.0.1, uptime 00:15:01, expires never
Group: 224.0.1.40, RP: 192.168.0.1, uptime 00:15:01, expires never

```


### Verification:

On R1 we can ping the destination group address 239.1.1.10

```bash
R1#ping 239.1.1.10 source 172.19.0.1 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 239.1.1.10, timeout is 2 seconds:
Packet sent with a source address of 172.19.0.1

Reply to request 0 from 172.19.1.1, 3 ms
Reply to request 0 from 172.19.1.1, 4 ms
Reply to request 1 from 172.19.1.1, 2 ms
Reply to request 2 from 172.19.1.1, 3 ms
Reply to request 3 from 172.19.1.1, 2 ms
Reply to request 4 from 172.19.1.1, 1 ms
Reply to request 5 from 172.19.1.1, 1 ms
Reply to request 6 from 172.19.1.1, 1 ms

```

### Verifying the Multicast routing table:

To verify Multicast routing table (S,G) the shortest path tree:

``` bash
R5#sh ip mroute
!output ommitted for brevity!

(*, 239.1.1.10), 00:18:18/stopped, RP 192.168.0.1, flags: SJCL
  Incoming interface: Ethernet0/0, RPF nbr 10.0.0.17
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:18:16/00:02:42

(172.19.0.1, 239.1.1.10), 00:00:05/00:02:54, flags: LJT
  Incoming interface: Ethernet0/0, RPF nbr 10.0.0.17
  Outgoing interface list:
    Ethernet0/1, Forward/Sparse, 00:00:05/00:02:54

(*, 224.0.1.40), 00:18:18/00:02:23, RP 192.168.0.1, flags: SJPCL
  Incoming interface: Ethernet0/0, RPF nbr 10.0.0.17
  Outgoing interface list: Null


```
### Verifying bandwidth Utilization:
```bash
R2#sh ip mfib active
Active Multicast Sources - sending >= 4 kbps
Default
```

