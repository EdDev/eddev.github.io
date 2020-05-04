---
title: "Static IPv6 Playground"
date: 2020-05-03
categories:
  - blog
tags:
  - ipv6
---

This post is all about setting up an IPv6 enviorment for learning and testing 
of varios setups and scenarios.

We will use network namespaces for our setup and proceed to define static
IP addresses.
(we will cover dynamic IP addresses in a following post)

## Basic Setup
In order to prepare the ground for our IPv6 playground, we will use network
namespaces to simulate different network stacks.
A network namespace is an isolated network stack that includes interfaces,
ip addresses and routes.

Throught this post, we will use the `ip` command which is part of the
[iproute2](https://wiki.linuxfoundation.org/networking/iproute2) utilities.

### Namespace creation
For our setup, we will use two namespaces: red & blue.

Lets create our two namespaces:
```
sudo ip netns add red
sudo ip netns add blue
```
Each namespace is created with a loopback interface which requires an explicit
enablement:
```
sudo ip netns exec red ip link set lo up
sudo ip netns exec blue ip link set lo up
```

### L2 connectivity (the veth)
In order to enable connectivity between the two namespaces, we will use a
veth interface. A veth interface type comes always in pairs, anything that
ingress one edge, egress the other edge and vice versa. It provides a L2 local
connectivity between the peers.

We will create the veth interface at the root namespace and then place each
peer in one of the namespaces.

Lets create the veth interface (and enable its links):
```
sudo ip link add veth00 type veth peer name veth10
sudo ip link set veth00 up
sudo ip link set veth10 up
```
And place each in the relevant namespace:
```
sudo ip link set dev veth00 netns red
sudo ip link set dev veth10 netns blue
```

At this point, assuming that IPv6 is enabled on the host, each namespace
should show a loopback interface and another veth type interface which
has a [link-local IPv6 address](https://tools.ietf.org/html/rfc4291#page-11).

Lets check the interfaces and their addresses:
```
$ sudo ip netns exec red ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
181: veth00@if180: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 92:3a:37:75:36:40 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::903a:37ff:fe75:3640/64 scope link
       valid_lft forever preferred_lft forever
```
```
$ sudo ip netns exec blue ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
180: veth10@if181: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 8e:6a:6b:83:93:ea brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::8c6a:6bff:fe83:93ea/64 scope link
       valid_lft forever preferred_lft forever
```

## L3 connectivity
With the IPv6 link-local addresses in place, we can already check the
connectivity between the two namespaces.

Using the infromation gathered in the previous `ip addr` commands, learn
the link-local IPv6 address of each peer and use it in the ping command.

Note: As the addresses have link-local scope, a zone must be added to the
destination address.
A link-local address has a default subnet of 64 bits with a default network
address of `fe80`. Therefore, on a node with multiple interfaces, an explicit
egress interface needs to be provided in order for the packet to know through
which interface to exit. 
See [here](https://tools.ietf.org/html/rfc4007) for more information
about zones.

Lets run an IPv6 ping:
```
sudo ip netns exec red ping -6 fe80::<the-peer-last-64-bits-address>%veth00
``` 

## IPv6 with global scope connectivity
The previous IPv6 link-local addresses may be used to check L3 connectivity
betweem two directly connected peers (i.e. interfaces connected to the same
physical LAN). Routers are required not to forward link-local addresses.

Therefore, in order to enable IPv6 connectivity beyond the physical LAN,
a global scoped address needs to be defined on the interface.
Such address may be set statically or dynamically.

### IPv6 Global Static Address
In order to enable connectivity without a router, we need both peers to be
set on the same network subnet, i.e. the network prefix need to be identical
for both peers and the host part needs to be unique.

We will use a 64 bit network subnet with a network address of `fd00`.
Resulting in the following addresses:
- red: `fd00::11/64`
- blue: `fd00::22/64`

Lets define a static address for each namespace:
```
sudo ip netns exec red ip addr add fd00::11/64 dev veth00
sudo ip netns exec blue ip addr add fd00::22/64 dev veth10
```
With this behind us, we can check the connectivity (without the zone part
this time):
```
sudo ip netns exec red ping -6 fd00::22
```

Next we will expore [dynamic IPv6](../dynamic-ipv6-playground).
