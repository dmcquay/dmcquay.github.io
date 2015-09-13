---
layout: post
title:  "Running ElasticSearch on Amazon EC2 Container Service"
date:   2015-09-12 21:01:05
category: DevOps
tags:   DevOps AWS ELB Docker
---
ElasticSearch is a search and analytics engine with native support for clustering. This can be a nice fit 
for deploying with Amazon EC2 Container Service (ECS) since Docker makes it so easy to install and 
configure ElasticSearch and ECS makes it so easy to deploy these docker containers as a cluster, scale them
up and down and monitor them.

I will walk you through the follow steps to get it all set up correctly:

1. Configure ElasticSearch discovery to work in EC2/ECS
1. Create your ECS task
1. Create an ECS cluster, service
1. Configure your firewalls
1. Troubleshoot, view logs
1. Scale

## Discovery

By default, ElasticSearch is able to discover other nodes and form a cluster using master election. It uses
multicast to send out a message saying "Hey, I'm an ElasticSearch node and I want to join a cluster". If 
nobody responds, then it assumes it is the only one and elects itself as master. When the next node sends 
this message, this first node will respond saying "I'm here and I'm the master" and so the second node will
join the cluster and note that the other node is master. There are more details, but that's the gist of it.

The problem is, AWS doesn't support multicast. Fear not, this problem has already been solved by the official 
[EC2 Discovery Plugin][ec2-discovery]. The [official ElasticSearch Docker Image][es-official-image] does 
not have this plugin installed, so we either have to trust someone else or build our own. I don't like to 
trust images made by random people, so I think we're best to roll our own in this case.
 
Here's an example Dockerfile for creating such an image.

    FROM elasticsearch:1.7
    MAINTAINER Dustin McQuay <dustin@steals.com>
    WORKDIR /usr/share/elasticsearch
    RUN bin/plugin -i elasticsearch/elasticsearch-cloud-aws/2.7.1
    COPY docker-entrypoint.sh /docker-entrypoint.sh

Simple, right? We start with the official elasticsearch image so we don't even have to install elasticsearch.
Then we use the elasticsearch plugin installer to install our aws plugin (you must pick the right version to
match your elasticsearch version, as I have done here).

Wait, what's that docker-entrypoint.sh? Let me explain before I show you that file. Docker creates its own 
network for running containers. If you fire up a container and run `ifconfig` inside it, you'll see an 
interface called docker0. This is problematic for ElasticSearch discovery because this is the IP address 
that nodes will publish when announcing themselves. Therefore, with the EC2 discovery plugin enabled, the 
nodes will be able to find each other initially, but when they start trying to communicate with each other 
after that, they will fail because they will be using these inaccessible IP addresses.

To fix that, we have to manually tell ElasticSearch what address to use when publishing itself. We do this 
by calling the [EC2 metadata service][ec2-metadata] and passing that as a parameter to elasticsearch.
 
Then entire docker-entrypoint.sh file looks like this:

    #!/bin/bash
    
    set -e
    
    # Add elasticsearch as command if needed
    if [ "${1:0:1}" = '-' ]; then
            set -- elasticsearch "$@"
    fi
    
    # Drop root privileges if we are running elasticsearch
    if [ "$1" = 'elasticsearch' ]; then
            # Change the ownership of /usr/share/elasticsearch/data to elasticsearch
            chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data
            exec gosu elasticsearch "$@"
    fi
    
    # ECS will report the docker interface without help, so we override that with host's private ip
    AWS_PRIVATE_IP=`curl http://169.254.169.254/latest/meta-data/local-ipv4`
    set -- "$@" --network.publish_host=$AWS_PRIVATE_IP
    
    # As argument is not related to elasticsearch,
    # then assume that user wants to run his own process,
    # for example a `bash` shell to explore this image
    exec "$@"

Most of that is just copied from the official elasticsearch image. Here's what we added:

    # ECS will report the docker interface without help, so we override that with host's private ip
    AWS_PRIVATE_IP=`curl http://169.254.169.254/latest/meta-data/local-ipv4`
    set -- "$@" --network.publish_host=$AWS_PRIVATE_IP

Now build that image and push it to your docker image repository of choice with some commands like these:

    docker build -t mycompany/elasticsearch-ecs .
    docker push mycompany/elasticsearch

## Create the ECS Task

If you're not familiar with ECS yet, an ECS Task is kind of like a docker-compose yaml file. It specifies 
not only what image you're running, but also the volumes, environment variables, etc. Tasks are versioned 
so you can easily rollback to a previous version in an emergency, or whatever other use case you may have. 
It is nice to be able to version not only your application code but also the entire environment.

