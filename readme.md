# Complete Minecraft Server Deployment (CloudFormation)

The template contained within this repository can be used to deploy a Minecraft server to Amazon Web Services (AWS) in minutes. As the solution leverages "Spot Pricing", the server should cost less than a cent an hour to run, and you can even turn it off when you and your friends aren't playing - saving even more money.

## Prerequisites

1. A basic understanding of Amazon Web Services, specifically CloudFormation.
2. An AWS Account.
3. Basic knowledge of Linux administration (no more than what would be required to just use the `itzg/docker-minecraft-server` Docker image).

## Overview

All of this is shamelessly copied and tweaked from the [Factorio Spot Pricing](https://github.com/m-chandler/factorio-spot-pricing) template.

The solution builds upon the [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server) Docker image, so generously curated by [the contributors](https://github.com/itzg/docker-minecraft-server/graphs/contributors) (thank you!). 

In a nutshell, the CloudFormation template launches an _ephemeral_ instance which joins itself to an Elastic Container Service (ECS) Cluster. Within this ECS Cluster, an ECS Service is configured to run a Minecraft Docker image. The ephemeral instance does not store any saves, mods, Minecraft config, data etc. - all of this state is stored on a network file system (Elastic File System - EFS).

The CloudFormation template is configured to launch this ephemeral instance using spot pricing. What is spot pricing you might ask? It's a way to save up to 90% on regular "on demand" pricing in AWS. There are drawbacks however. You're effectively participating in an auction to get a cheap instance. If demand increases and someone else puts in a higher bid than you, your instance will terminate in a matter of minutes. 

A few notes on the services we're using...

* **EFS** - Elastic File System is used to store Minecraft config, save games, mods etc. None of this is stored on the server itself, as it may terminate at any time.
* **Auto Scaling** - An Auto Scaling Group is used to maintain a single instance via spot pricing.
* **VPC** - The template deploys a very basic VPC, purely for use by the Minecraft server. This doesn't cost you a cent.

## Getting Started

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=minecraft)

1. Click the above link, you'll need to log into your AWS account if you haven't already.
2. Ensure you've selected a suitable AWS Region (closest to you) via the selector at the top right.
3. Upload the cf.yml file.
4. Click Next to proceed through the CloudFormation deployment, provide parameters on the following page. You'll need a Key Pair and your Public IP address if you want to access the instance remotely via SSH (recommended). Refer to the Remote Access section below. There should be no need to touch any other parameters unless you have reason to do so. Continue through the rest of the deployment. 

## Next Steps

All things going well, your Minecraft server should be running in five minutes or so. Wait until CloudFormation reports the stack status as `CREATE_COMPLETE`. Go to the [EC2 dashboard in the AWS console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=instanceId) and you should see a Minecraft server running. Take note of the public IP address. You should be able to fire up Minecraft, and join via this IP address. No need to provide a port number, we're using Minecraft's default. *Bonus points* - Public IP addresses are ugly. Refer to Custom Domain Name within Optional Features for a better solution. 

At this point you should *really* configure remote access as per the below section, so that you can access the server and modify `server.properties` (e.g. view-distance, allow nether, etc.).

## Optional Features

### Remote Access

If you know what you're doing, you might want to SSH onto the Linux instance to see what's going on / debug / make improvements. For security, SSH should be locked down to a known IP address (i.e. you), preventing malicious users from trying to break in (or worse - succeeeding). You'll need to create a Key Pair in AWS, find your public IP address, and then provide both of the parameters in the Remote Access (SSH) Configuration (Optional) section.

For remote access, you'll need to:

1. Create a [Key Pair](https://console.aws.amazon.com/ec2/v2/home#KeyPairs:sort=keyName) (Services > EC2 > Key Pairs). You'll need to use this to connect to the instance for additional setup.
2. [Find your public IP address]((https://whatismyipaddress.com/)). You'll need this to connect to the instance for additional setup.

If you're creating a new Minecraft deployment, provide these parameters when creating the stack. Otherwise, update your existing stack and provide these parameters.

### Custom Domain Name

Every time your Minecraft server starts it'll have a new public IP address. This can be a pain to keep dishing out to your friends. If you're prepared to register a domain name (maybe you've already got one) and create a Route 53 hosted zone, this problem is easily fixed. You'll need to provide both of the parameters under the DNS Configuration (Optional) section. Whenever your instance is launched, a Lambda function fires off and creates / updates the record of your choosing. This way, you can have a custom domain name such as "minecraft.mydomain.com". Note that it may take a few minutes for the new IP to propagate to your friends computers. Have patience. Failing that just go to the EC2 console, and give them the new public IP address of your instance.

## FAQ

**Do I need a VPC, or Subnets, or other networking config in AWS?** 

Nope. The stack creates everything you need.

**What if my server is terminated due to my Spot Request being outbid?** 

Everything is stored on EFS, so don't worry you won't lose anything (well, that's partially true - you might lose up to 5 minutes of gameplay depending on when the server last saved). There is every chance your instance will come back in a few minutes. If not you can either select a different instance type, increase your spot price, or completely disable spot pricing and revert to on demand pricing. All of these options can be performed by updating your CloudFormation stack parameters.

**My server keeps getting terminated. I don't like Spot Pricing. Take me back to the good old days.** 

That's fine; update your CloudFormation stack and set the SpotPrice parameter to an empty value. Voila, you'll now be using On Demand pricing (and paying significantly more).

**How do I change my instance type?** 

Update your CloudFormation stack. Enter a different instance type.

**How do I change my spot price limit?** 

Update your CloudFormation stack. Enter a different limit. 

**I'm done for the night / week / month / year. How do I turn off my Minecraft server?** 

Update your CloudFormation stack. Change the server state parameter from "Running" to "Stopped".

**I'm done with Minecraft, how do I delete this server?** 

Delete the CloudFormation stack. Done.

**I'm running the "latest" version, and a new version has just been released. How do I update my server?** 

You can force a redeployment of the service via ECS. [Update the service](https://console.aws.amazon.com/ecs/home?#/clusters/minecraft/services/minecraft/update), and select `Force new deployment`. 

**How can I change map settings, server settings etc.** 

You'll need to have remote access to the server (refer to Optional Features). You can make whatever changes you want to the configuration in `/opt/minecraft/`. Once done, restart the container:

1. Go to ECS (Elastic Container Service) in the AWS Console
2. Click the minecraft cluster
3. Tick the minecraft service, and select update
4. Tick "Force new deployment"
5. Click Next 3 times, and finally Update service

**I can no longer connect to my instance via SSH?** 

Your public IP address has probably changed. [Check your public IP address]((https://whatismyipaddress.com/)) and update the stack, providing your new public IP address.

## What's Missing / Not Supported?

* I didn't exhaustively copy over all the environment variables available, feel free to edit the cf.yml as needed by copying and renaming the appropriate blocks (like 'Whitelist/WhitelistProvided')

## Expected Costs

The two key components that will attract charges are:

* **EC2** - If you're using spot pricing (and the t3.medium instance as per the default in the template), I've been running my server nonstop for a month and the total cost was around $10/mo
* **EFS** - Charged per Gigabyte stored per month (GB-Month). Varies based on region, but typically less than 50c per gigabyte. My EFS file system for Minecraft cost 7 cents per month

AWS do charge for data egress (i.e. data being sent from your Minecraft server to clients), but again this should be barely noticeable.

## Help / Support

This has been tested in the Ohio AWS region (verify your AWS region of choice includes t3.medium @ https://aws.amazon.com/ec2/spot/pricing and/or change as needed). Your mileage may vary. If you get stuck, create an issue and myself or someone else may come along and assist.

Be sure to check out the repository on GitHub. Unless your question is specifically related to the AWS deployment, you may find the information you're after there:

- GitHub: https://github.com/itzg/docker-minecraft-server

### Stack gets stuck on CREATE_IN_PROGRESS

It may be because the `t3.medium` instance is not available in your region. As a result of this, an auto-scaling group gets successfully created - however it never launches an instance. This means the ECS service cannot ever start, as it has nowhere to place the container. I would suggest going to the AWS Console > EC2 > Spot Requests > Pricing History, and find a suitable instance type that's cost effective and has little to no fluctuation in price.

In the below example (Paris), `m5.large` looks like a good option. Try to create the CloudFormation stack again, changing the InstanceType CloudFormation parameter to `m5.large`.

![Spot pricing history](readme-spot-pricing.jpg?raw=true)

### Restarting the Container

Visit the ECS dashboard in the AWS Console.
1. Clusters
2. Click on the minecraft Cluster
3. Tick the minecraft Service, click Update
4. Tick the "Force new deployment" option
5. Click Next step (three times)
7. Click Update Service

### Basic Docker Debugging

If you SSH onto the server, you can run the following commands for debugging purposes:

* `sudo docker logs $(docker ps -q --filter ancestor=itzg/docker-minecraft-server)` - Check Minecraft container logs.

DO NOT restart the Minecraft docker container via SSH. This will cause ECS to lose track of the container, and effectively kill the restarted container and create a new one. Refer to Restarting the Container above for the right method.

## Thanks

Thanks goes out to [itzg](https://github.com/itzg/docker-minecraft-server) ([and contributors](https://github.com/itzg/docker-minecraft-server/graphs/contributors)) for maintaining the Minecraft Docker images.

Big shoutout to [Factorio Spot Pricing](https://github.com/m-chandler/factorio-spot-pricing). Couldn't have done this without it.
