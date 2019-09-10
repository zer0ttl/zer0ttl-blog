---
title: "CCNP Switch Session Notes - Part 1"
date: 2019-09-10T09:49:35+05:30
draft: false
description: "Recently I had a CCNP Switch refresher session for a friend who is studying for his CCNP. Notes are divided into two parts. This is the first part."
author: "zer0ttl"
tags: ["networking", "ccnp", "notes", "switching"]
---

# tldr;

Recently I had a CCNP Switch refresher session for a friend who is studying for his CCNP. These are the notes from that session.

## Topology

This is the topology that I used for the session. All configurations and examples are based on this topology.

![ccnp_topology](/img/ccnp_switch_topology.png)

**IMPORTANT :** If you think, your configuration can cause a disruption in service or you may lose access to the switch, use the following fail-safe mechanism. You can reload a switch after `n` minutes.

```text
SWITCH-A#reload in 00:10 I am going to make changes

! cancel the scheduled reload

SWITCH-A#reload cancel
```

## Basic Configuration

A VLAN is a **single broadcast domain**, meaning all hosts can reach each other by *broadcasting* at the data link layer.

* Configure a VLAN and name it

```text
SWITCH-A#vlan 10
SWITCH-A(config-vlan)#name VLAN10
```

<a href="https://packetlife.net/media/library/20/VLANs.pdf" target="_blank">Sweet VLAN cheatsheet : packetlife.net</a>


## Trunking, DTP, VTP

#### Trunking

An Ethernet frame is *tagged* with VLAN information. For frames to flow between switches, a **trunk** port must be established. A **trunk** port allows multiple VLANs through itself.

* Configure a trunk port :

```text
! SWITCH-A
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#switchport trunk encapsulation dot1q
SWITCH-A(config-if)#switchport mode trunk

! SWITCH-B
SWITCH-B(config)#int ethernet 1/1
SWITCH-B(config-if)#switchport trunk encapsulation dot1q
SWITCH-B(config-if)#switchport mode trunk

! Note: the 'switchport trunk encapsulation dot1q' command may not be required
! on modern switches like 2960X/XR, 3850, etc.
```

Sometimes only a set of VLANs are required to be propagated to another switch. In such cases VLAN pruning should be configured so that the overhead of processing VLANs on the switch is reduced.

* Configure VLAN pruning :

```text
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#switchport trunk allowed vlan 1-10
```

* Whenever you make changes to the VLAN pruning configuration, make sure that you use the keywords `add`, `remove` either to add VLANs or remove VLANs from the existing trunked VLANs list. By not doing so, you will override the current VLANs allowed on the trunk

* Editing VLAN pruning configuration :

```text
! add vlan 20 to existing config
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#switchport trunk allowed vlan add 20

! remove vlan 20 from the existing config
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#switchport trunk allowed vlan remove 20
```

#### DTP - Dynamic Trunking Protocol

DTP is a proprietary networking protocol developed by Cisco for the purpose of automatic creation of trunk links between two Cisco switches.

No sane engineer would want switches to configure trunk links on their own in their network. It is best to disable DTP on all trunking interfaces.


* Disable DTP

```text
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#switchport nonegotiate
```

<a href="https://packetlife.net/blog/2008/sep/30/disabling-dynamic-trunking-protocol-dtp/" target="_blank">More on DTP : packetlife.net</a>

#### VTP - VLAN Trunking Protocol

VTP is another proprietary networking protocol developed by Cisco to automatic propagation of VLANs across the network. This is another nightmare that should keep network engineers awake at night if not handled properly.

I personally disable it in the networks I configure and also advise against usage of VTP.

<a href="https://www.reddit.com/r/networking/comments/1jf832/vtp_disaster/" target="_blank">VTP disaster : reddit.com</a> : *Friends don't let friends use VTP*

* VTP refresher for just in case some *smart* interviewer wants to test your knowledge.
     * VTP creates, updates and deletes VLANS. For every change the revision number will be incremented by one.
     * **VTP Domain** : Should be same on all switches participating in VTP. Change this to reset revision number.
     * **VTP Server mode** : Default mode, generate and propagates VTP advertisements to clients
     * **VTP Client mode** : Receives and forwards advertisements from servers; cannot create, change, or delete VLANs.
     * **VTP Transparent mode** : The *good mode*. Does not participate in VTP, does not share their VLAN configuration, however forwards advertisements from servers.

* Disable VTP :

```text
SWITCH-A(config)#vtp mode transparent
```

## STP - Spanning Tree Protocol

Spanning tree protocol will help you build loop free networks. Bridge protocol data units (BPDU) are exchanged between switches to synchronize spanning tree. **BPDUs will flow from the root bridge downwards to all switches.**


#### BPDU Format :

![BPDU Frame Format](/img/bpdu_frame_format.png)

Spanning tree elects a root bridge. Switch with lowest bridge identifier becomes the root bridge.

Bridge Identifier = Bridge Priority + Extended System ID + MAC Address. Extended System ID is nothing but the VLAN number.

In our topology, `Switch-A` becomes the **root bridge**, as the tie-breaker is the mac-address of the switches. All other switches become **non-root** switches.

```text
! output of show spanning-tree vlan 10 on Switch-A
! some output omitted

bridge priority     :   32778
extended system id  :   sys-id-ext 10
mac address         :   aabb.cc00.0100

! output of show spanning-tree vlan 10 on Switch-B
! some output omitted

bridge priority     :   32778
extended system id  :   sys-id-ext 10
mac address         :   aabb.cc00.0200

! output of show spanning-tree vlan 10 on Switch-C
! some output omitted

bridge priority     :   32778
extended system id  :   sys-id-ext 10
mac address         :   aabb.cc00.0300
```

