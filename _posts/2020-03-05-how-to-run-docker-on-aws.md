---
layout: post
title: How to run docker container on AWS using CLI.
tags: devops aws infrastructure docker
---

# How to run docker container on AWS using Elastic Container Service(ECS) and CLI.

## Motivation.

When migrating our infrastructure on AWS I found it is not very clear on how to run the single docker container on AWS cloud.
I couldn't find any brief step-by-step guide on how to setup the project using only (or mostly) CLI.
The usage of CLI makes our development workflow faster and allows us to write simply automation scripts and instructions instead of creating heavy GUI-guided guides.

In order to run our container we need to go through the following steps:

1. Configure AWS CLI and ECS CLI
2. Create docker registry and push container
3. Create configuration and run our docker container on ECS.
4. Update code in the running containers

## Installation

Installation instructions for ECS CLI for all the systems may be found on [ECS CLI Installation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html)
Installation instructions for AWS CLI for all the systems may be found on [AWS CLI Installation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

The following installation instructions are provided for macOS.

Install AWS CLI

```shell
$ brew install awscli
```

Install ECS CLI

```shell
$ sudo curl -o /usr/local/bin/ecs-cli https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-darwin-amd64-latest
$ sudo chmod +x /usr/local/bin/ecs-cli
$ ecs-cli --version
```

If you see something like this, you are fine:

```shell
$ ecs-cli --version
ecs-cli version 1.18.1 (7e9df84)
```

## Getting keys

Get you `AWS_ACCESS_KEY` and `AWS_SECRET_KEY`

They may be found in AWS Console(this requires GUI interaction):

![iam]({{ "/assets/aws_iam_console.png" | absolute_url }})
**Pic.1.** _AWS Console. Proceed to IAM._

Proceed to users and click on you username:

![IAM]({{ "/assets/aws_iam.png" | absolute_url }})
**Pic.2.** IAM Users

Go to "Security Credentials" and create the key.
![IAM]({{ "/assets/aws_key.png" | absolute_url }})
**Pic.3.** Create Access Key

Copy ID and KEY to safe place.

## Configure CLI

Run:

```shell
AWS_PROFILE_NAME=my-profile
aws configure --profile $AWS_PROFILE_NAME
```

You will be prompted with:

```shell
AWS Access Key ID [****************M2WF]:
AWS Secret Access Key [****************i1Y2]:
Default region name [eu-central-1]:
Default output format [text]:
```

Then configure ECS CLI:

```shell
$ AWS_ACCESS_KEY_ID=key
$ CLUSTER_NAME=cluster
$ ecs-cli configure profile \
  --cluster $CLUSTER_NAME \
  --profile-name $AWS_PROFILE_NAME \
  --access-key $AWS_PROFILE_NAME \
  --secret-key $AWS_SECRET_ACCESS_KEY
```

## Build and push container

Firstly, we need to create the docker registry repo [AWS ECR](https://aws.amazon.com/ru/ecr/):

I just used instructions provided here: [Instructions](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html)

Or using CLI:

```shell
$ REPOSITORY_NAME=repo-name
$ aws ecr create-repository --repository-name=$REPOSITORY_NAME --profile $AWS_PROFILE_NAME
```

and you will see in output:

```
arn:aws:ecr:eu-central-1:149855362562:repository/repo-name        repo-name 123123123123.dkr.ecr.eu-central-1.amazonaws.com/repo-name
```

`123123123123.dkr.ecr.eu-central-1.amazonaws.com/repo-name` - is and address of our repo/registry

Once the repository is ready we can push our container there:

1. Login to the registry

```shell
$ aws ecr get-login-password --profile $AWS_PROFILE_NAME | docker login --username AWS --password-stdin 123123123123.dkr.ecr.eu-central-1.amazonaws.com/repo-name
```

2. `cd` into the directory with `Dockerfile` and execute the following command to push your container:

```shell
LOCAL_TAG=my_app/app
REGISTRY_TAG=123123123.dkr.eu-central-1.amazonaws.com/my-repo
$ docker build -t $LOCAL_TAG . && docker tag $LOCAL_TAG $REGISTRY_TAG && docker push $REGISTRY_PATH
```

`LOCAL_TAG` - just a container tag

## Create cluster

## Create task definition

## Create service

## Update Task definition and a service

## CI/CD

## Useful commands
