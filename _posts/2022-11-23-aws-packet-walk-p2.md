---
title: Part 2 -  Packet Walking in AWS
tags: [Networking, AWS]
style: fill
color: primary
description: This post explains how to walk through an Internet Connectivity scenario.
---

{% include elements/button.html link="/blog/aws-packet-walk-p1.html" text="If you haven't read part 1 then click here!" style="warning" %}


# Preamble

In this post we are going to explore an AWS Networking Scenario in which an EC2 Instance in a VPC needs Internet Connectivity. You've been tagged as the individual responsible for troubleshooting this and you need to apply your Packet Walking skills to understand what, if anything, could be at fault. Below we will first analyize the forward path, stopping at any major decision points and asking ourselfs if the packet will be forwarded or discarded at any point. We will then do the same for the return path. We will use the theory from part 1 to help in this analysis.

## Architecture

As always you want to start with identifying the source and destination IP, Port. For our example we want the EC2 Instance "Red" to connect to google's DNS. Therefore our source and destination IP, Port will be as such:

> Source IP: 10.0.142.154 <br/>
> Source Port: 47129 (Ephemeral port) <br/>
> Destination IP: 8.8.8.8 <br/>
> Destination Port: 53 udp/tcp <br/>

Our network diagram looks as such:

{% include elements/figure.html image="/media/2022-11-28/p2-architecture.svg" caption="Network Diagram" %}

While this scenario may feel boringly simple, it's less about what we're connecting to and more about performing the packet walk. The same packet walk process can be applied for significantly more complex networking scenarios. We're just introducing the process for now.

## Performing our Packet Walk

A packet walk involves tracing each hop of the network connectivity and taking note. It also includes picturing in your mind what is happening to the packet at each step. To do this I frequently write notes similar to below. Now to get started navigate to the "Red" instance and click on it's ENI ID and note it the same as below.

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.142.154 <br/>
> Source Port: 47129 (Ephemeral port) <br/>
> Destination: 8.8.8.8 <br/>
> Destination Port: 53/udp <br/>

{% include elements/figure.html image="/media/2022-11-28/p2-red-eni-id.png" caption="Red's ENI" %}

After clicking the ENI we are sent to the Network Interface page. We want to click on the security group ID.

{% include elements/figure.html image="/media/2022-11-28/p2-red-eni-page-sg.png" caption="Red's SG" %}

Since this is an outbound flow, we want to review the "Outbound rules" of the security group. Reviewing our packet flow notes, we are using the UDP protocol, on port 53 to the destination 8.8.8.8. In this case the traffic is allowed because we have an entry for all ports, protocols and destinations.

{% include elements/figure.html image="/media/2022-11-28/p2-red-sg.png" caption="Outbound rules" %}

Navigate back to the ENI. Click to open the Subnet ID.

{% include elements/figure.html image="/media/2022-11-28/p2-red-eni-page-sub.png" caption="Red's Subnet ID" %}

After clicking the Subnet ID we are sent to the subnet page. We want to click on the Route Table tab.

{% include elements/figure.html image="/media/2022-11-28/p2-red-subnet-rt.png" caption="Red's Route Table" %}

This route table indicates that the next hop is the NAT Gateway, which should always exist in a different subnet. However since this is in a different subnet we must next check the Network ACL first. As it is still possible that the Red's Network ACL can block the outgoing traffic.

Navigate back to the subnet and click on the Network ACL tab. Check the outbound rules and confirm if the traffic is allowed. In this case it is allowed because all protocols, port and destinations have an allowed entry.

{% include elements/figure.html image="/media/2022-11-28/p2-red-subnet-nacl.png" caption="Red's Network ACL" %}

Now go back to the Route Table and click on the NAT Gateway. Remember that the NAT Gateway ID as shown in the route table is a intent based route. It does not show us the next-hop but rather where we intend to send this packet. Therefore we need to look at the NAT Gateway page where we want to locate the NAT Gateway's ENI.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-eni-id.png" caption="NAT's ENI ID" %}

This NAT Gateway is in another subnet and therefore the Network ACL must permit the traffic inbound. Checking our NAT's Network ACL we can confirm that the traffic is allowed.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-nacl-return.png" caption="NAT's Network ACL" %}

Notice on the ENI Page for the NAT Gateway that there is no security group, therefore we can skip that check. Also note how "Source/dest. check" is "False." This is a default of the NAT Gateway service and if it were changed to "True" then traffic would be discarded. As the "Source/dest. check" option reviews each packet header to ensure either the source or destination IP matches the IP assigned to the ENI. Next we want to update our packet walking flow notes. Two things have happened.

1. Our packet was received on the ENI of the NAT Gateway.
2. The NAT Gateway performed Masquerading which changes our source IP and Port.

So we must update our forward path and our Packet Headers notes. Our source IP becomes the IP assigned to the NAT Gateway's ENI and our source port is randomized. 

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> eni-0bfbff2e9481ce666 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.8.143 <--- NAT's IP <br/>
> Source Port: 5259 (Ephemeral port) <--- NAT's Port <br/>
> Destination: 8.8.8.8 <br/>
> Destination Port: 53/udp (well-known DNS port)<br/>

{% include elements/figure.html image="/media/2022-11-28/p2-nat-eni-page.png" caption="NAT's ENI ID" %}

Next click on the Subnet ID for the NAT Gateway.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-eni-page-sub.png" caption="NAT's Subnet ID" %}

