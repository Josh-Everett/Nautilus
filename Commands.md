Interesting commands used and sources of them.

Not all inclusive list of commands.



Installation guide of RHEL on Mustang X-GENE
1) Create 16Gb FAT32 partition on SSD attached to SATA with only empty pace
2) Place dud ISO image of RHELL on MMC
3) Enter anaconda set and complete in reverse order
	ie Root password first, timzone last

### Configuring static networking on CentOS Pi ###

Determine device by observing IP command output
-> My wired connection was eth0
```bash
ip a
```

#### Modify the ifcfg file of the device to a static network and assign the IP ####
```bash
vim /etc/sysconfig/network-scripts/ifcfg-eth0
```
``` 
BOOTPROTO=static
IPADDR=192.168.1.110
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
```
https://github.com/ceph/ceph-ansible

### Configuring static networking on Raspbian ###

#### Networking configuration on Debian based machines found in /etc/dhcpcd.conf ####

```bash
sudo vim /etc/dhcpcd.conf
```
Configure the section on static IPs

```
interface eth0

static ip_address=192.168.1.2/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```

_In both cases the networking service will need to be restarted._

#### Raspbian ####
```bash 
sudo ifdown eth0
sudo ifup eth0
```

#### CentOS ####
```bash
systemctl restart network
systemctl restart network.service
```


## Initial configuration on Pis: ##
4 nodes running CentOS
This would be the most reasonable and realistic OS for a Pi.
The cost of a single RHEL subscription for a Pi is incredibly unreaslistic. 
CentOS is a downstream open source product and free to use.
Configuration of the Ceph nodes would completely be carried out thru ansible.

## End result ##
Old school SSH and manual configuration.
Ceph deployment accomplished via ceph-deploy
Legacy python app that automates deployment
Easy to use.


#### Create a file to give sudo access to ceph: ####

```bash
echo 'ceph ALL = (root) NOPASSWD:ALL' > /etc/sudoers.d/ceph
```
https://blog.raveland.org/post/raspian_ceph_en/


#### Preparing images ####
```bash
xzcat CentOS.raw.xz | sudo dd of=/dev/sde status=progress bs=4M

dd if=/dev/sdd of=configured-image.img bs=4M status=progress

sync
```

### Major issue encountered ###
Ceph no longer supports 32 bit builds.
Pis, despite running 64 bit capabale ARMv8 processors,
run ARMv7, a 32 bit flavor.
Multiple solutions exist but seemingly most stable is the unsupported build process.
12 hours to rebuild packages on pi.
.debs precompiled from
https://louwrentius.com/compiling-ceph-on-the-raspberry-pi-3b-armhf-using-clangllvm.html

```bash
dpkg --install *.deb
apt-get install --fix-missing
apt --fix-broken install
```


### Major issue encountered ###
Trivial in nature, ssh-keys
When ceph is installed it modifies the ceph user to have no shell and 
assigns its home directory to be in /var/lib/ceph.
So even with passwordless sudo privs on Ceph user you won't be able to 
write to that thru ssh-copy-id. Also connection issues that weren't immediately
apparent:

#### Can't connect to Pi via SSH ####
```bash
sudo raspi-config
```
https://raspberrypi.stackexchange.com/questions/41595/why-cant-i-ssh-to-raspbian-anymore


#### Modify the homedir of Ceph so you can appropriately generate SSH keys ####
```bash
usermod -m -d /home/ceph ceph
```
https://stackoverflow.com/questions/20797819/command-to-change-the-default-home-directory-of-a-user


#### Switch user with BASH shell ####
```bash
sudo su -s /bin/bash ceph
```
https://stackoverflow.com/questions/15174194/jenkins-host-key-verification-failed

#### Copying SSH key to other machines for passwordless login ####
_Needed to be done as Ceph user_

```bash
ssh-keygen
ssh-copy-id -i ~/.ssh/mykey user@host
```
https://www.ssh.com/ssh/copy-id


### Another Major issue: ###
_Need to create a host file with the nodes and hostnames matching as following. Without, ceph-deploy will get hung on connections_
```
Host pi1  
   Hostname pi1  
   User ceph  
```


### Major issue which followed: ###
Problem: the dpkg install from the files provided result in dependency issues that
result in a function ceph-deploy tool, but a broken implementation of Ceph in the background

Solution:
Need to reinstall all the OSs with clean dependency trees, do a total reinstal of the OS
across the entire cluster.

```bash
apt-get update
apt-get upgrade
```

### Ceph Administration ###
```bash
mkdir ceph-deploy && cd ceph-deploy
ceph-deploy new --public-network 192.168.1.0/25 ceph01 ceph02 ceph03 # Of course, adapt the names and the network
ceph-deploy mon create-initial # Deploy *mon* on all the machines
ceph admin pi1 pi2 pi3 # Copy conf on all machines
```

Now, we need to add OSD to our cluster. For it we will use our usb keys like this:

```
pi1 : 128 Gb USB 3.0 ( /dev/sda )
pi2 : 128 Gb USB 3.0  ( /dev/sda )
pi3 : 128 Gb USB 3.0  ( /dev/sda )
pi4 : 32 Gb USB 2.0 (/dev/sda)
```

### Performance testing of USB keys ###
```bash
sudo hdparm -tT /dev/sdb
sudo hdparm -W 0 /dev/sdb
lsusb -v -s 2:6 | grep bcdUSB
```




### CRUSH algorithm ###
CRUSH is designed to reconcile two
competing goals: efficiency and scalability of the mapping
algorithm, and minimal data migration to restore a balanced
distribution when the cluster changes due to the addition
or removal of devices.


https://ceph.com/wp-content/uploads/2016/08/weil-crush-sc06.pdf
