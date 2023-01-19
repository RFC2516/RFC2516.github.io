---
title: Learning EIGRP for Cisco ENCOR
tags: [certifications, cisco, encor, eigrp]
style: fill
color: success
description: Notes and thoughts on learning EIGRP for the Cisco ENCOR Certification
---

# Preamble

This post represents my learning efforts for the Cisco ENCOR Ceritifcation. Specifically for me to learn EIGRP and attempting to encode the data into a structured way that I can remember. I have prior experience in learning EIGRP from the ICND2 200-105 Exam which is now a retired exam. 

Here's the topics I've come up with regarding EIGRP. Throughout this post I will be using a Containerlab Topology File which consists of three Cisco CSR1000v routers. You can find this lab on my [github](https://github.com/RFC2516/clab-eigrp).

{% capture list_items %}
Initializing the Lab
EIGRP Features
Neighbor Adjacency
Timers
Metrics
Path Selection
{% endcapture %}
{% include elements/list.html title="Table of Contents" type="toc" %}

## [Initializing the Lab](#preamble)

To start I'll be initializing the lab in my pre-existing lab host by using the command `clab dep -t eigrp.yaml`. This will provide details regarding each node as shown below.

{% include elements/figure.html image="/media/2022-12-28/clab-node-details.png" caption="Node Details" %}

Next I will use `clab graph -t eigrp.yaml` to display the topology graphically.

{% include elements/figure.html image="/media/2022-12-28/clab-graph-details.png" caption="Topology Graph" %}

## [EIGRP Features](#preamble)

Cisco's public documentation on the subject is located [here](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/configuration/xe-16/ire-xe-16-book/ire-enhanced-igrp.html#GUID-57813511-FA91-4DE8-9E5F-2C1ABBDF39F0). There are several use-cases or benefits to using EIGRP in your network.

* Increased network width when compared with the RIP Routing Protocol. This use-case or benefit requires knowledge regarding RIP which has a limitation of routing updates only containing a maximum of 15 hops.
* Fast Convergence to configured neighbors when a routing update occurs. This is useful for when an outage schedules itself or you are scheduling a change.
* Partial updates to neighbors for routes learned. The goal is to minimize the amount of bandwidth use by the EIGRP process. Which in the world of 10 Gbps interfaces is likely not significant.
* Scaling for large networks. The goal is to add options to the EIGRP Routing Process which allow you to summarize and segment routing domains which is necessary in very large networks.

## [Neighbor Adjacency](#preamble)

From the same [EIGRP documentation](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/configuration/xe-16/ire-xe-16-book/ire-enhanced-igrp.html#GUID-07AC554E-631D-4884-B459-8FDB30162D79). The configuration for neighbors is fairly striaght forward. There are two options for configuring EIGRP

* Autonomous System Configuration
* Named Configuration

Let's explore more about these two options.

### Autonomous System Configuration

With this configuration option you use the `router eigrp` statement which ends with any choosen autonomous system number. For my lab I choose to use `64512`. This is also known as EIGRP Classic Mode in the documentation.

```
rtr1#show run | s router eigrp
router eigrp 64512
 network 172.16.0.0 0.15.255.255
```

The `router eigrp 64512` line creates an distinct EIGRP Routing Process on the router. The `network 172.16.0.0 0.15.255.255` line configures the EIGRP Routing Process to begin sending updates to interfaces which match this network. All EIGRP updates to any other neighbor going forward routing information regarding these interfaces. This allows other EIGRP Routers to know that the network can be reached through this portion of the topology.


<div class="alert alert-warning" role="alert">
  This configuration represents a legacy way of configuring EIGRP on a router. <a href="https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/configuration/xe-17/ire-xe-17-book/ire-classic-to-named.html#:~:text=IPv4%20and%20IPv6.-,We%20recommend,-that%20you%20upgrade">Cisco recommends</a> that you use Named Configuration for EIGRP because all new features are only available in Named Mode.
</div>


### Named Configuration

With this configuration option you use the `router eigrp` statement which ends with a `instance-name`, for example `router eigrp corpnet` where **corpnet** represents the instance-name. The difference however is that the EIGRP Named Configuration does not create a EIGRP Routing Instance yet. You must also specify an `address-family` to complete the configuration. An example of how to configure a router for EIGRP Named Configuration is as such:

```
rtr1#show run | s router
router eigrp corpnet
 !
 address-family ipv4 unicast autonomous-system 64512
  !
  topology base
  exit-af-topology
  network 172.16.0.0 0.15.255.255
 exit-address-family
```


### Checking Adjacency

Neighbor Adjacency requirements are shown in the table.

| Requirement | - |
| ------ | -----|
| The routers must be able to send packets to the multicast address 224.0.0.10 | Yes |
| A primary address must be matched by a `network` statement. | Yes |
| The interfaace connecting to the neighbor must not be passive. | Yes |
| Must use the same autonomous system. | Yes |
| Hello and Hold Timers must match | No |
| Neighbors must authenticate if configured | Yes |
| IP MTU must match | No |
| K-Values must match | Yes |
| Router IDs must be unique | No [1]|

<font size="1">  *[1] While EIGRP will still form an Adjacency with a router with the same Router-ID, the EIGRP topology table will reject routes advertised from the same Router-ID. This can be identified as occuring by running `show ip eigrp events | i dup` which will return a message similar to `Ignored route, dup routerid int: 10.0.0.15`.* </font>

<br/>

Checking for these adjancies can be done at a global level as shown below.

```
rtr1#show ip eigrp neighbors 
EIGRP-IPv4 Neighbors for AS(64512)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
1   172.30.0.6              Gi2                      14 00:01:08    3   100  0  7
0   172.30.0.2              Gi3                      14 00:01:11 1276  5000  0  6
```

Additionally this can be checked at an interface level as shown below.

```
rtr1#show ip eigrp neighbors gi2             
EIGRP-IPv4 Neighbors for AS(64512)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
1   172.30.0.6              Gi2                      13 00:02:44    3   100  0  7
```

## [Timers](#preamble)

EIGRP uses two timers to to contorl it's neighbor relationships. The hello timer is used to configure the interval between hello packets. The hold timer indicates how long the routing process should wait before declaring an EIGRP Neighbor down.

By default the hello timer is 5 seconds and the hold timer is 15 seconds. This can be seen in the `show run all | s router eigrp` command.

```
rtr1#show run all | s router eigrp   
router eigrp corpnet
 no shutdown
 !
 address-family ipv4 unicast autonomous-system 64512
  !
  af-interface default
   ! additional config ommited for brevity
   hello-interval 5
   hold-time 15
   !
  exit-af-interface
  !
  ! additional config ommited for brevity.
```

## [Metrics](#preamble)

EIGRP uses metrics to compare learned route information from neighbor. It can be configured using the `metric weights 0 1 0 1 0 0 0` command under the `router eigrp <name>` and `address-family` context.

Mismatched K Values can prevent neighbor relationships from forming therefore if using the metrics weight command the same value should be specified by all EIGRP neighboring routers.

There is an excessively complex formula which is frequently touted infront of aspiring network engineers, but I see no practical reason to memorize it.

{% include elements/figure.html image="/media/2022-12-28/eigrp-formula.png" caption="Practical Networking's EIGRP Metric Graphic" %}

[Graphic credit](https://pracnet.net)

The practical purpose of the metrics is to both predict and influence routing paths. Let's take the current state of our lab into consideration.

```
rtr1#show ip route eigrp
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       H - NHRP, G - NHRP registered, g - NHRP registration summary
       o - ODR, P - periodic downloaded static route, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR
       & - replicated local route overrides by connected

Gateway of last resort is 10.0.0.2 to network 0.0.0.0

      172.30.0.0/16 is variably subnetted, 5 subnets, 2 masks
D        172.30.0.8/30 [90/15360] via 172.30.0.6, 00:02:49, GigabitEthernet2
                       [90/15360] via 172.30.0.2, 00:02:49, GigabitEthernet3
rtr1#
```

To reach the EIGRP (D) learned route `172.30.0.8/30` from rtr1 we have two acceptable paths. Either through `172.30.0.6` or `172.30.0.2`. Both have a tied metric `15360` and hence the EIGRP process has installed the route. Inside of the EIGRP process there is a database entry for the route:

```
rtr1#show ip eigrp topology 172.30.0.8 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.1) for 172.30.0.8/30
  State is Passive, Query origin flag is 1, 2 Successor(s), FD is 1966080, RIB is 15360
  Descriptor Blocks:
  172.30.0.2 (GigabitEthernet3), from 172.30.0.2, Send flag is 0x0
      Composite metric is (1966080/1310720), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
  172.30.0.6 (GigabitEthernet2), from 172.30.0.6, Send flag is 0x0
      Composite metric is (1966080/1310720), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.6
```

We can see that EIGRP describes this route as having a **metric** which is comprised of Minimum Bandwidth, Total Delay, Reliablity, Load, MTU and Hop Count. Let's, through experimentation, learn how the metric is calculated. Let's manipulate the route to `172.30.0.6` over interface GigabitEthernet2.

By default the config is:
```
rtr1#show int gi2 | i BW|relia
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, 
     reliability 255/255, txload 1/255, rxload 1/255
rtr1#
```
Let's modify it to

```
rtr1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
rtr1(config)#int gi2
rtr1(config-if)#bandwidth 100000 
rtr1(config-if)#end
```
Now if we check the same topology database we will see a significant change in the metric.
```
rtr1#show ip eigrp topology 172.30.0.8 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.1) for 172.30.0.8/30
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 1966080, RIB is 15360
  Descriptor Blocks:
  172.30.0.2 (GigabitEthernet3), from 172.30.0.2, Send flag is 0x0
      Composite metric is (1966080/1310720), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
  172.30.0.6 (GigabitEthernet2), from 172.30.0.6, Send flag is 0x0
      Composite metric is (7864320/1310720), route is Internal
      Vector metric:
        Minimum bandwidth is 100000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.6
```


## [Path Selection](#preamble)

A requirement of any routing protocol, including EIGRP, is to select the most optimal loop free path. The EIGRP protocol uses something called the **EIGRP Fesiability Condition** to ensure the path is loop free and it uses a **Metric** to determine the most optimal path. First we will discuss the Feasiability Condition and then we will show the metric. 

Now in order to understand the condition however you must be familiar with the terminology used for EIGRP.

| Name | Description |
| ---| ---|
| Reported Distance (RD) | The total metric value from the remote router's interface to the network. |
| Feasible Distance (FD) | The total metric value from the local interface to the remote router and to the network. |

Therefore the condition can be understood as, *For any given route, the neighbor's Reported Distance (RD) must be smaller than the local router's feasible distance (FD) for the same route*. 

Let's consider our diagram to better understand this.

{% include elements/figure.html image="/media/2022-12-28/clab-graph-distances.png" caption="Topology Graph with Distances" %}

Both distances (RD & FD) are a measurement of the metric from a different point of view in the network. The `Reported Distance` is the metric for the network as seen by another router. The `Feasible Distance` is the metric from the point of view by the local router.

So let's consider a thought experiment. rtr2 has an EIGRP Neighborship with both rtr1 & rtr3. It receives the `Reported Distance` from both routes. rtr1 has a RD of `200` and rtr3 has a RD of `10`. `rtr2` will then apply it's own metric of `10` to each RD to create it's own `Feasible Distance`. Through the rtr1 router the FD is `210` and through the `rtr3` router the FD is `20`.

I've simplified the metrics to a simple `10`, however on an actual router the metric will not be so human readable. I've edited both the Feasible Distance and Reported Distance to simplify the numbering. On rtr1 we have the directly connected network `172.30.0.0/30`. If we query our EIGRP table we have the following:

```
rtr2#show ip eigrp topology 172.30.0.12 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.6) for 172.30.0.12/30
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 20, RIB is 2
  Descriptor Blocks:
  172.30.0.10 (GigabitEthernet4), from 172.30.0.10, Send flag is 0x0
      Composite metric is (20/10), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
  172.30.0.5 (GigabitEthernet2), from 172.30.0.5, Send flag is 0x0
      Composite metric is (30/20), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 30000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
        Originating router is 172.30.0.2
rtr2# 
```
rtr2 is learning the `172.30.0.12/30` network from two EIGRP Neighbors. The first neighbor is `172.30.0.10` which represents rtr3 and the second neighbor is `172.30.0.5` which represents rtr1. 

rtr2 prefers the route through `172.30.0.10` because it has the lowest metric value. Indicating that it the best route, which only the best routes are installed in the Routing Table.

There are multiple ways we can influence this. We could modify any enabled K Value which by default is only **Bandwidth** and **Delay** or use an offset-list command on an interface.

First let's try by modifying the bandwidth on `gi4` interface from `1000000 Kbit` to `100000 Kbit` the feasible distance (FD) went from `20` to `80`.

```
rtr2#show run int gi4
Building configuration...

Current configuration : 162 bytes
!
interface GigabitEthernet4
 description "TO_RTR3"
 bandwidth 100000
 ip address 172.30.0.9 255.255.255.252
 negotiation auto
 no mop enabled
 no mop sysid
end
rtr2#
rtr2#show ip eigrp topology 172.30.0.12 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.6) for 172.30.0.12/30
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 30, RIB is 3
  Descriptor Blocks:
  172.30.0.5 (GigabitEthernet2), from 172.30.0.5, Send flag is 0x0
      Composite metric is (30/20), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 30000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
        Originating router is 172.30.0.2
  172.30.0.10 (GigabitEthernet4), from 172.30.0.10, Send flag is 0x0
      Composite metric is (80/10), route is Internal
      Vector metric:
        Minimum bandwidth is 100000 Kbit
        Total delay is 20000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
rtr2#
```

The above topology table output shows that our preferred route to the `172.30.0.12/30` network is now `gi2` instead of `gi4` because the metric for reaching the network through the interface increased.

We can perform the same by increasing the delay on the interface.

```
rtr2#show run int gi4
Building configuration...

Current configuration : 160 bytes
!
interface GigabitEthernet4
 description "TO_RTR3"
 ip address 172.30.0.9 255.255.255.252
 delay 10000000
 negotiation auto
 no mop enabled
 no mop sysid
end         
rtr2#
rtr2#show ip eigrp topology 172.30.0.12 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.6) for 172.30.0.12/30
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 2621440, RIB is 20480
  Descriptor Blocks:
  172.30.0.5 (GigabitEthernet2), from 172.30.0.5, Send flag is 0x0
      Composite metric is (30/20), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 30000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
        Originating router is 172.30.0.2
  172.30.0.10 (GigabitEthernet4), from 172.30.0.10, Send flag is 0x0
      Composite metric is (80000020/80000010), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 100015268789063 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
rtr2#
```

Again our preference is now `gi2` and our metric for `gi4` has changed.

We can perform the same by using an offset-list command in the EIGRP process.

```
rtr2#show run | s router eigrp                             
router eigrp corpnet
 !
 address-family ipv4 unicast autonomous-system 64512
  !
  topology base
   offset-list EIGRP-OFFSET-GI4 in 1000000000 GigabitEthernet4 
  exit-af-topology
  network 172.16.0.0 0.15.255.255
  eigrp router-id 172.30.0.6
 exit-address-family
rtr2#
rtr2#
rtr2#
rtr2#show ip eigrp topology 172.30.0.12 255.255.255.252
EIGRP-IPv4 VR(corpnet) Topology Entry for AS(64512)/ID(172.30.0.6) for 172.30.0.12/30
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 30, RIB is 3
  Descriptor Blocks:
  172.30.0.5 (GigabitEthernet2), from 172.30.0.5, Send flag is 0x0
      Composite metric is (30/20), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 30000000 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
        Originating router is 172.30.0.2
  172.30.0.10 (GigabitEthernet4), from 172.30.0.10, Send flag is 0x0
      Composite metric is (1000000020/10), route is Internal
      Vector metric:
        Minimum bandwidth is 1000000 Kbit
        Total delay is 15278789063 picoseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 172.30.0.2
rtr2#
rtr2#
rtr2#
rtr2#
rtr2#show ip access-list EIGRP-OFFSET-GI4
Standard IP access list EIGRP-OFFSET-GI4
    10 permit 172.30.0.12, wildcard bits 0.0.0.3 (10 matches)
```

Again our preference is now `gi2` and our metric for `gi4` has changed. Interestingly the offset-list command appears to modify the delay as well so there's not a significant amount of difference between the two methods.