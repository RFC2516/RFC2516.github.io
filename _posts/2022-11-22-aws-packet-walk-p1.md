---
title: Part 1 -  Packet Walking in AWS
tags: [Networking, AWS]
style: fill
color: primary
description: Performing a packet walk can be the most important step to troubleshooting a network issue. This post talks about the terminology and theory.
---

# Preamble

Performing a packet walk can be the most important step to troubleshooting a network issue. This guide explains how to walk through the AWS Network topology. This post expects that you are familiar with AWS Networking terminology.

## What makes AWS Networking so different?

Traditional networking devices performing routing or switching use a vector based mechamisms and in this sense they route or switch the packet by choosing an egress interface which connects to an intermediary device. This means that traditionally in order to perform a packet walk you perform the following steps,

1. Log into the source device and determine if destination is same LAN or different.
2. If the destination is on a different LAN, then log into the Default Gateway device and issue a command to determine next-hop or interface
3. Log into the next intermediary device that represents that next-hop or other side of interface and issue a command to determine next-hop or interface.
4. Repeat until your next-hop is the your intended destination.

AWS Networking is different because it is intent based. Intent based indicates that you specify your end destination as the target rather then an intermediary hop. Let's review the key services.

## AWS Network Gateway Concepts

---

### Transitive Routing

Transitive Routingis a major concept in AWS Networking. Transitive Routing refers to using a target as an intermediate hop rather than directly routing to the target. For example if you are 'X' and go through 'Y' to reach 'Z' then 'Y' is a intermediate hop. Instead of routing 'X' -> 'Y' -> 'Z', if you are 'X' should go directly to 'Z'.

{% include elements/figure.html image="https://docs.aws.amazon.com/images/vpc/latest/peering/images/transitive-peering-diagram.png" caption="Incorrect Configuration: https://docs.aws.amazon.com/images/vpc/latest/peering/images/transitive-peering-diagram.png" %}

{% include elements/figure.html image="https://docs.aws.amazon.com/images/vpc/latest/peering/images/three-vpcs-peered.png" caption="Correct Configuration: https://docs.aws.amazon.com/images/vpc/latest/peering/images/three-vpcs-peered.png" %}

AWS enforces this limitation on their gateways by reviewing the source and destination IP address of the packet. The source or destination IP address is then used to determine if the packet is transitive or is being directly routed.

### Local Route

The Local Route is an intent based route that is limited to the same VPC that the route table exists in. By default your route tables will send all traffic for your VPC's CIDR to the Local Route.

{% include elements/figure.html image="/media/2022-11-22/local-gw-flow.svg" caption="Internet Gateway packet flow logic" %}

### NAT Gateway

The NAT Gateway target is an intent based route that performs a One to Many NAT which is comestimes referred to as Masquerading. 

No special rules are applied for sending traffic to this gateway.

### Network Interface

The Network Interface target is an vector based route that is limited to the same VPC that the route table exists in. It is important to note that routing to a network interface in a VPC requires that [source and destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#modify-source-dest-check) is turned off on the interface itself. This is a common gateway target when using a Virtual Network Appliance such as a Firewall or Software Defined thingamajig.

### Peering Connection

The Peering Connection target is an intent based route that enables you to reach another target VPC. For traffic sent to the Peering Connection the destination IP must exist as a Network Interface in the target VPC. A Peering Connection is not transitive, which is a commonly tested for topic during AWS's Certification Exams.

{% include elements/figure.html image="/media/2022-11-22/peer-conn-flow.svg" caption="Peering Connection packet flow logic" %}

### Virtual Private Gateway

The Virtual Private Gateway (VGW) is an intent and vector based route target that is commonly used to enable connectivity between a VPC and a Customer Gateway Device. It is intent based in the sense that you do not choose which VPN Endpoint the traffic is sent inbound to. When a packet is received the does not perform source or destination checking. It is a single container which hosts two VPN Endpoints with BGP as an optional feature. It is vector based in the sense that you can influence outbound direction to a specific next-hop or interface.

No special rules are applied for sending traffic to this gateway.

<!---
vgw vpn randomly chooses tunnel interface for egress

-->

### Transit Gateway

The Transit Gateway (TGW) is an intent based route target that is commonly used for generally connectivity use-cases including VPC to VPC, VPC to VPN, and VPC to Direct Connect. It is important to note that [TGW is a zonal service](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html#tgw-az-overview), meaning when creating the attachment you are supposed to enable all availability zones. An availability zone that are not enabled with an attachment cannot reach the transit gateway. When creating an attachment for each availability zone within a VPC, the TGW service creates Network Interfaces in the respective subnets.

{% include elements/figure.html image="/media/2022-11-22/tgw-flow.svg" caption="Transit Gateway packet flow logic" %}

### Internet Gateway

An Internet Gateway is an intent based route that is used to achieve access to the Public Internet. The Internet Gateway performs the same source check as previously mentioned with other gateways but will also check to see if either a Public IPv4 Address, an Elastic IPv4 Address or a GUA IPv6 Address is associated with the source resource. This Gateway also performs a One to One NAT, translating the Private IPv4 Address into the associated Public / Elastic IPv4.

{% include elements/figure.html image="/media/2022-11-22/inet-gw-flow.svg" caption="Internet Gateway packet flow logic" %}

### Egress Only Internet Gateway

The Egress only Internet Gateway is an intent based route used for VPCs which are using IPv6 rather than IPv4. IPv6 has several distinct groups of address types. IPv6 Globally Unique Unicast Addresses are by default routable on the internet. The purpose of the Egress Only Internet Gateway is to allow IPv6 GUA addresses to route to the internet but deny unsolicited IPv6 GUA address from reaching your resources.

{% include elements/figure.html image="/media/2022-11-22/eigw-flow.svg" caption="Internet Gateway packet flow logic" %}

### Gateway Load Balancer Endpoint

The Gateway Load Balancer Endpoint is an intent based route that is used to send traffic to a Virtual Network Appliance such as a Firewall or Software Defined thingamajig via a Load Balancer. This Load Balancer is a highly available target which will statefully send traffic to specific backend appliances for the duration of the entire flow.

No special rules are applied for sending traffic to this gateway.

---

# Thanks!

I hope to write more regarding performing a packet walk within Amazon Web Service's Networking services. It is truely vital in any troubleshooting scenario and a general internet search yields next to nothing, nor does any certificate or study material offer any insight on the method. 

This part 1 is filled with a significant amount of theory and matter of fact statements, which I hope to avoid in further posts. Coming next watch out for the scenario based posts which should really solidify this method in your mind!