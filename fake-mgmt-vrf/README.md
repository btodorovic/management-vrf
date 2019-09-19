# Fake Management VRF

Example showing management traffic termination in a non-default routing instance, with the router being managed in-band.
The customer has a VRF for management (here we call it MGMT) spreading all over their network.
Management traffic from BSS/OSS/NMS to the routers and back flows via the VRF MGMT.

## Network Topology

<pre>

     +--------+       rr
     |        |        |
    srv      noc ----- p ----- pe
     |        |
     +--------+

</pre>

* **srv** - is a Linux (CentOS) VM, simulating the BSS/OSS/NMS server, connected to the **noc** router.
* **noc** - VMX, connecting **srv** (connections being terminated in the routing-instance **MGMT**).
* All other VMs are also VMX

See [Master README file](../README.md) for more information on this.
