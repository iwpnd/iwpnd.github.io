---
layout: post
title: Docker Swarm on AWS EC2 Part1&#58; Initial provisioning and setup
tags: [docker, swarm, aws ec2]
categories: docker swarm aws ec2
date: 2020-05-13 13:37:00 +0200
toc: true
---

If you work with data, chances are, you came across or worked with a compute cluster of some sorts. There is not a conference, meetup, github repository or whatnot that doesn't mention Docker containers and/or their orchestration. If you've read some of my blog posts you know that I too, try to develop in and with Docker containers. But I have never actually deployed an application to a compute cluster other than a locally setup Docker Swarm. 
Let's change that, shall we?

## What will we learn in this series?
In this series we're going to find answers to some questions like:

How do I provision and setup an AWS EC2 instance?
How do I setup a cluster in the cloud? 
How do I route my applications? 
How do I manage my applications 
How do I even do CI/CD with a Docker Swarm?

The first part of this series will focus the initial steps of the setup. Like how to setup a Virtual Private Cloud (VPC) and attach security groups. I will also show how to launch an AWS EC2 instance and do the necessary steps to set up a Docker Swarm on it.

In the second part of the series I will show how to set up a [Traefik](https://containo.us/traefik/) Proxy with HTTPS that will handle incoming connections, expose your services and applications based on their domain name, manage multiple domains, handle and acquire HTTPS certificates with [Let's Encrypt](https://letsencrypt.org/). I will also show how to setup [Portainer](https://www.portainer.io/)) as the first service.

Part three will focus on how to deploy services to your Docker Swarm in a CI/CD pipeline using [GitHub](https://github.com) and [GitHub Actions](https://github.com/features/actions).

## What will I not cover in this series?
I love automation. However what I will not be doing, is showing you how to set up this cluster using Terraform or AWS Cloudformation (yet). I feel doing all of this from scratch, even if you only do it once, will go a long way when it comes to understanding it.

## Prerequisites
You will need to have access to the AWS management console. Your user will also need the permission to create resources within AWS e.g. a VPC, security groups, EC2 instances. For the purpose of this exercise I will be using MacOS. Windows users will have to read up on how to setup and ssh into a remote server, e.g. via [PuTTY](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html). 

### Virtual Private Cloud
As Elastic Compute Resources (EC2 Instances) can only be launched inside a [Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/), we will need a VPC. Accounts older than 04.12.2013 will have to create a [Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/), as described [here](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html#create-default-vpc). Newer accounts already have a default VPC setup that we can launch the EC2 instance for our Docker Swarm into.

### Security Group
An AWS security group acts as a virtual firewall for your AWS resources inside of your VPC. As such it it highly recommended to read up on some best practises for setting up security groups and how to manage each layer of your infrastructure properly using security groups (e.g. [best practices](https://www.stratoscale.com/blog/compute/aws-security-groups-5-best-practices/), [AWS intro on security groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)). For the purpose of this demo, we need to setup a security group that is open to inbound traffic from a couple of ports and protocols. Therefore, in the Management Console go to Services > search for VPC > under Security go to Security Groups and hit Create Security Group. Give it a significant name and description add inbound rules for:

| Protocol | Port | Source |
|-------|--------|---------|
| TCP | 2377 | 0.0.0.0/0 |
| TCP | 7946 | 0.0.0.0/0 |
| TCP | 8501 | 0.0.0.0/0 |
| UDP | 4789 | 0.0.0.0/0 |
| UDP | 7946 | 0.0.0.0/0 |
| SSH | 7946 | 0.0.0.0/0 |
| HTTPS | 443 | 0.0.0.0/0 |
| HTTP | 80 | 0.0.0.0/0 |

