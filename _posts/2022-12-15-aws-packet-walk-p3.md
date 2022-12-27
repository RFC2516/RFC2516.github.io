---
title: Packet Walking in AWS - Part 3
tags: [Networking, AWS]
style: fill
color: primary
description: This post explains how to walk through an VPC to VPC Connectivity scenario
---

{% include elements/button.html link="/blog/aws-packet-walk-p1.html" text="If you haven't read part 1 then click here!" style="warning" %}

# Preamble

In this post we are going to explore an AWS Networking Scenario in which an [EC2 Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html) in a [VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) needs to communicate with another EC2 Instance in a different VPC. You've been tagged as the individual responsible for troubleshooting this and you need to apply your Packet Walking skills to understand what, if anything, could be at fault. Below we will first analyize the forward path, stopping at any major decision points and asking ourselfs if the packet will be forwarded or discarded at any point. We will then do the same for the return path. We will use the theory from part 1 to help in this analysis.

# Architecture

To begin your packet walking you need to identify the source IP, destination IP, source port, destination port and protocol. Take some time to become familiar with the architecture. Consider right clicking and opening the image in a new tab to see it at maximum resolution.

Our architecture looks as such:

{% include elements/figure.html image="/media/2022-12-15/p3-architecture.svg" caption="Network Diagram" %}

While the scenario may not involve any actual problems, your goal while reading this post should be to practice your packet walking skill. This skill is absolutely necessary for anyone in charge of any networking configuration in AWS. You can think of this as doing your stretches before a big problem.

In this example we want to connect from HOST-A to HOST-B's HTTP Service. Let's start:

## Performing our Packet Walk

A packet walk involves tracing each hop of the network connectivity and taking note. To do this you could picture in your mind what is happening to the packet as well as the devices that exist at each hop. For now, while you're practicing I recommend writing notes similar to below.

> == Packet Path == <br/>
> Forward: ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

Let's start by locate the HOST-A's Instance ID in the console and clicking it.

{% include elements/figure.html image="/media/2022-12-15/locate-ec2-host-a.png" caption="HOST-A" %}

Click on HOST-A's Instance ID and then scroll down, open the "Networking Tab" and then open the click the ENI ID Link.

{% include elements/figure.html image="/media/2022-12-15/host-a-networking-page.png" caption="HOST-A's Networking Page" %}

This [ENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) represents the first hop for our packet. Let's click on the ENI ID and then open both the Subnet ID and the Security Group links on the next page. Additionally now that we know the first hop let's record it in our packet walking notes.

> == Packet Path == <br/>
> Forward: **eni-084d826e4b1e0e190** -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

On the next page we see,

{% include elements/figure.html image="/media/2022-12-15/host-a-eni-page.png" caption="HOST-A's ENI Page" %}

Open both the subnet and the security group. First check the security group for an Outbound Rule which allows this traffic. Since the destination of the traffic is "10.1.8.131" this matches the outbound rule for "0.0.0.0/0" for all protocols. Therefore the traffic is allowed outbound.

{% include elements/figure.html image="/media/2022-12-15/host-a-sg-outbound.png" caption="HOST-A's Security Group Outbound" %}

Check the subnet's route table to see where the next hop is.

{% include elements/figure.html image="/media/2022-12-15/host-a-subnet-route-table.png" caption="HOST-A's Route Table" %}

Let's look back at our packet walking notes.

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

The destination is 10.1.8.131 which matches the default route which is pointed at the [TGW](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html). However before being allowed to leave the subnet we have to open the Network ACL's outbound rules to check if this packet will be allowed out.

{% include elements/figure.html image="/media/2022-12-15/host-a-subnet-nacl-outbound.png" caption="HOST-A's Network ACL Outbound" %}

The destination 10.1.8.131 matches the rule for destination 0.0.0.0 with an action of Allow. Therefore the packet is allowed to go to the transit gateway as determined by the Route Table. The traffic is sent to the tgw via intent based routing. 

Let's check out TGW Logic from part 1:

{% include elements/figure.html image="/media/2022-11-22/tgw-flow.svg" caption="Transit Gateway packet flow logic" %}

