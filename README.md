
# MANGEMENT TRAFFIC IN A NON-DEFAULT ROUTING INSTANCE

## Options

1. [Non-default management VRF - with route leaking](#fake) - see sample [VMM configuration](fake-mgmt-vrf)
2. [Non-default management VRF - with instance-aware services](#mgmt)
3. [Routing instance **mgmt_junos**](#mgmt_junos) - see sample [VMM configurations](mgmt_junos)
4. [Routing instance **mgmt_junos** - with route leaking into inet.0](#mgmt_junos_inet0) - VMM configurations same as previous.

## Applicability Matrix

Depending on your Junos OS Release and router management method (in-band/out-of-band), you should go for the following options:

| **Junos OS Release**  | **IN-BAND**   |   **OUT-OF-BAND**      |
|:----------------------|:-------------:|:----------------------:|
|  Older than 17.3R1    |  [1](#fake)   |  N/A (inet.0 only)     |
|  17.3R1 up to 17.4R3  |  [1](#fake)   | [4](#mgmt_junos_inet0) |
|  18.1R1 or newer      |  [2](#mgmt)   | [3](#mgmt_junos)       |

<img src="ib-oob.png" width="75%" height="75%">

<a name="fake">
## 1. Non-default Management VRF - with Route Leaking
</a>

Suppose the customer has a VRF for management (here we call it MGMT) spreading all over their network.
Management traffic from BSS/OSS/NMS to the routers and back flows via the VRF MGMT.
The routers are not managed OOB (so, fxp0 interfaces are not relevant).
We want to allow IN-BAND management via VRF MGMT.

### Principle

* We define routing-instance MGMT on the router - standard VRF configuration:
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

* Set the lo0.1 address to be identical to lo0.0 - e.g.:
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

* In [edit routing-options] - we need to let the router know that BSS/OSS/NMS is within VRF MGMT, so NTP/SNMP/TACACS+ etc. responses go into VRF MGMT.
  For that part we need to use the [**next-table**](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/static-edit-routing-options.html)
  knob at the **[edit routing-options static route <\*>]** configuration level:
<pre>
    routing-options {
        static {
            route 100.0.1.0/24 next-table MGMT.inet.0; ### BSS/OSS/NMS range
        }
    }
</pre>

* BSS/OSS/NMS - is connected on the "noc" router, to ge-0/0/2 (in this demo - Linux server)
* No special additions required on the "noc" router, this works out-of-the-box.

**Effect:** Management traffic comes via the **MGMT** routing instance, where it hits the **lo0.1** loopback.
This traffic is then passed towards the appropriate target daemon on the router (e.g. snmpd, ntpd etc.).
The daemon receives the incoming messages and sends the responses back. However, since daemons operate at the
underlying FreeBSD level, they are not VRF aware, so they send the traffic back using the default **inet**
routing table. However, in the **inet** routing table, in the next-hop options, [next-table](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/static-edit-routing-options.html)
knob is present pointing to the **MGMT.inet.0** routing table, so the management traffic from the router towards the BSS/OSS/NMS servers will be redirected to
the **MGMT** routing instance.

### Services Tested

* NTP
* DNS
* TACACS+
* SYSLOG
* SNMP (gives the full access to all MIBs!)

<a name="mgmt">
## 2. Non-default Management VRF - with Instance-Aware Services
</a>
This method is just an add-on to the [non-default management VRF](#fake) use case, discussed in the previous section.
Starting from Junos OS Relase 17.4R1 a **routing-instance** knob was introduced for TACACS+, allowing TACACS+
servers to reside outside **inet.0**, in a non-default routing instance (e.g. **MGMT.inet.0**). In Junos OS
Release 18, this knob was added to other services as well. See table below showing the minimum
Junos OS Release required for each particular service to be configured as VRF-aware:

<a name="table">
| **Service**                    | **Minimum Junos OS Relase** |
|--------------------------------|-----------------------------|
| Automation Scripts             | 18.1R1                      |
| BGP Moniotoring Protocol (BMP) | 18.3R1                      |
| NTP                            | 18.1R1                      |
| RADIUS                         | 18.1R1                      |
| TACACS+                        | 17.4R1                      |
| SYSLOG                         | 18.4R1                      |
| DNS                            | 18.4R1                      |
</a>

Suppose the customer has a VRF for management (here we call it **MGMT**) spreading all over their network.
If DNS servers are located within that VRF, as of Junos OS Relase 18.4R1 we can use **routing-instance MGMT**
within the DNS configuration to instruct Junos to look for DNS in that routing instance - e.g.:

<pre>
    system {
        name-server {
            10.102.149.51 routing-instance MGMT;
            10.102.149.52 routing-instance MGMT;
        }
    }
</pre>

<a name="mgmt_junos">
## 3. Routing Instance **mgmt_junos**
</a>

Starting with Junos OS Release 17.3R1, you can confine the management interface in a nondefault virtual routing and forwarding instance, the mgmt_junos routing instance.
More information is available on the relevant <a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/management-interface-in-non-default-instance.html">Junos OS Documentation page</a>.
Although this feature was introduced in Junos 17.3R1, none of the services was routing-instance-aware in that release, so in that case
an additional step is required for services not operating within the **mgmt_junos** routing instance.
See [table above](#table) for the full overview of the minimum Junos OS Release where certain services became routing-instance-aware.

Similar to the [non-default management VRF](#fake) use case, we need to use the [**next-table**](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/static-edit-routing-options.html) knob
at the **[edit routing-options static route <\*>]** configuration level, to leak various BSS/OSS/NMS routes from **mgmt_junos.inet.0** into **inet.0**.
This option should be applied after all steps described in this chapter are applied. More information about this can be found [here](#mgmt_junos_inet0).

The name **mgmt_junos** is **RESERVED** in the Junos OS for the management routing instance purpose only.

Although often called "management VRF", the routing instance **mgmt_junos** is actually not a VRF. Junos OS hardcoded its **instance-type** to be **forwarding** and this cannot be changed.
Any attempt to change the **instance-type** parameter to anything else than **forwarding** will report an error and the configuration will not commit.

### Principle

Junos OS offers a hardcoded routing instance **mgmt_junos** for management traffic, which is activated by using the
**management-instance** knob at the **[edit system]** configuration level. Once configured, **DO NOT COMMIT IMMEDIATELY**,
but follow strictly the instructions provided in the <a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/management-interface-in-non-default-instance.html">Junos OS Documentation</a>:
Otherwise, you'll be cut off from the remote session and the only way to get back into the router will be from the
same LAN segment where fxp0 interface is connected to, or via the serial console port.

* Make a list of all static routes:

<pre>
    user@host&gt; show configuration routing-options static | display set relative
</pre>

* You will get the output in the form:

<pre>
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

* Select only routes having the next-hop on the network where **fxp0** is connected and configure them in the **mgmt_junos** routing instance (**DO NOT COMMIT YET!**): 

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

* Remove those same static routes from the master routing instance (**DO NOT COMMIT YET!**):

<pre>
    edit routing-options static
    delete route 172.16.0.0/12
    delete route 10.0.0.0/8
    delete route 66.129.255.62/32
</pre>

* Activate the knob (**DO NOT COMMIT YET!**):

<pre>
    [edit]
    user@host# set system management-instance
</pre>

* Now, **COMMIT** the configuration (use **confirmed** just in case) and you're done:

<pre>
    commit confirmed 1 and-quit
</pre>

You will lose remote access to the router, but that's only tempoprary.
If everything was done fine, you will be able to login into the router again. Then **COMMIT DEFINITELY**:

<pre>
    commit and-quit
</pre>

Otherwise, the **confirmed** option will rollback automatically to the previous configuration in 1 minute.

### Services Tested

Each of the services in the table below has a **routing-instance** knob, signaling the service to perform
route lookup in a specific routing instance. See table [above](#mgmt) specifying the minimum Junos OS Release
required for each service to be configured as VRF-aware.

In this case, use **routing-instance mgmt_junos** to have service daemons perform the route lookup in **mgmt_junos**.
For instance, to instruct DNS to use routing instance **mgmt_junos** to reach the DNS server, use:

<pre>
    system {
        name-server {
            10.102.149.51 routing-instance mgmt_junos;
            10.102.149.52 routing-instance mgmt_junos;
        }
    }
</pre>

For services not being capable of working within a non-default routing-instance, use
[route leaking into inet.0](#mgmt_junos_inet0) option, described in the next chapter.

<a name="mgmt_junos_inet0">
## 4. Routing Instance **mgmt_junos** - with Route Leaking into inet.0
</a>

This option is implemented in two stages:

* First, implement the [routing instance **mgmt_junos**](#mgmt_junos), according to the instructions provided in the [previous chapter](#mgmt_junos).
* Secondly, for any service not being capable of working within the **mgmt_junos** routing instance, add a static route in **inet.0** with the
  [**next-table**](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/static-edit-routing-options.html) knob pointing to
  the **mgmt_junos** routing instance - e.g.:

<pre>
    routing-options {
        static {
            route 100.0.1.3/32 next-table mgmt_junos.inet.0; ### VRF-unaware service (e.g. DNS)
        }
    }
</pre>

**Effect:** Management traffic will come via the **mgmt_junos** routing instance, while the host outbound (return)
traffic will try to find its next hop in the default routing instance (**inet.0**); however, the static route
to the traffic destination, with the **next-table** knob pointing back to the routing table **mgmt_junos.inet.0**
will instruct the router to perform the next hop lookup in the routing instance **mgmt_junos** ... Actually, the
same configuration applicable for the [non-default management VRF](#fake) use case.

