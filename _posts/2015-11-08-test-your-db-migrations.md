---
layout: post
title:  "Test Your Database Migrations"
date:   2015-11-08 15:00:00
category: Misc
tags:   SequalizeJS Testing
---
I like to use schema migration tools like Liquibase, Sequlize Migrations, etc to make database schema changes.
A critical part of deploying is the ability to quickly and reliably rollback. We go to great lengths to
prepare for this. VCS allows us to rollback our code. We might package our app up into tarballs, deb packages
or Docker containers to make it even easier to deploy new versions and rollback. 

Probably the most critical part of this process is database migrations, both schema and data. Why? Because
while apps are often ephemeral in nature, our databases are very much not. For this reason, database migration
tools (e.g. Liquibase, Sequalize Migrations, Django Migrations, etc) provide rollback capability.

Despite how critical it is to be able to rollback your database migrations, I have found that they are often
untested. I have a very simple tip to fix this problem. Create a wrapper script of some sort that applies
the migration, rolls it back, and then applies it again and always use this in development. Therefore, without
having to think about it, you will always be testing not only your migrations, but the very critical migration
rollbacks.

I'm currently working on a project in NodeJS that uses Sequelize Migrations. I have the following entry
in my package.json scripts section:

`"migrate-test": "sequelize db:migrate --config sequelize-config.js && sequelize db:migrate:undo --config sequelize-config.js && sequelize db:migrate --config sequelize-config.js"`