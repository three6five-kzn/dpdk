# KZNNOG 5 - DPDK Software Router

This guide shows the steps to setting up an [FRR Software Router](https://frrouting.org/) with [VPP - Vector Packet Processing](https://github.com/FDio/vpp) and of course [DPDK](https://www.dpdk.org/)!

This guide is based heavily on the work from the FRR Wiki post on [Alternative Forwarding Planes](https://github.com/FRRouting/frr/wiki/Alternate-forwarding-planes:-VPP).

The BGP Full Table howto is based on [this](https://www.stubarea51.net/2016/01/21/put-500000-bgp-routes-in-your-lab-network-download-this-vm-and-become-your-own-upstream-bgp-isp-for-testing/)

If you want to download OVFs for this they can be found [http://dpdk.ahouston.net](http://dpdk.ahouston.net)

# Install VPP

These steps were completed on a vanilla install of Ubuntu 18.04 LTS - Bionic Beaver - the server edition of course.

For the purposes of this guide I installed on an ESXi 5.5 compatible VM, with three NICs - one for management and two for the forwarding interfaces that will be claimed by the DPDK driver.


**1) First add make sure you have the universe repo in your sources.list**

````
echo "deb http://ubuntu.mirror.ac.za/ubuntu/ bionic universe
deb http://ubuntu.mirror.ac.za/ubuntu/ bionic-updates universe" | sudo tee -a "/etc/apt/sources.list"
````

**2) Then add the VPP Repo**
````
curl -s https://packagecloud.io/install/repositories/fdio/release/script.deb.sh | sudo bash
sudo apt update
````

**3) Install the VPP packages**

We're going to be installing the VPP packages now, including the ```vpp-dev``` to allow us to build the VPP Sandbox plugins later, as there doesn't appear to be an easy binary install for these.

````
sudo apt install libmbedtls-dev vpp-lib vpp vpp-plugins vpp-dev
````

## Start VPP and verify

The VPP service can be started with the ````service vpp start```` command - if it starts with no error then you're probably good. You can use the ````vppctl show interface```` command to see the interfaces that have been claimed.

````
kznnog@frr-dpdk:~$ sudo service vpp start
kznnog@frr-dpdk:~$ sudo vppctl show interface
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
GigabitEthernet13/0/0             2     down         9000/0/0/0     
GigabitEthernetb/0/0              1     down         9000/0/0/0     
local0                            0     down          0/0/0/0       
kznnog@frr-dpdk:~$ 
````

## Configure VPP

As you can see, VPP has claimed two interfaces, named GigabitEthernet13/0/0 and GigabitEthernetb/0/0 in my example.

It is probably a good idea to set these manually in the configuration. **Note:** When importing an OVF/OVA template you can almost certainly ensure that Ubuntu and/or VMWare are going to mess around your interfaces. The 13 and 0b values relate to their position on the virtual PCI bus, so it's going to differ from VM to VM.

The first thing to do is find out what position they occupy on your instance:

````
kznnog@frr-dpdk:~$ sudo lshw -class network -businfo
Bus info          Device      Class      Description
====================================================
pci@0000:03:00.0  ens160      network    VMXNET3 Ethernet Controller
pci@0000:0b:00.0              network    VMXNET3 Ethernet Controller
pci@0000:13:00.0              network    VMXNET3 Ethernet Controller
kznnog@frr-dpdk:~$ 
````

Here you can clearly see that the bus addresses of my last two NICs are **0000:0b:00.0** and **0000:13:00.0**. Keep in mind that in hexadecimal **13** is greater than **0b**, but when you sort alphabetically the same isn't true. Basically the order may be different in your config. 

Lets add the following section to ```/etc/vpp/startup.conf``` using your favorite text editor, and obviously substituting in the values you got from the output above.

````
dpdk {
  dev 0000:0b:00.0
  dev 0000:13:00.0
}
````

Now lets restart VPP and make sure that all is still well:

````
kznnog@frr-dpdk:~$ sudo service vpp restart
kznnog@frr-dpdk:~$ sudo vppctl show interface
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
GigabitEthernet13/0/0             2     down         9000/0/0/0     
GigabitEthernetb/0/0              1     down         9000/0/0/0     
local0                            0     down          0/0/0/0       
kznnog@frr-dpdk:~$
````

## Install the VPP Sandbox 

We need a few dependencies for the build, so lets do that first:

````
sudo apt install build-essential libtool m4 automake python-cffi python-pycparser
````

Now we clone the vppsb to our home directory:

```
cd ~
git clone https://gerrit.fd.io/r/vppsb
cd vppsb
```
First lets build the **netlink** plugin, as the **router** plugin depends on it.
````
cd netlink
libtoolize
aclocal
autoconf
automake --add-missing
./configure
make
sudo make install
````

Now lets do the **router** plugin:

````
cd ../router
libtoolize
aclocal
autoconf
automake --add-missing
./configure
make
sudo make install
cp /usr/local/lib/router.so.0.0.0 /usr/lib/vpp_plugins/router.so
````

Reload the libraries, restart VPP and check that the plugin is a candidate for inclusion:

````
ldconfig
service vpp restart
sudo vppctl show plugins | grep router
````

Finally enable the router plugin:

````
vppctl enable tap-inject
````





## Sync VPP and the Kernel network

VPP provides a way for your DPDK interfaces to talk to the kernel interfaces, so one of the first things we want to do is give them some IP addresses.

Lets set these in VPP first:

````
sudo vppctl create loopback interface
sudo vppctl set interface state loop0 up
sudo vppctl set interface state GigabitEthernetb/0/0 up
sudo vppctl set interface state GigabitEthernet13/0/0 up
sudo vppctl set interface ip address loop0 2.2.2.2/32
sudo vppctl set interface ip address GigabitEthernetb/0/0 10.0.10.2/24
sudo vppctl set interface ip address GigabitEthernet13/0/0 10.0.20.2/24
````

We can look at the status of the interfaces with the ```vppctl show interfaces```` command:

````
kznnog@frr-dpdk:/usr/lib/vpp_plugins# sudo vppctl show interface
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
GigabitEthernet13/0/0             2      up          9000/0/0/0     rx packets                    14
                                                                    rx bytes                    1968
                                                                    drops                          6
                                                                    punt                           8
                                                                    ip4                            6
GigabitEthernetb/0/0              1      up          9000/0/0/0     rx packets                     5
                                                                    rx bytes                     864
                                                                    drops                          4
                                                                    punt                           1
                                                                    ip4                            4
local0                            0     down          0/0/0/0       
loop0                             3      up          9000/0/0/0

````

Now it's time to mirror this configuration on the kernel interfaces. **A quick explanation here**: the **tap-inject** part is the router plugin creating the **vpp interfaces** in the kernel which map back to the **DPDK** ones.

In a "normal" DPDK application, the interfaces are not directly visible or addressable in the kernel.

You can see the vpp interfaces with the ````vppctl show tap-inject```` command like so:

````
kznnog@frr-dpdk:~$ sudo vppctl show tap-inject
loop0 -> vpp2
GigabitEthernet13/0/0 -> vpp1
GigabitEthernetb/0/0 -> vpp0
````

Here we can see that we have interfaces vpp0, vpp1 and vpp2 which map to the DPDK interfaces within VPP.

Let's put the same IP configuration on the kernel side now, this might seem counter-intuitive, but it is needed.

````
ip addr add 10.0.10.2/24 dev vpp0
ip addr add 10.0.20.2/24 dev vpp1
ip addr add 2.2.2.2/32 dev vpp2
ip link set dev vpp0 up
ip link set dev vpp1 up
ip link set dev vpp2 up
````

Finally, lets verify that we're seeing everything as it should be:

````
root@frr-dpdk:~# ifconfig
<snip>
vpp0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.10.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::20c:29ff:fea2:63d2  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:a2:63:d2  txqueuelen 1000  (Ethernet)
        RX packets 3  bytes 190 (190.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6  bytes 516 (516.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vpp1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.20.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::20c:29ff:fea2:63dc  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:a2:63:dc  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6  bytes 516 (516.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vpp2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 2.2.2.2  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::dcad:ff:fe00:0  prefixlen 64  scopeid 0x20<link>
        ether de:ad:00:00:00:00  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6  bytes 516 (516.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

````
## Install FRR

In this guide I'm installing [FRR 6.0-1 for Ubuntu 18.04](https://github.com/FRRouting/frr/releases/download/frr-6.0/frr_6.0-1.ubuntu18.04+1_amd64.deb), but as long as you're matching your VPP and FRR binary versions it should still work.

We need a few dependencies installed ahead of this though, so lets do that first:

````
sudo apt install iproute2 libc-ares2
````

Now lets fetch the .deb file and install it:
````
wget https://github.com/FRRouting/frr/releases/download/frr-6.0/frr_6.0-1.ubuntu18.04+1_amd64.deb
sudo dpkg -i frr_6.0-1.ubuntu18.04+1_amd64.deb
````


# Configure FRR

First lets enable the Zebra, OSPF and BGP daemons by editing the /etc/frr/daemons file

**/etc/frr/daemons**:
```
zebra=yes
bgpd=yes
ospfd=yes
```
Now lets add some config to the following files:

**/etc/frr/zebra.conf**:
```
hostname frr-dpdk
password kznnog
log stdout
````

**/etc/frr/ospfd.conf**:
```
hostname ospfd
password kznnog
log stdout
!
router ospf
 network 2.2.2.2/32 area 0
 network 10.0.10.2/24 area 0
 network 10.0.20.2/24 area 0
!
````

**/etc/frr/bgpd.conf**:
```
hostname bgpd
password kznnog
log stdout
!
router bgp 65535
 bgp router-id 2.2.2.2
 redistribute connected route-map CONNECTED-TO-BGP
 neighbor 10.0.10.100 remote-as 65512
 neighbor 10.0.10.100 timers 5 30
 neighbor 10.0.10.100 soft-reconfiguration inbound
!
ip prefix-list CONNECTED-TO-BGP seq 5 permit 10.0.10.0/24
ip prefix-list CONNECTED-TO-BGP seq 10 permit 10.0.20.0/24
ip prefix-list CONNECTED-TO-BGP seq 15 permit 10.0.20.0/24
!
route-map CONNECTED-TO-BGP permit 10
 match ip address prefix-list CONNECTED-TO-BGP
!
````

Now we fix up the ownership and permissions, restart FRR and take a look at the config:

```

sudo rm -rf /etc/frr/frr.conf
echo "no service integrated-vtysh-config" | sudo tee /etc/frr/vtysh.conf
sudo chown -R frr:frr /etc/frr/*.conf
sudo chown -R root:frrvty /etc/frr/vtysh.conf
sudo chmod 640 bgpd.conf ospfd.conf zebra.conf vtysh.conf
sudo service frr restart
````

Now check that the config is being loaded correctly:

````
kznnog@frr-dpdk:/etc/frr$ sudo vtysh

Hello, this is FRRouting (version 6.0).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

frr-dpdk# sho run
Building configuration...

Current configuration:
!
frr version 6.0
frr defaults traditional
hostname frr-dpdk
log stdout
no ip forwarding
no ipv6 forwarding
hostname ospfd
hostname bgpd
no service integrated-vtysh-config
!
password kznnog
!
router bgp 65535
 bgp router-id 2.2.2.2
 neighbor 10.0.10.100 remote-as 65512
 neighbor 10.0.10.100 timers 5 30
 !
 address-family ipv4 unicast
  redistribute connected route-map CONNECTED-TO-BGP
  neighbor 10.0.10.100 soft-reconfiguration inbound
 exit-address-family
!
router ospf
 network 2.2.2.2/32 area 0
 network 10.0.10.0/24 area 0
 network 10.0.20.0/24 area 0
!
ip prefix-list CONNECTED-TO-BGP seq 5 permit 10.0.10.0/24
ip prefix-list CONNECTED-TO-BGP seq 10 permit 10.0.20.0/24
!
route-map CONNECTED-TO-BGP permit 10
 match ip address prefix-list CONNECTED-TO-BGP
!
line vty
!
end
frr-dpdk# 
frr-dpdk# 
frr-dpdk# 
frr-dpdk# 
frr-dpdk# show ip bgp summ

IPv4 Unicast Summary:
BGP router identifier 2.2.2.2, local AS number 65535 vrf-id 0
BGP table version 2
RIB entries 3, using 480 bytes of memory
Peers 1, using 21 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
10.0.10.100     4      65512       0       0        0    0    0    never       Active

Total number of neighbors 1
frr-dpdk# 
````

**At this point the FRR + VPP router is ready for action**

Next you can follow the guide for deploying the test VM with a full routing table in order to lab the performance in a peering edge deployment.
