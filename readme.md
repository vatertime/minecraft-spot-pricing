# Complete Minecraft Server Deployment (CloudFormation)

The template contained within this repository can be used to deploy a Minecraft server to Amazon Web Services (AWS) in minutes. As the solution leverages "Spot Pricing", the server should cost around 2 cents an hour to run, and you can even turn it off when you and your friends aren't playing - saving even more money.

## Prerequisites

1. A basic understanding of Amazon Web Services, specifically CloudFormation.
2. An AWS Account.
3. Basic knowledge of Linux administration (no more than what would be required to just use the `itzg/docker-minecraft-server` Docker image).

## Overview

All of this is shamelessly copied and tweaked from the [Factorio Spot Pricing](https://github.com/m-chandler/factorio-spot-pricing) template.

The solution builds upon the [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server) Docker image, so generously curated by [the contributors](https://github.com/itzg/docker-minecraft-server/graphs/contributors) (thank you!). 

In a nutshell, the CloudFormation template launches an _ephemeral_ instance which joins itself to an Elastic Container Service (ECS) Cluster. Within this ECS Cluster, an ECS Service is configured to run a Minecraft Docker image. The ephemeral instance does not store any saves, mods, Minecraft config, data etc. - all of this state is stored on a network file system (Elastic File System - EFS).

The CloudFormation template is configured to launch this ephemeral instance using spot pricing. What is spot pricing you might ask? It's a way to save up to 90% on regular "on demand" pricing in AWS. There are drawbacks however. Spot instances use spare AWS capacity, and AWS can take that capacity back at any time with a two minute warning, or when the spot price rises above the maximum you've set. When that happens your instance is terminated and the Auto Scaling Group launches a replacement as soon as capacity is available. If that's not for you, the template can also run On Demand (see below).

A few notes on the services we're using...

* **EFS** - Elastic File System is used to store Minecraft config, save games, mods etc. None of this is stored on the server itself, as it may terminate at any time.
* **Auto Scaling** - An Auto Scaling Group is used to maintain a single instance (spot or on demand), and to scale it to zero when the server is stopped.
* **VPC** - The template deploys a very basic VPC, purely for use by the Minecraft server. This doesn't cost you a cent.

## Getting Started

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=minecraft)

1. Click the above link, you'll need to log into your AWS account if you haven't already.
2. Ensure you've selected a suitable AWS Region (closest to you) via the selector at the top right.
3. Download [cf.yml](https://raw.githubusercontent.com/vatertime/minecraft-spot-pricing/master/cf.yml) (right click, Save Link As) and upload it. Make sure you save the raw file and not the GitHub web page around it; uploading the web page gives a "Cannot convert the template" error.
4. Click Next to proceed through the CloudFormation deployment, provide parameters on the following page. Remote access works out of the box through AWS Systems Manager Session Manager; you only need a Key Pair and your Public IP address if you prefer plain SSH. Refer to the Remote Access section below. There should be no need to touch any other parameters unless you have reason to do so. Continue through the rest of the deployment. 

## On Demand vs Spot

You may switch between On Demand and Spot via the InstancePurchaseMode CloudFormation parameter. It is not necessary to specify an InstanceType. Simply adjust MinMemoryMiB and MinVCpuCount to say how much memory and CPU you would like on your instance (the defaults, 4096 MiB and 2 vCPUs, match a t3.medium). AWS will find you the cheapest suitable instance, below the SpotPrice you have specified when using Spot. Should you wish to use a specific instance, you can specify it via the InstanceType parameter.

Leaving SpotPrice blank also selects On Demand pricing, as it did in earlier versions of this template.

## Next Steps

All things going well, your Minecraft server should be running in five minutes or so. Wait until CloudFormation reports the stack status as `CREATE_COMPLETE`. Open the stack's Outputs tab and follow the CheckInstanceIp link (or go to the [EC2 dashboard in the AWS console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=instanceId)) and you should see a Minecraft server running. Take note of the public IP address. You should be able to fire up Minecraft, and join via this IP address. No need to provide a port number, we're using Minecraft's default. *Bonus points* - Public IP addresses are ugly. Refer to Custom Domain Name within Optional Features for a better solution. 

At this point you should *really* configure remote access as per the below section, so that you can access the server and modify `server.properties` (e.g. allow nether, spawn protection, etc.). Settings that are also stack parameters (Difficulty, GameMode, MaxPlayers, ViewDistance and so on) are written by the container on every start, so change those by updating the stack instead.

## Optional Features

### Remote Access

The easiest way to get a shell on the instance is AWS Systems Manager Session Manager, which needs no Key Pair, no open SSH port and no IP address. Go to the [EC2 console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=instanceId), select the instance, click Connect, choose the Session Manager tab and click Connect. Your world and config live in `/opt/minecraft`. Session Manager starts you as `ssm-user`, so use `sudo` to edit those files or run `docker` commands.

If you'd rather use SSH: if you know what you're doing, you might want to SSH onto the Linux instance to see what's going on / debug / make improvements. For security, SSH should be locked down to a known IP address (i.e. you), preventing malicious users from trying to break in (or worse - succeeeding). You'll need to create a Key Pair in AWS, find your public IP address, and then provide the KeyPairName parameter plus YourIPv4 and/or YourIPv6 in the Optional Remote Access (SSH) Configuration section. Port 22 is only opened to the address(es) you provide.

For remote access, you'll need to:

1. Create a [Key Pair](https://console.aws.amazon.com/ec2/v2/home#KeyPairs:sort=keyName) (Services > EC2 > Key Pairs). You'll need to use this to connect to the instance for additional setup.
2. [Find your public IP address](https://whatismyipaddress.com/). You'll need this to connect to the instance for additional setup.

If you're creating a new Minecraft deployment, provide these parameters when creating the stack. Otherwise, update your existing stack and provide these parameters. Connect as `ec2-user`, e.g. `ssh -i mykey.pem ec2-user@<public ip>`. A "Permission denied (publickey)" error usually means the wrong user name, a different key than the one selected in the KeyPairName parameter, or a key file whose permissions are too open (`chmod 400 mykey.pem`).

### Custom Domain Name

Every time your Minecraft server starts it'll have a new public IP address. This can be a pain to keep dishing out to your friends. If you're prepared to register a domain name (maybe you've already got one) and create a Route 53 hosted zone, this problem is easily fixed. You'll need to provide both of the parameters (HostedZoneId and RecordName) under the Optional DNS Configuration section. Whenever your instance is launched, a Lambda function fires off and creates / updates the record of your choosing. This way, you can have a custom domain name such as "minecraft.mydomain.com". Note that it may take a few minutes for the new IP to propagate to your friends computers. Have patience. Failing that just go to the EC2 console, and give them the new public IP address of your instance.

If your domain is registered and hosted somewhere else (e.g. alongside your website and email), you don't need to move it. Create a Route 53 hosted zone for a subdomain only, such as `minecraft.mydomain.com`, then at your current DNS provider add NS records for that subdomain pointing at the four name servers Route 53 gives you. Use the new hosted zone's ID and `minecraft.mydomain.com` as the stack's DNS parameters. A hosted zone costs about $0.50 a month, which is cheaper than keeping an Elastic IP around while the server is stopped.

### Backups

By default (the EfsBackups parameter), AWS Backup takes a daily backup of the EFS file system that holds your world and keeps each backup for 35 days. To restore, go to [AWS Backup](https://console.aws.amazon.com/backup/home#/backupvaults), open the `aws/efs/automatic-backup-vault` vault, pick a recovery point and restore it (to a new directory in the same file system is the simplest). Set EfsBackups to Disabled if you don't want them.

## FAQ

**Do I need a VPC, or Subnets, or other networking config in AWS?** 

Nope. The stack creates everything you need.

**What if my server is terminated because AWS reclaimed my spot instance?** 

Everything is stored on EFS, so don't worry you won't lose anything (well, that's partially true - you might lose up to 5 minutes of gameplay depending on when the server last saved). There is every chance your instance will come back in a few minutes. If not you can leave InstanceType blank (so AWS can choose from more instance types), increase your spot price, or switch InstancePurchaseMode to "On Demand". All of these options can be performed by updating your CloudFormation stack parameters.

**My server keeps getting terminated. I don't like Spot Pricing. Take me back to the good old days.** 

That's fine; update your CloudFormation stack and set the InstancePurchaseMode parameter to "On Demand". Voila, you'll now be using On Demand pricing (and paying significantly more).

**How do I change my instance type?** 

Update your CloudFormation stack. Enter a different instance type, or leave InstanceType blank and change MinMemoryMiB / MinVCpuCount to let AWS pick one for you.

**How do I change my spot price limit?** 

Update your CloudFormation stack. Enter a different limit. 

**I'm done for the night / week / month / year. How do I turn off my Minecraft server?** 

Update your CloudFormation stack. Change the server state parameter from "Running" to "Stopped".

**How do I turn my stack on and off from the terminal?**

You can write a bash script using the AWS CLI like so (replace `minecraft` with your stack name). It keeps every other parameter at its current value, since any parameter left out of `update-stack` would otherwise be reset to its default:

``` bash
#!/bin/bash

STACK_NAME=minecraft

update_stack() {
    local state=$1
    local params=()
    for key in $(aws cloudformation describe-stacks --stack-name "$STACK_NAME" \
            --query 'Stacks[0].Parameters[].ParameterKey' --output text); do
        if [ "$key" = "ServerState" ]; then
            params+=("ParameterKey=ServerState,ParameterValue=$state")
        else
            params+=("ParameterKey=$key,UsePreviousValue=true")
        fi
    done
    aws cloudformation update-stack \
        --stack-name "$STACK_NAME" \
        --use-previous-template \
        --parameters "${params[@]}" \
        --capabilities CAPABILITY_IAM
}

case "$1" in
    start)
        update_stack "Running"
        ;;
    stop)
        update_stack "Stopped"
        ;;
    *)
        echo "Usage: $0 {start|stop}"
        exit 1
        ;;
esac
```

**I'm done with Minecraft, how do I delete this server?** 

Delete the CloudFormation stack. Except for the EFS, done. The EFS is retained when the CloudFormation stack is deleted to preserve your world, but can then be manually deleted from the [EFS console](https://console.aws.amazon.com/efs/home#/file-systems). Any AWS Backup recovery points also stay in the `aws/efs/automatic-backup-vault` vault until they expire (35 days), or you can delete them there.

**My plugins need a MySQL database. Can I add one?**

Not from this template. The cheapest option is usually to run MySQL/MariaDB on the same instance as a second container (edit the task definition in cf.yml and store its data under `/opt/minecraft` so it lands on EFS). Amazon RDS also works but costs considerably more than the Minecraft server itself, and keeps running when the server is stopped.

**I'm running the "latest" version, and a new version has just been released. How do I update my server?** 

Restart the container (see Restarting the Container below). The container downloads the new server version when it starts.

**How can I change map settings, server settings etc.** 

You'll need to have remote access to the server (refer to Optional Features). You can make whatever changes you want to the configuration in `/opt/minecraft/`. Once done, restart the container (see Restarting the Container below).

**I can no longer connect to my instance via SSH?** 

Use Session Manager (see Remote Access) instead. If you want SSH, your public IP address has probably changed. [Check your public IP address](https://whatismyipaddress.com/) and update the stack, providing your new public IP address.

## What's Missing / Not Supported?

* I didn't exhaustively copy over all the environment variables available, feel free to edit the cf.yml as needed by copying and renaming the appropriate blocks (like 'Whitelist/WhitelistProvided')

## Expected Costs

Prices below are for US East (Ohio), checked October 2026 against the AWS price list. Other regions differ a little; check the [EC2](https://aws.amazon.com/ec2/pricing/on-demand/), [Spot](https://aws.amazon.com/ec2/spot/pricing/) and [EFS](https://aws.amazon.com/efs/pricing/) pricing pages for yours. While the server is running nonstop on Spot, expect roughly $15 to $20 a month all in. While it is stopped, you only pay for EFS storage and backups, usually well under $1 a month.

* **EC2** - A t3.medium-sized instance (the template's default of 4096 MiB and 2 vCPUs) costs $0.0416 an hour On Demand, about $30 a month nonstop. Spot prices change constantly but are typically around $0.012 to $0.016 an hour, about $9 to $12 a month nonstop. With InstanceType left blank, AWS may pick a different instance that is cheaper at the time.
* **Public IPv4 address** - $0.005 an hour (about $3.65 a month) for the instance's public IP address while it is running.
* **EBS** - The instance's root volume (30 GiB gp3 on the ECS-optimized AMI) costs about $0.08 per GB-month, roughly $2.40 a month while running. It is deleted along with the instance when the server is stopped.
* **EFS** - $0.30 per GB-month for Standard storage. Files not accessed for 7 days move to Infrequent Access ($0.025 per GB-month, plus $0.01 per GB when read), and move back as soon as they are read. A typical world is well under a gigabyte, so this is usually a few cents a month.
* **Backups** - With EfsBackups enabled, AWS Backup charges $0.05 per GB-month. Backups are incremental, so you pay for one full copy of your world plus whatever changed each day, kept for 35 days.

AWS also charge for data egress (i.e. data being sent from your Minecraft server to clients) at $0.09 per GB, but the first 100 GB a month across your account are free, which a small server rarely exceeds.

## Help / Support

This has been tested in the Ohio AWS region. If you set a specific InstanceType, verify your AWS region of choice offers it @ https://aws.amazon.com/ec2/spot/pricing. Your mileage may vary. If you get stuck, create an issue and myself or someone else may come along and assist.

Be sure to check out the repository on GitHub. Unless your question is specifically related to the AWS deployment, you may find the information you're after there:

- GitHub: https://github.com/itzg/docker-minecraft-server

### Stack gets stuck on CREATE_IN_PROGRESS

NOTE: This should rarely be an issue when InstanceType is left blank, since AWS picks from every instance type that meets MinMemoryMiB and MinVCpuCount.

It may be because the instance type you chose is not available in your region, or because your SpotPrice is below the current spot price for every suitable instance. As a result of this, an auto-scaling group gets successfully created - however it never launches an instance. This means the ECS service cannot ever start, as it has nowhere to place the container. Try leaving InstanceType blank, or raising SpotPrice. Otherwise, go to the AWS Console > EC2 > Spot Requests > Pricing history, find a suitable instance type that's cost effective and has little to no fluctuation in price, and set it as the InstanceType parameter.

### Restarting the Container

Visit the ECS dashboard in the AWS Console.
1. Clusters
2. Click on the `<stack name>-cluster` cluster (e.g. `minecraft-cluster`)
3. Tick the `<stack name>-ecs-service` service, click Update
4. Tick the "Force new deployment" option
5. Click Update

From the terminal: `aws ecs update-service --cluster minecraft-cluster --service minecraft-ecs-service --force-new-deployment` (replace `minecraft` with your stack name).

### Basic Docker Debugging

If you connect to the server (Session Manager or SSH), you can run the following commands for debugging purposes:

* `sudo docker logs $(sudo docker ps -q --filter name=minecraft)` - Check Minecraft container logs.

DO NOT restart the Minecraft docker container via SSH. This will cause ECS to lose track of the container, and effectively kill the restarted container and create a new one. Refer to Restarting the Container above for the right method.

## Changelog

Oct-2026
  * Instances now run the Amazon Linux 2023 ECS-optimized AMI.
  * Daily EFS backups via AWS Backup, retained for 35 days (EfsBackups parameter).
  * Remote access via AWS Systems Manager Session Manager, no Key Pair or open SSH port needed.
  * Ported from upstream Factorio Spot Pricing: migrate from Launch Configuration to Launch Template, as Launch Configuration is unavailable in AWS accounts created after 01-Oct-2024.
  * Ported from upstream: InstanceType is now optional. Specify the memory and vCPUs you need and AWS picks the cheapest instance; InstancePurchaseMode switches between Spot and On Demand.
  * Ported from upstream: the EFS file system is retained when the stack is deleted, and uses lifecycle management to lower cost.
  * DNS Lambda runtime updated to Python 3.13.

## Thanks

Thanks goes out to [itzg](https://github.com/itzg/docker-minecraft-server) ([and contributors](https://github.com/itzg/docker-minecraft-server/graphs/contributors)) for maintaining the Minecraft Docker images.

Big shoutout to [Factorio Spot Pricing](https://github.com/m-chandler/factorio-spot-pricing). Couldn't have done this without it.
