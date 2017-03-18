# Hydra and AWS in Docker Swarm
Running Hydra on AWS in a Docker Swarm

## Table of Contents

* [Introduction](#introduction)
* [AWS Setup](#aws-setup)
  * [Creating Nodes](#creating-nodes)
  * [Removing Nodes](#removing-nodes)
* [Swarm Create](#swarm-create)

---

<a name="introduction"></a> 
## Introduction

<a name="aws-setup"></a> 
## AWS Setup

<a name="creating-nodes"></a> 
### Creating Nodes


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

