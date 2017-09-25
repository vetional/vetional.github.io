---
layout: post
title: "(Incomplete) Docker basics"
comments: true
description: "Learn docker"
keywords: "docker"
---

### Think : Docker seems usefull how can I use it to leverage the multitude of areas I work in 

Well the common problem I face is every time I need to deploy or simply switch between the work I am doing as a hobby or for a client I mess up  my work environment for other projects. Even if i am using virtualenv or conda this problem often haunts me for hours and costs me a lot of otherwise good developemnt time. Docker promises to be a solution to this and I intend to learn and teach what it is and how can it allievate deployment and management troubles.

Docker can help manage deployment ANGER. It solves the problem "but it's working on my system". So let's get to it

Installing docker is pretty well documnted on the docker site


### Learn: Key terminonlogy

#### Docker engine: 

### The docker-machine command

What Docker engine does on a locally docker-machine let's you do the same thing on a remote machine it could be on any provider like Amazon EC2, Digital Ocean, IBM's softlayer etc. It also saves you the trouble of installing docker on the remote machine. Docker gives you an abstraction over the comandline interface for all major providers through docker-machine. In this example we will use IBM's softlayer to configure an instance.

#### Prerequisites
Access to a docker and docker-machine installed Mac or linux machine.


#### Step 1

To create a docker instance do

```bash
$ docker-machine create --driver softlayer --softlayer-user=<USER_NAME> \
--softlayer-api-key=<YOUR_API\_KEY> \
--softlayer-memory "1024" \
--softlayer-disk-size "25" \
--softlayer-region "sng01" \
--softlayer-domain "softlayer.com" \
--softlayer-hostname "sld" \
--softlayer-cpu "1" \
--softlayer-hourly-billing \
--softlayer-image="UBUNTU_14\_64" \
--softlayer-local-disk \
softlayer-docker-test
```
This will create and install the latest Docker on a fresh Ubuntu 14.04 machine with the name softlayer-docker-test.

**Note:** It's good to start with hourly billing for demoing docker but do remember to remove the -softlayer-hourly-billing flag in production.

#### Step 2

Now naturally you would like to do something on the machine, to connect to it, first set a few environment variables using 

```bash
$ eval "$(docker-machine env softlayer-docker-test)"
```
This will take all your docker commands on the local machine to be run on the remote machine.

#### Step 3
The remote machine and the local environment are now ready to execute all docker related commands, simply run to test your installation

```bash
$ docker run -d -p 8000:80 --name webserver kitematic/hello-world-nginx
``` 

to Get the ip of the remote machine

```bash
$ docker ip softlayer-docker-test
```
If you visit <machine-ip>:8000 this should bring up kitematic page on your fresh nginx server.

#### Stoping and removing the remote machine

To stop and remove the remote-machine:

```bash
$ docker-machine stop softalayer-docker-test
$ docker-machine rm softalayer-docker-test
```
Now that you have removed the machine you can use the slcli to confirm the removal using

```bash
$ pip install softlayer
$ slcli vs list
```

To unset the environment on the local machine do

```bash
$ eval $(docker-machine env -u)
```


#### NOTE:
In case:
```bash
Error creating machine: Error in driver during machine creation: Error launching instance: InstanceLimitExceeded: Your quota allows for 0 more running instance(s). You requested at least 1
```

Assuming that you've provisioned a machine with Ubuntu as the operating system, execute the following command from your local system to update the package database on the Docker host:

```bash
$ docker-machine ssh machine-name apt-get update
```

You can even apply available updates using:
```bash
$ docker-machine ssh machine-name apt-get upgrade
```

Not sure what kernel your remote Docker host is using? Type the following:
```bash
$ docker-machine ssh machine-name uname -r
```

Besides using the ssh subcommand to execute commands on the remote Docker host, you can also use it to log into the machine itself. It's as easy as typing:

```bash
$ docker-machine ssh machine-name
```

Your command prompt will change to reflect the fact that you're logged into the remote host:

```bash
Output
$ root@machine-name#
```

To exit from the remote host, simply type:

```bash
$ exit
```

#### Docker compose:
helps you connect multiple simple containers to make a complex pipeline.

#### Swarm : 
A group of docker containers groped togather as one single entity. It's like a bigger box with lots of small boxes but it has all the properties of a box, ie. it can support compose, jenkins, docker machine etc. (Very advanced: https://www.youtube.com/watch?v=JgUyI-MIKZ0)

Service = images + commands














### Code: 

**Refernces**
