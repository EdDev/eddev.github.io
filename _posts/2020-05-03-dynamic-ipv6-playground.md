---                            
title: "Dynamic IPv6 Playground"
date: 2020-05-03
categories:
  - blog
tags:
  - ipv6
---

In this post we will take a tour into the dynamic IPv6 setup, examining the
various options and scenarios it presents.

This post is a direct continuation to the
[Static IPv6 Playground](../static-ipv6-playground) post.
The tooling and the basic setup will remain the same.

## Basic setup
Please revisit the setup description in the
[static IPv6 basic setup](../static-ipv6-playground/#basic-setup) section.

## The Server
An IPv6 dynamic setup consists of a Router Advertisement (RA) service and an
optional DHCPv6 service which serves an entire network.

### Router Advertisement
In an IPv6 network, routers send RA messages that include the needed
information for setting up a host interface
[IPv6 configuration](https://tools.ietf.org/html/rfc4862).
The data included in the RA message includes the network address
(with its prefix length), routing information and potentially other extended
network information (e.g. [DNS](https://tools.ietf.org/html/rfc8106) entries).

### DHCPv6
Although RA is sufficient for setting up a client host interface with IPv6
configuration, DHCPv6 complements it with additional information in
scenarios where additional options are in need.
The latest [RFC](https://tools.ietf.org/html/rfc8415) even proposes a full
alternative to the stateless address autoconfiguration (SLAAC), but is probably
less common in the industry.

Stateful DHCPv6 complements RA by allowing a specific IPv6 address to be set
on an interface (i.e. stateful dhcp) and the distribution of DNS information.
It is also designed like its DHCPv4 origin with extendable options in mind,
making it more accommodated for quick custom extensions.
See [RA and DHCPv6 combinations](#ra-and-dhcpv6-combinations) for more
information.

As the DNS information is already available in RA messages, DHCPv6 is mainly
left suited for administrators that need to control the exact IPv6 full address
(unlike SLAAC where only the network is distributed by RA and the host address
is auto-generated).

### dnsmasq
We will use [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) to setup
both an RA and DHCPv6 server to provide dynamic IPv6 services to the client.

The server configuration is set on the *blue* namespace.

- dnsmasq-config-with-ra-only
```config
leasefile-ro
interface=veth10
enable-ra
dhcp-range=::,constructor:veth10,ra-only
```

- dnsmasq-config-with-slaac-and-dhcpv6
```config
leasefile-ro
interface=veth10
enable-ra
dhcp-range=::2220,::2222,constructor:veth10,slaac
```
> **Note**: The DHCP server may also be configured to provide "static"
addresses per specific identifiers (e.g. the interface mac address).
This is useful for controlling the exact IPv6 address a requesting interface.
With dnsmasq this translates to the following lines:
```
dhcp-host=12:34:56:78:90:ab,[::22]
dhcp-range=::10,::ffff,static
```

### Run Server

> **Warning**: These operations may cause the existing DNS entries to get
overwritten (i.e `resolv.conf`) by `dhclient`.
Therefore, it is recommended to run this setup in a container or VM.

dnsmasq requires the interface it runs on to have an IPv6 address in the same
network subnet of the address offer.
Therefore, a global IPv6 address needs to be defined:
```
sudo ip netns exec blue ip addr add fd::1/64 dev veth00
```
Lets examine the result:
```
$ sudo ip netns exec blue ip addr show veth10
180: veth10@if181: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 8e:6a:6b:83:93:ea brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fd::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::8c6a:6bff:fe83:93ea/64 scope link
       valid_lft forever preferred_lft forever
```

- The ra-only server may be started as follow (in the blue namespace):
```
sudo ip netns exec blue dnsmasq \
    --leasefile-ro \
    --interface=veth10 \
    --enable-ra \
    --dhcp-range=::,constructor:veth10,ra-only
```

- To combine it with DHCPv6, run:
```
sudo ip netns exec blue dnsmasq \
    --leasefile-ro \
    --interface=veth10 \
    --enable-ra \
    --dhcp-range=::2220,::2222,constructor:veth10,slaac
```

To check that the server is running, we can use the `ss` tool:
```
$ sudo ip netns exec blue ss -ap
Netid State      Recv-Q Send-Q Local Address:Port Peer Address:Portd
u_dgr UNCONN     0      0      * 7482662          * 12569 users:(("dnsmasq",pid=15612,fd=13))
raw   UNCONN     0      0      :::ipv6-icmp       :::*    users:(("dnsmasq",pid=15612,fd=3))
udp   UNCONN     0      0      *:domain           *:*     users:(("dnsmasq",pid=15612,fd=6))
udp   UNCONN     0      0      :::domain          :::*    users:(("dnsmasq",pid=15612,fd=8))
udp   UNCONN     0      0      :::dhcpv6-server   :::*    users:(("dnsmasq",pid=15612,fd=4))
tcp   LISTEN     0      5      *:domain           *:*     users:(("dnsmasq",pid=15612,fd=7))
tcp   LISTEN     0      5      :::domain          :::*    users:(("dnsmasq",pid=15612,fd=9))
```

To stop the server, just send a `kill` signal to the process pid:
```
$ sudo ip netns exec blue kill <dnsmasq pid id>
```

## The Client

On the client side (the red namespace) the IPv6 address should be already
in place. This is the SLAAC one which its host bits have been auto-generated
locally and the network bits delivered through the RA message.

```
$ sudo ip netns exec red ip addr show veth00
181: veth00@if180: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 92:3a:37:75:36:40 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fd::903a:37ff:fe75:3640/64 scope global mngtmpaddr dynamic
       valid_lft 3594sec preferred_lft 3594sec
    inet6 fe80::903a:37ff:fe75:3640/64 scope link
       valid_lft forever preferred_lft forever
```

When DHCPv6 is also enabled, one can run a DHCPv6 client and the appropriate
IPv6 address from the DHCPv6 range will be assigned.
```
sudo ip netns exec red dhclient -6 veth00
```
Lets re-examine the interface addresses:
```
$ sudo ip netns exec red ip addr show veth00
181: veth00@if180: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 92:3a:37:75:36:40 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fd::2220/64 scope global dynamic
       valid_lft 3596sec preferred_lft 3596sec
    inet6 fd::903a:37ff:fe75:3640/64 scope global mngtmpaddr dynamic
       valid_lft 3176sec preferred_lft 3176sec
    inet6 fe80::903a:37ff:fe75:3640/64 scope link
       valid_lft forever preferred_lft forever
```

> **Note**: A RA SLAAC address is identified by the `mngtmpaddr` label
assigned on it.

> **Note**: The RA message also supplies the default route, which is pointed
to the link-local address of the server side (i.e. the blue namespace
interface).

---

## RA and DHCPv6 combinations

There are the useful combinations:

- Only RA, called SLAAC,
  - client managed IPv6 address
  - RA contains prefix length and dns server
- RA, and no client specific DHCPv6 information, called "stateless DHCP"
  - client managed IPv6 address,
  - additional info like NTP server via DHCPv6
- RA, and client specific information in DHCP, called "stateful DHCP"
  -  server managed IP address,

Information in DHCPv6 might overlap with information from RA.
If they are not consistent,  different clients might behave in a different way.
