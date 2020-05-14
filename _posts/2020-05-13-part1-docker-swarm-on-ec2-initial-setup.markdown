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

## Launch an EC2 instance
With the prerequisites out of the way, we can go a head and launch our first instance. In the AWS Management Console go to Services > Compute > EC2 and spank that Launch button.

### Choose an Amazon Machine Image (AMI)
First, you will have to select an AMI to run on your instance. I opted for Amazon Linux 2 AMI (HVM) 64-bit (x86) for no particular reason. It's free tier eligible, if you still can make use of that, and is built by Amazon to have optimal performance on EC2. You can use another AMI that suits you. Just make sure to adapt things like package management that I will be using in this post to the OS of your choice.

### Choose an Instance Type
Next up, you're prompted to select an instance type. Now this is where you can either go all out, or start reasonably small. I was planning to use the Docker Swarm cluster for small services that I want to test in a production environment. I do not expect any users, nor do I run computational heavy stuff on it. You can however, that is up to you and your budget. A `t2.small` is roughly 18,00 Euro a month including storage and traffic if it's running 24/7 on eu-west-1. If you registered an account with AWS because you're on a learning path, then you will most likely still be eligible for the free tier, which means that `t2.micro` instance is free of charge for a year. A `t2.micro`, for the purpose of this tutorial, is all we need.

### Configure your Instance

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-config.png" alt="aws ec2 configuration">
</p>

There is not much to change here. If you're just starting out with your AWS account the Network will be prepopulated with your VPC already. The subnet can also stay on default. We want to, however _enable_ *Auto-assign Public IP*. If you plan to create resources on AWS with this particular EC2 instance, e.g. create an Amazon S3 Bucket, you will need to attach an IAM role to your instance for the proper permissions set up. In our case, we will not create any other resources WITH the EC2 Instance, so we leave that blank. *Stop - Hibernate behaviour* is interesting, if you plan on often shutting an instance down if you don't use it, and bring it back up when you use it. In this case on *Stop* all your instances Files and RAM will be dumped to an attached file system and you can use this if you bring it up another time to have everything where you left it off. Keep in mind however, that you will still pay for the attached storage 24/7. Monitoring is also pretty good with CloudWatch. It does come with a cost however, and since we will utilize Traefik and Portainer, we will have our own monitoring in place. Leave the rest as seen in the Screenshot and continue with *Next: Add Storage*.

### Add Storage
Pretty much just leave as is, unless you need more storage or have other requirements that you think will not be fulfilled with the default settings. Continue

### Add Tags
This will be the first of 