---
title: Packet Walking in AWS - Part 3
tags: [Networking, AWS]
style: fill
color: primary
description: This post explains how to walk through an VPC to VPC Connectivity scenario
---

{% include elements/button.html link="/blog/aws-packet-walk-p1.html" text="If you haven't read part 1 then click here!" style="warning" %}

# Preamble

In this post we are going to explore an AWS Networking Scenario in which an EC2 Instance in a VPC communicate with another EC2 Instance in a different VPC. Again you've been tagged as the individual responsible for troubleshooting this and you need to apply your Packet Walking skills to understand what, if anything, could be at fault. Below we will first analyize the forward path, stopping at any major decision points and asking ourselfs if the packet will be forwarded or discarded at any point. We will then do the same for the return path. We will use the theory from part 1 to help in this analysis.

# Architecture

To begin your packet walking you need to identify the source IP, destination IP, source port, destination port and protocol. In this example we want to connect from HOST-A to HOST-B with just an ICMP Ping.

> Source IP: 10.0.142.154 <br/>
> Source Port: ICMP <br/>
> Destination IP: 8.8.8.8 <br/>
> Destination Port: ICMP <br/>

Our diagram looks as such:

<!---

<img of vpc to tgw to vpc architecture>

--->

While the scenario may not involve any actual problems, the goal of this post is to practice the packet walking skill. This skill is absolutely necessary for anyone in charge of any networking configuration in AWS. You can think of this as doing your stretches before a big problem.

## Performing our Packet Walk

A packet walk involves tracing each hop of the network connectivity and taking note. To do this you could picture in your mind what is happening to the packet as well as the devices that exist at each hop. For now, while you're practicing I recommend writing notes similar to below.

> == Packet Path ==
> Forward: ?
>
> == Packet Headers ==
> Source:
> Source Port: 
> Destination: 
> Destination Port:

Let's start by locate the HOST-A EC2 Instance in the console.

<locate-ec2-host-a>

Click on the EC2 Instance ID and then scroll down, open the "Networking Tab" and then open the click the ENI ID Link.

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B)

<host-a-networking-page>

On the ENI page open both the Subnet ID and the Security Group links.

<host-a-eni-page>

Check the security group to ensure we have an Outbound Rule which allows this traffic.

<host-a-security-group-outbound>

Check the subnet's route table to see where the next hop is.

<host-a-subnet-route-table>

Since this is leaving the subnet, check the Network ACL's outbound rules

<host-a-subnet-nacl-outbound>

Since this is permitted, the traffic is sent to the tgw via intent based routing. Locate the TGW ENI in the same availability zone. 

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B)

<VPC-A-ATTACH-1B-ENI>

Note no security group and open it's subnet.

<vpc-a-attach-1b-subnet>

Check the network ACL's Inbound rules. No need to to check route table.

<vpc-a-attach-1b-nacl-inbound>

Find the TGW Attachment in VPC-B's 1b AZ and open's it's subnet

<note-that-the-traffic-crosses-the-tgw>

<vpc-b-attach-1b-eni>

On the subnet page check the route table.

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B)

<vpc-b-attach-1b-route-table>

this traffic is destined to "local" which is to an ec2 server outside of this subnet. Confirm the network acl outbound rules.

<vpc-b-attach-1b-nacl-outbound>

Okay nacl rules allows this traffic. Locate the ENI of the destination EC2 server

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)

<locate-ec2-host-b>

Click on the EC2 Instance ID and then scroll down, open the "Networking Tab" and then open the click the ENI ID Link.

<host-b-networking-page>

On the ENI page open both the Subnet ID and the Security Group links.

<host-b-eni-page>

Check the security group to ensure we have an Inbound Rule which allows this traffic.

<host-b-security-group-inbound>

Check the subnet's route table to see where the next hop is.

<host-b-subnet-route-table>

Since this is leaving the subnet, check the Network ACL's outbound rules

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)

<host-b-subnet-nacl-outbound>

Since this is permitted, the traffic is returned to the tgw via intent based routing. Locate the TGW ENI in the same availability zone as this instance. 

<vpc-b-attach-1a-eni>

Find the TGW Attachment in VPC-B's 1b AZ and open's it's subnet

<vpc-b-attach-1a-subnet>

Check the network ACL's Inbound rules. No need to to check route table.

<vpc-b-attach-1a-nacl-inbound>

Find the TGW Attachment in VPC-A's 1a AZ and open's it's subnet

<vpc-b-attach-1a-eni-page>

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A) -> eni-05888d6f1ed0595bb (VPC-B-ATTACH-1A)

<note-that-the-traffic-crosses-the-tgw>

