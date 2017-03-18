# Hydra and AWS in Docker Swarm
Running Hydra on AWS in a Docker Swarm

## Table of Contents

* [Introduction](#introduction)
* [AWS Setup](#aws-setup)
  * [Creating Nodes](#creating-nodes)

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