Okay let's confirm if there is a TGW Attachment in the same availability zone. This behavior is a key component of Transit Gateway's behavior and is why it's considered a [zonal service](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html#tgw-az-overview).

You can the EC2 Console's Network Interface page to locate the Attachment. I've already named mine but you can use the Description and the Availability Zone columns to identify it.

{% include elements/figure.html image="/media/2022-12-15/all-network-interfaces.png" caption="Locating the Attachment in US-EAST-1B" %}

Once found click on the ENI ID to open it's page.

{% include elements/figure.html image="/media/2022-12-15/vpc-a-attach-1b-eni.png" caption="VPC-A's Attachment ENI" %}

Note that there is no security group. Next we should click on it's subnet ID so we may inspect it's Network ACL. There is no need to inspect it's route table as the Transit Gateway attachment will always send traffic incoming traffic to the Transit Gateway regardless of it's route table's configuration.

{% include elements/figure.html image="/media/2022-12-15/vpc-a-attach-1b-nacl-inbound.png" caption="VPC-A's Attachment Network ACL" %}

Since this traffic is incoming to this attachment subnet from HOST-A's subnet we should check the inbound rules. The source IP 10.0.9.132 matches the rule for source 0.0.0.0/0 with an action of allow. Therefore the inbound rule will permit the traffic and we can apply this to our packet walking notes. 

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> **eni-02632fc1186f1e023** -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

Check the transit gateway's route table to determine where this packet will be sent.

{% include elements/figure.html image="/media/2022-12-15/tgw-route-table.png" caption="Transit Gateway Route Table" %}

