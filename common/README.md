# Linux (CentOS) VM Configuration Files

Various configuration elements applicable to both VMM configurations

* **ifcfg-ens3f1** - interface **ens3f1** configuration for the Linux VM
* **ifcfg-ens3f2** - interface **ens3f2** configuration for the Linux VM
* **shadow** - file to overwrite the **/etc/shadow** file on the Linux VM - defines the **root** password (well-known "E\*" password used on VMM)
* **ssh-key.txt** - file to be installed as **/root/.ssh/authorized_keys** on the Linux VM
* **rc.local** - file to be installed in **/etc/rc.d/rc.local** on the Linux VM - forces the system to:
  - Update itself (**yum -y update**)
  - Install some additional packages not being present in the default CentOS distribution
  - Remove the **/usr/sbin/cloud-init** script, to prevent the Linux VM from doing some crazy stuff
  - Fix permissions on the **/root/.ssh/** directory and all its files (e.g. **authorized_keys**), so public key authentication works correctly.
  - Finally, this initial **rc.local** script auto-destroys itself, by replacing itself with the default **rc.local** script.
