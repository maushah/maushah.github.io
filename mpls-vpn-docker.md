# Introduction

This page goes over how to setup Docker networking to terminate an MPLS VPN on a docker container using a Linux based switch and hosts running Docker.

[MPLS VPN definition](https://en.wikipedia.org/wiki/MPLS_VPN)

## Basic Architecture

Technology in use:

* Linux Switches to terminate MPLS circuits with fiber or 1G/10G ethernet physical connectors
* Linux blade servers to run as Docker hosts
  * Docker containers to do per MPLS VRF / VLAN (customer) segregation
  * IPTABLEs to NAT the customer traffic to a BJN IP address per Docker container
  * Quagga to run BGP per Docker container

## Configuration

### Configuring Linux Switch

On the Linux Switch setup for MPLS, do the below:
* Connect the provider port from patch panel to a port on switch, say swp48 in this case
* Connect a 10G port from the switch to the Docker host, say swp1 on this case - it will go to the first 10G interface on Docker host
* Login to the switch and configure /etc/network/interfaces as below.
* Basic idea is to create the 2 10G interfaces (swp1 and swp48 in this example)

``` sudo vi /etc/network/interfaces ```

``` source /etc/network/interfaces.d/swp_defaults
auto swp1 
iface swp1 
alias "dc1-dc11-docker-01"

auto swp48
iface swp48
alias "LT18614D/10GIG/ASBNVAEG/ASBNVAEG"
```
* Then bring up the interfaces

```
sudo ifup swp1
sudo ifup swp48
```

* Verify they are all up using
```
sudo ip link show 
sudo ethtool -m swp1 
sudo brctl show
```

### Configuring Docker Host

* Connect the Docker Host such that:
  * Eth0 goes to a switch in a Mgmt VLAN
  * Eth1 (1st PCI 10G nic) goes to the Linux switch as configured in previous section
  * Eth3 (2nd PCI 10G nic) goes to the existing internal Linux switch fabric as a 10G port in the Public VLAN
NOTE: To find ethernet interface naming use
```
~# lshw -c network |grep 'product\|logical'
       product: I350 Gigabit Network Connection
       logical name: eth0
       product: I350 Gigabit Network Connection
       logical name: eth1
       product: 82599EB 10-Gigabit SFI/SFP+ Network Connection
       logical name: eth2
       product: 82599EB 10-Gigabit SFI/SFP+ Network Connection
       logical name: eth3
```
* Update Kernel and base packages
```
sudo apt-get update
sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring ethtool vlan tcpdump traceroute bridge-utils
sudo reboot
```
* DOCKER base install
```
sudo wget -qO- https://get.docker.com/ | sh
sudo sh -c "echo deb https://get.docker.io/ubuntu docker main\ >> /etc/apt/sources.list.d/docker.list" 
wget -qO- https://get.docker.com/gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install lxc-docker
sudo docker pull chef/ubuntu-20.04 
sudo docker images
```

* PIPEWORK - install Docker networking tool
```
cd /usr/share
sudo git clone https://github.com/jpetazzo/pipework
sudo ln -s /usr/share/pipework/pipework /usr/local/bin/pipework
* Bridge LXC0 pointing towards interface towards public IP servers

sudo brctl addbr lxc0
sudo brctl addif lxc0 eth1
```

* Edit DOCKER defaults
```
sudo vi /etc/default/docker
#DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4"
DOCKER_OPTS="-b=lxc0 --dns 8.8.8.8 --dns 8.8.4.4 --iptables=false"
sudo service docker restart
```
* Add 802.1q to interface towards MPLS provider interface
```
sudo modprobe 8021q
```
* Make all interfaces on host promiscous
```
  sudo ifconfig eth0 promisc
  sudo ifconfig eth1 promisc
  sudo ifconfig lxc0 promisc
  sudo ifconfig eth2 promisc
  sudo ifconfig eth3 promisc
```
* Configure network interfaces on Docker host as below:

```
~$ sudo vi /etc/network/interfaces 
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The mgmt interface, only routes to 10.x.x.x for management
auto eth0
iface eth0 inet static
  address x.x.x.x
  netmask y.y.y.y
  up route add -net 10.0.0.0/8 gw x.x.x.1

# interface to connect to public IP servers
auto eth1
iface eth1 inet manual

# 10G interface to Docker SW which connects to provider MPLS circuit, random bogon IP
auto eth2
iface eth2 inet static
   address 169.z.z.z
   netmask 255.255.255.255

# Docker interface bridged to public IP server interface, has a public IP subnet that you own
auto lxc0
iface lxc0 inet static
        address A.A.A.A
        netmask 255.255.255.0
        gateway A.A.A.1
        dns-nameservers 10.100.0.10 10.100.0.9        
        bridge_ports eth1
```
* Restart networking as below:
```
sudo /etc/init.d/networking restart
```

### Configuring Docker host per customer config for each new VLAN:

* Login to docker host
* Add VLAN to Linux switch on both interfaces. In this example the VLAN being added is 100 on port swp1 and swp48 for ISP1 as below
```
vi /etc/network/interfaces.d/ISP1-VLANS

auto swp1.100
iface swp1.100

auto swp48.100
iface swp48.100

iface level3-br100 
alias bridge for ISP1 VLAN 100
bridge-ports swp1.100 swp48.100 
bridge-stp on
* Then bring up the interfaces

sudo ifup swp48.100 swp1.100 isp1-br100
```

* Add VLAN to interface going from Docker Host to MPLS Linux switch (in this case VLAN 100 on eth2), make interface promiscuous and edit /etc/network/interfaces
```
sudo vconfig add eth2 100  
sudo ifconfig eth2.100 promisc

sudo vi /etc/network/interfaces
auto eth2.100
iface eth2.100 inet manual
vlan-raw-device eth2
```

### Configuring Docker Container per customer based on support provisioning

* Start container with the right name <ISP><VLAN TAG> using Ubuntu 20.04
```
 sudo docker run -i --name="L3TestVLAN100" --privileged --net=none -t ubuntu:20.04 /bin/bash
```

* Add the interfaces as below - 1st configure the ISP1 interface and 2nd configure the interface towards public IP servers.
NOTE: Order is important as if you reverse it the default gateway for the container will be incorrect

* Here eth3.100 is the Vlan subinterface from docker host, eth3 is the new interface name in the container, L3TestVLAN100 is the container name, 100.65.0.10/30 is the IP given to the internal side in the customers MPLS VPN and 100.65.0.9 is the IP on the ISP side for the MPLS VPN
```
sudo pipework eth3.100 -i eth3  L3TestVLAN100 100.65.0.10/30@100.65.0.9
```

* Here lxc0 is the interface on Docker host that connects to public IP servers vlan, eth1 is the interface name in the container, L3TestVLAN100 is the container name, 31.xxx.xxx.31 is an IP in the EXT VLAN of your subnet, 31.xxx.xxx.1 is the default gw in that subnet.
(Replace xxx with real IP subnet info)

```
sudo pipework lxc0 -i eth1   L3TestVLAN100  31.171.210.31/24@31.xxx.xxx.1
```
* Re attach to the container and install packages and add NAT entry
```
sudo docker attach  L3TestVLAN100 
apt-get update
apt-get -y install openssh-server vim iptables conntrack  inetutils-ping traceroute quagga nano vim less tcpdump libconfig-tiny-perl make nagios-nrpe-server libnet-telnet-perl telnet
iptables -t nat -A POSTROUTING -o eth1 -j SNAT --to-source 31.xxx.xxx.31
```
The iptables rule basically NATs everything coming from eth3 (interface pointing to the ISP on container) to your public IP which is the IP on eth1 in this case (31.xxx.xxx.31)

* Configure quagga in the container
```
chown quagga:quagga /etc/quagga/bgpd.conf && chmod 640 /etc/quagga/bgpd.conf 
chown quagga:quaggavty /etc/quagga/vtysh.conf && chmod 660 /etc/quagga/vtysh.conf 
chown quagga:quagga /etc/quagga/zebra.conf && chmod 640 /etc/quagga/zebra.conf   
echo "password denim123" >> /etc/quagga/zebra.conf
echo "password denim123" >> /etc/quagga/bgpd.conf

vi /etc/quagga/daemons 
zebra=yes
bgpd=yes

vi /etc/quagga/debian.conf
zebra_options=" --daemon -A 127.0.0.1 -P 2601 -u quagga -g quagga --keep_kernel --retain"
bgpd_options=" --daemon -A 127.0.0.1 -P 2605 -u quagga -g quagga --retain -p 179"

service quagga restart
```
* BGP config on Quagga - example config for IPS1 for DC1 NNI (DC1 POP) where this POP is primary for DC1 and GLOBAL subnet, secondary for DC2 subnet and tertiary for DC3 subnet
```
root@3bd03afce191:/# vi /etc/quagga/bgpd.conf
!
password bgppassword
!
enable password bgppassword
!
hostname L3TestVLAN100
! BGP ASN FOR DC1 - replace with the right value
router bgp 65535
! 100.65.0.10 is the IP GIVEN BY ISP
 bgp router-id 100.65.0.10
 bgp log-neighbor-changes
! Define 4 subnets DC3 31.xxx.xxx.0/24, DC1 199.xxx.1.0/24, DC2 199.xxx.2.0/24, GLOBAL 199.xxx.100.0/24
 network 31.xxx.xxx.0/24
 network 199.xxx.2.0/24
 network 199.xxx.1.0/24
 network 199.xxx.100.0/24
! BGP NEIGHBOR IP IS ISP IP, REMOTE-AS IS BGP ASN FOR ISP
 neighbor 100.65.0.9 remote-as 3549
 neighbor 100.65.0.9 interface eth2
 neighbor 100.65.0.9 update-source eth2
! ADD ROUTE MAP FOR SETTING COMMUNITY STRINGS
 neighbor 100.65.0.9 route-map POP out
 neighbor 100.65.0.9 description L3TestVLAN100
!
access-list 101 permit ip 199.xxx.1.0 0.0.0.255 any
access-list 102 permit ip 199.xxx.2.0 0.0.0.255 any
!
! DEFINE PREFIX LIST FOR REMOTE POP - DC2 IN THIS CASE
ip prefix-list 1 seq 5 permit 199.xxx.2.0/24
! DEFINE PREFIX LIST FOR REMOTE POP - DC3 IN THIS CASE
ip prefix-list 2 seq 5 permit 31.xxx.xxx.0/24
! DEFINE PREFIX LIST FOR GLOBAL SUBNET AND LOCAL POP - DC1 IN THIS CASE
ip prefix-list 3 seq 5 permit 199.xxx.1.0/24
ip prefix-list 3 seq 10 permit 199.xxx.100.0/24
!SET COMMUNITY STRINGS AS BELOW FOR REMOTE / LOCAL POPS
route-map POP permit 10
 match ip address prefix-list 1
 set community 6745:700
!
route-map POP permit 20
 match ip address prefix-list 2
 set community 6745:600
!
route-map POP permit 30
 match ip address prefix-list 3
 set community 6745:750
!
The community should be 6745:xxx where 750 should be for global subnet and local POP subnet, 700 for closest POP and 600 for furthest POP. For example for DC1, DC3 subnet should be 600 and DC2 subnet should be 700 while for DC2 both DC1 and DC3 should be 700.
* If provider is AT&T and POP is SJ:

router bgp 65535
 bgp router-id 10.56.1.6
 bgp log-neighbor-changes
 network 31.xxx.xxx.0/24
 network 199.xxx.1.0/24
 network 199.xxx.2.0/24
 network 199.xxx.100.0/24
 neighbor 10.56.1.5 remote-as 8030
 neighbor 10.56.1.5 interface eth2
 neighbor 10.56.1.5 update-source eth2
 neighbor 10.56.1.5 timers 15 45
 neighbor 10.56.1.5 route-map POP out
!
ip prefix-list 1 seq 5 permit 199.xxx.100.0/24
ip prefix-list 2 seq 5 permit 31.xxx.xxx.0/24
ip prefix-list 3 seq 5 permit 199.xxx.1.0/24
ip prefix-list 3 seq 10 permit 199.xxx.2.0/24
!
route-map POP permit 10
 match ip address prefix-list 1
 set as-path prepend 65535 65535 65535
!
route-map POP permit 20
 match ip address prefix-list 2
 set as-path prepend 65535 65535 65535 65535 65535
!
route-map POP permit 30
 match ip address prefix-list 3
 as-path prepend 65535
!
line vty 
```
* Restart quagga daemon
```
service quagga restart
```
* Verify BGP is up on the quagga daemon in container
```
telnet localhost bgpd

3bd03afce191(config-router)# do sh ip bgp 
BGP table version is 0, local router ID is 100.65.0.10
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/30      100.65.0.9                             0 3549 i
*> 10.251.17.64/26  100.65.0.9                             0 3549 i
*  31.xxx.xxx.0/24  100.65.0.9                             0 3549 393490 i
*>                  0.0.0.0                  0         32768 i
*> 100.65.0.0/30    100.65.0.9                             0 3549 i
*> 100.65.0.4/30    100.65.0.9                             0 3549 i
*> 100.65.0.8/30    100.65.0.9                             0 3549 i
*> 199.xxx.1.0     0.0.0.0                  0         32768 i
*> 199.xxx.2.0     0.0.0.0                  0         32768 i
*> 199.xxx.100.0     0.0.0.0                  0         32768 i
```
