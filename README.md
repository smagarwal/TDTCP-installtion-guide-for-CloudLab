# TDTCP Installation Guide for CloudLab

This guide provides instructions for installing TDTCP on CloudLab using the Utah d6515 hardware. 

**By: Tessa Hammond and Shourya Agarwal**

**Special thanks to Dr. Ahmed Saeed, Shawn Chen, and CloudLab**

## Hardware Selection and Profile Creation

We have selected CloudLab Utah d6515 hardware for this project as it provides the closest match to the hardware used in the TDTCP experiment. Using the CloudLab topology editor connect all the four machines with three separate links.   

## Initial Setup

One issue with the CloudLab machines is that they do not automatically provide enough space on the partition for the TDTCP experiment. To check the available space, use the command `df -h`.

When using the d6515 machines with the standard Ubuntu 18.04 image, the disk partition is `/dev/sda1`. To expand the partition to the maximum available space, use the `setup-grow-rootfs.sh` script [1] and execute the following commands:

```
$ sudo export RESIZEROOT=0

$ sudo RESIZEROOT=0 ./setup-grow-root.sh
```

After reloading a node on CloudLab following the above disk partition expansion, the `df -H` command will incorrectly show that the disk partition has returned to the original, smaller size. However, running `sudo fdisk -l` should show that the disk partition is still the maximum size. 

Note: Once the partition is expanded using this method, creating a disk image will not work.

[1] https://gitlab.flux.utah.edu/johnsond/openstack-build-ubuntu/-/blob/master/setup-grow-rootfs.sh

## Setting up TDTCP: 

`TDTCP/etalon_init.sh`: After cloning the TDTCP repository, initialize the `etalon_init.sh` file with the FQDNs of the machines and the interface names. The interface names can be found in the Manifest tab on the CloudLab experiment page. The CloudLab page for each node will show the MAC address of the interfaces and the corresponding bandwidth.  

Note: After running `etalon_init.sh`, it updates the `etalon/etc/handles` file. However, in our experiment, it incorrectly wrote the FQDNs to the file. The file should be checked manually after the `etalon_init.sh` file is run. Also, we recommend manually verifying the `etalon/etc/netplan/99-etalon.yaml` file.  

`TDTCP/etalon/bin/node_install.sh`: Near the end of the file, add the following command: `export HOME=[path to directory where TDTCP was downloaded]`. Then change the commands for the RSA file to include the $HOME variable as shown: 
`$ cat /etalon/vhost/config/ssh/id_rsa.pub >> "$HOME/.ssh/authorized_keys"` 

`TDTCP/etalon/bin/switch_install.sh`: Add the following command to set the home directory: export HOME=[path to directory where TDTCP was downloaded]. Change the commands for the RSA files to include the $HOME variable as shown:  
```
cp -fv /etalon/vhost/config/ssh/id_rsa "$HOME/.ssh/"  

cp -fv /etalon/vhost/config/ssh/id_rsa.pub "$HOME/.ssh/" 

chmod 600 "$HOME/.ssh/id_rsa"

chmod 600 "$HOME/.ssh/id_rsa.pub"
```

`TDTCP/etalon/bin/install.sh`:  The standard Ubuntu 18.04 image on CloudLab did not have the `/etc/hostname` file. As such, we commented out the line: `sudo sed -i "s/$OLD_HOSTNAME/$NEW_HOSTNAME/g" /etc/hostname` . Additionally, there is no file `/etc/apt/apt.conf.d/20auto-upgrades`, as such, comment out the line: `sudo sed -i "s/1/0/g" /etc/apt/apt.conf.d/20auto-upgrades`. 

Running the `install.sh` file typically results in the ssh connection being severed. As such, in the `install.sh` file before commands that trigger a reboot, add the following command: `sudo dhclient [ssh interface name] -v`.  

`TDTCP/etalon/bin/node_install.sh`: Near the end of the file, add the following command: `export HOME=[path to directory where TDTCP was downloaded]`. 

Running `install.sh`: When prompted `usr.sbin.tcpdump (Y/I/N/O/D/Z) [default=N] ?`, select` N`. 

If the machine does not restart after rebooting, the ssh connection can be manually restored via the node’s Console on the CloudLab experiment page. You can login as the root using the provided password. Then, run the command `dhclient [ssh interface name] -v`. If the machine still does not restart, power cycling can be used. 

Note: After rebooting, the interface names may change. If so, run the following commands: 

``` 
$ sudo ip link set [old interface name] down 

$ sudo ip link set [old interface name] name [new interface name] 

$ sudo ip link set [new interface name] up 

``` 

The associated DNS files are reset to default versions after rebooting. As such, rerun the `install.sh` file after writing an exit command at line 96. Validate the hostname command shows the new name, the `/etc/hosts` file shows the correct IP addresses and hostnames, and the `/etc/netplan/99-etalon.yaml` shows the correct interface names and IP addresses. The `inconfig -a` command should show the new IP addresses assigned to the interface.  

`TDTCP/etalon/bin/gen_switch.py`:  Lines 128 through 130 have the ports hardcoded as 0 and 1. Change them to the PCIe addresses of the interfaces. An example is shown below: 

``` 
print 'in :: FromDPDKDevice(0000:41:0.0, MTU 9000)' 

print 'out :: ToDPDKDevice(0000:41:0.0)' 

print 'mgtout :: ToDPDKDevice(0000:41:0.1)' 
``` 

 
## Current Issues: 

Some of the experiments ran successfully in our environment, however, some experiments failed. After a failure, some or all of the host machines typically lost the SSH connection and failed to reboot. Additionally, booting a host machine into a specific kernel often failed. The hosts would fail to restart and forcing them to restart resulted in booting into the generic kernel. We are unsure of the root cause of these issues, although resetting the SSH connection with the dhclient command before rebooting into a specific kernel sometimes helped. Additionally, for the experiments that we were able to run successfully, some of the graphs did not show the expected output. There was likely an issue in the environment. 
 