Locate the ENI for the TGW Attachment in VPC-A for the availability zone 1a.

<vpc-a-attach-1a-eni>

open the vpc-a-attachment's subnet and network acl

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A) -> eni-05888d6f1ed0595bb (VPC-B-ATTACH-1A) -> TGW -> eni-062a260949710dda8 (VPC-A-ATTACH-1A)

<vpc-a-attach-1a-nacl-outbound>

locate the ENI of the destination EC2 server

<vpc-a-attach-1a-route-table>

<locate-ec2-host-a>

Click on the EC2 Instance ID and then scroll down, open the "Networking Tab" and then open the click the ENI ID Link.

<host-a-networking-page>

On the ENI page open both the Subnet ID and the Security Group links.

<host-a-eni-page>

Host-A's security group should allow the traffic as part of it's stateful inspection. Meaning the same "Outbound" rule allows the traffic to return.

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A) -> eni-05888d6f1ed0595bb (VPC-B-ATTACH-1A) -> TGW -> eni-062a260949710dda8 (VPC-A-ATTACH-1A) -> eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B)





Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A) -> eni-05888d6f1ed0595bb (VPC-B-ATTACH-1A) -> TGW -> eni-062a260949710dda8 (VPC-A-ATTACH-1A) -> eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B)

<!---

HTTP Server = HOST-B = i-0705e0c6a791ed085

logs filter path taken: logs lost

fields start as Time, srcAddr as Source, dstAddr as Destination, srcPort as SourcePort, dstPort as DestinationPort, interfaceId as ENI
| filter SourcePort = '49999' and DestinationPort = '8000' or SourcePort = '8000' and DestinationPort = '49999'
| sort by Time

eni-02632fc1186f1e023 = VPC-A-ATTACH-1b
eni-0092f8e2d29917a4c = HOST-B 
eni-0092f8e2d29917a4c = HOST-B
eni-062a260949710dda8 = VPC-A-ATTACH-1a
eni-084d826e4b1e0e190 = HOST-A
eni-084d826e4b1e0e190 = HOST-A
eni-05888d6f1ed0595bb = VPC-B-ATTACH-1a




HTTP Server = HOST-B = i-0705e0c6a791ed085

Forward: eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B) -> eni-02632fc1186f1e023 (VPC-A-ATTACH-1B) -> TGW -> eni-09cd3bddbd4bb7b1b (VPC-B-ATTACH-1B) -> eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A)
Return: eni-0092f8e2d29917a4c (HOST-B-VPC-B-1A) -> eni-05888d6f1ed0595bb (VPC-B-ATTACH-1A) -> TGW -> eni-062a260949710dda8 (VPC-A-ATTACH-1A) -> eni-084d826e4b1e0e190 (HOST-A-VPC-A-1B)

What's happening?

The TGW has an AZ Affinity so.

fields start as Time, srcAddr as Source, dstAddr as Destination, srcPort as SourcePort, dstPort as DestinationPort, interfaceId as ENI
| filter SourcePort = '49997' and DestinationPort = '8000' or SourcePort = '8000' and DestinationPort = '49997'
| sort by Time







1. here is the packet walk template
2. locate the EC2-A server
3. locate it's private IP to update template
4. navigate to network interfaces link
5. locate subnet and security group links.
6. update template with the name of the ENI-A
7. check the security group / network acl (outgoing)
8. check the route table, target is tgw
9. what availability zone are we in?
10. locate the transit gateway attachment ENI-A in the same availabilty zone
11. locate the network acl (incoming & outgoing) of the vpc-a tgw attachment eni subnet
12. update template

talk about transit gateway az affinity and it's intent based routing behavior

1. locate the tgw route table based on what the source resource is using.
2. what attachment / resource do we end up at?
3. locate the eni of the vpc-b attachment in the same az (az affinity reminder)
4. check the security group / networkacl (incoming & outgoing)
5 update the template

1. locate the route table of the ENI
2. check the ENI's route table, target is local
3. explain local
4. check network acl (incoming) of subnet for target ec2
5. check security group of ENI for EC2-B
6. update the template
7. This completes our forward path

--

starting our return path

1. locate the ec2 server
2. locate it's eni
3. check it's route table
4. check security group and network acl (outgoing)
5. update template.

1. locate the ENI of the TGW Attachment in the same AZ
2. check the route table for the ENI of the TGW Attachment in the same AZ 
3. Check the security group / network acl (incoming & outgoing)

1. Review the TGW's RT
2. Locate the ENI of the TGW Attachment in the same AZ
3. Review Network ACL (outgoing)

1. Locate the ENI of the source EC2
2. Review the network acl (incoming)

# Thanks

-->