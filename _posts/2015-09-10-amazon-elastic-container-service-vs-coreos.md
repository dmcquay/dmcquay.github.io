---
layout: post
title:  "AWS Elastic Container Service"
date:   2015-09-10 08:30:00
tags:   DevOps AWS CoreOS Docker
---
About a year ago I was visiting a friend of mine who is a software developer. We talked about kids, running,
biking. But, as it often did, the conversation drifted to cool new technologies. He told me about how Gaia
GPS was using Docker and CoreOS.

The next day I started researching about Docker and CoreOS and I got excited about how it could improve my
own architecture. At Steals.com, the development environment is a bit complex. It is simply a PHP app, so it
isn't terribly hard just to turn it on. But to get anywhere near what production looks like, it is a lot of
work. The result was that developers avoided certain bugs because they were hard to work on in their local
environment. Maybe they had Apache + PHP, but never bothered to set up nginx. Maybe their Apache configs were
too different from production and they couldn't reproduce the error. Perhaps the bug was related 
specifically to memcached, but they were using file cache. We also use MySQL, ElasticSearch and ImageSquish.
We have blogs running on a whole other stack. We have 8 or more domains that all have different 
configurations, so perhaps the dev only had the most common ones set up.

The list goes on, but I think you get the point. We had a strong need for a development environment that met
the follow requirements:

 1. Easy, automated setup
 2. Consistent with production
 3. Consistent for all developers

I had been trying to solve this problem using Vagrant, which is an excellent product. But as I learned about
Docker, it had one critical advantage over Vagrant that initially caught my attention. With Vagrant, I could
be consistent with production so long as I kept them in sync, which I had to do manually. With Docker, I can
actually run the same image in dev and prod.

So I switched gears and created a standardized development environment using Docker. And I began to deploy
lower risk services in production using Docker + CoreOS with the intention to close the loop by deploying
our main application that way as soon as we felt comfortable that Docker + CoreOS was solid and we were
experts.

Problem: I kept running into things I couldn't explain with CoreOS. I had units that would turn off and 
wouldn't start back up on their own. I had images that got stuck in weird states. Some things were CoreOS 
being buggy. Others may have been user error. It became clear to me that either CoreOS wasn't read for us or
we weren't ready for CoreOS. Either way, I needed a suitable replacement.

Enter ECS. Amazon created an offering called Elastic Container Service, which [launched in November 
2014][ecs-launch]. I was interested in ECS because I wanted a container service that required very little
expertise to manage. I had my eye on [Rancher][rancher], and I still do, but it is still in beta at the time
of this writing. As much as I love Docker, it was not a good time for my team to be an early adopter of a
risky technology. I needed something very stable, easy to use and backed by someone I trust.

By October 2015, I felt that ECS had become what I was searching for. They had worked out the kinks. They 
added the monitoring that I needed so that I wasn't flying blind. And so I decided to give it a try.

Here's what I've found so far.

Pros

 - Dead simple. Almost no expertise required (a few minor exceptions which I have noted in Cons section)
 - Decent monitoring OOB.
 - Automatic integration with elastic load balancers
 - Robust deploys OOB. When you deploy a new version, your previous version stays live until the new version
   comes up and is healthy.
 - Fantastic rollback support. Each task version contains not only the docker image, but also all the 
   environment variables and everything else about how it is executed. Reliable rollback is available at the
   click of a button.
 - Of course, full API support if you want to take the automation one level further.
 - But you can also do everything from the console. Very nice in case you need to do something in an emergency
   that you didn't think of when creating your deploy scripts.
 - Backed by Amazon. Built on solid technologies (EC2, ELB, etc).
 - Plain Docker. They didn't reinvent anything. All they did is run docker and add some nice features to allow
   you to manage your cluster. If we want to switch to Rancher or something else later, we're not in too deep
   with ECS.

Cons

 - Docker is a cool way to run a bunch of staging/test environments on a single host. It is a great way to 
   fully replicate production with very little cost. However, ECS requires you to reserve CPU shares and 
   memory, so the containers can't share the resources. On the bright side, this is best practice for 
   production environments, so it forces you to be wise in that case.
 - The robust deployment technique is great. It brings up the new version before it takes down the old 
   version. But that comes with a cost. You have to have extra resources laying around to run all of this. 
   If you run 5 instances of your app, then you need 10 hosts so that the new version can be brought up. 
   Technically you only need 1 extra host, but then it can only bring up 1 instance at a time. Then it 
   waits until it is stable, then drains connections on one of the old containers and then shuts it down. 
   The whole process might take a minute per container. With a large deployment and only one extra host, 
   that takes a long time. But you do have options. Scaling is easy with auto-scaling groups, so you could 
   just spin up a few extra hosts just before you do deploys.
 - This leads to my last complaint. The ECS agent seems a bit slow to pick up on pending changes. Having only
   one extra host wouldn't be too bad if the process of launching a container, verifying it and then taking
   down an old container were much faster. I wonder if they are using simple polling instead of something
   like SNS or SQS.

All in all, I think that AWS ECS is the best offering right now IF you are looking for something that requires
very little ops expertise. CoreOS is pretty solid, but you better be prepared to become a CoreOS expert and
 I don't just mean the high level stuff. You need to be ready to get into the guts of a new and radically 
 different operating system as an early adopter.

[ecs-launch]:   https://aws.amazon.com/blogs/aws/cloud-container-management/
[rancher]:      http://rancher.com/