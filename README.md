# Hydra and AWS in Docker Swarm
Running Hydra on AWS in a Docker Swarm

## Table of Contents

* [Introduction](#introduction)
* [AWS Setup](#aws-setup)
  * [Creating Nodes](#creating-nodes)
  * [Removing Nodes](#removing-nodes)
* [Swarm Create](#swarm-create)
* [Configuration Management](#configuration-management)
* [Services](#services)
* [Troubleshooting](#troubleshooting)
* [Redis and ELK instances](#redis-and-elk-instances)

---

<a name="introduction"></a>
## Introduction

<a name="aws-setup"></a>
## AWS Setup

* Sign into AWS and create a new IAM role for a programmatic user with `AmazonEC2FullAccess` credentials
* Grab the Access Key and Secret Key as you'll need them when using docker-machine
* make sure that the VPC used contains security group named `docker-machine` and that it has the following ports open (see https://gist.github.com/BretFisher/7233b7ecf14bc49eb47715bbeb2a2769):
  * TCP port 2377 for cluster management communications
  * TCP and UDP port 7946 for communication among nodes
  * TCP and UDP port 4789 for overlay network traffic

<a name="creating-nodes"></a>
### Creating Nodes

![](images/swarm.png)

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

<a name="removing-nodes"></a>
### Removing Nodes

```shell
#!/bin/bash
docker-machine rm -f $1
```

<a name="swarm-create"></a>
## Swarm Create

```shell
# create swarm masters
#
for i in 1 2 3
do
  ./create-node master0$i t2.small
done

# create swarm workers
#
for i in 1 2 3 4 5
do
  ./create-node worker0$i t2.small
done
```

#### List created machines

```shell
$ docker-machine ls -t "30"
```

### Assign swarm managers

```shell
$ docker-machine ssh master01
$ sudo docker swarm init --advertise-addr 34.205.252.209:2377
Swarm initialized: current node (f15m9npvwumliqoe6wzor8tvh) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2ohfpmuvx39e2o7wzqg1qcohoti8layg0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
    34.205.252.209:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ sudo docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2ohfpmuvx39e2o7wzqg1qcohoti8layg0vk7ivoebncmw37p9y-08zcw2jht968k1tdnf8dofcha \
    34.205.252.209:2377
```

```shell
$ docker-machine ssh master02
$ sudo docker swarm join \
--token SWMTKN-1-2ohfpmuvx39e2o7wzqg1qcohoti8layg0vk7ivoebncmw37p9y-08zcw2jht968k1tdnf8dofcha \
34.205.252.209:2377
$ exit

$ docker-machine ssh master03
$ sudo docker swarm join \
--token SWMTKN-1-2ohfpmuvx39e2o7wzqg1qcohoti8layg0vk7ivoebncmw37p9y-08zcw2jht968k1tdnf8dofcha \
34.205.252.209:2377
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

### Assign swarm workers

```shell
for i in 1 2 3 4 5
do
  docker-machine ssh worker0$i sudo docker swarm join \
  --token SWMTKN-1-2ohfpmuvx39e2o7wzqg1qcohoti8layg0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
  34.205.252.209:2377
done
```

From a manager node:

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

```shell
$ docker network create servicenet \
  --driver overlay \
  --subnet 10.0.9.0/24
```

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

| network | usage | scope |
| --- | --- | --- |
| docker_gwbridge | A network created by Docker to allow containers to connect to their host | local |
| ingress | Network created by Docker to externally expose services and utilize the routing mesh | swarm |
| servicenet | Overlay network we created to enable container to container communication | swarm |

#### References

* https://docs.docker.com/engine/swarm/networking/
* https://docs.docker.com/engine/swarm/ingress/
* https://docs.docker.com/engine/userguide/networking/get-started-overlay/
* https://github.com/docker/docker/issues/25295
* http://neuvector.com/blog/docker-swarm-container-networking/

### Swarm visualization service

```shell
$ docker-machine ssh master01
$ docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  manomarks/visualizer
```


<a name="configuration-management"></a>
## Configuration management

### upload config.json

```shell
$ hydra-cli use gemini
$ hydra-cli cfg hydra-router:0.15.4 config.json
```

#### Sample config

```javascript
{
  "cluster": false,
  "processes": 0,
  "externalRoutes": {},
  "risingStack": false,
  "debugLogging": true,
  "hydra": {
    "serviceName": "hydra-router",
    "serviceDescription": "Service Router",
    "serviceIP": "",
    "servicePort": 80,
    "serviceType": "router",
    "plugins": {
      "logger": {
        "logRequests": true,
        "redact": ["password"],
        "elasticsearch": {
          "host": "10.0.0.56",
          "port": 9200,
          "index": "hydra"
        }
      }
    },
    "redis": {
      "url": "redis://10.0.0.154:6379/15"
    }
  },
  "queuer": {
    "url": "10.0.0.154",
    "port": 6379,
    "db": 3
  }
}
```

<a name="services"></a>
## Services

```shell
$ docker service create \
    --name hydra-router \
    --network servicenet \
    --constraint=node.role==manager \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hydra-router:0.15.4" \
    --publish 80:80 \
    --replicas=1 \
    flywheelsports/fwsp-hydra-router:0.15.6
```

> When a service is created which using `--publish` it is automatically added to the `ingress` network.

```shell
$ docker service create \
    --name hello-service \
    --network servicenet \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hello-service:0.0.2" \
    --replicas=1 \
    cjus/hello-service:0.0.5
```

> Creating a service which does not use `--publish` places the service in the `servicenet`.  The service can still listen on a port for inter-service communication.

## Removing services

```shell
$ docker service rm hydra-router
$ docker service rm hello-service
```

## Scaling Services

### Scale up

```shell
$ docker service scale hydra-router=3
```

### Scale down

```shell
$ docker service scale hydra-router=0
```

<a name="troubleshooting"></a>
## Troubleshooting

```shell
$ docker exec -it eacb530274c7 bash
$ ip addr
```

<a name="redis-and-elk-instances"></a>
## Redis and ELK instances

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
