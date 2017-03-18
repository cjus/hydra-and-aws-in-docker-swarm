# Hydra, AWS, and Docker: a microservices deployment story

Document status: Work in progress - version 20170318

<a name="toc"></a>
## Table of Contents

* [Introduction](#introduction)
* [Dockerizing Hydra](#dockerizing-hydra)
* [AWS Setup](#aws-setup)
  * [Creating EC2 Nodes](#creating-ec2-nodes)
* [Creating and configuring the swarm](#creating-a-swarm)
* [Configuration Management](#configuration-management)
* [Services](#services)
* [Test drive](#testdrive)
* [Troubleshooting](#troubleshooting)
* [Redis and ELK instances](#redis-and-elk-instances)

---

<a name="introduction"></a>
## Introduction
[TOC](#toc)

In this document, we'll look at one way to use Hydra within Amazon's AWS while leveraging Docker and in particular Docker's Swarm mode for container orchestration.

The use of Docker and containerization is a topic which requires a fair amount of understanding. This document isn't intended to be an introduction to Docker or even a thorough examination of Docker's Swarm Mode. The same limitation applies to the Amazon AWS services described herein.  In particular, we won't cover the creation and configuration of an Amazon Application Load Balancer.  You should already know how to do that if tackling the creation of a docker swarm.

Instead, our goal here is to examine an approach to deploying your hydra-enabled services within Docker containers on AWS infrastructure. Your exact deployment strategy depends on your particular needs - the approach described here simply represents the most common use case.

The biggest challenge in thinking about the use of Hydra in the context of AWS and Docker is understanding how it all comes together. Hopefully, this document succeeds in that regard.

### Why Hydra?
[TOC](#toc)

[Hydra](https://github.com/flywheelsports/fwsp-hydra) is a library for facilitating the construction of distributed Node.js based applications. The most common usage is building microservices. Much of what Hydra offers is already provided by other NPM packages and services. So a common question is "why do I need Hydra?". Hydra was created to make it easy to use a single NPM package rather than a host of solutions. This makes conceiving and building microservices easier to reason about. Because Hydra is a library - rather than a framework, you're in control of which features to use and which to ignore or defer to other solutions.

A key feature of Hydra is its size. We've used Hydra on tiny Raspberry Pi Zeros and in a [cluster of SBCs](https://github.com/cjus/hydra-cluster) in addition to production-level applications running on cloud infrastructure.

### Why Docker, Docker Swarm, and Hydra?
[TOC](#toc)

Virtual Machine software has ushered in the age of containerization where applications can be packaged as containers making them portable and easier to manage. Docker is a significant evolution of that trend.

Running Hydra-enabled microservices inside of containers make them portable. We're able to run containers locally on our laptops and run the same containers in the cloud.  This greatly simplifies the construction of larger applications that consist of many moving service parts, as you're able to debug locally.

Getting containers to work together is non-trivial and requires the use of container orchestration tools such as Kubernetes, Mesos, Consul, Zookeeper and the like. This presented a formable learning curve for developers and so in version 1.12 of Docker, container orchestration was baked into the product under the name [Docker Swarm Mode](https://docs.docker.com/engine/swarm/).

### Cutting to the chase
[TOC](#toc)

In this document, we're going to look at building an eight-node cluster accessible via an Amazon Application Load Balancer (ALB). Our cluster will accept HTTP traffic and load balance between three master nodes which host our service-aware Application API Gateway, [hydra-router](https://github.com/flywheelsports/fwsp-hydra-router). Hydra-router, itself a microservice, is the only service listening on port 80.  It's responsible for routing service calls to individual services within the cluster.

Hydra-router will only run on master nodes 01 - 03, which are directly accessible via the ALB.  Our microservices will run on worker nodes 01-05.  Services running on worker nodes will not publish ports for use outside of the network that the container is running in.

<img src="./images/swarm-roles.png" alt="swarm roles" width="600px" />

Referring to the above diagram, the master nodes in the Ingress network communicate with one another in support of high availability. If one master node dies another is elected the active master. We can also scale up or down as required.  Each Hydra-router running inside of a master node is able to communicate with microservices running in containers on the `service network`. Additionally, each service is able to communicate with the outside world (external API services) and with its internal peers and hydra-routers.

Using Docker swarm mode, we'll be able to scale our services with a simple command. We'll also be able to add and remove EC2 instances participating in a swarm and redistribute our services accordingly.

The remainder of this document provides the specifics.

<a name="dockerizing-hydra"></a>
## Dockerizing Hydra  
[TOC](#toc)

If you've created your microservices using the [Yeoman generator for Hydra services](https://github.com/flywheelsports/generator-fwsp-hydra) then your service is already set for containerization.

```shell
$ npm run docker build
```

Otherwise you can create a Dockerfile in your project's root folder such as:

```
FROM node:6.9.4-alpine
MAINTAINER Carlos Justiniano cjus34@gmail.com
EXPOSE 2100  
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
ADD . /usr/src/app
RUN npm install -g pino-elasticsearch
RUN npm install --production
CMD ["npm", "start"]
```

> 🍺 **PubTip**: specifying the working directory above is ideal for later mapping in an external folder so you can update your config files at runtime.

Then run something like:

```shell
$ docker build --no-cache=true -t flywheelsports/fwsp-hydra-router:0.15.6 .
$ docker push flywheelsports/fwsp-hydra-router:0.15.6
```

If you're not thinking "yeah duh!" then you can learn more about docker [here](https://docs.docker.com/learn/).

<a name="aws-setup"></a>
## AWS Setup
[TOC](#toc)

Setting up our Docker Swarm requires the use of AWS EC2 instances. We're going to create and configure those instances from the command-line rather than from the AWS web interface. In preparation for this, we'll first create a new IAM role for a programmatic user with `AmazonEC2FullAccess` credentials.

> <img src="./images/IAM_Management_Console.png" alt="swarm roles" style="width: 600px;"/>

Make sure to grab the Access Key and Secret Key as you'll need those shortly.

To assist with the creation and configuration of EC2 instances we'll create a shell script called `create_node` which uses the [docker-machine](https://docs.docker.com/machine/overview/) command to create an EC2 instance and install the docker engine.

```shell
#!/bin/bash

AWS_AK="FAKE-AKIAISQOOR3SGN3RPCBT"
AWS_SK="FAKE-CfZ77GEyomrp7+VmRCX+yq+D6uJoKv8lufbEH8Jl"
AWS_VPC='vpc-{here}'
AWS_REGION='us-east-1'

NODE_NAME=$1
MTYPE=$2
MACHINE_TYPE=${MTYPE:='t2.small'}

docker-machine create --driver amazonec2 \
    --amazonec2-access-key $AWS_AK \
    --amazonec2-secret-key $AWS_SK \
    --amazonec2-vpc-id "${AWS_VPC}" \
    --amazonec2-region "${AWS_REGION}" \
    --amazonec2-instance-type "${MACHINE_TYPE}" \
    ${NODE_NAME}

echo "${NODE_NAME} should be available in a minute"
```

In the script, we've defined the AWS Access token key (AWS_AK) and the Secret token key (AWS_SK). Additionally, we define the AWS VPC id (AWS_VPC) and the AWS Region (AWS_REGION).  These tokens can be defined and exported outside of the script as a good practice - it's embedded in this example for clarity.

The above script also allows you to specify the type of EC2 instance to use. The default is `t2.small` but could be `t2.micro` or larger depending on your needs.

Using the script is as easy as:

```shell
$ ./create_node node01 t2.micro
```

As a complement to the above script, we'll also create a `remove_node` script.

```shell
#!/bin/bash
docker-machine rm -f $1
```

So we can remove EC2 instances created using `create-node`:

```shell
$ ./remove_node node01
```

<a name="creating-ec2-nodes"></a>
### Creating EC2 Nodes
[TOC](#toc)

As a recap here is the breakdown of the EC2 instances we'll create.

<img src="./images/swarm.png" alt="swarm" style="width: 500px;"/>

Using our `create-node` script we're able to automate the creation and configuration of our EC2 instances.

```shell
# create master nodes
#
for i in 1 2 3
do
  ./create-node master0$i t2.small
done

# create worker nodes
#
for i in 1 2 3 4 5
do
  ./create-node worker0$i t2.small
done
```
> 🍺 **PubTip**: Consider running each section above in different terminal shells. At this stage, the master and worker nodes do not depend on one another and can be created in parallel.

Once the above commands complete we can view a list of created machines.

```shell
$ docker-machine ls -t "30"
```

#### AWS security group setup
[TOC](#toc)

After creating your first EC2 node above you should see a `docker-machine` security group in the VPC you specified. It's a basic setup suitable for simple uses, but we'll need to update it for use with our swarm.

Here's a summary of the changes we need to make:

  * SSH port 22
  * TCP port 2377 for cluster management communications
  * TCP and UDP port 7946 for communication among nodes
  * TCP and UDP port 4789 for overlay network traffic

Here is what your enhanced security group should look like on AWS:

> <img src="./images/EC2_DockerSecurityGroup.png" alt="AWS security group" style="width: 800px;"/>

With these change in place, we can proceed to configure our swarm.

<a name="creating-a-swarm"></a>
## Creating and configuring the swarm
[TOC](#toc)

We're now ready to configure our swarm.  This process will involve creating a swarm manager and assigning workers.

We begin configuring our swarm by requesting the external IP address of our the master01 node.

```shell
$ docker-machine ip master01
35.128.252.201
```

We'll use that IP to initialize our swarm.

```shell
$ docker-machine ssh master01
$ sudo docker swarm init --advertise-addr 35.128.252.201:2377
Swarm initialized: current node (f15m9npvwumliqoe6wzor8tvh) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2ohfpmuvx34e2o7wzag1qcohoti8layd0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
    35.128.252.201:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
    35.128.252.201:2377
```

We have two other master nodes to turn into managers.  Sadly, they won't get a pay raise.

```shell
$ docker-machine ssh master02
$ sudo docker swarm join \
--token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
35.128.252.201:2377
$ exit

$ docker-machine ssh master03
$ sudo docker swarm join \
--token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
35.128.252.201:2377
$ exit
```

From any swarm manager node you can view the status of managers:

```shell
$ sudo docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
f15m9npvwumliqoe6wzor8tvh *  master01  Ready   Active        Leader
t77rsrfdrq9u3v4rftldyzsgj    master02  Ready   Active        Reachable
ye7iq8hswgacvkz8il51v6je1    master03  Ready   Active        Reachable
```
Here we see that our master01 node is the leader, but should something happen to it - one of the other managers will be elected the new leader.  If our master01 node later recovers from its untimely accident it won't resume as the leader but it will be marked as Reachable and eligible for promotion should something happen to another master node.

Now we're ready to configure our worker nodes.

```shell
for i in 1 2 3 4 5
do
  docker-machine ssh worker0$i sudo docker swarm join \
  --token SWMTKN-1-2ohfpmuvx34e2o7wzag1qcohoti8layd0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
  35.128.252.201:2377
done
```

From a manager node, we can see the status of our swarm cluster. We see that our master01 node is the leader, with two mangers reachable and waiting in the wings for their shot at the corner office.  We also see that none of our worker nodes are managers.

```
$ sudo docker node ls -t "30"
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
8caeo3nvjfa5d3jrqamciyijv    worker04  Ready   Active
c4nc3wnr45ii53hli5yomw234    worker03  Ready   Active
dfjrl5767thytai4lz9dfk360    worker05  Ready   Active
f15m9npvwumliqoe6wzor8tvh *  master01  Ready   Active        Leader
fcvzbgziv3ptso1r9egazizqv    worker01  Ready   Active
t77rsrfdrq9u3v4rftldyzsgj    master02  Ready   Active        Reachable
vz489z1vywrthlt4r9bw94zda    worker02  Ready   Active
ye7iq8hswgacvkz8il51v6je1    master03  Ready   Active        Reachable
```

### Swarm networking
[TOC](#toc)

At this stage, we have EC2 instances participating in a swarm as either managers or workers. We're not ready to create a network on which each node can communicate.  This type of network is referred to as an overlay network.

```shell
$ docker network create servicenet \
  --driver overlay \
  --subnet 10.0.9.0/24
```

You can list available networks with:

```shell
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
7ffba041b5b9        bridge              bridge              local
90d25bc2e419        docker_gwbridge     bridge              local
7af9c7ddd8f6        host                host                local
p5f0bg197oia        ingress             overlay             swarm
e5f86522a1d0        none                null                local
z6vut7t9439u        servicenet          overlay             swarm
```

Notice that there are two overlay networks, `ingress` and our newly created `servicenet` - both have a scope of `swarm`.

<img src="./images/networks.png" alt="networks" style="width: 400px;"/>

| network | usage | scope |
| --- | --- | --- |
| docker_gwbridge | A network created by Docker to allow containers to connect to their host | local |
| ingress | Network created by Docker to externally expose services and utilize the routing mesh | swarm |
| servicenet | Overlay network we created to enable container to container communication | swarm |


### Swarm visualization service
[TOC](#toc)

Mano Marks has created a handy [docker swarm visualizer](https://github.com/ManoMarks/docker-swarm-visualizer) that we'll install onto our master01 node.

```shell
$ docker-machine ssh master01
$ docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  manomarks/visualizer
```

To view it, make sure to open port 8080 on the master01 node using an AWS security group that restricts access to your IP address.

<img src="./images/Visualizer.png" alt="AWS security group" style="width: 900px;"/>

<a name="configuration-management"></a>
## Configuration management
[TOC](#toc)

Hydra-based applications are initialized using a JavaScript object which contains the service name, description, IP and Port information and the location of the Redis server that Hydra depends on. Most often that information is loaded from a remote config.json file. In the case of a containerized hydra-based application, you have the option of overriding the packaged config.json file with one mapped to a volume using the `-v` fragment in the example below:

```shell
docker run -d \
  --workdir=/usr/src/app \
  -p 1337:1337 \
  --restart always \
  --add-host host:$HOST \
  --add-host redis:$DBS \
  --name auth-svcs \
  -v /usr/local/etc/auth-svcs:/usr/src/app/config \
  someco/auth-svcs:0.2.7
```

This can can work fine in dockerized deployments which use ECS optimized EC2 images.  You simply have to ensure that the config files are present on the machine before running the dockerized containers.

However, this isn't convenient for use with Docker Swarm where you don't actually know what machine your container will run on.

Starting with [fwsp-hydra](https://github.com/flywheelsports/fwsp-hydra) 0.15.10 and [fwsp-hydra-express](https://github.com/flywheelsports/fwsp-hydra-express)  0.15.11 your hydra service can request its config directly from your Redis instance. Naturally, that implies that you've loaded the config into Redis in the first place.

To do this you'll need [hydra-cli](https://github.com/flywheelsports/hydra-cli) version 0.5.4 or greater.

```shell
$ hydra-cli cfg push hydra-router:0.15.4 config.json
```

You're expected to provide the service name separated by a version string and a local config.json file whose contents will be uploaded.

Later you can retrieve a stored file using:

```shell
$ hydra-cli cfg pull hydra-router:0.15.4 > config.json
```

This is useful when you want to make changes to an existing config file or when you'd like to upload a new config based on an older copy.

<a name="services"></a>
## Services
[TOC](#toc)

We can now use the Docker service create command to push containers into our swarm. In the example below we specify `--env HYDRA_REDIS` to point to the Redis server the service will use to retrieve its configuration file from.  In production, the Redis instance would likely be an Amazon Elastic Cache cluster or one at RedisLabs.

```shell
$ docker service create \
    --name hydra-router \
    --network servicenet \
    --constraint=node.role==manager \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hydra-router:0.15.4" \
    --publish 80:80 \
    --replicas=3 \
    flywheelsports/fwsp-hydra-router:0.15.6
```

> When a service is created which uses `--publish` it is automatically added to the `ingress` network. This is because publishing a port says you want the container to be remotely accessible.

```shell
$ docker service create \
    --name hello-service \
    --network servicenet \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hello-service:0.0.2" \
    --replicas=5 \
    cjus/hello-service:0.0.5
```

> Creating a service which does not use `--publish` places the service in the `servicenet`, our private subnet. The service can still listen on a port for inter-service communication.

If you'd like to play around with this, both the flywheelsports/fwsp-hydra-router and cjus/hello-service containers can be pulled from Docker Hub.

### Removing services

You can easily remove services using:

```shell
$ docker service rm hydra-router
$ docker service rm hello-service
```

### Scaling services

Scaling services is just a matter of using the Docker service scale command and specifying the service name and the number of required replicas.  This allows you to scale a service up or down.

```shell
$ docker service scale hydra-router=3
```

```shell
$ docker service scale hydra-router=0
```

<a name="testdrive"></a>
## Test drive
[TOC](#toc)

In order to try all of this out you'll need to obtain the IP address of your Amazon ALB from the AWS dashboard.


<img src="./images/EC2_AEB_Console.png" alt="networks" style="width: 700px;"/>

You can direct traffic to the load balancer doing something like this:

<img src="./images/hydra-alb-test_us-east-1_elb_amazonaws_com_v1_hello_test.png" alt="networks" style="width: 700px;"/>

Refreshing the browser page would display different service IDs as the traffic is load balanced to our five hello services.

<a name="troubleshooting"></a>
## Troubleshooting
[TOC](#toc)

While debugging the swarm networking options I found it useful to be able to enter a container, check its IP address using `ip addr` and pinging and using curl to access other services.

```shell
$ docker exec -it eacb530274c7 bash
$ ip addr
```

<a name="redis-and-elk-instances"></a>
## Redis and ELK instances
[TOC](#toc)

Not all of the services a swarm requires need to be inside of the swarm itself. You can add resources to the same VPC where the swarm lives in order for them to be internally accessible.  Of course, you're also free to access remote and publicly accessible services.

In this example, we add a Redis instance and a container for an ELK stack for use by our swarm nodes.

Create two t2.small EC2 instances using Amazon ECS optimized image.

Setup Redis:

```shell
$ docker pull redis:3.0.7
$ sudo mkdir /data

Add this to /etc/rc.local

docker rm -f redis
docker run -d -p 6379:6379 --restart always -v /data:/data --name redis redis:3.0.7

Then:

$ sudo reboot
```

Setup ELK:

```shell
$ sudo docker pull elasticsearch:2
$ sudo docker pull kibana:4
$ sudo mkdir /data

Add this to /etc/rc.local

HOST=`ifconfig eth0 | grep "inet addr" | cut -d ':' -f 2 | cut -d ' ' -f 1`
docker rm -f elasticsearch
docker run -d -p 9200:9200 -p 9300:9300 --restart always --ulimit nofile=2048:2048 -v /data:/data --name elasticsearch elasticsearch:2
docker rm -f kibana
docker run -d -p 5601:5601 --restart always --add-host elasticsearch:$HOST -v /data:/data --name kibana kibana:4

Then:

$ sudo reboot
```

---
[TOC](#toc)
