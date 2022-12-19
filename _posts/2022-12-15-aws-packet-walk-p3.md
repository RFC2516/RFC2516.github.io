---
title: Part 3 -  Packet Walking in AWS
tags: [Networking, AWS]
style: fill
color: primary
description: This post explains how to walk through an VPC to VPC Connectivity scenario
---

{% include elements/button.html link="/blog/aws-packet-walk-p1.html" text="If you haven't read part 1 then click here!" style="warning" %}

# Preamble

In this post we are going to explore an AWS Networking Scenario in which an EC2 Instance in a VPC communicate with another EC2 Instance in a different VPC. Again you've been tagged as the individual responsible for troubleshooting this and you need to apply your Packet Walking skills to understand what, if anything, could be at fault. Below we will first analyize the forward path, stopping at any major decision points and asking ourselfs if the packet will be forwarded or discarded at any point. We will then do the same for the return path. We will use the theory from part 1 to help in this analysis.

# Architecture

<!---

<img of vpc to tgw to vpc architecture>

## Performing the Packet Walk

1. here is the packet walk formula
2. locate the EC2 server
3. locate it's private IP to update template
4. navigate to network interfaces link
5. locate ENI for EC2
6. update template

1. locate the subnet of the ENI
2. check the security group / network acl
3. check the route table, target is tgw
4. what availability zone are we in?
5. locate the transit gateway attachment ENI in the same availabilty zone
6. locate the network acl of the tgw attachment eni subnet
7. update template

talk about transit gateway az affinity

1. locate the tgw route table
2. what attachment / resource do we end up at?
3. locate the eni of the attachment in the same az (az affinity reminder)
4. update the template

5. locate the route table of the ENI
6. check the network acl of the ENI's subnet
7. check the ENI's route table, target is local
8. explain local
9. check network acl of subnet for target ec2
10. check security group of ENI for EC2
11. update the template



# Thanks

-->