# What is it about - What has been done

To learn how to make AWS AMI image with packer and ansible.

Goal: 
Run wordpress instance with docker. Wordpress should use AWS RDS DB (MySQL).

For wordpress we will use: https://hub.docker.com/_/wordpress/ image

This was done on MacOs High Sierra where we already had ansible installed. 

WordPress is accessable through port 8000 after you provision image!

# Prerequisites

You need AWS account and enabled:
- your security key and secret
- keypair
- created security group(I give it name wordpress) that has enabled ports(inbound rules): 22,8000, 3306
- base ami is from reagon us-east-2 so you nave to be aware that in other region base ami image is with different name and not ami-6a003c0f (Ubuntu Server 16.04 LTS (HVM), SSD Volume Type)

I presume that RDB DB on AWS is already running and that you have all needed data(username, password and endpoint)
I tested on MySQL 5.6.39 DB instance
You will need db instance, master username and password later in your Dockerfile.
Your DB instance for learning purposes needs to be in same security group as your running instance

Set up packer https://www.packer.io/docs/install/index.html


# How to run it

## prepare file myami.json

Replace < ...> :
```json
"variables":{
        "aws_access_key": "<your access key>",
        "aws_secret_key": "<your secret kez>"
    },
    "builders": [{
        "type": "amazon-ebs",
        "access_key": "{{user `aws_access_key`}}",
        "secret_key": "{{user `aws_secret_key`}}",
        "region": "us-east-2",
        "source_ami": "ami-6a003c0f",
        "ami_virtualization_type": "hvm",
        "instance_type": "t2.micro",
        "ssh_username": "ubuntu",
        "ssh_private_key_file": "<your private .pem key>",
        "ssh_keypair_name": "<your keypair name>",
        "ami_name": "packer-wordpress",
        "ami_description": "EC2 AMI Ubuntu", 
            "tags": {
            "role": "wordpress"
            },
            "run_tags": {
            "role": "wordpress"
            }
            }
    ],
```

## Prepare file run.sh

Replace < ...> :

```bash
docker run -p 8000:80 -e WORDPRESS_DB_HOST=<add your Endpoint>:3306  -e WORDPRESS_DB_USER=<your DB username> -e WORDPRESS_DB_PASSWORD=<your DB pass> -d wordpress
```

## Construct AMI
run command: ```packer build myami.json```

This command will provision t2.micro instance with ubuntu OS. It will install:
  - python
  - ansible
  - docker
and then create AMI image with name **packer-wordpress**

## Create EC2 Instance
Select your constructed AMI image and click Launch. Select your tier and modify data that you need. I just clicked next, till came to security group where I select my group that was prepared for this exercise. Do not forget to enable your public IP!

After instance was running it may take a minute or two that everything is up and running.

You can see your instance in browser with <you public ip>:8000

# How everything interact 

RDB DB instance is communicating with docker image that is running in our provisioned instance.
To simplify it we added both RDS DB and instance in same security group so that they can see each other.

Packer with help of shell script(so that ansible is run there) and ansible is used to provision AWS EC2 instance and then transform it to AMI image.

To ensure that dockker image is run at machine start we used crontab engine.

# Problems

Had problems installing ansible with command: sudo pip install ansible
Used wget method at the end!

Testing is slow. 

Sometimes for unknown reasons shell part has problems. Just rerun it again. 

Due that this is my first time working with packer, probably could be done differently.

# Improvements
List what will be nice to have:

- First also configuring RDS DB with provisioning tools
- More secure. Creating two secure groups and different subnets so that DB and machine are not visible directly
- When you bring EC2 instance up, image should run through https
- Se Container section

## Container

It would be better to do it more properly. That is:
  - Dockerfile
  - if container failes it should fix it by itself - kubernetes,...
  - running image in crontab is not something for production 


# Hints for next time

Divide and conquer. First test AMI on small chuncks before provision complete machine in one go! 
  - Prepare image first with smaller stuff and store it. Progress then from stored AMI and then after next steps go through combine it and then again proceed with next implementation.
  - containers should be more secure. We should prepare image that has everything inside so that we do not expose passwords
  - cleanup when you are done
  - controlling if container is running should be done properly not with crontab. I will use also some clustering if this is will be production solution.
  - add some monitor solution of the final running machine. 
