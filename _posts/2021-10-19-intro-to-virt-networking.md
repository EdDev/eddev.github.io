---
title: "Introduction to Virtual Networking"
date: 2021-10-19
categories:
  - blog
tags:
  - network 
---

TBD (scope of this intro)

## The OSI model
The [OSI model](https://en.wikipedia.org/wiki/OSI_model) and its encarnations have been insived to seperate and decouple layers of communication functions.
This layering model allowed the creation of multiple protocols and implementation to be created which can still operate in a networking stack.

- Physical (L1): Defines cable types, voltage, connectors, etc.
- Data Link (L2): Defines direct link communication between network nodes.
- Network (L3): Defines connectivity means between nodes on different local area netowrks (LAN).
- Transport (l4): Defines how to transfer data sequences between nodes, with options of reliability and flow control.
- Session (L5): Defines how to distingwish and manage a "connection" between two nodes.
- Presentation (L6): Defines means of translating between application data and network data.
- Application (L7): Defines services in the application that are dedicated to serve the communication function.

### TCP/IP
Nowdays, the set of communication protocols which are commonly used in the industry is the [Internet Protocol Suite](https://en.wikipedia.org/wiki/Internet_protocol_suite) (also known as TCP/IP).

- Link: Equivalent to the OSI L2.
- Internet: Equivalent to the OSI L3.
- Transport: Equivalent to the OSI L4.
- Application: Equivalent to the OSI L5-L7.

### Ethernet frame
[Ehternet](https://en.wikipedia.org/wiki/Ethernet_frame) is a L2 protocol that is used to transfer data between nodes in a network segment (LAN).

Initially, LAN/s connected network devices which were located in a close physical area. Nowdays, LAN/s may be distributed accross distance areas using tunneling protocols.

Noticable metadata passed in an ethernet frame includes:
- Desitnation and source MAC addresses.
- Next protocol ID.
- CRC
- An optional VLAN tag and priority.

### IP packet
The Internet Protocol defines an IP packet which encapsulates the payload data that is passed between network nodes.
The IP protocol operates at L3 and currently comes in two versions: [IPv4](https://en.wikipedia.org/wiki/IPv4) and [IPv6](https://en.wikipedia.org/wiki/IPv6_packet).

Noticable metadata passed in an ethernet frame includes:
- Version
- Desitination and source IP addresses.
- Priority.
- Time To Live (TTL) and Hop Limit.
- Next protocol ID.

## Bridge
A [bridge](https://en.wikipedia.org/wiki/Bridging_(networking)) is a network device that connects and unifies two or more network segments (LAN). It is considered a L2 device.

Before bridges came to be, networks used to connect network nodes with just cables and [hubs](https://en.wikipedia.org/wiki/Ethernet_hub). These were operating at the physical layer (L1) and just amplified the signals recieved at their ports.
One of its main characteristics was that all nodes in the LAN "heared" all others, even for unicast communication.

Bridges improved the situation by filtering out unicast traffic and allowing only broadcast and multicast to pass.
It does so by managing a [mac table (also known as fib)](https://en.wikipedia.org/wiki/Forwarding_information_base).

### Switch
A switch is in fact a bridge with enchanced capabilities.
It has multiple ports, each may be considered as a bridge.

Some basic switch capabilities include [port aggregation](https://en.wikipedia.org/wiki/Link_aggregation), [VLAN](https://en.wikipedia.org/wiki/Virtual_LAN) and [trunking](https://en.wikipedia.org/wiki/Trunking) support.

## Router
A [router](https://en.wikipedia.org/wiki/Router_(computing)) is a network device that provides connectivity between LAN networks (without unifying them). Its main function us to forward L3 packets.

Routers operate based on a table named the routing-table. It contains network addresses prefixes and a next-hop destination to reach them.
A router will look at each packet destination IP and determine based on its routing table through which port to pass it on (if at all).
This routing table is populated by:
- Directly connected links network addresses.
- Static entries (provided by a user).
- Routing protocols that distribute routing entries between routers (e.g. [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)).

### Network Address Translation (NAT)
The intense growth of the internet has introduced the limitation of the IPv4 public address pool size. Very quickly the public network addresses have been exhausted and required workarrounds.

While IPv6 was consived to resolve the address pool size, the intermediate solution came in the form of [NAT](https://en.wikipedia.org/wiki/Network_address_translation).

NAT allows a private network (e.g. 10.10.0.0/16) to access the internet (public) through a router that will replace the source address of each packet to a single (or a range) IP address.
This router will preserve the entry of each mapping to translate the destination IP on incomming packets (from the internet).

NAT has two flavors: static translation (1:1 address mapping) and dynamic trnaslation (n:1 address translation). The later uses the L4 (tcp/udp) ports to assist in this task.

## Firewall
Firewalls are network entities that aim to secure networks from malicious attacks and misuse.
It does so by limiting incoming and outgoing traffic based on various characteristics.

## Interface Types
An interface in Linux is a representation of a network device.
Some interface represent physical NIC/s and some software based ones.

The list below includes some known interface types:
- ethernet: Usually represent physical devices.
- bridge: This is a software based bridge implementation.
- bond: This is a software based link-aggregation implementation.
- vlan: Represents a VLAN, when the base interface is a trunk.
- veth: A special pair of interfaces, representing peers of a "cable". Such a pair can connect two bridges or serve to connect two network namespaces.
- tap:
- tun:
- macvlan/macvtap:

For more details see [here](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking) or search the web for Linux interfaces.

## Network Namespace
Linux [namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) is a broad subject which is worth a dedicated post by itself.

A network namespace (netns) partitions the kernel network stack into self contained and individual stacks, such that almost no resource is shared between each one (e.g. routing tables, interfaces).

It is common to interconnect different netns using veth interfaces and bridges.

Linux provides the `ip netns` CLI tool to work with them from the terminal.

## Containers
[Containers](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction) cover a broad range of subjects, too wide to be covered here.
At its base, it is a technology that allows to run applications in a light-weight virtual environment which is isolated using namespaces.

Unlike virtual machines (VM), it shares the host kernel and has no emulation in place. Making it lighter, consuming less resources from the system.

There are several tools which help work with containers, one of whihch is [podman](https://docs.podman.io/en/latest/) and its ecosystem tools.

### A bridge setup
The following setup illustrates how a bridge can connect two network namespaces.

The setup looks like this:
[RED NS {veth01}]---[{veth00}--{Bridge br99}--{veth10}]---[{veth11} BLUE NS]

IP addresses are defined on the {veth01} and {veth11} interfaces, sharing the same subnet.

br99, veth00 and veth10 reside in the root NS.

```bash
sudo ip netns add red
sudo ip netns add blue

udo ip netns exec red ip link set lo up
sudo ip netns exec blue ip link set lo up

sudo ip link add veth00 type veth peer name veth01
sudo ip link set veth00 up
sudo ip link set veth01 up

sudo ip link add veth10 type veth peer name veth11
sudo ip link set veth10 up
sudo ip link set veth11 up

sudo ip link set dev veth01 netns red
sudo ip link set dev veth11 netns blue

sudo ip link add br99 type bridge
sudo ip link set br99 up
sudo ip link set veth00 master br99
sudo ip link set veth10 master br99

sudo ip netns exec red ip addr add 10.10.10.1/24 dev veth01
sudo ip netns exec blue ip addr add 10.10.10.11/24 dev veth11
```

Now ping between the red and blue namespaces:

```bash
sudo ip netns exec red ping 10.10.10.11
```

See an IPv6 setup [here](https://ehaas.net/blog/static-ipv6-playground/).

### A route setup (?)
The following route setup illustrates routing of packets between two networks. The networks are using different subnets, requiring from the router to perform a forwarding action.

> *_Note_*: To enable route forwarding in Linux, a special kernel stack option needs to be enabled.

The setup is very similar to the previous bridge setup.
The only changes are with the IP addresses required and the enablement of forwarding.

After setting up the L2 connectivity as outlined in the bridge setup, configure the following IP addresses and enable ip forwarding:
```bash
sysctl -w net.ipv4.ip_forward=1
cat /proc/sys/net/ipv4/ip_forward

sudo ip addr add 10.10.0.1/24 dev br99
sudo ip addr add 10.10.1.1/24 dev br99
sudo ip netns exec red ip addr add 10.10.0.2/24 dev veth01
sudo ip netns exec blue ip addr add 10.10.1.2/24 dev veth11
```

Now ping between the red and blue namespaces:

```bash
sudo ip netns exec red ping 10.10.1.2
```

## Virtual Machines
A [virtual machine](https://en.wikipedia.org/wiki/Virtual_machine) allows users to run guests with various OS on a hosting machine (the host).

Hypervisors like [qemu](https://en.wikipedia.org/wiki/QEMU) are used to emulate HW using
software and the host HW virtualization options.

A VM is often defined with virtual NIC/s (vNIC) which are wired through the host machine.

There are several options to connect vNIC/s to the host, out of them the most common are the tap and macvtap devices.

For a comprehensive overview of how networking is implemented in qemu, review this [post](https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net).
### tap device
A [tap device](https://en.wikipedia.org/wiki/TUN/TAP) is a virtual network device that allows packets to travese between kernel and user space. An application in user space is bounded to such devices and is able recieve and transmit data through.

tap devices serve L2 traffic and are mainly used with hypervisors like qemu.

### macvtap device
Similar to the tap device, a [macvtap device](https://virt.kernelnewbies.org/MacVTap) is bridging between the kernel and user space and mainly serve hypervisors like qemu.

Unlike the tap device, the macvtap includes a simple bridge implementation and therefore it does not need a full blown bridge.

It does come with a few limitations, like inability to serve more than one mac address (guest side) and to establish communication with the host itself.

Multiple macvtap devices may use the same host NIC.
Depending on the work mode, they can communicate between each other but not to the node itself through the base NIC.

## Packet Analyzer
Also known as sniffers, these tools are capturing traffic and are able to present them to the user for analysis.

[tcpdump](https://en.wikipedia.org/wiki/Tcpdump) and [wireshark](https://en.wikipedia.org/wiki/Wireshark) are among the most common ones used.

### tcpdump
tcpdump is CLI based tool which is a great friend when debugging and troubleshooting network issue.

With a very simple command, one is able to see the traffic passing throuhg a specific interface:
```bash
sudo tcpdump -i eth0
```