#### Spanning tree port types

**Designated ports** : Interfaces that forward traffic. All ports on **root** bridge are always in forwarding mode.

**Root port** : All **non-root** switches have to find the shortest path to the root bridge. The interface that leads the switch to the root bridge is called the **root port** and is in forwarding state.

**Non-designated port** : In order to break the loop, spanning-tree has to block a port. In our topology, `Switch-B` and `Switch-C` will compare their bridge identifier to find out whose port will get blocked. The lowest bridge identifier wins so port on `Switch-C` will get blocked. A port that is blocking traffic is called **non-designated port**.

Priority is **32768** by default on all switches. You can set the priority manually on the switches. Lower the priority, higher are the chances of the switch becoming the root bridge.

The cost to root bridge is determined using a cost table as below:

| Bandwidth | Cost |
|-----------|------|
|  10 Mbps  |  100 |
| 100 Mbps  | 19   |
| 1G        | 4    |
| 10G       | 2    |

In the BPDU, the field named **root path cost** will contain a switch's cost of its shortest path to the root bridge.

#### Spanning tree decision making algorithm

Whenever spanning-tree has to make a decision, following list will be used:

1. **Lowest bridge ID** : the switch with the lowest bridge becomes the root bridge.
2. **Lowest path cost to root bridge** : when the switch receives multiple BPDUs, it will select the interface with the lowest cost to root bridge as the root port
3. **Lowest sender bridge ID** : when a switch is connected to two switches that it can use to reach the root bridge and cost to reach the root bridge is the same, it will select the interface connecting to the switch with the lowest bridge ID as the root port.
4. **Lowest sender port ID** : when the switch has two interfaces connecting to the same switch, and the cost to reach the root bridge is the same, it will use the interface with the lowest number as the root port.

#### Root bridge config

A switch can be configured as root bridge in two ways: configure the switch as root bridge or set the priority of the switch to a lower value than the rest of the switches.

* Configure a switch as root bridge for a VLAN using the `root` command

```text
SWITCH-B(config)#spanning-tree vlan 20 root primary
```

* Configure lower priority on switch so that it becomes the root bridge

```text
SWITCH-A(config)#spanning-tree vlan 20 priority 4096
```

#### Port States in spanning tree :

1. Listening
     * Only root or designated ports move to listening state. No data transmitted for **15 seconds**. Post 15 seconds, port moves to learning state.
2. Learning
     * At this state, the interface will process Ethernet frames. Frames are however not forwarded to the destination. Only the mac-address table will be populated. It takes **15 seconds** to move to the next state i.e forwarding state.
3. Forwarding
     * Final state of the interface and interface will forward Ethernet frames.
4. Blocking
     * This is the default state of a non-designated port. A non-designated port will remain in this state for **20 seconds** before transitioning to listening state in case of root port failure.

Altogether, if a port flaps, it will take the port a minimum of 15 + 15 = **30 seconds to start forwarding traffic**. Keep this in mind while troubleshooting issues in your network.

In order to bypass this and put a port directly to forwarding state, use the `portfast` command. This is usually configured on the ports connected to end devices like computers, phones, laptops, printers, etc. This can be configured globally as well as individually on a every interface.

```text
! enable portfast globally on a switch
SWITCH-A(config)#spanning-tree portfast default

! enable portfast on an interface
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-if)#spanning-tree portfast
```

#### Uplink fast

Uplink fast is used to recover from direct link failure.

Non-designated port stay in blocking state until a designated port goes down. In this case, the non-designated port will go from port states 1-4. It will take 50 seconds for that switch to be a part of the network. For fast transition of a non-designated port to a designated port, utilize the `uplinkfast` feature. This is a global config.

```text
SWITCH-A(config)#spanning-tree uplinkfast
```

#### Backbone fast

Backbone Fast is used to recover from an indirect link failures.

In our topology, Switch-A is the root bridge, port e0/1 on Switch-V and port e1/0 on Switch-C are the root ports. e2/0 on Switch-C is the non-designated port. Now imagine the link between Switch-A and Switch-B goes down. Now it will take 50 seconds for port e2/0 on Switch-C to go from blocking to forwarding state. In order to quicken this process use **Backbone fast** feature. This is global config.

```text
SWITCH-A(config)#spanning-tree backbonefast
```

#### Difference between classic spanning tree and rapid spanning tree

|  Classic Spanning Tree    |  Rapid Spanning Tree  |
|------------------------   |---------------------- |
| Blocking                  | Discarding            |
| Blocking                  | Discarding            |
| Learning                  | Learning              |
| Blocking                  | Blocking              |

RST has only three port states. **Discarding** is a new state that combines the blocking and listening port state.

#### Spanning tree protection

* BPDUGuard
     * This will disable (err-disable) an interface that has PortFast configured, it that port receives a BPDU

```text
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-int)#spanning-tree bpduguard enable
```

* BPDUFilter
     * This will stop sending or receiving BPDUs on an interface

```text
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-int)#spanning-tree bpdufilter enable
```

* Rootguard
     * This is enabled on the designated ports of root switch, so in the case a superior BPDU is received on those ports then those ports are put in inconsistent state.

```text
Configure RootGuard on a port
SWITCH-A(config)#int ethernet 1/1
SWITCH-A(config-int)#spanning-tree guard root
```

* Loopguard
     * When a switch is sending but not receiving BPDUs on the interface, LoopGuard will place the interface in the loop-inconsistent state and block all traffic

```text
SWITCH-A(config)#spanning-tree loopguard default
```

---

We will cover etherchannel, inter-vlan routing, SPAN, remote SPAN, and gateway redundancy in the next post.
