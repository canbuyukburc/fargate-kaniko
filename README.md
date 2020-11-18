# Fargate Kaniko

An experiment to run [Google
Kaniko](https://github.com/GoogleContainerTools/kaniko) as a Container Image
builder inside of AWS Fargate. Traditional Container Image Builders (docker
build) can not run in the default isolation boundary of a container and often
need host privileges (--privileged) or access to the underlying Container
Runtime. In AWS Fargate, you are not able to run a container with "privileges"
and you are not able to access the underlying container runtime. Therefore a
rootless builder, like Kaniko is required.

This repository contains a Task Definition and a Run Task instruction for Amazon
ECS. This Run Task could be triggered be a CI pipeline easily enough.

The Task definition includes 2 containers. A git container and a Kaniko
container. One of the design goals here was to use the upstream Kaniko image. I
did not want to have to maintain my own builder image. Running a seperate [Git
container](./git-container/) in the same task, which is an Alpine base with the
git client installed, allows me to achieve this design goal by not installing
additional tools in the Kaniko image.

## Prerequisites

This repo assumes some core AWS infrastructure is in place.

- A VPC, Subnets and Security Group (No inbound access is required in the
  security group)
- An ECS Cluster
- An ECR Repository
- An IAM Role with a Policy that can push to the ECR repository. This will be
  assumed by our ECS Task.

## Create ECS Task Definition

First we will create the Cloudwatch Log Group with a short retention policy for
our build logs to go to.

```
aws logs create-log-group \
    --log-group-name kaniko-builder

aws logs put-retention-policy \
    --log-group-name kaniko-builder \
    --retention-in-days 7
```

Create the ECS task defintion. This will need to be customised for your
environment with the relevant ARNs and ECR / Git Repos. Also the commands in the
Kaniko container definition set the Build Context and the location of the
Dockerfile, these will need to be updated depending on the application git
repository layout.

```
aws ecs register-task-definition \
    --family kaniko-builder \
    --cli-input-json file://kaniko-taskdef.json
```

Finally we can run the ECS Task. This Run Task definition will also need to
updated with the relevant AWS VPC, Subnet, Secruity Group and ECS Cluster.

```
aws ecs run-task \
    --task-definition kaniko-builder:14 \
    --cli-input-json file://kaniko-runtask.json
```

Now you can monitor the build in Cloudwatch Logs and watch the ECR repository
for your new container image :)