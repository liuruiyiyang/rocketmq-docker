# Introduction of distributed RocketMQ Docker deployment based on the swarm cluster

## Background

This document shows how to deploy a NameServer cluster and a master-only broker cluster on a docker swarm cluster. 

## Steps to deploy and run docker containers

1. Determine the IP and DNS information of the host (physical or virtual machine) to be deployed with NameServer or Broker, the storage file location in the hosted node, and ensure that the relevant ports (9876, 10911, 10912, 10909) are not occupied.
2. Deploy a docker swarm cluster
3. Configure the ```docker-compose.yml``` file
4. Start the cluster service stack in the docker way

## Deploy a docker swarm cluster
1. Install Docker on each node of your cluster. You can refer to [Docker Official Guides](https://docs.docker.com/install/linux/docker-ce/ubuntu/).

2. Choose a node as the swarm manager node, on this node, run the following command:

```
docker swarm init 
```

> Please insure that ports 2377 and 2376 are not used because they are required by docker. 

>If you have multiple interface on the node, you may experience error, and please use ```docker swarm init --advertise-addr <swarm-manager-host ip>``` to specify the host ip of your swarm manager node, for example ```docker swarm init --advertise-addr 192.168.130.5```

If the current node become a swarm manager successfully, the look for output like this:
```
Swarm initialized: current node (7a6xktf3g09mivup8rvy6p8kr) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5qwapmsrxk9no11al34xjnczbmf583tflwazzfhj7z0ps5t1id-0y4btqkgopq5xdm1bfel9eec3 192.168.130.5:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

You will get a token and you can use the token to join other nodes to the swarm cluster.

3. Join the rest nodes into the swarm cluster as workers, run

```
docker swarm join --token <token> <manager node ip>:2377
```
on each node.

If the current node become a swarm worker successfully, the look for output like this:

```
This node joined a swarm as a worker.
```

4. Check the swarm cluster status. Login the swarm manager node, and run
```
docker node ls
```

The look for output like this:
```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
aasqtwnlh9hyjfahdjpa5129w     pc04                Ready               Active                                  18.06.3-ce
7a6xktf3g09mivup8rvy6p8kr *   s5                  Ready               Active              Leader              18.09.7
```
The output shows which node is the swarm manager and each node's status.

There it is, you have successfully deployed a swarm cluster!

For more information, please refer to docker swarm commands [official docs](https://docs.docker.com/engine/reference/commandline/swarm/)

## Configure the docker-compose.yml file
The default ```docker-compose.yml``` file is:
```
version: '3'
services:
  #Service for nameserver
  namesrv:
    container_name: namesrv
#    build:
#      context: ../image-build
#      dockerfile: Dockerfile-alpine
#      args:
#        version: 4.5.0
    image: rocketmqinc/rocketmq:4.5.0-alpine
    ports:
      - "9876:9876"
    networks:
      - backend
    deploy:
      placement:
        constraints:
          - node.role == manager
      mode: global
      restart_policy:
        condition: on-failure
#      resources:
#        limits:
#          cpus: "0.5"
#          memory: 6G
    volumes:
      - ./data/namesrv/logs:/home/rocketmq/logs
      - ./data/namesrv/store:/home/rocketmq/store
    command: sh mqnamesrv

  #Service for broker
  broker:
    container_name: broker
    image: rocketmqinc/rocketmq:4.5.0-alpine
    ports:
      - "10909:10909"
      - "10911:10911"
      - "10912:10912"
    networks:
      - backend
    deploy:
      mode: replicated
      replicas: 2
    environment:
      - NAMESRV_ADDR=namesrv:9876
    volumes:
      - ./data/broker/logs:/home/rocketmq/logs
      - ./data/broker/store:/home/rocketmq/store
      - ./broker.conf:/opt/rocketmq-4.5.0/conf/broker.conf
    command: sh mqbroker -c ./broker.conf

networks:
  backend:
```

This file defines a docker stack service of 2 master-brokers distributed on swarm cluster and a name service process on the swarm manager node.