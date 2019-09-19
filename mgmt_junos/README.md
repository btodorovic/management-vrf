# mgmt_junos - VMM Exapmle Configuration

Example of using **mgmt_junos** routing instance. Requires Junos OS Release **18.4R1** for all services to work.
Older releases can be fixed by using static routes in **inet.0** with the **next-table mgmt_junos.inet.0** knob
to redirect host outbound traffic into the **mgmt_junos** routing instance.

Services fully working within **mgmt_Junos** routing instance are:

| **Service**                    | **Minimum Junos OS Relase** |
|--------------------------------|-----------------------------|
| Automation Scripts             | 18.1R1                      |
| BGP Moniotoring Protocol (BMP) | 18.3R1                      |
| NTP                            | 18.1R1                      |
| RADIUS                         | 18.1R1                      |
| TACACS+                        | 17.4R1                      |
| SYSLOG                         | 18.4R1                      |
| DNS                            | 18.4R1                      |

The testbed for this use case consists only of one VMX and one Linux VM, communicating via management Ethernet only.

