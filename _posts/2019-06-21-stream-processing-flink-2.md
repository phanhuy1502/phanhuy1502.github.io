---
layout: post
title: Stream processing with Flink (Part 2)
subtitle: Part 2 introduces an stream processing example with Flink and Kafka
tags: [code, flink, tutorial]
comments: true
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
---

Other posts in the series:

- [Part 1]({% post_url 2019-05-10-stream-processing-flink %})

In this part, we will redo the example in part 1 using Kafka as our message queue and Flink as our stream processing framework.
While part 1 focuses more on concepts, part 2 will introduce some popular frameworks for stream data processing (Kafka / Flink), which helps to handle common issues in data engineering, so developers can focus on the actual processing logic.

Notice that the tutorial here runs everything on a single machine, which is not a proper setup for a production environment
(where you almost always want to run your data pipelines on a cluster of machines).
The focus here is on introducing Kafka and Flink and how they fit in to the example in part 1 (of a data pipeline without any fancy technology).

The tutorial is tested on Debian GNU/Linux 9.9.
Ability to read Java code will be useful to follow this part.

## Installation

(If you're just reading this tutorial without doing hands-on, skip the scripts in this part.
There are some brief introduction of the technology we're using so it's still worth reading the Installation part even if you're not doing hands-on)

We'll run all our dependencies as Docker containers.

### (1) Install Docker & Docker Compose

Docker is a container technology. You can think of container as a packaging tool for the applications, so the applications can be delivered/deployed/installed as a package/container.
(like how you install software on Linux with tools like `yum` or `apt-get`).
Docker offers more than that, but in this tutorial, we'll use Docker mostly as a tool for installing our softwares (like Kafka / Zookeeper).

Find the installation guide for your system [here](https://docs.docker.com/install/overview/)

For debian based linux:

```sh
# extracted from: https://docs.docker.com/install/linux/docker-ce/debian/

# install packages for apt to add private repo
sudo apt-get update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common

# add docker repo
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"

# install docker-ce
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Verify the installation

```sh
docker version

>>>
...
 Version:           18.09.7
...
```

### (2) Start Zookeeper

Kafka & Flink uses Zookeeper for distributed coordination.
In this post, all installations are in a single machine, so the roles of Zookeeper is not clear.
But when we deploy Kafka/Flink on a cluster (= multiple machines), the role of Zookeeper will be clearer, helping the different nodes in the cluster to communicate.

Start a Zookeeper server as a Docker container on your local env.
We are using this [official Zookeeper image by Apache](https://hub.docker.com/_/zookeeper)

```bash
# this command start the zookeeper as a docker container
# --name zk: specify the name of the zookeeper container to be zk
# -p 2181:2181: map the container port 2181 to the host port 2181. This allows the zookeeper service to be accessible from localhost:2181
# zookeeper: is the Docker image name. This Docker image name is fetched from the docker hub to your local env
docker run --name zk -d -p 2181:2181 zookeeper
```

### (3) Start Kafka

Kafka is used as a asynchronous message queue.
Producer can write messages to Kafka to different topics (queues), while multiple consumer can read these messages in order from Kafka.

Recall our data pipeline from (Part 1)({% post_url 2019-05-10-stream-processing-flink %}).
The Kafka replaces the role of the text files (stream 1/ stream 2) in the pipeline.

```txt
 _____________     __________     ______________     __________
|  stream.sh  |-> |  ids.txt |-> |  process.sh  |-> |names.txt |
|(data source)|   |(stream 1)|   |(process unit)|   |(stream 2)|
 -------------     ----------     --------------     ----------
```

We'll use the [Kafka image provided by Spotify](https://hub.docker.com/r/spotify/kafka).
Again, it's a minimal setting with a single Kafka server with all the default settings.

```bash
docker run -p 2181:2181 -p 9092:9092 -d --name kafka spotify/kafka
```

### Start Flink

[Flink](https://ci.apache.org/projects/flink/flink-docs-master/) is a platform for streaming data processing.
Flink helps to manage and run our data processing unit.
In the diagram above, Flink plays the role of the process unit.

We'll use the [official Flink image for deployment](https://hub.docker.com/_/flink).
Again, we deploy with a standalone (single machine) minimal configuration.

A super brief intro to Flink's architecture:
Flink's architecture is a typical master-slave architecture in distributed system.
Master-slave architecture includes several (usually 3-5) master nodes and lots of slave node.
Slave node (also known as worker node or executor node) execute the actual processing work (like process data, run server).
Master node (also known by other names like: cordinator node or scheduler node) helps to coordinate the worker.
A Flink cluster include `job managers` and `task managers`.
Job managers are the master nodes, manage and cooridate the task managers - the slave nodes which run your data pipelines.

Start a single job manager and a single task manager as two Docker containers

```bash

# create a docker "network" named flink, for different containers to communicate
docker network create --driver bridge flink

docker run -d \
    --name flink_jobmanager \
    --network flink \
    --expose 6123 \
    -p 8081:8081 \
    -e JOB_MANAGER_RPC_ADDRESS=flink_jobmanager \
    flink:1.9.1 jobmanager

docker run --name flink_taskmanager \
    --network flink \
    --expose 6121 --expose 6122 \
    -e JOB_MANAGER_RPC_ADDRESS=flink_jobmanager \
    -d flink:1.9.1 taskmanager

docker run -it amouat/network-utils bash
```
