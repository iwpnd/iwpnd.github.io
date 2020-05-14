---
layout: post
title: Docker Swarm on AWS EC2 Part 1&#58; Initial provisioning and setup
tags: [docker, swarm, aws ec2]
categories: docker swarm aws ec2
date: 2020-05-13 13:37:00 +0200
toc: true
---

If you work with data, chances are, you came across or worked with a compute cluster of some sorts. There is not a conference, meetup, GitHub repository, or whatnot that doesn't mention Docker containers and/or their orchestration. If you've read some of my blog posts you know that I too, try to develop in and with Docker containers. But I have never actually deployed an application to a compute cluster other than a local setup Docker Swarm. 
Let's change that, shall we?

## What will we learn in this series?
In this series, we're going to find answers to some questions like:

How do I provision and set up an AWS EC2 instance?
How do I set up a cluster in the cloud? 
How do I route my applications? 
How do I manage my applications 
How do I even do CI/CD with a Docker Swarm?

The first part of this series will focus on the initial steps of the setup. Like how to set up a Virtual Private Cloud (VPC) and attach security groups. I will also show how to launch an AWS EC2 instance and do the necessary steps to set up a Docker Swarm on it.

In the second part of the series I will show how to set up a [Traefik](https://containo.us/traefik/) Proxy with HTTPS that will handle incoming connections, expose your services and applications based on their domain name, manage multiple domains, handle and acquire HTTPS certificates with [Let's Encrypt](https://letsencrypt.org/). I will also show how to set up [Portainer](https://www.portainer.io/)) as the first service.

Part three will focus on how to deploy services to your Docker Swarm in a CI/CD pipeline using [GitHub](https://github.com) and [GitHub Actions](https://github.com/features/actions).

## What will I not cover in this series?
I love automation. However, what I will not be doing, is showing you how to set up this cluster using Terraform or AWS Cloudformation (yet). I feel doing all of this from scratch, even if you only do it once, will go a long way when it comes to understanding it.

## Prerequisites
You will need to have access to the AWS management console. Your user will also need permission to create resources within AWS e.g. a VPC, security groups, EC2 instances. For this exercise, I will be using macOS. Windows users will have to read up on how to set up and ssh into a remote server, e.g. via [PuTTY](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html). 

### Virtual Private Cloud
As Elastic Compute Resources (EC2 Instances) can only be launched inside a [Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/), we will need a VPC. Accounts older than 04.12.2013 will have to create a [Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/), as described [here](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html#create-default-vpc). Newer accounts already have a default VPC setup that we can launch the EC2 instance for our Docker Swarm into.

## Launch an EC2 instance
With the prerequisites out of the way, we can go ahead and launch our first instance. In the AWS Management Console go to Services > Compute > EC2 and spank that Launch button.

### Choose an Amazon Machine Image (AMI)
First, you will have to select an AMI to run on your instance. I opted for Amazon Linux 2 AMI (HVM) 64-bit (x86) for no particular reason. It's free tier eligible, if you still can make use of that, and is built by Amazon to have optimal performance on EC2. You can use another AMI that suits you. Just make sure to adapt things like package management that I will be using in this post to the OS of your choice.

### Choose an Instance Type
Next up, you're prompted to select an instance type. Now, this is where you can either go all out or start reasonably small. I was planning to use the Docker Swarm cluster for small services that I want to test in a production environment. I do not expect any users, nor do I run computational heavy stuff on it. You can, however, that is up to you and your budget. A `t2.small` is roughly 18,00 Euro a month including storage and traffic if it's running 24/7 on eu-west-1. If you registered an account with AWS because you're on a learning path, then you will most likely still be eligible for the free tier, which means that `t2.micro` instance is free of charge for a year. A `t2.micro`, for this tutorial, is all we need.

### Configure your Instance

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-config.png" alt="aws ec2 configuration">
</p>

There is not much to change here. If you're just starting with your AWS account the Network will be prepopulated with your VPC already. The subnet can also stay on default. We want to, however, **enable** **Auto-assign Public IP**. If you plan to create resources on AWS with this particular EC2 instance, e.g. create an Amazon S3 Bucket, you will need to attach an IAM role to your instance for the proper permissions set up. In our case, we will not create any other resources WITH the EC2 Instance, so we leave that blank. **Stop - Hibernate behavior** is interesting, if you plan on often shutting an instance down if you don't use it, and bring it back up when you use it. In this case, on **Stop** all your instances Files and RAM will be dumped to an attached file system and you can use this if you bring it up another time to have everything where you left it off. Keep in mind, however, that you will still pay for the attached storage 24/7. Monitoring is also pretty good with CloudWatch. It does come with a cost, however, and since we will utilize [Traefik](https://containo.us/traefik/) and [Portainer](https://www.portainer.io/), we will have our monitoring in place. Leave the rest as seen in the Screenshot and continue with **Next: Add Storage**.

### Add Storage
Pretty much just leave as is, unless you need more storage or have other requirements that you think will not be fulfilled with the default settings. Continue

### Add Tags
This EC2 instance will function as the Manager Node of your Docker Swarm Cluster that receives commands on behalf of the cluster and assign containers to other Swarm nodes (if you plan on having more than 1). Tags will only help you to distinguish your instance from each other in the AWS Management Console. It will not have an effect on your Instances hostname or whatever. Do yourself a favor and Tag your AWS resources and continue.

### Configure Security Group
An AWS security group acts as a virtual firewall for your AWS resources inside of your VPC. As such it is highly recommended to read up on some best practices for setting up security groups and how to manage each layer of your infrastructure properly using security groups (e.g. [best practices](https://www.stratoscale.com/blog/compute/aws-security-groups-5-best-practices/), [AWS intro on security groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)). For this demo, we need to set up a security group that is open to inbound traffic from a couple of ports and protocols. You can set it up right now or in the Management Console go to Services > search for VPC > under Security go to Security Groups and hit Create Security Group. Give it a significant name and description add inbound rules for:

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

Now make sure to attach the newly created security group to your instance and continue.

### Review your instance and launch

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-config-keypair.png" alt="aws ec2 configuration keypair">
</p>

Upon launch, you will be asked to create a keypair, a **private** key, and a **public** key. AWS EC2 uses public-key cryptography to encrypt and decrypt login information and therefore allows you to access your instance using a private key instead of a password. For a detailed description of the concept of Public Key Authentification, I refer you to [ssh.com](https://www.ssh.com/ssh/public-key-authentication).
AWS will ask you to name and create a key pair. The private key will be stored in a `.pem` file and should be downloaded to a secure location on your local file system or trusted storage of your choice. I store it in `~/.ssh/`. If you create an EC2 instance only this file/this key will let you access it. AWS does not store a duplicate it anywhere, so if you have lost it, you lost it and you will have to shut down your instance for good. The other part, the public key, will be stored on your Linux EC2 instance in `~/.ssh/authorized_keys`.

## Congratulations, you just set up your first EC2 instance
Now open a terminal, you will need it from now on.

### Configure SSH to access your instance
For convenience's sake, I create a `~/.ssh/config` file and store my remote instances information there for easy access from the terminal.

```bash
vim ~/.ssh/config
```

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-ssh-config.png" alt="aws ec2 configuration keypair">
</p>

For your hostname, you will set up the Public DNS (IPv4) of your EC2 instance. The **User** depends on your OS. Fox Amazon Linux 2 it's `ec2-user`, for other AMIs you can look up the default user [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs.html).  Your **IdentityFile** will be the `.pem` file with your private key you downloaded earlier. Save and close the file `:wq!`.

```bash
ssh swarm
```

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-ssh.png" alt="ssh into AWS EC2">
</p>

### Change your Instances hostname
First, we will change the hostname of the instance using a subdomain of a domain you plan on using for your swarm. Changing the hostname will require you to reboot your instance. Don't worry, if you're only in it for this part of the tutorial and only want to set up your first swarm for the lulz, you can skip this step.

```bash
sudo hostnamectl set-hostname webserver.mydomain.com
sudo reboot
```

Wait for your instance to reboot, ssh back in, and check the result using `echo $HOSTNAME`.

### Install and setup Docker
Now, this part comes down to the OS you chose for your AMI. Amazon Linux 2 AMI uses `yum` package manager. Adapt the next steps to your chosen OS.

```bash
# update 
sudo yum update

# install docker 
sudo yum install docker

# start docker
sudo service docker start

# check if its installed properly
docker --version

# add ec2-user to docker group to allow docker commands without sudo
sudo usermod -a -G docker ec2-user

# add docker to autostart and reboot to see if it's working
sudo chkconfig docker on
sudo reboot

# test user rights and autostart
docker version
```

### Init your Docker swarm
With the setup of Docker out of the way, we can finally initiate our Docker Swarm.

```bash
docker swarm init
```

This will greet you with this:

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-docker-swarm-init.png" alt="Docker swarm init">
</p>

If you want to add another node to your cluster, you can go back to the AWS Management Console, and do all of this again. If that EC2 instance is ready and docker installed, you can just use the command in the screenshot above to join this new instance to your Docker swarm as a worker node.
Now you can check the nodes in your swarm via:

```bash
docker node ls
```

<p align="center">
<img src="/img/2020-05-docker-swarm/aws-ec2-docker-swarm-node-ls.png" alt="Docker swarm init">
</p>

As you can see I only have set up a single node, which is my leading node - the manager.

Congratulations! You just set up your first Docker Swarm on AWS EC2!

The next part will focus on how to setup [Traefik](https://containo.us/traefik/) and [Portainer](https://www.portainer.io/)) to handle connections in your cluster, manage HTTPS certificates and routing as well as monitoring your Docker Swarm.