This route table indicates that it will sent packets destined for the 10.1.0.0/20 network to the attachment ID ending in "05bd" aka the Resource "VPC-B". Remember that [the transit gateway is a zonal service](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html#tgw-az-overview), it will always by default deliver packets to the same availability zone in which the packet enters from. Since our HOST-A exists in US-EAST-1B availability zone therefore we need to find the TGW Attachment in VPC-B's US-EAST-1B availability zone and open's it's subnet

You can the EC2 Console's Network Interface page to locate the Attachment. I've already named mine but you can use the Description and the Availability Zone columns to identify it.

{% include elements/figure.html image="/media/2022-12-15/all-network-interfaces.png" caption="Locating the Attachment in US-EAST-1B" %}

Click on the VPC-B's Attachment in the US-EAST-1B availability zone to open it's ENI Page.

{% include elements/figure.html image="/media/2022-12-15/vpc-b-attach-1b-eni.png" caption="VPC-B's Attachment in 1B." %}

Note that there is no security group to check. Next click on the subnet page check the route table. It is necessary to check the route table [because this is traffic coming from the transit gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-nacls.html) rather than traffic going to the transit gateway. Since traffic is not filtered by the Network ACL when coming from the TGW to an Attachment's ENI we can update our packet flow notes.

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> **eni-09cd3bddbd4bb7b1b** -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

{% include elements/figure.html image="/media/2022-12-15/vpc-b-attach-1b-route-table.png" caption="VPC-B's Attachment Route Table." %}

The route table indicates that this traffic is destined to "local" which is to HOST-B which is outside of this subnet. Let's check it's routing logic that we learned from part 1.

{% include elements/figure.html image="/media/2022-11-22/local-gw-flow.svg" caption="Local packet flow logic" %}

The IP address does exist in this VPC and the ENI is in-use by the HOST-B. Next we need to confirm the TGW Attachment's Network ACL outbound rules because the traffic is leaving the attachment subnet to go to HOST-B's subnet.

{% include elements/figure.html image="/media/2022-12-15/vpc-b-attach-1b-nacl-outbound.png" caption="VPC-B's Attachment Network ACL Outbound." %}

Let's compare our packet walking notes:

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> **eni-09cd3bddbd4bb7b1b** -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

Okay Network ACL allows this traffic, because "10.1.8.131" matches the rule with Destination 0.0.0.0/0 with an action of Allow. Next we need to confirm that HOST-B's Network ACL will allow this traffic inbound. 

{% include elements/figure.html image="/media/2022-12-15/host-b-subnet-nacl-inbound.png" caption="HOST-B's Network ACL Inbound" %}

The source of the traffic is "10.0.9.132" which matches the source 0.0.0.0/0 with an action of Allow. This means that the traffic is allowed to the ENI of HOST-B. We can locate it's ENI by navigating to the EC2 Console.

{% include elements/figure.html image="/media/2022-12-15/locate-ec2-host-b.png" caption="Locating HOST-B" %}

Click on the HOST-B's Instance ID and then scroll down, open the "Networking Tab" and then open the click the ENI ID Link.

{% include elements/figure.html image="/media/2022-12-15/host-b-networking-page.png" caption="HOST-B's Networking Page" %}

On the ENI page open both the Subnet ID and the Security Group links. We need to confirm that the security group's inbound rules will allow this traffic.

{% include elements/figure.html image="/media/2022-12-15/host-b-eni-page.png" caption="HOST-B's Network Interface" %}

Check the security group to ensure we have an Inbound Rule which allows this traffic.

{% include elements/figure.html image="/media/2022-12-15/host-b-security-group-inbound.png" caption="HOST-B's Security Group Inbound" %}

Since the security group has a rule which allows all traffic from all sources then the traffic is received on the OS and the HTTP Service can respond. Let's update our packet walking notes to reflect what we've confirmed.

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> **eni-0092f8e2d29917a4c** <br/>
> Return: <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.9.132<br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.1.8.131  <br/>
> Destination Port: 443/tcp <br/>

Since we reached our destination, this finishes our forward path. Next we will follow the return path. This is connectivity from HOST-B to HOST-A. We will start by checking HOST-B's subnet route table to see where the next hop is. This can be located by first locating Host-B in the EC2 Console.

{% include elements/figure.html image="/media/2022-12-15/locate-ec2-host-b.png" caption="HOST-B" %}

Next lets click on HOST-B's Instance ID and then scroll down to the networking tab/page.

{% include elements/figure.html image="/media/2022-12-15/host-b-networking-page.png" caption="HOST-B's Networking Page" %}

On this page we can click on the subnet ID to locate it's route table and network acl.

{% include elements/figure.html image="/media/2022-12-15/host-b-subnet-route-table.png" caption="HOST-B's Route Table" %}

Because our traffic is reversed on the return way we are looking for the route which matches "10.0.9.132".

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c <br/>
> Return: **eni-0092f8e2d29917a4c** -> ?<br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.1.8.131 <br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.0.9.132 <br/>
> Destination Port: 443/tcp <br/>

The only matching route is the default route to the transit gateway. Therefore the traffic is leaving this subnet and Network ACL's outbound rules needs to be checked.

{% include elements/figure.html image="/media/2022-12-15/host-b-subnet-nacl-outbound.png" caption="HOST-B's Network ACL Outbound" %}

The destination "10.0.9.132" matches the outbound rule for destination 0.0.0.0/0 with an action of Allow.

Next the traffic is returned to the tgw via intent based routing. The transit gateway is a zonal service it will always by default deliver packets to the same availability zone in which the packet enters from. Since our HOST-B exists in US-EAST-1A therefore we need to find the TGW Attachment in VPC-B’s US-EAST-1A availability zone and open’s it’s subnet. Locate the TGW ENI in the same availability zone as this instance. 

You can the EC2 Console’s Network Interface page to locate the Attachment. I’ve already named mine but you can use the Description and the Availability Zone columns to identify it.

{% include elements/figure.html image="/media/2022-12-15/all-network-interfaces.png" caption="Locating VPC-B's Attachment in US-EAST-1A" %}

Now let's click on it's ENI ID.

{% include elements/figure.html image="/media/2022-12-15/vpc-b-attach-1a-eni.png" caption="VPC-B's Attachment's ENI" %}

Note that there is no security group for us to inspect. Next let's open the subnet to inspect the Network ACL. 

{% include elements/figure.html image="/media/2022-12-15/vpc-b-attach-1a-nacl-inbound.png" caption="VPC-B's Attachment's Network ACL Inbound" %}

Let's check our packet walking notes:

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c <br/>
> Return: eni-0092f8e2d29917a4c -> **eni-05888d6f1ed0595bb** -> ?<br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.1.8.131 <br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.0.9.132 <br/>
> Destination Port: 443/tcp <br/>

The source "10.1.8.131" matches the source 0.0.0.0/0 rule with an action of Allow. Therefore the packet is received by the ENI and updated in our packet walking notes. The transit gateway is a zonal service it will always by default deliver packets to the same availability zone in which the packet enters from. Since our traffic entered the TGW via US-EAST-1A then it is received on VPC-A's attachment in US-EAST-1A.

Find the TGW Attachment in VPC-A's US-EAST-1A availability zone and open's it's subnet.

{% include elements/figure.html image="/media/2022-12-15/vpc-a-attach-1a-eni.png" caption="VPC-A's Attachment's ENI" %}

This is the ENI that receives our traffic from the TGW. Let's update our packet walking notes.

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c <br/>
> Return: eni-0092f8e2d29917a4c -> eni-05888d6f1ed0595bb -> eni-062a260949710dda8 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.1.8.131 <br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.0.9.132 <br/>
> Destination Port: 443/tcp <br/>

Now let's open the attachment's route table by navigating to the subnet ID.

{% include elements/figure.html image="/media/2022-12-15/vpc-a-attach-1a-route-table.png" caption="VPC-A's Attachment's Route Table" %}

The route table indicates that traffic to "10.1.8.131" should be sent to "Local" which means directly to HOST-A's ENI. Next we have to confirm if the attachment's Network ACL will allow this traffic outbound.

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c <br/>
> Return: eni-0092f8e2d29917a4c -> eni-05888d6f1ed0595bb -> eni-062a260949710dda8 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.1.8.131 <br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.0.9.132 <br/>
> Destination Port: 443/tcp <br/>

{% include elements/figure.html image="/media/2022-12-15/vpc-a-attach-1a-nacl-outbound.png" caption="VPC-A's Attachment Network ACL Outbound" %}

The destination is "10.0.9.132" which matches the "0.0.0.0/0" rule with an action of Allow. This means that the traffic is allowed to exit the subnet. Now we must confirm if the traffic is permitted inbound to the HOST-A's Subnet via it's Network ACL's inbound rules.

{% include elements/figure.html image="/media/2022-12-15/host-a-subnet-nacl-inbound.png" caption="HOST-A's Network ACL Inbound" %} 

The source IP of the traffic is "10.1.8.131" which matches the "0.0.0.0/0" rule with an action of Allow. This means that the traffic is allowed to enter the subnet. Now that we've confirmed that the traffic is allowed into HOST-A's Subnet then the traffic should be received on the operating system itself.

There is no need to check the security group because the security group is a stateful inspection service. Meaning the same "Outbound" rule allows the traffic to return. Let's update our Packet Walking notes:

> == Packet Path == <br/>
> Forward: eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c <br/>
> Return: eni-0092f8e2d29917a4c -> eni-05888d6f1ed0595bb -> eni-062a260949710dda8 -> eni-084d826e4b1e0e190 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.1.8.131 <br/>
> Source Port: Ephemeral  <br/>
> Destination: 10.0.9.132 <br/>
> Destination Port: 443/tcp <br/>

# Bonus Section

This is all very theory based and there must be a way to prove the accurancy of the information above. Well, this can all be confirmed via logging of course! VPC Flow Logs can be used to log traffic received at any ENI. However is very important to understand that the VPC Flow Log service is not guarenteed to deliver every log and additionally it is in no way ordered. Let's see this in action, I've confirmed VPC Flow Logging on both VPCs used to create this scenario and both are being logged to the same CloudWatch Log Group.

We can use the following Log Insights query to pull a specific TCP Flow from these logs and structure the output to show each ENI that received these flows.

> fields start as Time, srcAddr as Source, dstAddr as Destination, srcPort as SourcePort, dstPort as DestinationPort, interfaceId as ENI <br/>
> | filter SourcePort = '49999' and DestinationPort = '8000' or SourcePort = '8000' and DestinationPort = '49999' <br/>
> | sort by Time <br/>

After plugging the same into the Log Insights page let's see the output.

{% include elements/figure.html image="/media/2022-12-15/flow-logs-tcp-tuple.png" caption="Log Insights query on VPC Flow Logs" %} 

Let's check out the first column which is labelled as "Time". Each row contains a value which represents time in Unix Seconds. The VPC Flow Logs documentation indicates that "This might be up to 60 seconds after the packet was transmitted or received on the network interface." So if we copy these out:

* 1671497064 = 18:44:24 <br/>
* 1671497072 = 18:44:32 <br/>
* 1671497072 = 18:44:32 <br/>
* 1671497078 = 18:44:38 <br/>
* 1671497083 = 18:44:43 <br/>
* 1671497088 = 18:44:48 <br/>
* 1671497092 = 18:44:52 <br/>
* 1671497092 = 18:44:52 <br/>

You can use any tool online just google for "[unix seconds converter](https://www.google.com/search?q=unix+seconds+converter&sxsrf=ALiCzsZzEcKB5o2rS6qU_z5MVS_pWcNLgg%3A1672152745414&source=hp&ei=qQarY52oFJWoqtsP2a6iiAY&iflsig=AJiK0e8AAAAAY6sUuWYq48Koo5I9WBhtrOP0G9UdXSDo&ved=0ahUKEwjdoJWzhpr8AhUVlGoFHVmXCGEQ4dUDCAo&uact=5&oq=unix+seconds+converter&gs_lcp=Cgdnd3Mtd2l6EAMyBQgAEIAEMgYIABAWEB4yBggAEBYQHjIGCAAQFhAeMgYIABAWEB4yBQgAEIYDMgUIABCGAzoRCC4QgwEQxwEQsQMQ0QMQgAQ6CwgAEIAEELEDEIMBOhEILhCABBCxAxCDARDHARDRAzoICAAQgAQQsQM6BQguEIAEOg4ILhCABBCxAxDHARDRAzoICC4QgAQQ1AI6CAguEIAEELEDOggILhDUAhCABDoNCAAQFhAeEA8Q8QQQCjoICAAQFhAeEA9QAFiEFGDRFGgAcAB4AIABSogB7QiSAQIyMpgBAKABAQ&sclient=gws-wiz)" to convert these to human-readable format. Just like I did above. This proves that the [VPC Flow Logs are not real-time](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#:~:text=Flow%20logs%20do%20not%20capture%20real%2Dtime). Next let's copy the order of the ENI column out:

* eni-02632fc1186f1e023 = VPC-A-ATTACH-1b <br/>
* eni-0092f8e2d29917a4c = HOST-B  <br/>
* eni-0092f8e2d29917a4c = HOST-B <br/>
* eni-09cd3bddbd4bb7b1b = VPC-B-ATTACH-1b
* eni-05888d6f1ed0595bb = VPC-B-ATTACH-1a <br/>
* eni-062a260949710dda8 = VPC-A-ATTACH-1a <br/>
* eni-084d826e4b1e0e190 = HOST-A <br/>
* eni-084d826e4b1e0e190 = HOST-A <br/>

The order of these ENIs makes no sense from a routing perspective, but can be used to confirm that the packet was indeed received on that ENI. So let's confirm our Forward and Return path with the information above.

**Forward:** eni-084d826e4b1e0e190 -> eni-02632fc1186f1e023 -> eni-09cd3bddbd4bb7b1b -> eni-0092f8e2d29917a4c<br/>

**Return:** eni-0092f8e2d29917a4c -> eni-05888d6f1ed0595bb -> eni-062a260949710dda8 -> eni-084d826e4b1e0e190<br/>

We can see that each occurance exactly matches and only the order is not preserved. Here is a diagram that illustrates our path.

{% include elements/figure.html image="/media/2022-12-15/p3-packet-flow-diagram.svg" caption="Packet Flow Diagram" %} 

# Thanks

I hope you felt that the guided walk-through of this packet walk is useful for you. As you can see the packet walk has a lot of steps but as long as you put one step in front of the other you’ll be fine. Hopefully you'll be able to apply this one day during a troubleshooting scenario and it will save you a lot of time.