When you set up your task, you can create as many container definitions as you want. For our use case, we 
only need one. Let's name it "elasticsearch". Set the image name to the image you just created (e.g. 
"mycompany/elasticsearch-ecs:v1"). Note the "v1". It is best to tag your builds so that task versions 
reference a specific snapshot of your image. Neglecting this would defeat the purpose of versioning your 
task definitions.

For my use case, my index only contained about 5000 items and each was fairly small. I only needed about 
1GB of memory and I reserved 1 CPU (1024 "Units"). Many use cases will require more than this.

Other fields:

 - Essential: yes
 - Port Mappings: 9200:9200 and 9300:9300. 9200 is what your app will use to consume this service. 9300 is 
   use for elasticsearch nodes to communicate with each other.
 - Command: Leave blank for now. We will tweak this in a moment...
 - Leave all other fields blank/default

We need to pass some parameters to the elasticsearch executable to tell it to use the ec2 plugin 
for discovery. Also, 
we can tell it to filter for hosts that have a certain security group so that it isn't trying to form a 
cluster with every host in your account. The idea is to do this by overriding the `Command` and setting it 
to something like `/docker-entrypoint.sh --discovery.type=ec2 --discovery.ec2.groups=sg-XXXXXXXX`, but this
doesn't work. It just doesn't get passed through to Docker correctly for some reason. We need to use the 
array syntax to specify each component separately and to do that, we need to edit this section directly in 
the JSON-formatted task definition. To do that, just click "Add" to close the container edit dialog, then 
click the "JSON" tab. Scroll down to the "command" section and set it to something like this:
 
    "command": [
        "/docker-entrypoint.sh",
        "--discovery.type=ec2",
        "--discovery.ec2.groups=sg-XXXXXXXX"
    ],

Click "Create" and you now have an ECS Task definition that is ready for deploy!

## Create a Cluster, Service and Fire it Up!

I'm not going to walk through every detail of creating an ECS cluster or service since there is nothing about
it that is specific to this use case. Create the cluster. Create the service. Set the number of instances 
to at least 2 so that we can be sure that the cluster forms correctly.

Beware! Port 9200 allows you to do dangerous things like delete an index. Therefore you don't want to just 
leave port 9200 open to the world. You should only open it up to your intended consuming hosts. And if you 
want to be able to scale your app, you should grant access to your app's security group, not individual 
hosts. But, in order to do this, you must create an internal facing load balancer. Internet facing load 
balancers cannot grant access to security groups. However, if you have hosts outside of AWS that need to 
consume this service, then you'll need to use an internet facing load balaner. Decide this and create your 
load balancer before you create your ECS service, because you'll need to select the load balancer at that 
time.
 
## Firewall Rules

 - Each ElasticSearch node needs access to the other nodes on port 9300
 - The load balancer needs to be able to access all ES nodes on port 9200
 - Your app needs to be able to access the load balancer port 9200 (or whatever port you mapped it to)
 - For troubleshooting, you might want to temporarily open port 9200 on the load balancer and each node for
   your IP.

## Verify

Now grab the endpoint for your load balancer and open it up in your browser. You should see a status 
response similar to this:
 
    {
      "status" : 200,
      "name" : "Virako",
      "cluster_name" : "elasticsearch",
      "version" : {
        "number" : "1.7.1",
        "build_hash" : "b88f43fc40b0bcd7f173a1f9ee2e97816de80b19",
        "build_timestamp" : "2015-07-29T09:54:16Z",
        "build_snapshot" : false,
        "lucene_version" : "4.10.4"
      },
      "tagline" : "You Know, for Search"
    }

If you see that, you know you've got at least one node up. If not, check your firewall rules. Check ECS to 
make sure the nodes actually deployed. You can also ssh into one of the instances and use `docker ps -a` to
find the name of the most recent container created for your task and then `docker logs [name]` to view the 
logs and hopefully figure out what is wrong.
 
Once that all looks good, then look at `/_cluster/state` to see if the nodes formed a cluster correctly. Look
for the `nodes` key and there should be as many nodes in that list as how many you configured when you 
created the ECS service. If not, click on one of the tasks in the ECS console to see what EC2 instance it 
is running on, SSH into that node and use the docker logs command to see what is going wrong.

## Scale

You can now edit your service to increase the instances as needed. Remember that you cannot schedule more 
instances that you have capacity for in your cluster, but you can easily scale up the cluster using the 
auto scaling group. And ECS provides per-cluster and per-service metrics within the ECS console, as well as
the CloudWatch metrics you get for the EC2 nodes themselves. Happy searching!

[ec2-discovery]: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-ec2.html
[es-official-image]: https://hub.docker.com/_/elasticsearch/
[ec2-metadata]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-data-retrieval