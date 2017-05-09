---
layout: post
title: "Application File Structures"
date: 2017-02-07
categories: Architecture
---

Files
Links
  Files

When you're starting a new project, how should you organize your files and directories?

You likely have some conflicting goals in mind:
 - Keep it simple
 - Avoid setting up the foundation for a new monolith

Working on monolithic code bases is difficult. Migrating off of them is even harder.
So when I'm starting a new project, I am often contemplating how to avoid creating
yet another monolith.

I have seen some great solutions to this problem. My favorite so far, even though
it is not perfect, is the architecture at Pluralsight. At Pluralsight, we have the
concept of a Bounded Context. Inside of a Bounded Context lives one or more applications,
possibly microservices, databases and whatever else is needed. That Bounded Context is
the source of truth for some data and for other data it is a consumer. For data that
it is the source of truth for, it will publish messages to a rabbit cluster for all
changes. For data that it consumes, it will listen to incoming messages from other
sources of truth. Therefore, it has a local cache of that data that is updated on
the fly without ever blocking a user request. All sources of truth must also provide
a Self-Healing API where consumers can fetch data they are missing, if any.

That was a very brief description of this architecture and probably has holes in it.
The architecture itself could also be argued for or against. I don't want to go too
deep into that though. Let it suffice that this architecture has served us quite well,
allowing teams to be autonomous, independent and avoid creating more monoliths.
I also hope to convey that it is not easy. This architecture requires more work to
maintain.

When I'm starting a new project, I feel that starting out with an architecture like
this is far too heavy. But traditional architectures set you up for disaster. I would
like to propose a middle ground.

Organize files into feature directories
You can have nested feature directories
  This conveys dependencies and enforces them to be uni-directional
Shared dependencies should generally be avoided. Better to duplicate some code, even a repository for example.
  For example, user logic like user management should probably be in a feature folder,
    but many other features are going to depend on user data. But that dependency should
    generally be limited and read-only. Therefore, create repo method in the respective folders
    that access the user table directly. Then, later if you want to break that feature folder
    into a separate microservice, it is nearly ready to go. And if you want it in a separate
    Bounded Context, you will need to find a way to replicate the user data to a new database
    and then you're ready to go.

As an alternative, you might consider allowing feature directories to depend on sibling feature directories? Via facade only?
  But beware that the more you do this, the more you are beginning to create a monolith. Also, this starts to become
  grey area very fast and hard to stand your ground. I think it's better to keep things black and white if possible.
