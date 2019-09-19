
# IN-BAND MANGEMENT VIA A NON-DEFAULT ROUTING INSTANCE

## Methods

1. [Fake management VRF - with route leaking](#fake)
2. [Routing instance **mgmt_junos**](#mgmt_junos)

## Applicability - Use Cases

In any of the following cases, use the former ([Fake management VRF](#fake)) method:

* JUNOS version older than 17.3, or:
* Routers managed **IN-BAND**, with management traffic terminated in a non-default routing-instance.

Otherwise, if neither of the above matches your use case - use the latter ([mgmt_junos](#mgmt_junos)) method.

<a name="fake">
## 1. Fake Management VRF - with Route Leaking
</a>

The customer has a VRF for management (here we call it MGMT) spreading all over their network.
Management traffic from BSS/OSS/NMS to the routers and back flows via the VRF MGMT.
The routers are not managed OOB (so, fxp0 interfaces are not relevant).
We want to allow IN-BAND management via VRF MGMT.

### Principle

* We define routing-instance MGMT on the router, classic one:
<pre>
routing-instances {
    MGMT {
        instance-type vrf;
        interface lo0.1;                  ### VRF-specific loopback
        route-distinguisher 100.0.0.1:1;
        vrf-import PL-MGMT-IMPORT;
        vrf-export PL-MGMT-EXPORT;
        vrf-table-label;
    }
}
</pre>

* The lo0.1 address is identical to lo0.0:
<pre>
interfaces {
    lo0 {
        unit 0 {
            family inet {
                address 100.0.0.3/32;
            }
        }
        unit 1 {
            family inet {
                address 100.0.0.3/32;
            }
        }
    }
}
</pre>

* In [edit routing-options] - we need to let the router know that BSS/OSS/NMS is within VRF MGMT, so NTP/SNMP/TACACS+ etc. responses go into VRF MGMT:
<pre>
routing-options {
    static {
        route 100.0.1.0/24 next-table MGMT.inet.0; ### BSS/OSS/NMS range
    }
}
</pre>

* BSS/OSS/NMS - is connected on the "noc" router, to ge-0/0/2 (in this demo - Linux server)
* No special additions required on the "noc" router, this works out-of-the-box.

### Services Tested

* NTP
* DNS
* TACACS+
* SYSLOG
* SNMP (gives the full access to all MIBs!)

<a name="mgmt_junos">
## 2. Routing Instance **mgmt_junos**
</a>

Starting with Junos OS Release 17.3R1, you can confine the management interface in a nondefault virtual routing and forwarding (VRF) instance, the mgmt_junos routing instance.
More information is available on the relevant <a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/management-interface-in-non-default-instance.html">Junos OS Documentation page</a>.

### Principle

Junos OS offers a hardcoded routing instance **mgmt_junos** for management traffic, which is activated by using the
**management-instance** knob at the **[edit system]** configuration level. Once configured, **DO NOT COMMIT IMMEDIATELY**,
but follow strictly the instructions provided in the <a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/management-interface-in-non-default-instance.html">Junos OS Documentation</a>:
Otherwise, you'll be cut off from the remote session and the only way to get back into the router will be from the
same LAN segment where fxp0 interface is connected to, or via the serial console port.

* Activate the knob (**DO NOT COMMIT YET!**):
<pre>
[edit]
user@host# set system management-instance
</pre>

* Make a list of all static routes:

<pre>
user@host&gt; show configuration routing-options static | display set relative
</pre>

* You will get the output in the form:
</pre>
set route 172.16.0.0/12 next-hop 10.102.175.254
set route 172.16.0.0/12 retain
set route 172.16.0.0/12 no-readvertise
set route 10.0.0.0/8 next-hop 10.102.175.254
set route 10.0.0.0/8 retain
set route 10.0.0.0/8 no-readvertise
set route 66.129.255.62/32 next-hop 10.102.175.254
set route 66.129.255.62/32 retain
set route 66.129.255.62/32 no-readvertise
</pre>

* Select only routes having the next-hop on the network where **fxp0** is connected and configure them in the **mgmt_junos** routing instance:

<pre>
edit routing-instances mgmt_junos routing-options static
set route 172.16.0.0/12 next-hop 10.102.175.254
set route 172.16.0.0/12 retain
set route 172.16.0.0/12 no-readvertise
set route 10.0.0.0/8 next-hop 10.102.175.254
set route 10.0.0.0/8 retain
set route 10.0.0.0/8 no-readvertise
set route 66.129.255.62/32 next-hop 10.102.175.254
set route 66.129.255.62/32 retain
set route 66.129.255.62/32 no-readvertise
</pre>

* Remove those same static routes from the master routing instance:

<pre>
edit routing-options static
delete route 172.16.0.0/12
delete route 10.0.0.0/8
delete route 66.129.255.62/32
</pre>

* Now, **COMMIT** the configuration and you're done:

<pre>
commit and-quit
</pre>

You will lose remote access to the router, but that's only tempoprary.
If everything was done fine, you will be able to login into the router again.