After clicking the Subnet ID we are sent to the subnet page. We want to click on the Route Table tab.

This route table indicates that the next hop is the Internet Gateway which is leaving the VPC all together. This is a intent based route as the actual resources that forward our Internet Traffic are managed by AWS and not visible to us. However since this is leaving the subnet we must next check the Network ACL.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-rt.png" caption="NAT's Route Table" %}

Navigate back to the subnet and click on the Network ACL tab. Check the outbound rules and confirm if the traffic is allowed. In this case it is allowed because all protocols, port and destinations have an allowed entry.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-nacl.png" caption="NAT's Network ACL" %}

Now go back to the Route Table and note the Internet Gateway ID. Now we want to update our packet walking flow notes. 

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> eni-0bfbff2e9481ce666 -> igw-0a64529194be520d2 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 10.0.8.143 <--- NAT's IP <br/>
> Source Port: 5259 (Ephemeral port) <--- NAT's Port <br/>
> Destination: 8.8.8.8 <br/>
> Destination Port: 53/udp (well-known DNS port)<br/>

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-rt.png" caption="NAT's Route Table" %}

Remember the Internet Gateway logic as discussed in part 1? The checks that are performed at the Internet Gateway including making sure that an Elastic or Public IP address is assigned to the resource. In this case we do have an Elastic IP assigned as one is automatically created when you create the NAT Gateway. 

{% include elements/figure.html image="/media/2022-11-22/inet-gw-flow.svg" caption="Internet Gateway Logic" %}

This check can be confirmed by navigating to the Elastic IP Page. Here you will see your Elastic IP to Private IP association.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-elastic-ip.png" caption="NAT's Route Table" %}

Now we must walk the return route. First the Internet Gateway undoes the 1:1 NAT. Then the Internet Gateway uses an implicit route table that returns traffic *Locally*, meaning directly to the ENI that owns the IP in the destination header. Our Packet Headers are now reversed as the flow of the packet is now from the internet resource to our EC2 resource.

This makes our packet walking notes as such:

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> eni-0bfbff2e9481ce666 -> igw-0a64529194be520d2 <br/>
> Return: igw-0a64529194be520d2 -> ? <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 8.8.8.8 <br/>
> Source Port:  53/udp (well-known DNS port)<br/>
> Destination: 10.0.8.143 <br/>
> Destination Port: 5259 (Ephemeral port)<br/>

We added the ENI of the NAT Gateway to our packet walking return path notes as it is the resource that owns the "10.0.8.143" IP address. Let's navigate to the EC2 Console and then click on the Network Interfaces link and then locate the ENI's Subnet.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-eni-page.png" caption="NAT's ENI ID" %}

Now let's go back and check the Network ACL for the NAT Gateway's Subnet. Since this is incoming from the Internet to this subnet we have to check the Network ACL of the NAT Gateway's subnet.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-nacl-return.png" caption="NAT's ENI ID" %}

Now that the traffic is allowed into the subnet, it is recieved on the ENI of the NAT Gateway. The NAT Gateway undoes the masquerading NAT and then tries to send the data. Now we must update our packet walking notes.

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> eni-0bfbff2e9481ce666 -> igw-0a64529194be520d2 <br/>
> Return: igw-0a64529194be520d2 -> eni-0bfbff2e9481ce666 -> ?<br/>
> <br/>
> == Packet Headers == <br/>
> Source: 8.8.8.8 <br/>
> Source Port:  53/udp (well-known DNS port)<br/>
> Destination: 10.0.142.154 <br/>
> Destination Port: 47129 (Ephemeral port)<br/>

 However before being allowed to send it is inspect by the Network ACL because it is leaving the subnet in order to get there.

{% include elements/figure.html image="/media/2022-11-28/p2-nat-sub-nacl.png" caption="NAT's Network ACL" %}

Now let's navigate to the ENI for our Red Instance and then open the subnet link.

{% include elements/figure.html image="/media/2022-11-28/p2-red-eni-page-sub.png" caption="Red's Subnet ID" %}

We need to check the Network ACL to ensure that the traffic is allowed back to our instance. In this case it is allowed.

{% include elements/figure.html image="/media/2022-11-28/p2-red-sg-inbound.png" caption="Red's Subnet ID" %}

Return traffic to the instance is automatically allowed back via the security group because the security group is stateful. This means that the same oubound rule allows the return traffic and thus our finished Packet Walking notes are as such:

> == Packet Path ==<br/>
> Forward: eni-0a47fb1a0de444f15 -> eni-0bfbff2e9481ce666 -> igw-0a64529194be520d2 <br/>
> Return: igw-0a64529194be520d2 -> eni-0bfbff2e9481ce666 -> eni-0a47fb1a0de444f15 <br/>
> <br/>
> == Packet Headers == <br/>
> Source: 8.8.8.8 <br/>
> Source Port:  53/udp (well-known DNS port)<br/>
> Destination: 10.0.142.154 <br/>
> Destination Port: 47129 (Ephemeral port)<br/>

We now have confidence in walking the network to determine the path of our packets as well as taking time to apply the theory we learned in part 1 to ensure that packets should be passed and not discarded due to some logical issue. With all of this being confirmed we can now confidently say that there is no network concern!

# Thanks!

I hope you felt that the guided walk-through of this packet walk is useful for you. As you can see the packet walk has a lot of steps but as long as you put one step in front of the other you'll be fine! This was a relatively simple scenario. I hope to increase the complexity as I work on more posts.